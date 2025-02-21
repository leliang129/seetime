---
title: Docker镜像存储机制
published: 2025-02-21
tags: [docker]
category: docker
draft: false
---
## Linux系统运行基础
> Linux系统正常运行，通常需要两个文件系统
### boot file system（bootfs）
1）包含Boot Loader与Kernel文件，用户不能修改这些文件。并且在系统启动过程完成之后，整个系统的内核都会被加载进内存。此时bootfs会被卸载，从而释放出所占用的系统内存。     
2）在容器中可以运行不同版本的Linux，说明对于同样内核版本的不同Linux发行版的bootfs都是一致的，否则会无法启动。因此可以推断，docker是需要内核支持的    
3）Linux系统中典型的bootfs目录：(核心)/boot/vmlinuz、(核心解压缩所需RAM Disk) /boot/initramfs   

### root file system (rootfs)
1）不同的Linux发行版本bootfs相同，rootfs不同（二进制文件）   
2）每个容器都有自己的rootfs，它来自不同的linux发行版的基础镜像，包括ubuntu，debian和suse等   
3）使用不同的rootfs就决定了在构建镜像的过程中可以使用哪些系统命令。    
4）典型的rootfs目录：/dev、/proc、/bin、/etc、/lib、/usr     

## OverlayFS层次
OverlayFS结构分为三个层：LowerDir、Upperdir、MergedDir   
- LowerDir(只读)
   - 只读的image layer，其实就是rootfs，在使用dockerfile构建镜像的时候，image layer可以分很多层，所以对应的lowerfir会很多（镜像源）
- UpperDir(读写)
   - upperdir则是在lowerdir之上的一层为读写层。容器在启动的时候会创建，所有对容器的修改都是在这层。比如容器启动写入的日志文件，或者是应用程序写入的临时文件。
- MergedDir(展示)
   - mergerd目录是容器的挂载点，在用户视角能够看到的所有文件，都是从这层展示。

## 查看容器的目录存储结构
### 启动容器
>前台启动，直接进入到容器
```shell
docker run -it wcjiang/reference:latest bash
```
### 查看容器挂载信息
>容器启动以后，挂载merged、lowerdir、upperdir以及workdir目录   
lowerdir是只读的image layer，其实就是rootfs

```
# 获取容器ID
docker ps
```

### 容器存储目录信息
>注意在所有的启动容器中会自动添加init目录，此目录用于存储系统的hostname与域名解析文件
```shell
docker inspect 63c8df68a5066   

        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/4a014b7d10ad06b0aa7366f4d702be3c5b22649712676af3855be43ef7a64655-init/diff:/var/lib/docker/overlay2/a84553f16629629beb74ff0f1331f21144a555a000963d42770fbd236bde7d72/diff:/var/lib/docker/overlay2/447de51ccdb8ba1840c5216cfbf5228a79e7cf111c874b5063bea9a5270f8050/diff:/var/lib/docker/overlay2/f64414cf101f6de64337107fd61db00d2a57c001853f682ea076a918dfd1acca/diff:/var/lib/docker/overlay2/b0a46b72afe239427a680beeab4ab973e9ca1780e9c4fd4c59fbc35306e72b45/diff:/var/lib/docker/overlay2/f0ed5980d82f24ddc621388aab9d8b93b7458278c10b1339b050fc2403660268/diff",
                "MergedDir": "/var/lib/docker/overlay2/4a014b7d10ad06b0aa7366f4d702be3c5b22649712676af3855be43ef7a64655/merged",
                "UpperDir": "/var/lib/docker/overlay2/4a014b7d10ad06b0aa7366f4d702be3c5b22649712676af3855be43ef7a64655/diff",
                "WorkDir": "/var/lib/docker/overlay2/4a014b7d10ad06b0aa7366f4d702be3c5b22649712676af3855be43ef7a64655/work"
            },
            "Name": "overlay2"
        },
```

## 容器文件系统

### 查看init层地址指向
>在容器启动过程中，Lower会自动挂载init的一些文件
```shell
root@localhost:~# ls /var/lib/docker/overlay2/4a014b7d10ad06b0aa7366f4d702be3c5b22649712676af3855be43ef7a64655-init/diff
dev  etc  proc  sys
```

### init层的主要内容是什么
init层是以一个uuid + -init结尾表示，放在只读层(Lower)和读写层(upperdir)之间，作用只是存放/etc/hosts、/etc/resolv.conf等文件。

### 为什么需要init层
1）容器在启动以后，默认情况下lower层是不能修改内容的，但是用户有需求需要修改`主机名与域名地址`，那么就需要添加init层中的文件(hostname，resolve.conf),用于解决此类问题。   
2）修改的内容只对当前容器生效，而在docker commit提交为镜像的时候，并不会将init层提交。   
3）init文件存放的目录为/var/lib/docker/overlay2/init-id/diff   

### 查看init层文件
>hostname与resolve.conf全部为空文件，在系统启动以后由系统写入
```shell
root@localhost:~# ls -l /var/lib/docker/overlay2/4a014b7d10ad06b0aa7366f4d702be3c5b22649712676af3855be43ef7a64655-init/diff/etc
total 0
-rwxr-xr-x 1 root root  0 Apr  9  2024 hostname
-rwxr-xr-x 1 root root  0 Apr  9  2024 hosts
lrwxrwxrwx 1 root root 12 Apr  9  2024 mtab -> /proc/mounts
-rwxr-xr-x 1 root root  0 Apr  9  2024 resolv.conf
```
:::note[总结]
1）镜像所挂载的目录为Lower层，然后通过Merged展示所有的文件目录与文件。用户写入的所有文件都是在UpperDir目录，并且会在UpperDir建立于Merged层展示的文件目录结构，所以用户就可以看到写入的文件。并且底层的镜像是不能被修改(如果挂载目录为UpperDir，则可以修改源镜像)     

2）在下次重新启动已经停止的容器的时候，如果容器的ID没有发生变化，那么所写入的文件是存在物理系统中的，反之就是一个新的容器，之前手动建立的文件是不存在的。    

3）基于容器创建的镜像，就相当于容器的快照，可以删除原来的容器，但是不能删除原来的镜像。

:::