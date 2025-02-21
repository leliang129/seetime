---
title: kubernetes安装NFS CSI Driver
published: 2025-02-21
tags: [k8s]
category: kubernetes
draft: false
---
## 安装NFS-SERVER
     
在kubernetes中集群安装部署一个nfs-server prod
      
> 参考: [Set up a NFS Server on a Kubernetes cluster](https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/deploy/example/nfs-provisioner/README.md)

```shell
# 下载安装文件
[root@master nfs]# wget https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/deploy/example/nfs-provisioner/nfs-server.yaml

# 安装资源对象
[root@master nfs]# kubectl apply -f nfs-server.yaml

# 查看POD状态
[root@master nfs]# kubectl get pods 
NAME                          READY   STATUS    RESTARTS   AGE
nfs-server-5847b99d99-l222n   1/1     Running   0          13m
```
     
## 安装NFS-CSI
> 参考: [Install NFS CSI driver on a kubernetes cluster](https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/docs/install-csi-driver-v4.2.0.md)

### Install with kubectl
```shell
[root@master nfs]# curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.2.0/deploy/install-driver.sh | bash -s v4.2.0 --
Installing NFS CSI driver, version: v4.2.0 ...
serviceaccount/csi-nfs-controller-sa configured
serviceaccount/csi-nfs-node-sa configured
clusterrole.rbac.authorization.k8s.io/nfs-external-provisioner-role configured
clusterrolebinding.rbac.authorization.k8s.io/nfs-csi-provisioner-binding configured
csidriver.storage.k8s.io/nfs.csi.k8s.io configured
deployment.apps/csi-nfs-controller created
daemonset.apps/csi-nfs-node created
NFS CSI driver installed successfully.
```

### clean up NFS CSI driver
```shell
[root@master nfs]# curl -skSL https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/v4.2.0/deploy/uninstall-driver.sh | bash -s v4.2.0 --
```
     
查看pod状态
    
```shell
[root@master nfs]# kubectl -n kube-system get pods 
NAME                                 READY   STATUS              RESTARTS       AGE
coredns-7b5dd5fb4-7dt6v              1/1     Running             1 (145m ago)   47h
coredns-7b5dd5fb4-br2vk              1/1     Running             1 (145m ago)   47h
csi-nfs-controller-d4ff49ff5-vj8rm   0/3     ContainerCreating   0              110s
csi-nfs-node-7p9t7                   0/3     ErrImagePull        0              107s
csi-nfs-node-js92p                   0/3     ContainerCreating   0              107s
csi-nfs-node-jwqgn                   0/3     ErrImagePull        0              107s
csi-nfs-node-ppbdj                   0/3     ErrImagePull        0              107s
csi-nfs-node-qzzsn                   0/3     ErrImagePull        0              107s
csi-nfs-node-v8xx9                   0/3     ErrImagePull        0              107s
```
     
修改镜像为国内镜像( registry.k8s.io --> uhub.service.ucloud.cn )
:::tip
可使用[ucloud](https://console.ucloud.cn/dashboard)的镜像加速功能
:::
    
```shell
[root@master nfs]# kubectl -n kube-system edit deployments.apps csi-nfs-controller 
deployment.apps/csi-nfs-controller edited

[root@master nfs]# kubectl -n kube-system edit daemonsets.apps csi-nfs-node 
daemonset.apps/csi-nfs-node edited
```

```shell
# 再次查看pod状态
[root@master nfs]# kubectl -n kube-system get pods -w
NAME                                  READY   STATUS              RESTARTS       AGE
coredns-7b5dd5fb4-7dt6v               1/1     Running             1 (154m ago)   47h
coredns-7b5dd5fb4-br2vk               1/1     Running             1 (154m ago)   47h
csi-nfs-controller-7c78cc596b-9sm7s   3/3     Running             0              5m53s
csi-nfs-node-5vgql                    0/3     ContainerCreating   0              2s
csi-nfs-node-jx79k                    0/3     ContainerCreating   0              1s
csi-nfs-node-n4cl9                    0/3     ContainerCreating   0              1s
csi-nfs-node-sb7fw                    0/3     ContainerCreating   0              2s
csi-nfs-node-w5gpx                    0/3     ContainerCreating   0              1s
csi-nfs-node-zg58g                    0/3     ContainerCreating   0              2s
```

## 创建storage class
> 参考：[storageclass-nfs.yaml](https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/deploy/example/storageclass-nfs.yaml)
```yaml 
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: nfs-server.default.svc.cluster.local
  share: /
  # csi.storage.k8s.io/provisioner-secret is only needed for providing mountOptions in DeleteVolume
  # csi.storage.k8s.io/provisioner-secret-name: "mount-options"
  # csi.storage.k8s.io/provisioner-secret-namespace: "default"
reclaimPolicy: Delete
volumeBindingMode: Immediate
mountOptions:
  - nfsvers=4.1
```
    
创建资源对象：   
```shell
[root@master nfs]# kubectl apply -f csi-nfs-sc.yaml 
storageclass.storage.k8s.io/nfs-csi created

# 查看资源对象
[root@master nfs]# kubectl get sc
NAME      PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-csi   nfs.csi.k8s.io   Delete          Immediate           false                  46s
```

:::caution
此次实验最终并未自动创建pvc，原因待查。
:::