---
layout: default
---

# Deep Learning学习笔记一

去年的VR/AR，今年的人工智能，都是大公司和资本追逐的热点。以前总觉得AI这事儿太缥缈，大多公司都是忽悠，直到前几天看到Geoff Hinton的事情，才知道近年来因为新算法的提出，这个领域确实被Deep Learning改变了。

这几天翻了不少资料，知乎上看到Naiyan Wang同学的回答“Deep Learning本质上是工程学科，而不是自然学科。”，感觉说的非常好。所以决定这段时间深入研究一下这块。

今天这篇学习笔记主要是写Theano在Mac下的安装，明天再看看怎么使用。

### Python与Anaconda

MacOS Sierra自带Python v2.7.x，所以这块基本无需准备。

但Anaconda是需要装的，不过300M的东西，在官网下载还挺费劲的，这里我们使用清华Tuna，访问：

[https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/]()

选择Anaconda2中的最新版（Anaconda3对应Python 3.x）下载，例如我下载的是 `Anaconda2-4.3.1-MacOSX-x86_64.sh` 。

完成之后直接执行：

```bash
$ bash Anaconda2-4.3.1-MacOSX-x86_64.sh
```

确认完license之后它会问你装到`$HOME/anaconda2`行不行，不行可以输入想要安装的路径作为`PREFIX`。安装完成后还会自动在`$HOME/.bash_profile`中追加`PATH`指向：

```bash
# added by Anaconda2 4.3.1 installer
export PATH="/Users/colin/Tools/anaconda2/bin:$PATH"
```

要使得此配置生效，可以重新开一个Terminal窗口，或执行`source`指令：

```bash
source ~/.bash_profile
```

### 



* * * *

# 新时代

自从2012年创业之后，就很少写博客了。

### 选择GitHub Pages

当年从博客大巴迁移到WordPress ，部署的服务器也从Amazon EC2 Tokyo到阿里云最后变成`backup.tgz`躺在硬盘里，中间还有一年域名被他人抢注，拿回来后又需要重新备案。几经波折，想想还是找一个稳定的服务提供商靠谱一些，毕竟现在已不是当年那个有着大把时间折腾的技术小青年了。

### 从头开始

2012年开始的第一段创业，技术用了不少，从Objective-C到C++到Node.js，从FFmpeg到Cocos2d-x到SocketIO，虽然做出来的产品都很不错，但一直没能沉下心去研究。因为创业逼迫着我思考许多问题，尤其是生存问题。

2015年开始第二段创业，一个重运营的O2O项目，更多的是团队管理、运营、公司治理，把原来很少接触的市场、运营、人力、财务、资本全部走了个遍。成长不少，但却在技术路上越行越远。

这段时间公司调整，慢慢闲了下来，看看现在流行的技术，已比两年前有了很大的变化，我想，是时候该做一些事情了。

就从这个博客开始，从头开始。
