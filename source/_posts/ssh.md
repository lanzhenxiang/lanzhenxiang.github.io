---
title: 解决SSH登录提示输入密码响应慢问题
date: 2019-08-29 17:50:16
tags:
  - SSH
  - DNS
  - Linux
---

最近在环境上遇到一个问题，就是 ssh 登录服务器时比较缓慢，常表现为需要等待 10 几秒才能提示输入密码的现象。其实这个是 debian 做的一个配置上的修改引起的。提供两个解决办法

# 方法一、取消 DNS 反向解析

使用的 Linux 用户可能觉得用 SSH 登陆时为什么反映这么慢，有的可能要几十秒才能登陆进系统。其实这是由于默认 sshd 服务开启了 DNS 反向解析，如果你的 sshd 没有使用域名等来作为限定时，可以取消此功能。

```
vi /etc/ssh/sshd_config
将 # UseDNS yes
改为 UseDNS no
```

没有的话自行添加

# 方法二

这个问题正是最后面那项 GSSAPIAuthentication 引起的,打开这个 ssh 的时候可能会先去尝试其他的认证方式.很多地方都会介绍说修改 /etc/ssh/ssh_config 文件,但是其实这并不是最好的办法,因为在下次升级的时候,也许会因为配置文件被修改过,而引起不必要的麻烦.我的解决办法是修改个人用户的配置文件,如 下:

```
echo “GSSAPIAuthentication no” >> ~/.ssh/config
```

# 修改超时时间

在 Asinanux 3.0 带 4.3sp2 版本 OpenSSH，默认超时连接时间比较短，这是出于安全的考虑，但对于需要长时间使用的用户来说很麻烦，每次都要重新连接。我们可以修改其设定参数：

```
# vi /etc/ssh/sshd_config
```

- 找到选项 #ClientAliveInterval 0
- 修改为 ClientAliveInterval 10

重启 sshd 服务

```
service sshd restart
```

这样，超过 10 秒没有动作的情况下，sshd 服务才会中断连接
