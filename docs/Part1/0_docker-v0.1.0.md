docker v0.1.0源码结构
=====================
##目录
为了了解`docker`的核心内容， 我们先切换到`v0.1.0`版本，看看基本目录。为了显示简单，部分文件夹和文件使用(...)省略，如`*_test.go`, `deb`打包相关的代码等。目录和文件如下

    auth
        auth.go
    contrib
        install.sh  
        mkimage-busybox.sh
        README
    deb
        debian
            source
                ...
            ...
        etc
            ...
        ...
    docker
        docker.go
    docs
        images-repositories-push-pull.md
        README.md
    puppet
        manifests
            ...
        modules
            docker
                ...
    rcli
        http.go
        tcp.go  
        types.go
    term
        term.go
        termios_darwin.go
        termios_linux.go
    archive.go
    changes.go
    commands.go
    container.go
    graph.go
    image.go
    lxc_template.go
    mount.go
    mount_darwin.go
    mount_linux.go
    network.go
    registry.go
    runtime.go
    state.go
    sysinit.go
    tags.go
    utils.go
    README.md
    Vagrantfile
    ...
##详解
其入口可以看作为 `docker/docker.go`， 我们先从上到下了解一下文件夹的用途。

1.  `auth` 里面只有 `auth.go` 和 `auth_test.go`， 主要实现服务器的认证。
2.  `contrib` 一些脚本的源码。
3.  `deb`  `Debian`系统打包源码。
4.  `docker` 包含`docker.go`， 实现docker核心功能。
5.  `docs` 文档目录
6.  `puppet` 基于`puppet`部署工具的相关源码。
7.  `rcli` `Remote Command-Line Interface` 一个简单协议的命令行界面。
8.  `term` Term 的跨平台支持。
9.  `README` 帮助文档，主要是介绍、特性、安装、使用、开发环境、容器的介绍等。
10.  根目录中其它.go文件各有用途，在源码阅读中我们会接触到。

####__author__ = 'luw2007@gmail.com'