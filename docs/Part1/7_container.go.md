Container的构造
==================
`image`生成的可用实例就是容器`Container`。
``` golang
type Container struct {
	root string

	Id string

	Created time.Time

	Path string
	Args []string

	Config *Config
	State  State
	Image  string

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