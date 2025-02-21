---
title: docker buildx构建多平台镜像
published: 2025-02-21
tags: [linux,docker]
category: linux
draft: false
---
# buildx构建多平台镜像
## 安装docker环境
### 环境准备
准备2台机器，分别是amd和arm架构
      
| 主机 | IP | 架构 | docker版本 | buildx版本 | buildkit镜像版本 |    
| :------: | :------: | :------: | :------: | :------: | :------: |    
| build-amd | 192.168.91.47 | amd64 | 25.0.4 | v0.13.0 | latest |
| build-arm | 192.168.91.48 | arm64 | 25.0.4 | v0.13.0 | latest |

### 安装环境
**build-amd**
```shell
# step 1: 安装必要的一些系统工具
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
# step 2: 安装GPG证书
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
# Step 3: 写入软件源信息
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
# Step 4: 更新并安装Docker-CE
sudo apt-get -y update
sudo apt-get -y install docker-ce

# 安装指定版本的Docker-CE:
# Step 1: 查找Docker-CE的版本:
# apt-cache madison docker-ce
#   docker-ce | 17.03.1~ce-0~ubuntu-xenial | https://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
#   docker-ce | 17.03.0~ce-0~ubuntu-xenial | https://mirrors.aliyun.com/docker-ce/linux/ubuntu xenial/stable amd64 Packages
# Step 2: 安装指定版本的Docker-CE: (VERSION例如上面的17.03.1~ce-0~ubuntu-xenial)
# sudo apt-get -y install docker-ce=[VERSION]
```
**版本验证**
```shell
# docker 版本
root@build-amd:~# docker version
Client: Docker Engine - Community
 Version:           25.0.4
 API version:       1.44
 Go version:        go1.21.8
 Git commit:        1a576c5
 Built:             Wed Mar  6 16:32:14 2024
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          25.0.4
  API version:      1.44 (minimum version 1.24)
  Go version:       go1.21.8
  Git commit:       061aa95
  Built:            Wed Mar  6 16:32:14 2024
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.28
  GitCommit:        ae07eda36dd25f8a1b98dfbf587313b99c0190bb
 runc:
  Version:          1.1.12
  GitCommit:        v1.1.12-0-g51d5e94
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

# docker buildx 版本
root@build-amd:~# docker buildx version
github.com/docker/buildx v0.13.0 0de5f1c
```
**build-arm**    
arm架构优先使用二进制安装
```shell
# 下载docker离线安装包
root@build-arm:~# wget https://download.docker.com/linux/static/stable/aarch64/docker-25.0.4.tgz

# 下载buildx离线包
root@build-arm:~# wget https://github.com/docker/buildx/releases/download/v0.13.0/buildx-v0.13.0.linux-arm64

# 解压缩
root@build-arm:~# tar xvf docker-25.0.4.tgz

# 安装前准备
root@build-arm:~# cd docker/

root@build-arm:~/docker# ls
containerd  containerd-shim-runc-v2  ctr  docker  dockerd  docker-init  docker-proxy  runc
# bin目录存放可执行文件，plugin目录存放buildx插件，systemd目录存放systemd管理文件
root@build-arm:~/docker# mkdir {bin,plugin,systemd}

# 移动可执行文件至bin目录
root@build-arm:~/docker# mv containerd containerd-shim-runc-v2 ctr docker dockerd docker-init docker-proxy runc bin/
root@build-arm:~/docker# ls bin/
containerd  containerd-shim-runc-v2  ctr  docker  dockerd  docker-init  docker-proxy  runc

# 移动buildx至plugin目录
root@build-arm:~# mv buildx-v0.13.0.linux-arm64 docker/plugin/docker-buildx
root@build-arm:~# chmod +x docker/plugin/docker-buildx
root@build-arm:~# ls docker/plugin/docker-buildx
docker/plugin/docker-buildx

# systemd
root@build-arm:~# cd docker/systemd/
root@build-arm:~/docker/systemd# ls
containerd.service  docker.service  docker.socket

# 查看目录层级
root@build-arm:~# tree docker
docker
├── bin
│   ├── containerd
│   ├── containerd-shim-runc-v2
│   ├── ctr
│   ├── docker
│   ├── dockerd
│   ├── docker-init
│   ├── docker-proxy
│   └── runc
├── plugin
│   └── docker-buildx
└── systemd
    ├── containerd.service
    ├── docker.service
    └── docker.socket

3 directories, 12 files
```
**执行安装脚本**    
```shell
#!/bin/bash

# 创建用户组
groupadd docker

# 创建buildx插件目录
mkdir -p /usr/local/lib/docker/cli-plugins

# 授权
chmod +x bin/* plugin/*

# 复制可执行文件至PATH路径
cp bin/* /usr/bin/

# 复制插件至插件目录
cp plugin/* /usr/local/lib/docker/cli-plugins

# 复制systemd文件到系统system目录下
cp systemd/* /etc/systemd/system/

# 重新加载生效
systemctl daemon-reload

# 开机自启动
systemctl enable containerd

# 服务启动
systemctl enable docker

# 开机自启动
systemctl start containerd

# 服务启动
systemctl start docker
```
:::note[buildx插件介绍]
**Linux**: 
  - $HOME/.docker/cli-plugins   
  - /usr/local/lib/docker/cli-plugins
  - /usr/local/libexec/docker/cli-plugins
  - /usr/lib/docker/cli-plugins
  - /usr/libexec/docker/cli-plugins
:::
     
**版本验证**   
```shell
# docker 版本
root@build-arm:~# docker version
Client: Docker Engine - Community
 Version:           25.0.4
 API version:       1.44
 Go version:        go1.21.8
 Git commit:        1a576c5
 Built:             Wed Mar  6 16:32:14 2024
 OS/Arch:           linux/arm64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          25.0.4
  API version:      1.44 (minimum version 1.24)
  Go version:       go1.21.8
  Git commit:       061aa95
  Built:            Wed Mar  6 16:32:14 2024
  OS/Arch:          linux/arm64
  Experimental:     false
 containerd:
  Version:          1.6.28
  GitCommit:        ae07eda36dd25f8a1b98dfbf587313b99c0190bb
 runc:
  Version:          1.1.12
  GitCommit:        v1.1.12-0-g51d5e94
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0

# docker buildx 版本
root@build-arm:~# docker buildx version
github.com/docker/buildx v0.13.0 0de5f1c
```
   
## 配置 Remote driver
### 生成证书
*证书生成amd机器上生成*
```shell
root@build-amd:~# SAN="127.0.0.1"
root@build-amd:~# docker buildx bake https://github.com/moby/buildkit.git#master:examples/create-certs

# 创建证书目录
root@build-amd:~# mkdir -p /etc/buildkit/certs
root@build-amd:~# cp -r .certs/* /etc/buildkit/certs
```
### 复制证书到build-arm机器
```shell
# 设置变量值
root@build-amd:~# arm_ip=192.168.91.48
root@build-amd:~# arm_user=root

# 远程登录创建证书目录
root@build-amd:~# ssh $arm_user@$arm_ip mkdir -p /etc/buildkit

# 复制证书
root@build-amd:~# scp -r /etc/buildkit/certs $arm_user@$arm_ip:/etc/buildkit/certs
```
### 配置builkit
```toml
root@build-amd:~# vim /etc/buildkitd.toml

[registry."dockerhub.example.com"]
  mirrors = ["192.168.91.49"]
  http = true
  insecure = true
```
### 运行容器
执行如下命令运行buildkitd容器
> buildx-amd和build-arm机器均需操作
```shell
docker run -d \
  --restart=always \
  --name=remote-buildkitd \
  --privileged \
  -p 1234:1234 \
  -v /etc/buildkit/:/etc/buildkit/ \
  moby/buildkit \
  --addr tcp://0.0.0.0:1234 \
  --tlscacert /etc/buildkit/certs/daemon/ca.pem \
  --tlscert /etc/buildkit/certs/daemon/cert.pem \
  --tlskey /etc/buildkit/certs/daemon/key.pem
```

### 注册Remote driver
> 在build-amd的runner上运行
```shell
root@build-amd:~# arm_ip=192.168.91.48
root@build-amd:~# SAN="127.0.0.1"

root@build-amd:~# docker buildx create --name remote-container --node remote-container0 --driver remote --driver-opt cacert=/etc/buildkit/certs/client/ca.pem,cert=/etc/buildkit/certs/client/cert.pem,key=/etc/buildkit/certs/client/key.pem,servername=$SAN tcp://127.0.0.1:1234 

root@build-amd:~# docker buildx create --append --name remote-container --node remote-container1 --driver remote --driver-opt cacert=/etc/buildkit/certs/client/ca.pem,cert=/etc/buildkit/certs/client/cert.pem,key=/etc/buildkit/certs/client/key.pem,servername=$SAN tcp://$arm_ip:1234 


# 配置docker buildx 使用remote-containerd
root@build-amd:~# docker buildx use remote-container
```
## systemd管理文件
>containerd.service
```shell
[Unit]
Description=containerd container runtime  
Documentation=https://containerd.io  
After=network.target local-fs.target  

[Service]  
ExecStartPre=-/sbin/modprobe overlay  
ExecStart=/usr/bin/containerd  
  
Type=notify  
Delegate=yes   
KillMode=process  
Restart=always  
RestartSec=5  
LimitNPROC=infinity  
LimitCORE=infinity  
LimitNOFILE=infinity  
TasksMax=infinity  
OOMScoreAdjust=-999  

[Install]  
WantedBy=multi-user.target
```
>docker.service
```shell
[Unit]  
Description=Docker Application Container Engine  
Documentation=https://docs.docker.com  
After=network-online.target docker.socket firewalld.service containerd.service time-set.target  
Wants=network-online.target containerd.service  
Requires=docker.socket  
     
[Service]  
Type=notify  
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock  
ExecReload=/bin/kill -s HUP $MAINPID  
TimeoutStartSec=0  
RestartSec=2  
Restart=always  

StartLimitBurst=3  

StartLimitInterval=60s  

LimitNPROC=infinity  
LimitCORE=infinity  

TasksMax=infinity  

Delegate=yes  

KillMode=process  
OOMScoreAdjust=-500  
    

[Install]  
WantedBy=multi-user.target  
```
    
>docker.socket
```shell
[Unit]  
Description=Docker Socket for the API  
   
[Socket]  
ListenStream=/run/docker.sock  
SocketMode=0660  
SocketUser=root  
SocketGroup=docker  
  
[Install]  
WantedBy=sockets.target  
```

## 下载地址
  - [buildx官方文档](https://docs.docker.com/reference/cli/docker/buildx/)
  - [docker离线包](https://download.docker.com)
  - [buildx项目地址](https://github.com/docker/buildx)
  - [buildkit配置文件](https://docs.docker.com/build/buildkit/toml-configuration/#buildkitdtoml)