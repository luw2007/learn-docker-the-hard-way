`docker`后台服务的主要实现在`runtime.go`中。因此梳理玩`docker`启动逻辑之后，我们来看看`runtime`的实现。

`runtime`中主要包含`runtime`和`history`的实现。

初始化
==================
``` golang
// 存储系统初始化路径
var sysInitPath string

func init() {
	//#	`SelfPath`是`docker`的绝对路径，具体参见`docker.go`。
	sysInitPath = SelfPath()
```

runtime
==================
##runtime的构造
``` goalng
type Runtime struct {
	root           string
	//# 运行环境仓库`path.Join(root, "containers")`
	repository     string
	//# 容器列表
	containers     *list.List
	//# 网络
	networkManager *NetworkManager
	//# 本地存储
	graph          *Graph
	//# `images·仓库
	repositories   *TagStore
	//# 权限配置
	authConfig     *auth.AuthConfig
}
```

##runtime的方法
``` golang
//# List 遍历`runtime.containers`, 生成`history`返回。
func (runtime *Runtime) List() []*Container {
	containers := new(History)
	for e := runtime.containers.Front(); e != nil; e = e.Next() {
		containers.Add(e.Value.(*Container))
	}
	return *containers
}

func (runtime *Runtime) getContainerElement(id string) *list.Element {
	for e := runtime.containers.Front(); e != nil; e = e.Next() {
		container := e.Value.(*Container)
		if container.Id == id {
			return e
		}
	}
	return nil
}
//# Get 调用`getContainerElement`遍历`runtime.containers`根据id找出`container`。不存在返回nil, 存在返回`Container`的指针。
func (runtime *Runtime) Get(id string) *Container {
	e := runtime.getContainerElement(id)
	if e == nil {
		return nil
	}
	return e.Value.(*Container)
}

//# Exists 判断id对应的`Container`是否存在
func (runtime *Runtime) Exists(id string) bool {
	return runtime.Get(id) != nil
}

func (runtime *Runtime) containerRoot(id string) string {
	return path.Join(runtime.repository, id)
}

//# Create 
func (runtime *Runtime) Create(config *Config) (*Container, error) {
	// 查找`Image`
	img, err := runtime.repositories.LookupImage(config.Image)
	if err != nil {
		return nil, err
	}
	container := &Container{
		// FIXME: we should generate the ID here instead of receiving it as an argument
		Id:              GenerateId(),
		Created:         time.Now(),
		Path:            config.Cmd[0],
		Args:            config.Cmd[1:], //FIXME: de-duplicate from config
		Config:          config,
		Image:           img.Id, // Always use the resolved image id
		NetworkSettings: &NetworkSettings{},
		// FIXME: do we need to store this in the container?
		SysInitPath: sysInitPath,
	}
	container.root = runtime.containerRoot(container.Id)
	//# 第1步：创建容器目录。
	// 同时还可以避免出现竞态条件。
	if err := os.Mkdir(container.root, 0700); err != nil {
		return nil, err
	}
	//# 第2步：保存容器JSON
	if err := container.ToDisk(); err != nil {
		return nil, err
	}
	//# 第3步：注册容器
	if err := runtime.Register(container); err != nil {
		return nil, err
	}
	return container, nil
}

func (runtime *Runtime) Load(id string) (*Container, error) {
	container := &Container{root: runtime.containerRoot(id)}
	if err := container.FromDisk(); err != nil {
		return nil, err
	}
	if container.Id != id {
		return container, fmt.Errorf("Container %s is stored at %s", container.Id, id)
	}
	if err := runtime.Register(container); err != nil {
		return nil, err
	}
	return container, nil
}

// Register makes a container object usable by the runtime as <container.Id>
func (runtime *Runtime) Register(container *Container) error {
	if container.runtime != nil || runtime.Exists(container.Id) {
		return fmt.Errorf("Container is already loaded")
	}
	if err := validateId(container.Id); err != nil {
		return err
	}
	container.runtime = runtime
	// Setup state lock (formerly in newState()
	lock := new(sync.Mutex)
	container.State.stateChangeLock = lock
	container.State.stateChangeCond = sync.NewCond(lock)
	// Attach to stdout and stderr
	container.stderr = newWriteBroadcaster()
	container.stdout = newWriteBroadcaster()
	// Attach to stdin
	if container.Config.OpenStdin {
		container.stdin, container.stdinPipe = io.Pipe()
	} else {
		container.stdinPipe = NopWriteCloser(ioutil.Discard) // Silently drop stdin
	}
	// Setup logging of stdout and stderr to disk
	if err := runtime.LogToDisk(container.stdout, container.logPath("stdout")); err != nil {
		return err
	}
	if err := runtime.LogToDisk(container.stderr, container.logPath("stderr")); err != nil {
		return err
	}
	// done
	runtime.containers.PushBack(container)
	return nil
}

func (runtime *Runtime) LogToDisk(src *writeBroadcaster, dst string) error {
	log, err := os.OpenFile(dst, os.O_RDWR|os.O_APPEND|os.O_CREATE, 0600)
	if err != nil {
		return err
	}
	src.AddWriter(NopWriteCloser(log))
	return nil
}

func (runtime *Runtime) Destroy(container *Container) error {
	element := runtime.getContainerElement(container.Id)
	if element == nil {
		return fmt.Errorf("Container %v not found - maybe it was already destroyed?", container.Id)
	}

	if err := container.Stop(); err != nil {
		return err
	}
	if mounted, err := container.Mounted(); err != nil {
		return err
	} else if mounted {
		if err := container.Unmount(); err != nil {
			return fmt.Errorf("Unable to unmount container %v: %v", container.Id, err)
		}
	}
	// Deregister the container before removing its directory, to avoid race conditions
	runtime.containers.Remove(element)
	if err := os.RemoveAll(container.root); err != nil {
		return fmt.Errorf("Unable to remove filesystem for %v: %v", container.Id, err)
	}
	return nil
}

// Commit creates a new filesystem image from the current state of a container.
// The image can optionally be tagged into a repository
func (runtime *Runtime) Commit(id, repository, tag string) (*Image, error) {
	container := runtime.Get(id)
	if container == nil {
		return nil, fmt.Errorf("No such container: %s", id)
	}
	// FIXME: freeze the container before copying it to avoid data corruption?
	// FIXME: this shouldn't be in commands.
	rwTar, err := container.ExportRw()
	if err != nil {
		return nil, err
	}
	// Create a new image from the container's base layers + a new layer from container changes
	img, err := runtime.graph.Create(rwTar, container, "")
	if err != nil {
		return nil, err
	}
	// Register the image if needed
	if repository != "" {
		if err := runtime.repositories.Set(repository, tag, img.Id, true); err != nil {
			return img, err
		}
	}
	return img, nil
}

func (runtime *Runtime) restore() error {
	dir, err := ioutil.ReadDir(runtime.repository)
	if err != nil {
		return err
	}
	for _, v := range dir {
		id := v.Name()
		container, err := runtime.Load(id)
		if err != nil {
			Debugf("Failed to load container %v: %v", id, err)
			continue
		}
		Debugf("Loaded container %v", container.Id)
	}
	return nil
}
```

history
==================
##history的构造

``` golang

//# 历史记录是一个存储`Container`指针的数组。`Contatiner`结构体将在`container.go`中详细介绍。
type History []*Container
```

##history的方法
``` golang

func (history *History) Len() int {
	return len(*history)
}

func (history *History) Less(i, j int) bool {
	containers := *history
	return containers[j].When().Before(containers[i].When())
}

func (history *History) Swap(i, j int) {
	containers := *history
	tmp := containers[i]
	containers[i] = containers[j]
	containers[j] = tmp
}

func (history *History) Add(container *Container) {
	*history = append(*history, container)
	sort.Sort(history)
}
```

####__author__ = 'luw2007@gmail.com'