---
title: 'SSH配置与使用'
date: 2020-07-18 10:28:00
updated: 2020-07-18 10:28:00
tags: Linux
---


### 免密登录

创建本地公私钥文件：

```shell
ssh-keygen -t rsa -b 4096 -C "youremail@example.com”
```

生成后的文件在`~/.ssh`目录内，`id_rsa`为私钥，`id_rsa.pub`为公钥。将公钥复制到远程机器对应用户的`~/.ssh/authorized_keys`内。


<!--more-->


### 远程服务器配置

为保证远程服务器安全，创建非root用户：

```shell
adduser custom_user
```

编辑服务器ssh服务配置文件`/etc/ssh/sshd_config`：

```ini
# 修改登陆端口号
Port 2210
# 禁止 root 登陆
PermitRootLogin no
# 关闭远程密码登陆
PasswordAuthentication no
UsePAM no
```

重启ssh服务：

```shell
sudo service ssh restart
```



### 本地SSH配置

ssh的配置文件有两个，分别对应全局`/etc/ssh/ssh_config`，当前用户`~/.ssh/config`，以下配置在这两个中都有效。

##### 保持连接

SSH连接长时间不使用，会断连。可以通过配置间隔向server发送keep-alive包，保持连接

```ini
# 每30秒发送一个keep-alive包
ServerAliveInterval 30
# 发送10次都无响应，断开连接
ServerAliveCountMax 10
```

##### 使用多个key

可以针对不同server设置不同的key，也可以配置多个key让ssh分别尝试

```ini
# 指定单个server的key文件
Host example.com
    IdentityFile ~/.ssh/id_rsa_other
    
# 通用配置多个key
Host *
    IdentityFile ~/.ssh/id_rsa_1
    IdentityFile ~/.ssh/id_rsa_2
```

##### 共享连接

同时打开多个同一个server的连接，连接间可以复用，不用再进行连接建立，加快连接速度。

```ini
Host *
    ControlMaster auto
    ControlPath /tmp/ssh-connection-%h-%p-%r
```

在会话结束后，可以让master连接继续在后台保持一段时间，加快下次连接速度。这个在git拉取推送代码时非常有用，可以加快每次的速度。

```ini
Host *
    ControlPersist 4h
```



### 端口转发

![ssh-forward.drawio](/images/ssh_forward.png)

##### 本地端口转发

将发送到本地端口的请求，通过中间服务器，转发到目标主机端口。

例如情况①，server3可以连接到server1，不能访问server2，两台机器server1、server2间可以通信；这时在server3上设置转发，通过中间服务器server1，将本地的端口转发到server2。

```shell
ssh -L [绑定地址:]本地端口:目标地址:目标端口 中间服务器信息
ssh -g -L [localhost:]8080:server2:80 root@server1
```

`-g`参数允许远程连接使用此端口转发，如果不设置，只能server3通过localhost:port来进行访问。

##### 远程端口转发

将发送到远程端口的请求，转发到目标端口。

例如情况②，server1可以连接server2与server3，server3不能访问server2；这时在server1上设置转发，使server3能够通过server1的端口，访问server2的端口。

```shell
ssh -R [绑定地址:]绑定端口:目标地址:目标端口 用户@主机地址
ssh -R [localhost:]8080:server2:80 root@server3
```

##### 动态端口转发

将发送到本地端口的请求，转发指定地址，目标地址和端口由发起的请求决定。

在开启转发后，ssh将在本地建立socket代理，将客户端的代理设置成本地代理即可使用。

例如情况①，若server3需要访问server2上的多个端口服务；这时在server3上设置动态转发，然后将需要转发客户端的代理设置成`127.0.0.1:8080`即可。

```shell
ssh -D [绑定地址:]绑定端口 用户@主机地址
ssh -D [localhost:]8080 root@server1
```
