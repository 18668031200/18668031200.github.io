---
title: docker exec 是怎么做到进入容器里的
author: ygdxd
type: 转载
date: 2021-08-10 15:56:44
tags: docker
categories: docker
---
转自极客时间深入剖析k8s
1.原理
-------------------
Linux Namespace 创建的隔离空间虽然看不见摸不着，但一个进程的 Namespace 信息在宿主机上是确确实实存在的，并且是以一个文件的方式存在。
我们可以通过docker inspect 命令获取到正在运行的 Docker 容器的进程号（PID）。
```bash
$ docker inspect --format '{{ .State.Pid }}' ${container id}

$ ls -l /proc/${Pid}/ns
```

可以看到，一个进程的每种 Linux Namespace，都在它对应的 /proc/[进程号]/ns 下有一个对应的虚拟文件，并且链接到一个真实的 Namespace 文件上。
这也就意味着：一个进程，可以选择加入到某个进程已有的 Namespace 当中，从而达到“进入”这个进程所在容器的目的，这正是 docker exec 的实现原理。
而这个操作所依赖的，乃是一个名叫 setns() 的 Linux 系统调用。
```C

#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

#define errExit(msg) do { perror(msg); exit(EXIT_FAILURE);} while (0)

int main(int argc, char *argv[]) {
    int fd;
    
    fd = open(argv[1], O_RDONLY);
    if (setns(fd, 0) == -1) {
        errExit("setns");
    }
    execvp(argv[2], &argv[2]); 
    errExit("execvp");
}
```
这段代码功能非常简单：它一共接收两个参数，第一个参数是 argv[1]，即当前进程要加入的 Namespace 文件的路径，比如 /proc/[进程号]/ns/net；
而第二个参数，则是你要在这个 Namespace 里运行的进程，比如 /bin/bash。
这段代码的核心操作，则是通过 open() 系统调用打开了指定的 Namespace 文件，并把这个文件的描述符 fd 交给 setns() 使用。
在 setns() 执行后，当前进程就加入了这个文件对应的 Linux Namespace 当中了。

同时，一旦一个进程加入到了另一个 Namespace 当中，在宿主机的 Namespace 文件上，也会有所体现。
你可以用 ps 指令找到这个 set_ns 程序执行的 /bin/bash 进程。查看一下这个 程序对应的进程的 Namespace。
2个进程指向的Network Namespace 文件完全一样。
```bash
$ ps aux | grep /bin/bash
$ ls -l /proc/[进程1]/ns/net
$ ls -l /proc/[进程2]/ns/net
```
我们可以在dockers 启动命令行中增加参数--net container:XXX来加入到另外一个容器的Network Namespace 中。


2.获取root 权限
-------------------

在公司执行了set_ns脚本后，竟然发现原有的用户变成了root。
执行id命令后
```bash
$ id
uid=0(root) gid=0(root) 组=0(root)
```
