Graph的构造
==================
``` golang

type Graph struct {
    //# `graph`目录绝对路径
	Root string
}
```

Graph的方法
==================

``` golang
//# 某个`Image`的`graph`是否存在
func (graph *Graph) Exists(id string) bool {
	if _, err := graph.Get(id); err != nil {
		return false
	}
	return true
}
//# Get 通过id，调用`LoadImage`获取`image`信息
func (graph *Graph) Get(id string) (*Image, error) {
	// FIXME: return nil when the image doesn't exist, instead of an error
	img, err := LoadImage(graph.imageRoot(id))
	if err != nil {
		return nil, err
	}
	if img.Id != id {
		return nil, fmt.Errorf("Image stored at '%s' has wrong id '%s'", id, img.Id)
	}
	img.graph = graph
	return img, nil
}

//# Create 生成`img`数据
func (graph *Graph) Create(layerData Archive, container *Container, comment string) (*Image, error) {
	img := &Image{
	    //# 使用`sha`算法生成的字符串
		Id:      GenerateId(),
		Comment: comment,
		Created: time.Now(),
	}
	//# 基于容器`Container`生成
	if container != nil {
		img.Parent = container.Image
		img.Container = container.Id
		img.ContainerConfig = *container.Config
	}
	//# 保存`Image`
	if err := graph.Register(layerData, img); err != nil {
		return nil, err
	}
	return img, nil
}

//# Register 调用`StoreImage存储`image`
func (graph *Graph) Register(layerData Archive, img *Image) error {
	if err := ValidateId(img.Id); err != nil {
		return err
	}
	// (This is a convenience to save time. Race conditions are taken care of by os.Rename)
	if graph.Exists(img.Id) {
		return fmt.Errorf("Image %s already exists", img.Id)
	}
	//# 创建临时文件夹 `graph/{imag.Id}`
	tmp, err := graph.Mktemp(img.Id)
	//# 函数执行完毕后删除临时文件夹`graph/{imag.Id}`内所有文件。
	defer os.RemoveAll(tmp)
	if err != nil {
		return fmt.Errorf("Mktemp failed: %s", err)
	}
	//# 解压`layer`并存储`json`数据。
	if err := StoreImage(img, layerData, tmp); err != nil {
		return err
	}
	// Commit
	if err := os.Rename(tmp, graph.imageRoot(img.Id)); err != nil {
		return err
	}
	//# 在img附上文件路径路径
	img.graph = graph
	return nil
}
//# Mktemp 生成临时`image`存储目录
func (graph *Graph) Mktemp(id string) (string, error) {
    //# 检查tmp目录是否存在， 不存在则创建， 并返回目录真实路径
	tmp, err := NewGraph(path.Join(graph.Root, ":tmp:"))
	if err != nil {
		return "", fmt.Errorf("Couldn't create temp: %s", err)
	}
	if tmp.Exists(id) {
		return "", fmt.Errorf("Image %d already exists", id)
	}
	return tmp.imageRoot(id), nil
}

//# Garbage 创建垃圾目录
func (graph *Graph) Garbage() (*Graph, error) {
	return NewGraph(path.Join(graph.Root, ":garbage:"))
}

//# Delete 删除，先将目录移动到`garbage`目录中
func (graph *Graph) Delete(id string) error {
	garbage, err := graph.Garbage()
	if err != nil {
		return err
	}
	return os.Rename(graph.imageRoot(id), garbage.imageRoot(id))
}

//# Undelete 取消删除， 从`garbage中移回删除的目录
func (graph *Graph) Undelete(id string) error {
	garbage, err := graph.Garbage()
	if err != nil {
		return err
	}
	return os.Rename(garbage.imageRoot(id), graph.imageRoot(id))
}

func (graph *Graph) GarbageCollect() error {
	garbage, err := graph.Garbage()
	if err != nil {
		return err
	}
	return os.RemoveAll(garbage.Root)
}

//# Map 返回遍历`images`返回一个{image.id: image}的map
func (graph *Graph) Map() (map[string]*Image, error) {
	// FIXME: this should replace All()
	all, err := graph.All()
	if err != nil {
		return nil, err
	}
	images := make(map[string]*Image, len(all))
	for _, image := range all {
		images[image.Id] = image
	}
	return images, nil
}

//# All 遍历`graph`目录，得到所有的`image`
func (graph *Graph) All() ([]*Image, error) {
	var images []*Image
	err := graph.WalkAll(func(image *Image) {
		images = append(images, image)
	})
	return images, err
}
//# WalkAll 遍历`graph`目录，使用Get方法得到所有的image
func (graph *Graph) WalkAll(handler func(*Image)) error {
	files, err := ioutil.ReadDir(graph.Root)
	if err != nil {
		return err
	}
	for _, st := range files {
		if img, err := graph.Get(st.Name()); err != nil {
			// Skip image
			continue
		} else if handler != nil {
			handler(img)
		}
	}
	return nil
}

//# ByParent 递推生成一个父子关系{image.Parent: [childre_image, ]}的map。
func (graph *Graph) ByParent() (map[string][]*Image, error) {
	byParent := make(map[string][]*Image)
	err := graph.WalkAll(func(image *Image) {
		image, err := graph.Get(image.Parent)
		if err != nil {
			return
		}
		if children, exists := byParent[image.Parent]; exists {
			byParent[image.Parent] = []*Image{image}
		} else {
			byParent[image.Parent] = append(children, image)
		}
	})
	return byParent, err
}

//# Heads 取出所有的`head`
func (graph *Graph) Heads() (map[string]*Image, error) {
	heads := make(map[string]*Image)
	byParent, err := graph.ByParent()
	if err != nil {
		return nil, err
	}
	err = graph.WalkAll(func(image *Image) {
	    //# 如果 不在 byParent 的搜索表中， 则表示它没有父`image`，既 这就是`head`。
		if _, exists := byParent[image.Id]; !exists {
			heads[image.Id] = image
		}
	})
	return heads, err
}

//# imageRoot 返回`image`
func (graph *Graph) imageRoot(id string) string {
	return path.Join(graph.Root, id)
}

```

##image.go:LoadImage
``` golang
//# 根据在graph中的目录获取`image`信息
func LoadImage(root string) (*Image, error) {
	// 读取`json`数据
	jsonData, err := ioutil.ReadFile(jsonPath(root))
	if err != nil {
		return nil, err
	}
	var img Image
	if err := json.Unmarshal(jsonData, &img); err != nil {
		return nil, err
	}
	if err := ValidateId(img.Id); err != nil {
		return nil, err
	}
	// 检查文件系统中`layer`目录是否存在
	if stat, err := os.Stat(layerPath(root)); err != nil {
		if os.IsNotExist(err) {
			return nil, fmt.Errorf("Couldn't load image %s: no filesystem layer", img.Id)
		} else {
			return nil, err
		}
	} else if !stat.IsDir() {
		return nil, fmt.Errorf("Couldn't load image %s: %s is not a directory", img.Id, layerPath(root))
	}
	return &img, nil
}

```

##image.go:StoreImage
``` golang

func layerPath(root string) string {
	return path.Join(root, "layer")
}

func GenerateId() string {
	// FIXME: don't seed every time
	rand.Seed(time.Now().UTC().UnixNano())
	randomBytes := bytes.NewBuffer([]byte(fmt.Sprintf("%x", rand.Int())))
	id, _ := ComputeId(randomBytes) // can't fail
	return id
}

// ComputeId reads from `content` until EOF, then returns a SHA of what it read, as a string.
func ComputeId(content io.Reader) (string, error) {
	h := sha256.New()
	if _, err := io.Copy(h, content); err != nil {
		return "", err
	}
	return fmt.Sprintf("%x", h.Sum(nil)[:8]), nil
}


//# 解压`layer`并存储`json`数据。
func StoreImage(img *Image, layerData Archive, root string) error {
	// Check that root doesn't already exist
	if _, err := os.Stat(root); err == nil {
		return fmt.Errorf("Image %s already exists", img.Id)
	} else if !os.IsNotExist(err) {
		return err
	}
	//# 存储`layer`
	layer := layerPath(root)
	if err := os.MkdirAll(layer, 0700); err != nil {
		return err
	}
	//# 解压`layer`到`graph/_tmp/{img.Id}/layer`目录
	if err := Untar(layerData, layer); err != nil {
		return err
	}
	// 存储json
	jsonData, err := json.Marshal(img)
	if err != nil {
		return err
	}
	if err := ioutil.WriteFile(jsonPath(root), jsonData, 0600); err != nil {
		return err
	}
	return nil
}
```
####__author__ = 'luw2007@gmail.com'