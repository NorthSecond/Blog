---
type: Blog
title: 使用 Docker 封印 EasyConnect 进行 SSH 内网链接
date: 2024-02-01 14:00:00 +0800
tags: ['Notes']
summary: 将 Ezconnect 封印在 Docker中
---

在折腾 Tailscale 的时候，初衷就是绕过 Easyconnect 访问内网特定的设备，然而没想到的是回家之后仍然遇上了需要使用 Easyconnect 进行内网连接的非校园网场景，但是我是实在不想在自己个人电脑上装这个王八蛋软件，于是就docker封装，暴露 SOCKv5 代理端口进行连接。

本文很短，仅作备忘。

## Docker 运行

我尝试使用 `cli` 版本的 Docker，但是发现不管是 `7.6.3` 还是 `7.6.7` 版本的 Docker 都总会各种断联，那就还是使用图形界面版吧。

```bash
docker run \
--env=IPTABLES_LEGACY=1  \
--env=PASSWORD=xxxx \
--env=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin \
--env=PING_INTERVAL=1800 \
--name=easyconnect \
--volume=C:\Codes\xxxx\.ecdata:/root \
--volume=/root/ \
--volume=/usr/share/sangfor/EasyConnect/resources/logs/ \
--cap-add=NET_ADMIN \
-p 127.0.0.1:1080:1080 \
-p 127.0.0.1:5901:5901 \
-p 127.0.0.1:8888:8888 \
--restart=no \
--rm \
--device /dev/net/tun:/dev/net/tun \ 
--runtime=runc \
-t \
-d hagb/docker-easyconnect:7.6.7 
```

运行之后使用 VNC 远程桌面客户端（我这里使用 RemoteRipple ）访问 `127.0.0.1:5901`，密码是 `xxxx` ，进行登录就好了。

## SSH 代理配置

在登录成功之后，会配置两个端口的代理，分别是在 `:8888` 端口的 HTTPS 代理和在 `:1080` 端口的 SOCKv5 代理。一般我们使用 SOCKv5 代理。

使用 Termius 或者 WindTerm 相关第三方命令行软件的可以在新建 Host 的时候直接设置代理为 SOCKv5，对应代理链接为 `127.0.0.1:1080`，使用 Windows 原生命令行或者 VSCode 这种借助原生 OpenSSH 的就要麻烦一点，有 `ncat` 和 `git` 两种方法。

### Git for Windows 方法

Git for Windows 中会有一个支持 SSH over HTTP 的 `connect.exe`。我们可以借助它进行代理的连接，在 `config` 文件中设置如下：

```bash
Host xxx
    HostName xxx
    Port xxx
    User xxx
    IdentityFile xxx
    ProxyCommand <path-to-git>\Git\mingw64\bin\connect.exe -S 127.0.0.1:1080 %h %p
```

这样就可以链接了。

### ncat 方法

由于 wireshark 自带了 `nmap` 中包含了 `ncat` ，也可以通过 ncat 来进行 SSH over HTTP 的代理设置。

```bash
Host xxx
    HostName xxx
    Port xxx
    User xxx
    IdentityFile xxx
    ProxyCommand ncat --verbose --proxy-type http --proxy 127.0.0.1:8888 %h %p
```

这里假设 `ncat` 被加入了 `PATH` 环境变量中，如果没有，那就需要将 `ncat` 改为 `<path-to-nmap>/ncat.exe`。

## 一些坑和注意点

1.   由于 Docker 中的 TUN 网卡冲突，如果使用 Clash 的 TUN Mode 会导致链接成功之后不能正常加载内容
2.   `cli` 版本目前还是奇奇怪怪地不能用