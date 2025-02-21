---
title: gitlab集成钉钉扫码登录
published: 2025-02-21
tags: [gitlab]
category: linux
draft: false
---
# gitlab集成钉钉扫码登录

## 安装gitlab-ce
```shell
# 01 更新软件源
root@ecs-e58f:~# apt update

# 02 下载gitlab-ce安装包
root@ecs-e58f:~# wget https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/ubuntu/pool/focal/main/g/gitlab-ce/gitlab-ce_14.6.3-ce.0_amd64.deb

# 03 安装gitlab
root@ecs-e58f:~# dpkg -i gitlab-ce_14.6.3-ce.0_amd64.deb 

# 04 编译配置文件
root@ecs-e58f:~# vim /etc/gitlab/gitlab.rb

# 如下修改 external_url 为本机地址或者域名
root@ecs-e58f:~# grep external_url /etc/gitlab/gitlab.rb
##! For more details on configuring external_url see:
external_url 'http://1.94.109.197'

# 05 重新加载配置文件
root@ecs-e58f:~# gitlab-ctl reconfigure

# 06 启动gitlab
root@ecs-e58f:~# gitlab-ctl start
ok: run: alertmanager: (pid 164302) 523s
ok: run: gitaly: (pid 164290) 523s
ok: run: gitlab-exporter: (pid 164263) 525s
ok: run: gitlab-workhorse: (pid 164243) 526s
ok: run: grafana: (pid 164395) 522s
ok: run: logrotate: (pid 163289) 663s
ok: run: nginx: (pid 163794) 599s
ok: run: node-exporter: (pid 164257) 526s
ok: run: postgres-exporter: (pid 164310) 523s
ok: run: postgresql: (pid 163521) 644s
ok: run: prometheus: (pid 164272) 524s
ok: run: puma: (pid 163711) 613s
ok: run: redis: (pid 163328) 657s
ok: run: redis-exporter: (pid 164265) 525s
ok: run: sidekiq: (pid 163734) 607s
```

## 访问gitlab
### 浏览器访问
```shell
# 浏览器访问
http://1.94.107.197
```
![](https://pic.imgdb.cn/item/65f831449f345e8d031f65a5.jpg)

### 初始密码登录
```shell
root@ecs-e58f:~# cat /etc/gitlab/initial_root_password 
# WARNING: This value is valid only in the following conditions
#          1. If provided manually (either via `GITLAB_ROOT_PASSWORD` environment variable or via `gitlab_rails['initial_root_password']` setting in `gitlab.rb`, it was provided before database was seeded for the first time (usually, the first reconfigure run).
#          2. Password hasn't been changed manually, either via UI or via command line.
#
#          If the password shown here doesn't work, you must reset the admin password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.

Password: LYwmW7opZs83gZpfv61ub6xnpV/Rxzekd+nKTUONPCM=

# NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.
```

### 成功登录界面
![](https://pic.imgdb.cn/item/65f830f99f345e8d031d86da.jpg)

## 配置钉钉登录
### 创建钉钉应用
```shell
# 参考文档
https://open.dingtalk.com/document/isvapp/create-an-application
```
### 获取密钥信息
![](https://pic.imgdb.cn/item/65f836229f345e8d033d4088.jpg)

### 配置回调域名
```shell
http://<Gitlab_url>/users/auth/dingtalk/callback
```
![](https://pic.imgdb.cn/item/65f836a09f345e8d03405d40.jpg)

### 编辑配置文件
```shell
root@ecs-e58f:~# vim /etc/gitlab/gitlab.rb

# 添加如下配置
gitlab_rails['omniauth_auto_link_ldap_user'] = true
gitlab_rails['omniauth_allow_single_sign_on'] = ['dingtalk']
gitlab_rails['omniauth_sync_email_from_provider'] = 'dingtalk'
gitlab_rails['omniauth_sync_profile_from_provider'] = ['dingtalk']
gitlab_rails['omniauth_sync_profile_attributes'] = ['name','email','location']
gitlab_rails['omniauth_providers'] = [
    {
      name: "dingtalk",
      # 登录按钮展示名称
      label: "钉钉",
      app_id: "dingxkrixr7yny3qauzn",
      app_secret: "l3c6rKxpeV3ZYT9AUEsw8D2Y9xh_bQBrZ4ICUfVLMGL4xSS-LP6xM8ebGJZg23sl"
    }
  ]

# 重新加载配置文件
root@ecs-e58f:~# gitlab-ctl reconfigure
```
### 访问gitlab
![](https://pic.imgdb.cn/item/65f837449f345e8d03447523.jpg)
     
如上所示，登录选项多了钉钉选项。
    
点击跳转至扫码登录界面
![](https://pic.imgdb.cn/item/65f838039f345e8d03490ad0.jpg)
     
出现如下信息即可：
![](https://pic.imgdb.cn/item/65f8383f9f345e8d034a7a6e.jpg)

### 审批
      
登录管理员账号审批同意钉钉登录用户
![](https://pic.imgdb.cn/item/65f839499f345e8d035144f0.jpg)

### 钉钉用户登录
     
审批完成后，返回扫码登录     
登录成功，设置个人信息后可正常使用。
![](https://pic.imgdb.cn/item/65f839ab9f345e8d0353f331.jpg)

:::tip[⚠注意]
文中所涉及云服务器以及钉钉应用均已删除。
:::