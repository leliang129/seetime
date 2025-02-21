---
title: tcpdump抓包与分析
published: 2025-02-21
tags: [linux]
category: linux
draft: false
---

## 抓包基础
```bash
# 安装tcpdump
[root@localhost ~]# yum install -y tcpdump

# 抓包内容实时显示到控制台
# 这个命令的意思是在网络接口 eth0 上捕获发送或接收到目标主机 IP 地址为 192.168.91.41 的所有数据包，并以可读的日期和时间的格式显示时间戳。
[root@localhost ~]# tcpdump -i eth0 host 192.168.91.41 -nn -tttt

# 抓包内容实时显示到控制台
[root@localhost ~]# tcpdump -i eth0 host 192.168.91.41 -nn -tttt
[root@localhost ~]# tcpdump -i any host 192.168.91.41 -nn -tttt
[root@localhost ~]# tcpdump -i any host 192.168.91.41 and port 8088 -nn -tttt
# 抓包存到文件
[root@localhost ~]# tcpdump -i eth0 -w test.pcap
# 读取抓包内容
[root@localhost ~]# tcpdump -r test.pcap -nn -tttt
```
:::tip 常用参数
`-r`: 指定包文件。    
`-nn`: 显示数字ip和端口，不转换成名字。   
`-tttt`: 显示时间戳格式: `2006-01-02 15:04:05.999999`。    
:::

## 轮转抓包
```bash
# 每100M轮转一次，最多保留200个文件 (推荐，文件大小可控，可通过文件修改时间判断报文时间范围)
[root@localhost ~]# tcpdump -i eth0 port 8880 -w cvm.pcap -C 100 -W 200

# 每2分钟轮转一次，后缀带上时间
[root@localhost ~]# tcpdump -i eth0 port 31780 -w node-10.70.10.101-%Y-%m%d-%H%M-%S.pcap -G 120
```

## 过滤连接超时的包(reset)
一般如果有连接超时发生，一般 client 会发送 reset 包，可以过滤下:    

```bash
[root@localhost ~]# tcpdump -r test.pcap 'tcp[tcpflags] & (tcp-rst) != 0' -nn -ttt
```

## 统计流量源IP
```bash
[root@localhost ~]# tcpdump -i eth0 dst port 60002 -c 10000|awk '{print $3}'|awk -F. -v OFS="." '{print $1,$2,$3,$4}'|sort |uniq -c|sort -k1 -n
```