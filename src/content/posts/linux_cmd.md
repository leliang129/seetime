---
title: Linux常用排查命令
published: 2025-02-21
tags: [linux]
category: linux
draft: false
---
## 查看 socket buffer
        
查看是否阻塞
```shell
root@ubuntu:~# netstat -antup | awk '{if($2>100||$3>100){print $0}}'
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
```
     
  - `Recv-Q` 是接收队列，如果持续有堆积，可能是高负载，应用处理不过来，也可能是程序的 bug，卡住了，导致没有从 buffer 中取数据，可以看看对应 pid 的 stack 卡在哪里了(`cat /proc/$PID/stack`)。
      
查看是否有 UDP buffer 满导致丢包
```shell
# 使用 netstat 查看统计
root@ubuntu:~# netstat -s | grep "buffer errors"
    0 receive buffer errors
    0 send buffer errors

# 也可以用 nstat 查看计数器
root@ubuntu:~# nstat -az | grep -E 'UdpRcvbufErrors|UdpSndbufErrors'
UdpRcvbufErrors                 0                  0.0
UdpSndbufErrors                 0                  0.0
```
对于 TCP，发送 buffer 慢不会导致丢包，只是会让程序发送数据包时卡住，等待缓冲区有足够空间释放出来，而接收 buffer 满了会导致丢包，可以通过计数器查看:
```shell
root@ubuntu:~# nstat -az | grep TcpExtTCPRcvQDrop
TcpExtTCPRcvQDrop               0                  0.0
```
     
查看当前 UDP buffer 的情况
```shell
root@ubuntu:~# ss -nump
Recv-Q    Send-Q          Local Address:Port         Peer Address:Port    Process
0         0             10.10.4.26%eth0:68              10.10.4.1:67       users:(("NetworkManager",pid=960,fd=22))
     skmem:(r0,rb212992,t0,tb212992,f0,w0,o640,bl0,d0)
```
  - rb212992 表示 UDP 接收缓冲区大小是 212992 字节，tb212992 表示 UDP 发送缓存区大小是 212992 字节。    
  - Recv-Q 和 Send-Q 分别表示当前接收和发送缓冲区中的数据包字节数。
     
查看当前 TCP buffer 的情况
```shell
root@ubuntu:~# ss -ntmp
State         Recv-Q         Send-Q                 Local Address:Port                   Peer Address:Port         Process                                                                                        
ESTAB         0              36                     192.168.91.47:22                   192.168.91.199:7916          users:(("sshd",pid=1327,fd=4))
	 skmem:(r0,rb131072,t0,tb87040,f2780,w1316,o0,bl0,d0)         
ESTAB         0              0                      192.168.91.47:22                   192.168.91.199:7917          users:(("sshd",pid=1468,fd=4))
	 skmem:(r0,rb131072,t0,tb87040,f4096,w0,o0,bl0,d0)
```
  - rb131072 表示 TCP 接收缓冲区大小是 131072 字节，tb87040 表示 TCP 发送缓存区大小是 87040 字节。     
  - Recv-Q 和 Send-Q 分别表示当前接收和发送缓冲区中的数据包字节数。
      
## 查看监听队列
```shell
root@ubuntu:~# ss -lnt
State                    Recv-Q                   Send-Q                                     Local Address:Port                                     Peer Address:Port                   Process                   
LISTEN                   0                        4096                                       127.0.0.53%lo:53                                            0.0.0.0:*                                                
LISTEN                   0                        128                                              0.0.0.0:22                                            0.0.0.0:*                                                
LISTEN                   0                        128                                            127.0.0.1:6010                                          0.0.0.0:*                                                
LISTEN                   0                        128                                                 [::]:22                                               [::]:*                                                
LISTEN                   0                        128                                                [::1]:6010                                             [::]:*   
```
     
`Recv-Q` 表示 accept queue 中的连接数，如果满了(Recv-Q的值比Send-Q大1)，要么是并发太大，或负载太高，程序处理不过来；要么是程序 bug，卡住了，导致没有从 accept queue 中取连接，可以看看对应 pid 的 stack 卡在哪里了(`cat /proc/$PID/stack`)。
    
##  查看网络计数器
```shell
root@ubuntu:~# nstat -az
#kernel
IpInReceives                    25253              0.0
IpInHdrErrors                   0                  0.0
IpInAddrErrors                  0                  0.0
IpForwDatagrams                 0                  0.0
IpInUnknownProtos               0                  0.0
IpInDiscards                    0                  0.0
IpInDelivers                    25253              0.0
IpOutRequests                   11348              0.0
IpOutDiscards                   20                 0.0
IpOutNoRoutes                   0                  0.0
IpReasmTimeout                  0                  0.0
......

root@ubuntu:~# netstat -s | grep -E 'drop|overflow'
    20 outgoing packets dropped
```
> 如果有 overflow，意味着 accept queue 有满过，可以查看监听队列看是否有现场。
    
## 查看 conntrack
```shell
# install
root@ubuntu:~# apt install -y conntrack

root@ubuntu:~# conntrack -S
cpu=0   	found=0 invalid=0 ignore=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=0 
cpu=1   	found=0 invalid=0 ignore=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=0 
cpu=2   	found=0 invalid=0 ignore=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=0 
cpu=3   	found=0 invalid=0 ignore=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=0 
cpu=4   	found=0 invalid=0 ignore=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=0 
cpu=5   	found=0 invalid=0 ignore=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=0 
cpu=6   	found=0 invalid=0 ignore=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=0 
cpu=7   	found=0 invalid=0 ignore=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=0 
cpu=8   	found=0 invalid=0 ignore=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=0 
cpu=9   	found=0 invalid=0 ignore=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=0 
cpu=10  	found=0 invalid=0 ignore=0 insert=0 insert_failed=0 drop=0 early_drop=0 error=0 search_restart=0
......
```
  - 若有 `insert_failed`，表示存在 conntrack 插入失败，会导致丢包。

## 查看连接数
     
如果有 `ss` 命令，可以使用 `ss -s` 统计
```shell
root@ubuntu:~# ss -s
Total: 156
TCP:   7 (estab 2, closed 0, orphaned 0, timewait 0)

Transport Total     IP        IPv6
RAW	  1         0         1        
UDP	  1         1         0        
TCP	  7         5         2        
INET	  9         6         3        
FRAG	  0         0         0 
```

如果没有 `ss`，也可以尝试用脚本统计当前各种状态的 TCP 连接数:
```shell
root@ubuntu:~# netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
ESTABLISHED 2
```
或者直接手动统计 `/proc`
```shell
root@ubuntu:~# cat /proc/net/tcp* | wc -l
9
```
## 测试网络连通性
不断 telnet 查看网络是否能通
```shell
root@ubuntu:~# while true; do echo "" | telnet 192.168.91.47 443; sleep 0.1; done
Trying 192.168.91.47...
telnet: Unable to connect to remote host: Connection refused
Trying 192.168.91.47...
telnet: Unable to connect to remote host: Connection refused
Trying 192.168.91.47...
telnet: Unable to connect to remote host: Connection refused
Trying 192.168.91.47...
telnet: Unable to connect to remote host: Connection refused
Trying 192.168.91.47...
telnet: Unable to connect to remote host: Connection refused
Trying 192.168.91.47...
telnet: Unable to connect to remote host: Connection refused
Trying 192.168.91.47...
telnet: Unable to connect to remote host: Connection refused
Trying 192.168.91.47...
telnet: Unable to connect to remote host: Connection refused
Trying 192.168.91.47...
telnet: Unable to connect to remote host: Connection refused
```
  - `ctrl+c` 终止测试   
  - 替换 `192.168.91.47` 与 `443` 为需要测试的 `IP/域名 和端口`
    
没有安装 telnet，也可以使用 nc 测试
```shell
root@ubuntu:~# nc -vz 192.168.91.47 443
nc: connect to 192.168.91.47 port 443 (tcp) failed: Connection refused
```
## 排查流量激增
### iftop纠出大流量IP
```shell
# install
root@ubuntu:~# apt install -y iftop

root@ubuntu:~# iftop
ubuntu  => 192.168.91.199  736b    838b    955b
        <=                 208b    208b    243b
```
### netstat查看大流量IP连接
```shell
root@ubuntu:~# netstat -np | grep 192.168.91.47
tcp        0     36 192.168.91.47:22        192.168.91.199:7916     ESTABLISHED 1327/sshd: root@pts 
tcp        0      0 192.168.91.47:22        192.168.91.199:7917     ESTABLISHED 1468/sshd: root@not
```

## 排查资源占用
### 文件被占用
看某个文件在被哪些进程读写
```shell
root@ubuntu:~# lsof <filename>
```
看某个进程打开了哪些文件
```shell
root@ubuntu:~# lsof -p <pid>
```
   
### 端口占用
查看 22 端口被谁占用
```shell
root@ubuntu:~# lsof -i :22
COMMAND  PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
sshd    1084 root    3u  IPv4  37331      0t0  TCP *:ssh (LISTEN)
sshd    1084 root    4u  IPv6  37333      0t0  TCP *:ssh (LISTEN)
sshd    1327 root    4u  IPv4  40063      0t0  TCP ubuntu:ssh->192.168.91.199:7916 (ESTABLISHED)
sshd    1468 root    4u  IPv4  40259      0t0  TCP ubuntu:ssh->192.168.91.199:7917 (ESTABLISHED)
```
或者
```shell
root@ubuntu:~# netstat -tunlp | grep 22
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1084/sshd: /usr/sbi 
tcp6       0      0 :::22                   :::*                    LISTEN      1084/sshd: /usr/sbi 
```
## 查看进程树
```shell
root@ubuntu:~# pstree
systemd─┬─VGAuthService
        ├─accounts-daemon───2*[{accounts-daemon}]
        ├─agetty
        ├─atd
        ├─cron
        ├─dbus-daemon
        ├─irqbalance───{irqbalance}
        ├─multipathd───6*[{multipathd}]
        ├─networkd-dispat
        ├─polkitd───2*[{polkitd}]
        ├─rsyslogd───3*[{rsyslogd}]
        ├─snapd───12*[{snapd}]
        ├─sshd─┬─sshd───bash─┬─iftop───3*[{iftop}]
        │      │             └─pstree
        │      └─sshd───sftp-server
        ├─systemd───(sd-pam)
        ├─systemd-journal
        ├─systemd-logind
        ├─systemd-network
        ├─systemd-resolve
        ├─systemd-timesyn───{systemd-timesyn}
        ├─systemd-udevd
        ├─udisksd───4*[{udisksd}]
        ├─unattended-upgr───{unattended-upgr}
        └─vmtoolsd───2*[{vmtoolsd}]
```
## 测试对比 CPU 性能
看计算圆周率耗时，耗时越短说明 CPU 性能越强
```shell
root@ubuntu:~# time echo "scale=5000; 4*a(1)"| bc -l -q
3.141592653589793238462643383279502884197169399375105820974944592307\
81640628620899862803482534211706798214808651328230664709384460955058\
22317253594081284811174502841027019385211055596446229489549303819644\
28810975665933446128475648233786783165271201909145648566923460348610\
45432664821339360726024914127372458700660631558817488152092096282925\
......

real	0m11.704s
user	0m11.689s
sys	0m0.004s
```

## 查看证书内容
查看 secret 里的证书内容
```shell
root@ubuntu:~# kubectl get secret test-crt-secret  -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
```
查看证书文件内容
```shell
root@ubuntu:~# openssl x509 -noout -text -in test.crt
```
查看远程地址的证书内容
```shell
root@ubuntu:~# echo | openssl s_client -connect 192.168.91.47:443 2>/dev/null | openssl x509 -noout -text
```

## 磁盘占用
### 空间占用
```shell
root@ubuntu:~# df -h
Filesystem                         Size  Used Avail Use% Mounted on
udev                               1.9G     0  1.9G   0% /dev
tmpfs                              391M  1.3M  390M   1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv   59G  6.3G   50G  12% /
tmpfs                              2.0G     0  2.0G   0% /dev/shm
tmpfs                              5.0M     0  5.0M   0% /run/lock
tmpfs                              2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/loop0                          33M   33M     0 100% /snap/snapd/12704
/dev/loop1                          71M   71M     0 100% /snap/lxd/21029
/dev/loop2                          56M   56M     0 100% /snap/core18/2128
/dev/sda2                          976M  107M  803M  12% /boot
/dev/loop3                          56M   56M     0 100% /snap/core18/2812
tmpfs                              391M     0  391M   0% /run/user/0
```
### inode占用
```shell
root@ubuntu:~# df -i
Filesystem                         Inodes IUsed   IFree IUse% Mounted on
udev                               488931   476  488455    1% /dev
tmpfs                              500220   771  499449    1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv 3899392 80416 3818976    3% /
tmpfs                              500220     1  500219    1% /dev/shm
tmpfs                              500220     3  500217    1% /run/lock
tmpfs                              500220    18  500202    1% /sys/fs/cgroup
/dev/loop0                            474   474       0  100% /snap/snapd/12704
/dev/loop1                           1602  1602       0  100% /snap/lxd/21029
/dev/loop2                          10803 10803       0  100% /snap/core18/2128
/dev/sda2                           65536   312   65224    1% /boot
/dev/loop3                          10944 10944       0  100% /snap/core18/2812
tmpfs                              500220    22  500198    1% /run/user/0
```
或者
```shell
root@ubuntu:~# tune2fs -l /dev/sda2 | grep -i inode
Filesystem features:      has_journal ext_attr resize_inode dir_index filetype needs_recovery extent 64bit flex_bg sparse_super large_file huge_file dir_nlink extra_isize metadata_csum
Inode count:              65536
Free inodes:              65224
Inodes per group:         8192
Inode blocks per group:   512
First inode:              11
Inode size:	          256
Journal inode:            8
Journal backup:           inode blocks
```