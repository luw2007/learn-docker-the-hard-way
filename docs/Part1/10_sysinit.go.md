sysinit.go
==========
系统初始化模块，对外暴露`SysInit`方法。
##SysInit
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

##setupNetworking
``` golang
// 设置网关
func setupNetworking(gw string) {
	if gw == "" {
		return
	}
	cmd := exec.Command("/sbin/route", "add", "default", "gw", gw)
	if err := cmd.Run(); err != nil {
		log.Fatalf("Unable to set up networking: %v", err)
	}
}
```

##changeUser
``` golang
// 降低权限以便照顾普通用户
func changeUser(u string) {
	if u == "" {
		return
	}
	userent, err := user.LookupId(u)
	if err != nil {
		userent, err = user.Lookup(u)
	}
	if err != nil {
		log.Fatalf("Unable to find user %v: %v", u, err)
	}

	uid, err := strconv.Atoi(userent.Uid)
	if err != nil {
		log.Fatalf("Invalid uid: %v", userent.Uid)
	}
	gid, err := strconv.Atoi(userent.Gid)
	if err != nil {
		log.Fatalf("Invalid gid: %v", userent.Gid)
	}

	if err := syscall.Setgid(gid); err != nil {
		log.Fatalf("setgid failed: %v", err)
	}
	if err := syscall.Setuid(uid); err != nil {
		log.Fatalf("setuid failed: %v", err)
	}
}

```

##executeProgram
``` golang
func executeProgram(name string, args []string) {
	path, err := exec.LookPath(name)
	if err != nil {
		log.Printf("Unable to locate %v", name)
		os.Exit(127)
	}

	if err := syscall.Exec(path, args, os.Environ()); err != nil {
		panic(err)
	}
}

```