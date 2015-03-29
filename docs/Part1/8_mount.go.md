mount.go
========
Mounted 已经在image.go中讲到, 这里看看`umount`。
``` golang
func Unmount(target string) error {
	if err := syscall.Unmount(target, 0); err != nil {
		return err
	}
	// Even though we just unmounted the filesystem, AUFS will prevent deleting the mntpoint
	// for some time. We'll just keep retrying until it succeeds.
	for retries := 0; retries < 1000; retries++ {
		err := os.Remove(target)
		if err == nil {
			// rm mntpoint succeeded
			return nil
		}
		if os.IsNotExist(err) {
			// mntpoint doesn't exist anymore. Success.
			return nil
		}
		// fmt.Printf("(%v) Remove %v returned: %v\n", retries, target, err)
		time.Sleep(10 * time.Millisecond)
	}
	return fmt.Errorf("Umount: Failed to umount %v", target)
}

```

mount_darwin.go
===============
这里很明确的表示不支持`darwin`操作系统。
``` golang
func mount(source string, target string, fstype string, flags uintptr, data string) (err error) {
	return errors.New("mount is not implemented on darwin")
}
```

mount_linux.go
==============
linux下直接使用系统mount 指令。
``` golang
func mount(source string, target string, fstype string, flags uintptr, data string) (err error) {
	return syscall.Mount(source, target, fstype, flags, data)
}
```
