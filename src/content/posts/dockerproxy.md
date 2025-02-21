---
title: Docker代理配置
published: 2025-02-21
tags: [docker]
category: docker
draft: false
---

## 配置docker代理
```shell
mkdir -p /etc/systemd/system/docker.service.d

vim /etc/systemd/system/docker.service.d/http-proxy.conf

[Service]
Environment="HTTP_PROXY=http://192.168.91.222:7890"
Environment="HTTPS_PROXY=http://192.168.91.222:7890"
Environment="NO_PROXY=localhost,127.0.0.0/8,svc,.cluster.local,192.168.0.0/16,10.96.0.0/16"

```
## containerd代理配置
> 在容器运行时需要代理上网，则配置
```shell
cat /etc/systemd/system/containerd.service.d/http-proxy.conf

[Service]
Environment="HTTP_PROXY=http://192.168.91.222:7890"
Environment="HTTPS_PROXY=http://192.168.91.222:7890"
Environment="NO_PROXY=localhost,127.0.0.0/8,svc,.cluster.local,192.168.0.0/16,10.96.0.0/16"
```
:::note
仅支持Docker 17.07及以上版本
:::

## Docker build代理配置
```shell
docker build . \
    --build-arg "HTTP_PROXY=http://192.168.91.1:7890/" \
    --build-arg "HTTPS_PROXY=http://192.168.91.1:7890/" \
    --build-arg "NO_PROXY=localhost,127.0.0.1,.example.com" \
    -t <image_name>:<image_tag>
```

:::info
Docker build构建镜像构成中需将变量http_proxy、https_proxy、no_proxy传递给构建环境
:::