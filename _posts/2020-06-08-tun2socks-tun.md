---
layout: post
title:  "摩登穿墙 · 二 · tun2socks的TUN"
background: '/assets/bg/puket.jpg'
date:   2020-06-08 22:19:00 +0800
categories: scinet
---

这篇主要讲`tun2socks`里的`TUN`。

# TUN

先讲TUN，根据[Wikipedia介绍](https://en.wikipedia.org/wiki/TUN/TAP)：

> TUN是`network TUNnel`的缩写，是一种以软件实现的虚拟网络设备，模拟的是7层中的第3层网络层（Network Layer）设备，处理IP数据包。

![TUN/TAP OSILayer](/images/2020/06/08/tun2socks/Tun-tap-osilayers-diagram.png)

简单来说，TUN是Linux内核自带的一个模块，要使用它分为几步：

1. 命令行启动系统内核中的TUN，会生成一个文件，如`/dev/net/tun0`
2. 为TUN设定IP地址，如`192.168.0.1/24`，这样这个网段收到的所有消息都会到TUN中，消息也可以被TUN模拟成网段中的某个IP发出来
3. 启动一个程序，读取tun文件（没有内容时会阻塞），收到数据后对数据进行处理，想返回数据包时就写入tun文件

# 玩一下TUN

*&lt;补齐代码&gt;*

参考资料：
1. [TUN/TAP - Wikipedia](https://en.wikipedia.org/wiki/TUN/TAP)
2. [TUN/TAP设备浅析(一) -- 原理浅析](https://www.jianshu.com/p/09f9375b7fa7)
