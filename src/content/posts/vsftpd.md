---
title: Ubuntu20.04安装vsftpd
published: 2025-02-21
tags: [linux]
category: linux
draft: false
---

### 修改主机名
```shell
root@ubuntu:~# hostnamectl set-hostname vsftpd
```
### 更新软件源
```shell
root@vsftpd:~# apt update
```
### 安装vsftpd服务
```shell
# 安装
root@vsftpd:~# apt install -y vsftpd

# 查看服务状态
root@vsftpd:~# systemctl status vsftpd.service 
● vsftpd.service - vsftpd FTP server
     Loaded: loaded (/lib/systemd/system/vsftpd.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2024-05-15 13:26:54 UTC; 29s ago
   Main PID: 3057 (vsftpd)
      Tasks: 1 (limit: 4583)
     Memory: 820.0K
     CGroup: /system.slice/vsftpd.service
             └─3057 /usr/sbin/vsftpd /etc/vsftpd.conf

May 15 13:26:54 vsftpd systemd[1]: Starting vsftpd FTP server...
May 15 13:26:54 vsftpd systemd[1]: Started vsftpd FTP server.
```

### 设置访问用户
>在FTP安装完成后，会默认为我们创建用户名为ftp的用户，默认无密码.
```shell
# 为ftp用户设置密码
root@vsftpd:~# passwd ftp
New password: 
Retype new password: 
passwd: password updated successfully

# 创建ftp用户的家目录，即ftp文件的存储路径
root@vsftpd:~# mkdir /home/ftp

# 设置ftp目录的权限
root@vsftpd:~# chmod 777 /home/ftp
```
### 配置FTP服务
配置文件存放路径：
```shell
/etc/vsftpd.conf
```
修改如下：
```shell
#取消如下配置前的注释符号：
local_enable=YES（是否允许本地用户登录）
write_enable=YES（是否允许本地用户写的权限）
chroot_local_user=YES（是否将所有用户限制在主目录）
chroot_list_enable=YES（是否启动限制用户的名单）
chroot_list_file=/etc/vsftpd.chroot_list（可在文件中设置多个账号）
#修改如下配置
anonymous_enable=NO （不允许匿名访问，必须登录）
chown_uploads=YES （允许上传改变）
#并且添加如下内容
local_root=/home/ftp （访问目录）
allow_writeable_chroot=YES
```
同时在/etc下创建vsftpd.chroot_list文件，这个文件创建完成保持为空即可
```shell
root@vsftpd:~# touch /etc/vsftpd.chroot_list
```

### 重启服务
```shell
root@vsftpd:~# systemctl restart vsftpd.service 
root@vsftpd:~# systemctl status vsftpd.service 
● vsftpd.service - vsftpd FTP server
     Loaded: loaded (/lib/systemd/system/vsftpd.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2024-05-15 13:36:08 UTC; 1s ago
    Process: 3791 ExecStartPre=/bin/mkdir -p /var/run/vsftpd/empty (code=exited, status=0/SUCCESS)
   Main PID: 3801 (vsftpd)
      Tasks: 1 (limit: 4583)
     Memory: 740.0K
     CGroup: /system.slice/vsftpd.service
             └─3801 /usr/sbin/vsftpd /etc/vsftpd.conf

May 15 13:36:08 vsftpd systemd[1]: Starting vsftpd FTP server...
May 15 13:36:08 vsftpd systemd[1]: Started vsftpd FTP server.
```
### 测试
```shell
root@vsftpd:~# ftp localhost
Connected to localhost.
220 (vsFTPd 3.0.5)
Name (localhost:root): ftp
331 Please specify the password.
Password:
530 Login incorrect.
Login failed.
ftp>
```
>如果登录ftp总是出现密码错误，可以将/etc/vsftpd.conf配置文件的pam_service_name=vsftpd改为pam_service_name=ftp，即可解决。
```shell
pam_service_name=ftp
```
修改完成之后再次重启服务，测试如下：
```shell
root@vsftpd:~# ftp localhost
Connected to localhost.
220 (vsFTPd 3.0.5)
Name (localhost:root): ftp
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp>
```

### FTP客户端测试
这里使用filezilla演示:    
新建连接站点：    
![](https://pic.imgdb.cn/item/6644bc2a0ea9cb14032baf29.jpg)
     
弹出提示，点击信任进入下一步：    
     
![](https://pic.imgdb.cn/item/6644bc5f0ea9cb14032bf1d6.jpg)
      
随即登录成功：   
    
![](https://pic.imgdb.cn/item/6644bc990ea9cb14032c3806.jpg)
     
创建文件测试：
     
![](https://pic.imgdb.cn/item/6644bcee0ea9cb14032c9fe9.jpg)
    
通过Linux服务端查看文件是否存在：
```shell
root@vsftpd:~# cd /home/ftp/
root@vsftpd:/home/ftp# ls
windows_client
root@vsftpd:/home/ftp#
```