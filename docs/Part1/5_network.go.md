常量
==================
``` golang
const (
	networkBridgeIface = "lxcbr0"
	portRangeStart     = 49153
	portRangeEnd       = 65535
)
```

PortMapper的构造
==================
//# 端口映射器`PortMapper`通过设置iptables规则，管理容器`container`的外部端口。
//# 它跟踪所有的映射关系，能够随意取消映射
type PortMapper struct {
	mapping map[int]net.TCPAddr
}

```

PortMapper的方法
==================
``` golang
//# cleanup 清理
func (mapper *PortMapper) cleanup() error {
	// 忽略错误 - 这意味着链接可能从未建立
	iptables("-t", "nat", "-D", "PREROUTING", "-j", "DOCKER")
	iptables("-t", "nat", "-D", "OUTPUT", "-j", "DOCKER")
	iptables("-t", "nat", "-F", "DOCKER")
	iptables("-t", "nat", "-X", "DOCKER")
	mapper.mapping = make(map[int]net.TCPAddr)
	return nil
}
//# setup 使用`iptables`设定`NAT`
func (mapper *PortMapper) setup() error {
	if err := iptables("-t", "nat", "-N", "DOCKER"); err != nil {
		return errors.New("Unable to setup port networking: Failed to create DOCKER chain")
	}
	if err := iptables("-t", "nat", "-A", "PREROUTING", "-j", "DOCKER"); err != nil {
		return errors.New("Unable to setup port networking: Failed to inject docker in PREROUTING chain")
	}
	if err := iptables("-t", "nat", "-A", "OUTPUT", "-j", "DOCKER"); err != nil {
		return errors.New("Unable to setup port networking: Failed to inject docker in OUTPUT chain")
	}
	return nil
}

//# iptablesForward `iptable`转发
func (mapper *PortMapper) iptablesForward(rule string, port int, dest net.TCPAddr) error {
	return iptables("-t", "nat", rule, "DOCKER", "-p", "tcp", "--dport", strconv.Itoa(port),
		"-j", "DNAT", "--to-destination", net.JoinHostPort(dest.IP.String(), strconv.Itoa(dest.Port)))
}

//# Map 设置对`dest`的转发， 并记录在`mapper.mapping`中
func (mapper *PortMapper) Map(port int, dest net.TCPAddr) error {
	if err := mapper.iptablesForward("-A", port, dest); err != nil {
		return err
	}
	mapper.mapping[port] = dest
	return nil
}

//# Unmap 取消端口`port`的转发设置
func (mapper *PortMapper) Unmap(port int) error {
	dest, ok := mapper.mapping[port]
	if !ok {
		return errors.New("Port is not mapped")
	}
	if err := mapper.iptablesForward("-D", port, dest); err != nil {
		return err
	}
	delete(mapper.mapping, port)
	return nil
}
```

NetworkManager的构造
==================
``` golang
//# 网络管理器管理的一组网络接口
//# 每台主机只有**一个**网络管理器被使用
type NetworkManager struct {
	bridgeIface   string
	bridgeNetwork *net.IPNet
	
	ipAllocator   *IPAllocator
	portAllocator *PortAllocator
	portMapper    *PortMapper
}
```

NetworkManager的方法
==================
``` golang
//# Allocate 分配一个网络接口
func (manager *NetworkManager) Allocate() (*NetworkInterface, error) {
	ip, err := manager.ipAllocator.Acquire()
	if err != nil {
		return nil, err
	}
	iface := &NetworkInterface{
		IPNet:   net.IPNet{IP: ip, Mask: manager.bridgeNetwork.Mask},
		Gateway: manager.bridgeNetwork.IP,
		manager: manager,
	}
	return iface, nil
}

```

ipAllocator的构造
==================
``` golang
// IP分配器：自动分配和释放网络端口
type IPAllocator struct {
	network *net.IPNet
	queue   chan (net.IP)
}
```

ipAllocator的方法
==================
``` golang
//# populate 
func (alloc *IPAllocator) populate() error {
	firstIP, _ := networkRange(alloc.network)
	size, err := networkSize(alloc.network.Mask)
	if err != nil {
		return err
	}
	//# 队列大小应该是网络规模 - 3
	//# 网络地址, 广播地址, 网关地址
	alloc.queue = make(chan net.IP, size-3)
	for i := int32(1); i < size-1; i++ {
		ipNum, err := ipToInt(firstIP)
		if err != nil {
			return err
		}
		ip, err := intToIp(ipNum + int32(i))
		if err != nil {
			return err
		}
		// Discard the network IP (that's the host IP address)
		if ip.Equal(alloc.network.IP) {
			continue
		}
		alloc.queue <- ip
	}
	return nil
}
//# Acquire 获取一个ip
func (alloc *IPAllocator) Acquire() (net.IP, error) {
	select {
	case ip := <-alloc.queue:
		return ip, nil
	default:
		return net.IP{}, errors.New("No more IP addresses available")
	}
	return net.IP{}, nil
}

func (alloc *IPAllocator) Release(ip net.IP) error {
	select {
	case alloc.queue <- ip:
		return nil
	default:
		return errors.New("Too many IP addresses have been released")
	}
	return nil
}
```

PortAllocator的构造
==================
``` golang
//# 端口分配器：自动分配和释放网络端口
type PortAllocator struct {
	ports chan (int)
}

```

PortAllocator的方法
==================
``` golang
func (alloc *PortAllocator) populate(start, end int) {
	alloc.ports = make(chan int, end-start)
	for port := start; port < end; port++ {
		alloc.ports <- port
	}
}

func (alloc *PortAllocator) Acquire() (int, error) {
	select {
	case port := <-alloc.ports:
		return port, nil
	default:
		return -1, errors.New("No more ports available")
	}
	return -1, nil
}

func (alloc *PortAllocator) Release(port int) error {
	select {
	case alloc.ports <- port:
		return nil
	default:
		return errors.New("Too many ports have been released")
	}
	return nil
}

```

NetworkInterface的构造
==================
``` golang

// 网络接口表示容器`container`的网络堆栈
type NetworkInterface struct {
	IPNet   net.IPNet
	Gateway net.IP

	manager  *NetworkManager
	extPorts []int
}

```

NetworkInterface的方法
==================
``` golang
//# AllocatePort 分配外部TCP端口，并将其映射到接口
func (iface *NetworkInterface) AllocatePort(port int) (int, error) {
	extPort, err := iface.manager.portAllocator.Acquire()
	if err != nil {
		return -1, err
	}
	if err := iface.manager.portMapper.Map(extPort, net.TCPAddr{IP: iface.IPNet.IP, Port: port}); err != nil {
		iface.manager.portAllocator.Release(extPort)
		return -1, err
	}
	iface.extPorts = append(iface.extPorts, extPort)
	return extPort, nil
}

//# Release 发布，网络清理，释放所有的资源
func (iface *NetworkInterface) Release() error {
	for _, port := range iface.extPorts {
		if err := iface.manager.portMapper.Unmap(port); err != nil {
			log.Printf("Unable to unmap port %v: %v", port, err)
		}
		if err := iface.manager.portAllocator.Release(port); err != nil {
			log.Printf("Unable to release port %v: %v", port, err)
		}

	}
	return iface.manager.ipAllocator.Release(iface.IPNet.IP)
}
```

###util
``` golang 
//# networkRange 计算网络`IPNET`的第一和最后一个IP地址
func networkRange(network *net.IPNet) (net.IP, net.IP) {
	netIP := network.IP.To4()
	firstIP := netIP.Mask(network.Mask)
	lastIP := net.IPv4(0, 0, 0, 0).To4()
	for i := 0; i < len(lastIP); i++ {
		lastIP[i] = netIP[i] | ^network.Mask[i]
	}
	return firstIP, lastIP
}

//# ipToInt 将一个IPv4地址转换为32位整数
func ipToInt(ip net.IP) (int32, error) {
	buf := bytes.NewBuffer(ip.To4())
	var n int32
	if err := binary.Read(buf, binary.BigEndian, &n); err != nil {
		return 0, err
	}
	return n, nil
}

// intToIp 将一个32位整数转换为IPv4地址
func intToIp(n int32) (net.IP, error) {
	var buf bytes.Buffer
	if err := binary.Write(&buf, binary.BigEndian, &n); err != nil {
		return net.IP{}, err
	}
	ip := net.IPv4(0, 0, 0, 0).To4()
	for i := 0; i < net.IPv4len; i++ {
		ip[i] = buf.Bytes()[i]
	}
	return ip, nil
}

//# networkSize 通过网络掩码计算可用的主机数
func networkSize(mask net.IPMask) (int32, error) {
	m := net.IPv4Mask(0, 0, 0, 0)
	for i := 0; i < net.IPv4len; i++ {
		m[i] = ^mask[i]
	}
	buf := bytes.NewBuffer(m)
	var n int32
	if err := binary.Read(buf, binary.BigEndian, &n); err != nil {
		return 0, err
	}
	return n + 1, nil
}

//# iptables 封装iptables命令
func iptables(args ...string) error {
	if err := exec.Command("/sbin/iptables", args...).Run(); err != nil {
		return fmt.Errorf("iptables failed: iptables %v", strings.Join(args, " "))
	}
	return nil
}

//# getIfaceAddr 返回一个网络接口的IPv4地址
func getIfaceAddr(name string) (net.Addr, error) {
	iface, err := net.InterfaceByName(name)
	if err != nil {
		return nil, err
	}
	addrs, err := iface.Addrs()
	if err != nil {
		return nil, err
	}
	var addrs4 []net.Addr
	for _, addr := range addrs {
		ip := (addr.(*net.IPNet)).IP
		if ip4 := ip.To4(); len(ip4) == net.IPv4len {
			addrs4 = append(addrs4, addr)
		}
	}
	switch {
	case len(addrs4) == 0:
		return nil, fmt.Errorf("Interface %v has no IP addresses", name)
	case len(addrs4) > 1:
		fmt.Printf("Interface %v has more than 1 IPv4 address. Defaulting to using %v\n",
			name, (addrs4[0].(*net.IPNet)).IP)
	}
	return addrs4[0], nil
}
```
