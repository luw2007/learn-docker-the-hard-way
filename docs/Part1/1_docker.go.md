####__author__ = 'luw2007@gmail.com'
`docker.go` 打包成`docker`命令行工具， 也就是我们最常接触到的`docker`命令。
里面有三个方法 `main`, `daemon`, `runCommand`。
	
	// Running in init mode
	//# `SysInit`在`sysinit.go`中。

上一句注释为源代码注释， 下面的注释为后加的解释。 
为了排版简介， 部分有中文注释简单的原文注释被删除了。 

docker.go:main
==================
``` golang 

func main() {	
	//# 根据路径判断是否需要系统级初始化 `SelfPath`在 
	//# `utils`中，返回二进制文件（即`docker`）的绝对路径。
	if docker.SelfPath() == "/sbin/init" {
		//# `SysInit`在`sysinit.go`中，完成系统初始化后退出。
		docker.SysInit()
		return
	}
	// FIXME: Switch d and D ? (to be more sshd like)
	fl_daemon := flag.Bool("d", false, "Daemon mode")
	fl_debug := flag.Bool("D", false, "Debug mode")
	flag.Parse()
	rcli.DEBUG_FLAG = *fl_debug
	if *fl_daemon {
		if flag.NArg() != 0 {
			flag.Usage()
			return
		}
		//# 启动daemon
		if err := daemon(); err != nil {
			log.Fatal(err)
		}
	} else {
		if err := runCommand(flag.Args()); err != nil {
			log.Fatal(err)
		}
	}
}
```

##sysinit.go:SysInit
``` golang
func SysInit() {
	//# 需要参数
	if len(os.Args) <= 1 {
		fmt.Println("You should not invoke docker-init manually")
		os.Exit(1)
	}
	//# 从参数中找到用户`username or uid`和网关`gateway address`
	var u = flag.String("u", "", "username or uid")
	var gw = flag.String("g", "", "gateway address")

	flag.Parse()
	
	//# 设置网络，改变用户，启动守护进程
	//# 如果`gw` != "", 则使用系统命令`/sbin/route add default gw GW`
	//# 设置网关
	setupNetworking(*gw)
	//# 降低权限以便照顾普通用户
	//# 如果·u· != "", 则通过`Setuid`，`Setgid`设置权限。
	changeUser(*u)
	//# 运行程序(Deamon/Debug 模式)
	executeProgram(flag.Arg(0), flag.Args())
}
```

docker.go:daemon
==================
``` golang 
func daemon() error {
	//# 启动服务
	service, err := docker.NewServer()
	if err != nil {
		return err
	}
	//# 监听本地4242端口， 并提供服务
	return rcli.ListenAndServe("tcp", "127.0.0.1:4242", service)
}
```

##commands.go:NewServer
```golang 
func NewServer() (*Server, error) {
	rand.Seed(time.Now().UTC().UnixNano())
	if runtime.GOARCH != "amd64" {
		log.Fatalf("The docker runtime currently only supports amd64 (not %s). This will change in the future. Aborting.", runtime.GOARCH)
	}
	//# 生成新的运行环境
	runtime, err := NewRuntime()
	if err != nil {
		return nil, err
	}
	//# 封装运行环境为服务
	srv := &Server{
		runtime: runtime,
	}
	return srv, nil
}

type Server struct {
	runtime *Runtime
}
```

##runtime.go:NewRuntime
``` golang
func NewRuntime() (*Runtime, error) {
	return NewRuntimeFromDirectory("/var/lib/docker")
}

func NewRuntimeFromDirectory(root string) (*Runtime, error) {
	runtime_repo := path.Join(root, "containers")

	if err := os.MkdirAll(runtime_repo, 0700); err != nil && !os.IsExist(err) {
		return nil, err
	}

	//# 生成Graph
	g, err := NewGraph(path.Join(root, "graph"))
	if err != nil {
		return nil, err
	}
	//# 生成tagstore
	repositories, err := NewTagStore(path.Join(root, "repositories"), g)
	if err != nil {
		return nil, fmt.Errorf("Couldn't create Tag store: %s", err)
	}
	//# 
	netManager, err := newNetworkManager(networkBridgeIface)
	if err != nil {
		return nil, err
	}
	authConfig, err := auth.LoadConfig(root)
	if err != nil && authConfig == nil {
		// If the auth file does not exist, keep going
		return nil, err
	}

	runtime := &Runtime{
		root:           root,
		repository:     runtime_repo,
		containers:     list.New(),
		networkManager: netManager,
		graph:          g,
		repositories:   repositories,
		authConfig:     authConfig,
	}

	if err := runtime.restore(); err != nil {
		return nil, err
	}
	return runtime, nil
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

##graph.go:NewGraph
``` golang
type Graph struct {
	Root string
}

func NewGraph(root string) (*Graph, error) {
	abspath, err := filepath.Abs(root)
	if err != nil {
		return nil, err
	}
	// Create the root directory if it doesn't exists
	if err := os.Mkdir(root, 0700); err != nil && !os.IsExist(err) {
		return nil, err
	}
	return &Graph{
		Root: abspath,
	}, nil
}
```

##tags.go:NewTagStore
``` golang
const DEFAULT_TAG = "latest"

type TagStore struct {
	path         string
	graph        *Graph
	Repositories map[string]Repository
}

type Repository map[string]string

func NewTagStore(path string, graph *Graph) (*TagStore, error) {
	abspath, err := filepath.Abs(path)
	if err != nil {
		return nil, err
	}
	store := &TagStore{
		path:         abspath,
		graph:        graph,
		Repositories: make(map[string]Repository),
	}
	// Load the json file if it exists, otherwise create it.
	if err := store.Reload(); os.IsNotExist(err) {
		if err := store.Save(); err != nil {
			return nil, err
		}
	} else if err != nil {
		return nil, err
	}
	return store, nil
}

```

##network.go:newNetworkManager
``` golang
func newNetworkManager(bridgeIface string) (*NetworkManager, error) {
	addr, err := getIfaceAddr(bridgeIface)
	if err != nil {
		return nil, err
	}
	network := addr.(*net.IPNet)

	ipAllocator, err := newIPAllocator(network)
	if err != nil {
		return nil, err
	}

	portAllocator, err := newPortAllocator(portRangeStart, portRangeEnd)
	if err != nil {
		return nil, err
	}

	portMapper, err := newPortMapper()

	manager := &NetworkManager{
		bridgeIface:   bridgeIface,
		bridgeNetwork: network,
		ipAllocator:   ipAllocator,
		portAllocator: portAllocator,
		portMapper:    portMapper,
	}
	return manager, nil
}
```

##auth.go:LoadConfig
```
// load up the auth config information and return values
// FIXME: use the internal golang config parser
func LoadConfig(rootPath string) (*AuthConfig, error) {
	confFile := path.Join(rootPath, CONFIGFILE)
	if _, err := os.Stat(confFile); err != nil {
		return &AuthConfig{}, fmt.Errorf("The Auth config file is missing")
	}
	b, err := ioutil.ReadFile(confFile)
	if err != nil {
		return nil, err
	}
	arr := strings.Split(string(b), "\n")
	orig_auth := strings.Split(arr[0], " = ")
	orig_email := strings.Split(arr[1], " = ")
	authConfig, err := DecodeAuth(orig_auth[1])
	if err != nil {
		return nil, err
	}
	authConfig.Email = orig_email[1]
	authConfig.rootPath = rootPath
	return authConfig, nil
}
```

docker.go:runCommand
==================
``` golang
func runCommand(args []string) error {
	var oldState *term.State
	var err error
	if term.IsTerminal(0) && os.Getenv("NORAW") == "" {
		oldState, err = term.MakeRaw(0)
		if err != nil {
			return err
		}
		defer term.Restore(0, oldState)
	}
	// FIXME: we want to use unix sockets here, but net.UnixConn doesn't expose
	// CloseWrite(), which we need to cleanly signal that stdin is closed without
	// closing the connection.
	// See http://code.google.com/p/go/issues/detail?id=3345
	if conn, err := rcli.Call("tcp", "127.0.0.1:4242", args...); err == nil {
		receive_stdout := docker.Go(func() error {
			_, err := io.Copy(os.Stdout, conn)
			return err
		})
		send_stdin := docker.Go(func() error {
			_, err := io.Copy(conn, os.Stdin)
			if err := conn.CloseWrite(); err != nil {
				log.Printf("Couldn't send EOF: " + err.Error())
			}
			return err
		})
		if err := <-receive_stdout; err != nil {
			return err
		}
		if !term.IsTerminal(0) {
			if err := <-send_stdin; err != nil {
				return err
			}
		}
	} else {
		service, err := docker.NewServer()
		if err != nil {
			return err
		}
		if err := rcli.LocalCall(service, os.Stdin, os.Stdout, args...); err != nil {
			return err
		}
	}
	if oldState != nil {
		term.Restore(0, oldState)
	}
	return nil
}
```