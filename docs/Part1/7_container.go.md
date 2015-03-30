Container的构造
==================
**docker 中最重要的模块**
`image`生成的可用实例就是容器`Container`。
``` golang
type Container struct {
	root string

	Id string
    //# 创建时间
	Created time.Time

	Path string
	//# 启动参数
	Args []string
    //# toDisk 写入配置文件
	Config *Config
	//# 状态
	State  State
	//# 镜像id
	Image  string

    //# 网络接口
	network         *NetworkInterface
	NetworkSettings *NetworkSettings

	SysInitPath string
	cmd         *exec.Cmd
	stdout      *writeBroadcaster
	stderr      *writeBroadcaster
	stdin       io.ReadCloser
	stdinPipe   io.WriteCloser

	stdoutLog *os.File
	stderrLog *os.File
	runtime   *Runtime
}
```

Container的方法
==================
``` golang

func (container *Container) Cmd() *exec.Cmd {
	return container.cmd
}

func (container *Container) When() time.Time {
	return container.Created
}

//# FromDisk 读取容器`json`配置
func (container *Container) FromDisk() error {
	data, err := ioutil.ReadFile(container.jsonPath())
	if err != nil {
		return err
	}
	//# 加载容器配置
	if err := json.Unmarshal(data, container); err != nil {
		return err
	}
	return nil
}

func (container *Container) ToDisk() (err error) {
	data, err := json.Marshal(container)
	if err != nil {
		return
	}
	return ioutil.WriteFile(container.jsonPath(), data, 0666)
}

//# generateLXCConfig 调用`lxc_template:LxcTemplateCompiled`得到LXC的配置
func (container *Container) generateLXCConfig() error {
	fo, err := os.Create(container.lxcConfigPath())
	if err != nil {
		return err
	}
	defer fo.Close()
	if err := LxcTemplateCompiled.Execute(fo, container); err != nil {
		return err
	}
	return nil
}

//# startPty 调用`github.com/kr/pty`启动一个终端
//# pty 介绍请查看 http://en.wikipedia.org/wiki/Pseudoterminal.
func (container *Container) startPty() error {
	stdout_master, stdout_slave, err := pty.Open()
	if err != nil {
		return err
	}
	container.cmd.Stdout = stdout_slave

	stderr_master, stderr_slave, err := pty.Open()
	if err != nil {
		return err
	}
	container.cmd.Stderr = stderr_slave

	// Copy the PTYs to our broadcasters
	//# TODO
	go func() {
		defer container.stdout.Close()
		io.Copy(container.stdout, stdout_master)
	}()

	go func() {
		defer container.stderr.Close()
		io.Copy(container.stderr, stderr_master)
	}()

	//# 输入
	var stdin_slave io.ReadCloser
	if container.Config.OpenStdin {
		stdin_master, stdin_slave, err := pty.Open()
		if err != nil {
			return err
		}
		container.cmd.Stdin = stdin_slave
		// FIXME: The following appears to be broken.
		// "cannot set terminal process group (-1): Inappropriate ioctl for device"
		// container.cmd.SysProcAttr = &syscall.SysProcAttr{Setctty: true, Setsid: true}
		go func() {
			defer container.stdin.Close()
			io.Copy(stdin_master, container.stdin)
		}()
	}
	if err := container.cmd.Start(); err != nil {
		return err
	}
	stdout_slave.Close()
	stderr_slave.Close()
	if stdin_slave != nil {
		stdin_slave.Close()
	}
	return nil
}

//# start 启动， 设置stdin, stdout, stderr句柄，并执行`cmd.Start`
func (container *Container) start() error {
	container.cmd.Stdout = container.stdout
	container.cmd.Stderr = container.stderr
	if container.Config.OpenStdin {
		stdin, err := container.cmd.StdinPipe()
		if err != nil {
			return err
		}
		go func() {
			defer stdin.Close()
			io.Copy(stdin, container.stdin)
		}()
	}
	return container.cmd.Start()
}

//# Start 启动 
//# 1. 确保镜像挂载成功
//# 2. 分配网络
//# 3. 加载LXC的配置
//# 4. 处理参数， -g 网络，-u 用户，-- 程序
//# 5. 设置环境
//# 6. 启动
//# 7. 在状态`State`中写入pid，把配置保存到`json`文件中，
//# 8. 启动监控`monitor`
func (container *Container) Start() error {
	if err := container.EnsureMounted(); err != nil {
		return err
	}
	if err := container.allocateNetwork(); err != nil {
		return err
	}
	if err := container.generateLXCConfig(); err != nil {
		return err
	}
	params := []string{
		"-n", container.Id,
		"-f", container.lxcConfigPath(),
		"--",
		"/sbin/init",
	}

	// 网络
	params = append(params, "-g", container.network.Gateway.String())

	// 用户
	if container.Config.User != "" {
		params = append(params, "-u", container.Config.User)
	}

	// 程序
	params = append(params, "--", container.Path)
	params = append(params, container.Args...)

	container.cmd = exec.Command("/usr/bin/lxc-start", params...)

	// 设置环境
	container.cmd.Env = append(
		[]string{
			"HOME=/",
			"PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
		},
		container.Config.Env...,
	)

	var err error
	if container.Config.Tty {
		err = container.startPty()
	} else {
		err = container.start()
	}
	if err != nil {
		return err
	}
	// FIXME: save state on disk *first*, then converge
	// this way disk state is used as a journal, eg. we can restore after crash etc.
	container.State.setRunning(container.cmd.Process.Pid)
	container.ToDisk()
	go container.monitor()
	return nil
}

func (container *Container) Run() error {
	if err := container.Start(); err != nil {
		return err
	}
	container.Wait()
	return nil
}

func (container *Container) Output() (output []byte, err error) {
	pipe, err := container.StdoutPipe()
	if err != nil {
		return nil, err
	}
	defer pipe.Close()
	if err := container.Start(); err != nil {
		return nil, err
	}
	output, err = ioutil.ReadAll(pipe)
	container.Wait()
	return output, err
}


//# StdinPipe 返回连接到容器活动进程的标准输入管道。
func (container *Container) StdinPipe() (io.WriteCloser, error) {
	return container.stdinPipe, nil
}

func (container *Container) StdoutPipe() (io.ReadCloser, error) {
	reader, writer := io.Pipe()
	container.stdout.AddWriter(writer)
	return newBufReader(reader), nil
}

func (container *Container) StderrPipe() (io.ReadCloser, error) {
	reader, writer := io.Pipe()
	container.stderr.AddWriter(writer)
	return newBufReader(reader), nil
}

//# allocateNetwork 按照容器的端口配置来分配网络
func (container *Container) allocateNetwork() error {
	iface, err := container.runtime.networkManager.Allocate()
	if err != nil {
		return err
	}
	container.NetworkSettings.PortMapping = make(map[string]string)
	for _, port := range container.Config.Ports {
		if extPort, err := iface.AllocatePort(port); err != nil {
			iface.Release()
			return err
		} else {
			container.NetworkSettings.PortMapping[strconv.Itoa(port)] = strconv.Itoa(extPort)
		}
	}
	container.network = iface
	container.NetworkSettings.IpAddress = iface.IPNet.IP.String()
	container.NetworkSettings.IpPrefixLen, _ = iface.IPNet.Mask.Size()
	container.NetworkSettings.Gateway = iface.Gateway.String()
	return nil
}

//# releaseNetwork 释放网络，见`network:NetworkInterface:Release`
func (container *Container) releaseNetwork() error {
	err := container.network.Release()
	container.network = nil
	container.NetworkSettings = &NetworkSettings{}
	return err
}

//# monitor 监控
//# 1. 等待程序退出
//# 2. 释放网络
//# 3. 关闭文件句柄，并卸载容器
//# 4. 重新打开输入
//# 5. 设置停止状态
//# 6. 写入容器配置
func (container *Container) monitor() {
	// 等待程序退出
	container.cmd.Wait()
	exitCode := container.cmd.ProcessState.Sys().(syscall.WaitStatus).ExitStatus()

	// 释放网络
	if err := container.releaseNetwork(); err != nil {
		log.Printf("%v: Failed to release network: %v", container.Id, err)
	}
	container.stdout.Close()
	container.stderr.Close()
	if err := container.Unmount(); err != nil {
		log.Printf("%v: Failed to umount filesystem: %v", container.Id, err)
	}

	// 一旦容器退出， 重新创建了一个全新的标准输入管道
	if container.Config.OpenStdin {
		container.stdin, container.stdinPipe = io.Pipe()
	}

	// 报告状态
	container.State.setStopped(exitCode)
	container.ToDisk()
}

func (container *Container) kill() error {
	if err := container.cmd.Process.Kill(); err != nil {
		return err
	}
	//# 等待容器确实停止
	container.Wait()
	return nil
}

func (container *Container) Kill() error {
	if !container.State.Running {
		return nil
	}
	return container.kill()
}

//# Stop 发一个`SIGTERM`信号， 10秒后条用`Kill`方法结束容器
func (container *Container) Stop() error {
	if !container.State.Running {
		return nil
	}

	//# 1. 发一个`SIGTERM`信号
	if output, err := exec.Command("/usr/bin/lxc-kill", "-n", container.Id, "15").CombinedOutput(); err != nil {
		log.Printf(string(output))
		log.Printf("Failed to send SIGTERM to the process, force killing")
		if err := container.Kill(); err != nil {
			return err
		}
	}

	//# 2.  等待容器进程自己结束
	if err := container.WaitTimeout(10 * time.Second); err != nil {
		log.Printf("Container %v failed to exit within 10 seconds of SIGTERM - using the force", container.Id)
		if err := container.Kill(); err != nil {
			return err
		}
	}
	return nil
}

func (container *Container) Restart() error {
	if err := container.Stop(); err != nil {
		return err
	}
	if err := container.Start(); err != nil {
		return err
	}
	return nil
}

// 阻塞到容器停止运行，然后返回它的退出代码。
func (container *Container) Wait() int {

	for container.State.Running {
		container.State.wait()
	}
	return container.State.ExitCode
}

//# ExportRw 压缩`rw`路径中的文件
func (container *Container) ExportRw() (Archive, error) {
	return Tar(container.rwPath(), Uncompressed)
}

//# Export 压缩`rootfs`路径中的文件
func (container *Container) Export() (Archive, error) {
	if err := container.EnsureMounted(); err != nil {
		return nil, err
	}
	return Tar(container.RootfsPath(), Uncompressed)
}

//# WaitTimeout 使用消息队列， 等待完成
func (container *Container) WaitTimeout(timeout time.Duration) error {
	done := make(chan bool)
	go func() {
		container.Wait()
		done <- true
	}()

	select {
	case <-time.After(timeout):
		return errors.New("Timed Out")
	case <-done:
		return nil
	}
	return nil
}

//# EnsureMounted 确认已经挂载
func (container *Container) EnsureMounted() error {
	if mounted, err := container.Mounted(); err != nil {
		return err
	} else if mounted {
		return nil
	}
	return container.Mount()
}

//# Mount 挂载镜像`image`到可读写目录中
func (container *Container) Mount() error {
	image, err := container.GetImage()
	if err != nil {
		return err
	}
	return image.Mount(container.RootfsPath(), container.rwPath())
}

//# Changes 出现修改，则返回镜像`image`的所有层`layers`的改变。 详见`image.Changes`
func (container *Container) Changes() ([]Change, error) {
	image, err := container.GetImage()
	if err != nil {
		return nil, err
	}
	return image.Changes(container.rwPath())
}

//# GetImage 返回容器对应的镜像`image`
func (container *Container) GetImage() (*Image, error) {
	if container.runtime == nil {
		return nil, fmt.Errorf("Can't get image of unregistered container")
	}
	return container.runtime.graph.Get(container.Image)
}

//# Mounted 已经挂载， 检查`rootfs`目录
func (container *Container) Mounted() (bool, error) {
	return Mounted(container.RootfsPath())
}

//# Unmount TODO
func (container *Container) Unmount() error {
	return Unmount(container.RootfsPath())
}

func (container *Container) logPath(name string) string {
	return path.Join(container.root, fmt.Sprintf("%s-%s.log", container.Id, name))
}

func (container *Container) ReadLog(name string) (io.Reader, error) {
	return os.Open(container.logPath(name))
}

func (container *Container) jsonPath() string {
	return path.Join(container.root, "config.json")
}

func (container *Container) lxcConfigPath() string {
	return path.Join(container.root, "config.lxc")
}

//# This method must be exported to be used from the lxc template
//# TODO 这个方法只能用在lxc 模板中被导出
func (container *Container) RootfsPath() string {
	return path.Join(container.root, "rootfs")
}

//# rwPath 读写单独放在一个`rw`目录
func (container *Container) rwPath() string {
	return path.Join(container.root, "rw")
}

//# validateId 验证id
func validateId(id string) error {
	if id == "" {
		return fmt.Errorf("Invalid empty id")
	}
	return nil
}
```