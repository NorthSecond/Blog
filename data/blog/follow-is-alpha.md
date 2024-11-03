---
type: Blog
title: "Follow RSS 阅读器 Alpha 测试体验"
date: 2024-11-03 19:00:00 +0800
tags: ['Notes']
summary: "算是能取代我常用的 Thunderbird 了"
---

前段时间上网冲浪，看见有人推荐了 [Follow](https://follow.is) 作为新的 RSS 阅读器出现，我本身也是 RSS 订阅和阅读者（此前一直在用 Thunderbird 在管理相关的订阅和进行阅读）and RSSHub 的使用者（但是在白嫖的阿里云学生免费服务器过期之后就没有自己搭建了），于是就找了个邀请码下载下来体验了一段时间。

看了一下应该是使用了一个月了，放一些代表性图片，总结了一下我在使用过程中发现的优缺点。

![image-20241103193535138](https://blog-img.yfyang.me/2024/11/c72208d4c5d0aca555c0ce5b70e62907.png)

### 体验好的地方

1.   界面很现代很美化，换到 Follow 感觉像是瞬间从上世纪走向 2024 年一样，阅读体验确实有一定的提升；
2.   不同类别的列表和分类展示做的很不错
     -   ![image-20241103195106815](https://blog-img.yfyang.me/2024/11/15034f426dcb47b2147638f0dc38669d.png)
3.   可以直接订阅别人的订阅列表，如果相关方向有人做好了列表的化可以直接订阅
4.   由于和 RSSHub 是同一个作者，所以对 RSSHub 等的支持做的还是不错的
     -   ![image-20241103195148937](https://blog-img.yfyang.me/2024/11/cdda9c89fa7a94276058bca4a25002ae.png)

### 体验不好的地方

1.   个别地方渲染还是有问题
     -   比如英伟达博客这个地方图片渲染就位置问题，没有按照预期一样渲染在页面中间
     -   ![image-20241103194252255](https://blog-img.yfyang.me/2024/11/3f45a68a3176465c4ba74e0f0cee8a51.png)
2.   RSS 订阅的内容是在线读取的
     -   这一点我是有点能理解开发者但是又有点不能理解开发者的
     -   尤其是订阅默认是公开的and虽然有私人订阅的功能，但是把所有订阅的内容和阅读状态直接上传也是有点不太放心，如果一些订阅列表命名为像是”自己的博客“or”朋友们的博客“，未免可能容易被开盒
     -   一些局域网内的 RSS 订阅就不能使用 Follow 进行阅读了
     -   应该是海外服务器吧，在网络环境不好的情况下感觉访问还是有点慢
3.   个别时候更新不成功
     -   感觉更新过程好像还是有一点失败的概率的
     -   自己手动下载过两三次更新包
4.   缺少一个在系统浏览器打开的选项
     -   对于部分订阅源，我可能在看见 RSS 的推送之后，需要打开对应的网页获取更多信息，目前的快捷方式只能在内嵌的 webview 中打开，我访问部分网站喜欢使用系统浏览器（有插件增强），但是目前好像只能通过复制链接的方式打开，还是有点麻烦的。

---

总的来说，这玩意基本上可以取代我以前使用的 Thunderbird RSS 订阅器，成为日常使用的阅读工具。同时也期待能够早日发一版安卓版客户端，在手机上也能进行对应的阅读



btw， 对于没有 RSS 阅读习惯的观众来说，RSS 阅读对于大多数人来说其实是个伪需求，它给你推送的内容实际上还是经过你自己选择的，并不能帮你脱离信息茧房。因此，没有需求的观众可以试用一下，但是最好还是不要找二道贩子掏钱购买了（）