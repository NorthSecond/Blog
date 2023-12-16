---
type: Blog
title: 在服务器上部署 Overleaf
date: 2023-11-13 19:00:00 +0800
tags: ['Notes']
summary: 虽然我也不知道阿里云和Overleaf哪个更稳定一点。
---

## 动机

大清早起来发现自己手上有个文档在 Overleaf 编译突然显示超时了，一查才发现 Overleaf 给免费账号的最大编译时限缩减到 20s 了（参考[官方 Blog](https://www.overleaf.com/blog/changes-to-free-compile-timeouts-and-servers)）。想着手上刚刚续了阿里云的学生 ECS，就想着本地部署一个吧，这样也方便一点。

## 配置过程

### 环境要求

需要 Docker 和 VIM。

### 初始化环境

Overleaf 的[官方Repo](https://github.com/overleaf/toolkit) 给了相对详尽的过程。

```bash
git clone https://github.com/overleaf/toolkit.git ./overleaf
cd overleaf
bin/init
```
此时配置文件就在 `./config/` 文件夹下了。于是就可以

```bash
vim config/overleaf.rc
```

主要更改的就是监听的IP源还是端口

```
SHARELATEX_LISTEN_IP=0.0.0.0
SHARELATEX_PORT=11880
```
默认配置是只监听本地的80端口，我们是在服务器上配置的，因此就不太对头，改成接收所有 IP 段的请求就可以了。如果本地有 TLS 的配置，也是在这个链接里面做的。

之后就可以运行

```bash
./bin/up
```
进行初始化和第一次运行了。在 log 稳定之后使用 `Ctrl+C` 退出这一次运行，以后就可以用

```bash
./bin/start
```
后台运行了。

### tex-live 安装

自带的 Tex-live 环境是不全的，不支持 XeLaTeX 不说，对于 pdfLaTeX 的支持也是各种奇奇怪怪，编译会有各种各样的问题。于是我们需要安装一下完整版的 tex-live。

```bash
docker exec -it sharelatex /bin/bash
tlmgr option repository https://mirrors.ustc.edu.cn/CTAN/systems/texlive/tlnet/
tlmgr update --self
tlmgr install scheme-full
```
### 字体安装

```bash
apt-get install ttf-mscorefonts-installer
fc-cache
```
这个 mscorefonts 包包含了 Arial、Georgia、Times Roman 等热门的 Microsoft’s Truetype fonts。

## 万恶之源

前段时间淘了一个 racknerd 的垃圾云服务器，顺手注册了一个域名，于是配了 DNS，然后想着这样方便一点。结果就是——

![](/img/62c91730e3f5f46b8ca4737b39c75232.png)

我………………直接拍桌子了。
## 尾声

发现直接公网 IP 访问就不存在这个问题了，那就公网访问吧……反正也就自己用。

> 安装过程参考了 [Blog1](https://blog.skywt.cn/posts/self-host-overleaf#toc_title5),[Blog2](https://www.tnnidm.com/build-and-use-overleaf-server/index.html) 和 [官方指引](https://github.com/overleaf/toolkit/blob/master/doc/quick-start-guide.md)，在此表示感谢。
>
> 虽然阿里云昨天搞了个大新闻，但是既然给我免费 ECS 服务器用，也感谢一下2333. 