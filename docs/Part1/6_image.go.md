Image的构造
==================
在`graph`中我们已经见过`image`的一些方法。比如`LoadImage``StoreImage`。
``` golang
type Image struct {
	Id              string    `json:"id"`
	//# 父镜像，该镜像基于某个父镜像创建。
	Parent          string    `json:"parent,omitempty"`
	Comment         string    `json:"comment,omitempty"`
	Created         time.Time `json:"created"`
	//# 容器名称
	Container       string    `json:"container,omitempty"`
	//# 容器配置
	ContainerConfig Config    `json:"container_config,omitempty"`
	//# Graph.Get 读取image 信息后，再把`graph`赋值给`image.graph`
	graph           *Graph
}
```

Image的方法
==================
``` golang
//# MountAUFS `docker`使用`AUFS`作为存储后端。 
//# 扩展阅读 [Why use AUFS as the default Docker storage backend instead of devicemapper?](http://stackoverflow.com/questions/24764908)
func MountAUFS(ro []string, rw string, target string) error {
	// FIXME: Now mount the layers
	rwBranch := fmt.Sprintf("%v=rw", rw)
	roBranches := ""
	for _, layer := range ro {
		roBranches += fmt.Sprintf("%v=ro:", layer)
	}
	branches := fmt.Sprintf("br:%v:%v", rwBranch, roBranches)
	return mount("none", target, "aufs", 0, branches)
}

//# Mount
func (image *Image) Mount(root, rw string) error {
	if mounted, err := Mounted(root); err != nil {
		return err
	} else if mounted {
		return fmt.Errorf("%s is already mounted", root)
	}
	layers, err := image.layers()
	if err != nil {
		return err
	}
	// 如果目标目录不存在，就创建它们。
	if err := os.Mkdir(root, 0755); err != nil && !os.IsExist(err) {
		return err
	}
	if err := os.Mkdir(rw, 0755); err != nil && !os.IsExist(err) {
		return err
	}
	// FIXME: @creack shouldn't we do this after going over changes?
	if err := MountAUFS(layers, rw, root); err != nil {
		return err
	}
	// FIXME: Create tests for deletion
	// FIXME: move this part to change.go
	// Retrieve the changeset from the parent and apply it to the container
	//  - Retrieve the changes
	changes, err := Changes(layers, layers[0])
	if err != nil {
		return err
	}
	// 遍历变化
	for _, c := range changes {
		// 如果这里是删除状态
		if c.Kind == ChangeDelete {
			// 确认目录已经存在
			file_path, file_name := path.Dir(c.Path), path.Base(c.Path)
			if err := os.MkdirAll(path.Join(rw, file_path), 0755); err != nil {
				return err
			}
			// 创建空文件（我们只需创建空文件，并继续） 
			if _, err := os.Create(path.Join(path.Join(rw, file_path),
				".wh."+path.Base(file_name))); err != nil {
				return err
			}
		}
	}
	return nil
}

//# Changes 返回镜像`images`的层`layers`
func (image *Image) Changes(rw string) ([]Change, error) {
	layers, err := image.layers()
	if err != nil {
		return nil, err
	}
	return Changes(layers, rw)
}

//# ValidateId 验证镜像`id`是否正确
func ValidateId(id string) error {
	if id == "" {
		return fmt.Errorf("Image id can't be empty")
	}
	if strings.Contains(id, ":") {
		return fmt.Errorf("Invalid character in image id: ':'")
	}
	return nil
}

//# GenerateId 生成镜像id, 当前时间的纳秒作为随机种子。
func GenerateId() string {
	// FIXME: don't seed every time
	rand.Seed(time.Now().UTC().UnixNano())
	randomBytes := bytes.NewBuffer([]byte(fmt.Sprintf("%x", rand.Int())))
	id, _ := ComputeId(randomBytes) // can't fail
	return id
}

//# ComputeId 从`content`读取到EOF，然后返回使用一个SHA加密成的字符串。 
func ComputeId(content io.Reader) (string, error) {
	h := sha256.New()
	if _, err := io.Copy(h, content); err != nil {
		return "", err
	}
	return fmt.Sprintf("%x", h.Sum(nil)[:8]), nil
}

//# History 镜像`Image`很方便地包括了图的代理函数
//# 如果镜像`Image`未注册这些函数将返回一个错误（如： if image.graph == nil）
func (img *Image) History() ([]*Image, error) {
	var parents []*Image
	if err := img.WalkHistory(
		func(img *Image) error {
			parents = append(parents, img)
			return nil
		},
	); err != nil {
		return nil, err
	}
	return parents, nil
}

//# layers `layers`返回一个镜像`image`挂载所需的所有文件层`layers`.
// FIXME: @shykes refactor this function with the new error handling
//        (I'll do it if I have time tonight, I focus on the rest)
func (img *Image) layers() ([]string, error) {
	var list []string
	var e error
	if err := img.WalkHistory(
		func(img *Image) (err error) {
			if layer, err := img.layer(); err != nil {
				e = err
			} else if layer != "" {
				list = append(list, layer)
			}
			return err
		},
	); err != nil {
		return nil, err
	} else if e != nil { // Did an error occur inside the handler?
		return nil, e
	}
	if len(list) == 0 {
		return nil, fmt.Errorf("No layer found for image %s\n", img.Id)
	}
	return list, nil
}

//# WalkHistory 遍历图像的结构，检查父镜像是否存在
func (img *Image) WalkHistory(handler func(*Image) error) (err error) {
	currentImg := img
	for currentImg != nil {
		if handler != nil {
			if err := handler(currentImg); err != nil {
				return err
			}
		}
		currentImg, err = currentImg.GetParent()
		if err != nil {
			return fmt.Errorf("Error while getting parent image: %v", err)
		}
	}
	return nil
}

//# GetParent 通过`img.Parent`获取父镜像
func (img *Image) GetParent() (*Image, error) {
	if img.Parent == "" {
		return nil, nil
	}
	if img.graph == nil {
		return nil, fmt.Errorf("Can't lookup parent of unregistered image")
	}
	return img.graph.Get(img.Parent)
}

func (img *Image) root() (string, error) {
	if img.graph == nil {
		return "", fmt.Errorf("Can't lookup root of unregistered image")
	}
	return img.graph.imageRoot(img.Id), nil
}

//# layer 返回镜像`Image`的层`layer`的路径
func (img *Image) layer() (string, error) {
	root, err := img.root()
	if err != nil {
		return "", err
	}
	return layerPath(root), nil
}
```

##mount.go:Mounted
``` golang
//# Mounted 是否已经挂载
func Mounted(mountpoint string) (bool, error) {
	mntpoint, err := os.Stat(mountpoint)
	if err != nil {
		if os.IsNotExist(err) {
			return false, nil
		}
		return false, err
	}
	parent, err := os.Stat(filepath.Join(mountpoint, ".."))
	if err != nil {
		return false, err
	}
	mntpointSt := mntpoint.Sys().(*syscall.Stat_t)
	parentSt := parent.Sys().(*syscall.Stat_t)
	return mntpointSt.Dev != parentSt.Dev, nil
}
```
