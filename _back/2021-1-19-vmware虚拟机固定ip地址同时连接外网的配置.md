---
layout: "post"
category: 环境配置
author: LJR
title: vmware虚拟机固定ip地址同时连接外网的配置
tags:
    - vmware
mathjax: true
---

为了避免vmware虚拟机每次启动后ip地址都会变化的情况，考虑固定其ip地址。这样的话，用vscode的remote-SSH插件也可以使用静态配置的config文件了。

## 固定虚拟机ip地址的配置

TODO

### NAT原理

TODO

## 手动配置DNS服务器地址

ip固定下来后，此时应该能够ping通宿主机。但是由于dns服务被自动取消需要自己手动配置dns服务器。

修改`/etc/systemd/resolved.conf`文件，更改`DNS`服务器的ip地址如下：

```shell
[Resolve]
DNS=8.8.8.8
#...
#...
...
```

然后重启resolved服务即可。

```shell
service systemd-resolved restart
```
