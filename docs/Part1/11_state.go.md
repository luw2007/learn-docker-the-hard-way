State的构造
==================
type State struct {
	Running   bool
	Pid       int
	ExitCode  int
	StartedAt time.Time
    //# 状态变化锁
	stateChangeLock *sync.Mutex
	//# `Condition`
	stateChangeCond *sync.Cond
}
```

Container的方法
==================
``` golang
// String 返回状态的可读描述
func (s *State) String() string {
	if s.Running {
		return fmt.Sprintf("Up %s", HumanDuration(time.Now().Sub(s.StartedAt)))
	}
	return fmt.Sprintf("Exit %d", s.ExitCode)
}

func (s *State) setRunning(pid int) {
	s.Running = true
	s.ExitCode = 0
	s.Pid = pid
	s.StartedAt = time.Now()
	s.broadcast()
}

func (s *State) setStopped(exitCode int) {
	s.Running = false
	s.Pid = 0
	s.ExitCode = exitCode
	s.broadcast()
}

//# broadcast 解除`Condition`等待
func (s *State) broadcast() {
	s.stateChangeLock.Lock()
	s.stateChangeCond.Broadcast()
	s.stateChangeLock.Unlock()
}

//# wait 设置`Condition`等待
func (s *State) wait() {
	s.stateChangeLock.Lock()
	s.stateChangeCond.Wait()
	s.stateChangeLock.Unlock()
}
```