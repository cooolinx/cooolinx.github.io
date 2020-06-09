---
layout: post
title:  "科学上网编程 · 二 · tun2socks的TUN"
background: '/assets/bg/puket.jpg'
date:   2020-06-08 22:19:00 +0800
categories: scinet
---

这篇主要讲`tun2socks`里的`TUN`。

# TUN

先讲TUN，[Wikipedia](https://en.wikipedia.org/wiki/TUN/TAP)是这么说的：

> TUN是`network TUNnel`的缩写，是一种以软件实现的虚拟网络设备，模拟的是OSI网络模型（OSI Model）中的第三层网络层（Network Layer）设备，处理IP数据包。

![TUN/TAP OSILayer](/assets/2020/scinet/tun2socks/Tun-tap-osilayers-diagram.png)

[socat](http://www.dest-unreach.org/socat/doc/socat-tun.html)上有个比较好的解释：

> Some operating systems allow the generation of virtual network interfaces that do not connect to a wire but to a process that simulates the network. Often these devices are called TUN or TAP.

通俗点讲：

1. TUN是Linux内核的一个模块，可以虚拟成网络层设备来处理IP包；
2. 我们可以设置此设备的IP地址或网段；
3. 所有与此网段交互的数据包都会到TUN里；
4. TUN同时会生成磁盘上一个字符设备（Character Device）用于程序读写。

<!-- CIDR -->
<!-- 再细节一些，要使用它分为几步：

1. 确认系统中有TUN模块
2. 用代码启动系统内核中的TUN，此时会生成一个文件（其实是一个字符设备`character device`），如`/dev/net/tun0`
3. 为TUN设定IP地址，如`192.168.0.1/24`，这样这个网段收到的所有消息都会到TUN中，消息也可以被TUN模拟成网段中的某个IP发出来
4. 启动一个程序，读取tun文件（没有内容时会阻塞），收到数据后对数据进行处理，想返回数据包时就写入tun文件
 -->

道理是这些道理，但还是很抽象，下面我们实战一下增强理解。

# 简单玩一下TUN

目标是**创建出一个TUN设备可以被查询到**。为了避免各位受环境影响奇怪问题，所有例子都基于docker，没有的去[安装一个](https://www.docker.com/)。

第一步，准备环境。：

```sh
docker run --name tun --rm -it -v $PWD:/app --privileged alpine sh
```

解释：

- 这里我们基于`alpine`（简单体积小），如果要换成`ubuntu`等其他的，只需修改一下拉取依赖的指令即可
- `--privileged`是为了让docker容器拥有完整的网络权限，否则会在第二步中看不到`/dev/net/tun`文件，参见[Docker - Runtime privilege and Linux capabilities](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities)
- `-v $PWD:/app`是将当前路径映射到容器的`/app`目录，为了避免等下写的代码在退出容器后丢失

第二步，确认TUN模块存在，在容器中执行

```sh
ls -l /dev/net
```

如果存在会看到：

```
crw-rw----    1 root     root       10, 200 Jun  9 04:23 tun
```

文件类型`crw-rw----`的第一个字是`c`，是`character device`即`字符设备`的意思。

> 如果前面docker启动时没有加`--privileged`参数，将会看不到文件，如果尝试用`mknod /dev/net/tun c 10 200`建立则会得到`Operation not permitted`。

第三步，安装环境依赖

```sh
# 使用阿里云国内源
sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
# 安装gcc编译环境
apk add g++ make
apk add linux-headers
# 安装后续使用工具
apk add tcpdump
```

第四步，编写代码并编译：

`/app/crtun.cc`

```c++
#include <unistd.h>
#include <stdio.h>  
#include <stdlib.h>  
#include <string.h>  
#include <unistd.h>  
#include <netinet/ip.h>  
#include <linux/if_ether.h>  
#include <linux/if_tun.h>
#include <linux/if.h>
#include <fcntl.h>
#include <sys/ioctl.h>

int tun_alloc(char dev[IFNAMSIZ]) {
  struct ifreq ifr;
  int fd, err;

  if ((fd = open("/dev/net/tun", O_RDWR)) < 0) {  // 打开文件
    perror("open");
    return -1;
  }

  bzero(&ifr, sizeof(ifr));
  ifr.ifr_flags = IFF_TUN | IFF_NO_PI;  // tun设备不包含以太网头部

  if (*dev) {
    strncpy(ifr.ifr_name, dev, IFNAMSIZ);
  }

  if ((err = ioctl(fd, TUNSETIFF, (void *) &ifr)) < 0) {  // 打开设备
    perror("ioctl TUNSETIFF");
    close(fd);
    return err;
  }
  strcpy(dev, ifr.ifr_name);
  return fd;
}

int main(int argc, char *argv[]) {
    char tun_name[IFNAMSIZ];
    tun_name[0] = '\0';
    tun_alloc(tun_name);
    getchar();
    return 0;
}
```

> 存储到`/app`中的文件会被映射到容器外部宿主机上，确保容器销毁后不丢失

保存好后编译：

```sh
cd app
gcc crtun.cc -o crtun
```

第五步，执行及验证

再开一个会话窗口，并进入tun容器：

```sh
docker exec -it tun sh
```

一个会话执行刚刚编译好的程序：

```sh
/app/crtun
```

另一个会话查看网络设备：

```sh
ip address
```

结果如下：

```
...
4: tun0: <POINTOPOINT,MULTICAST,NOARP> mtu 1500 qdisc noop state DOWN qlen 500
    link/[65534] 
...
```

`tun0`就是刚刚创建出来的TUN虚拟设备。回到第一个会话中，按任意键结束`crtun`，再次执行`ip address`，会看到`tun0`已消失。

现在我们创建过一个TUN了，那么TUN到底有什么用呢？下面再看一个栗子。

# 再玩一下TUN

目标是**做一个TUN监控`192.168.0.0/8`网段，使得此网段中所有IP均可接收ping**。

实现思路：

1. 按上一步创建一个TUN
2. IP地址为`192.168.0.1`，配置子网掩码`255.255.255.0`，这样等于`192.168.0.2 ~ 192.168.0.254`都会转发给TUN
3. 实现简单的ICMP echo request协议（ping使用的协议）使之可以被ping

第一步，还是在我们刚刚的docker容器中，建立3个文件：`faketcp.h`、`faketcp.cc`、`icmpecho.cc`

`faketcp.h`

```c++
#include <algorithm>  // std::swap

#include <assert.h>
#include <stdint.h>
#include <string.h>
#include <arpa/inet.h>  // inet_ntop
#include <net/if.h>

struct SocketAddr
{
  uint32_t saddr, daddr;  // 源地址和目的地址
  uint16_t sport, dport;  // 源端口和目的端口

  bool operator==(const SocketAddr& rhs) const
  {
    return saddr == rhs.saddr && daddr == rhs.daddr && 
        sport == rhs.sport && dport == rhs.dport;
  }

  bool operator<(const SocketAddr& rhs) const
  {
    return memcmp(this, &rhs, sizeof(rhs)) < 0;
  }
};

int tun_alloc(char dev[IFNAMSIZ]);
uint16_t in_checksum(const void* buf, int len);

void icmp_input(int fd, const void* input, const void* payload, int len);
```

`faketcp.cc`
```c++
#include "faketcp.h"

#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <linux/if_tun.h>
#include <netinet/in.h>
#include <netinet/ip_icmp.h>
#include <sys/ioctl.h>

int sethostaddr(const char* dev)
{
  struct ifreq ifr;
  bzero(&ifr, sizeof(ifr));
  strcpy(ifr.ifr_name, dev);
  struct sockaddr_in addr;
  bzero(&addr, sizeof addr);
  addr.sin_family = AF_INET;
  inet_pton(AF_INET, "192.168.0.1", &addr.sin_addr);
  //addr.sin_addr.s_addr = htonl(0xc0a80001);
  bcopy(&addr, &ifr.ifr_addr, sizeof addr);
  int sockfd = socket(AF_INET, SOCK_DGRAM, 0);
  if (sockfd < 0)
    return sockfd;
  int err = 0;
  // ifconfig tun0 192.168.0.1
  if ((err = ioctl(sockfd, SIOCSIFADDR, (void *) &ifr)) < 0)
  {
    perror("ioctl SIOCSIFADDR");
    goto done;
  }
  // ifup tun0 其实就是启动tun0
  if ((err = ioctl(sockfd, SIOCGIFFLAGS, (void *) &ifr)) < 0)
  {
    perror("ioctl SIOCGIFFLAGS");
    goto done;
  }
  ifr.ifr_flags |= IFF_UP;
  if ((err = ioctl(sockfd, SIOCSIFFLAGS, (void *) &ifr)) < 0)
  {
    perror("ioctl SIOCSIFFLAGS");
    goto done;
  }
  // ifconfig tun0 192.168.0.1/24 # 配置子网掩码
  inet_pton(AF_INET, "255.255.255.0", &addr.sin_addr);
  bcopy(&addr, &ifr.ifr_netmask, sizeof addr);
  if ((err = ioctl(sockfd, SIOCSIFNETMASK, (void *) &ifr)) < 0)
  {
    perror("ioctl SIOCSIFNETMASK");
    goto done;
  }
done:
  close(sockfd);
  return err;
}

int tun_alloc(char dev[IFNAMSIZ])
{
  struct ifreq ifr;
  int fd, err;

  if ((fd = open("/dev/net/tun", O_RDWR)) < 0)
  {
    perror("open");
    return -1;
  }

  bzero(&ifr, sizeof(ifr));
  ifr.ifr_flags = IFF_TUN | IFF_NO_PI; // tun设备不包含以太网头部,而tap包含,仅此而已

  if (*dev)
  {
    strncpy(ifr.ifr_name, dev, IFNAMSIZ); 
  }

  if ((err = ioctl(fd, TUNSETIFF, (void *) &ifr)) < 0)
  {
    perror("ioctl TUNSETIFF");
    close(fd);
    return err;
  }
  strcpy(dev, ifr.ifr_name);
  if ((err = sethostaddr(dev)) < 0) // 设定地址等信息
    return err;

  return fd;
}

uint16_t in_checksum(const void* buf, int len)
{
  assert(len % 2 == 0);
  const uint16_t* data = static_cast<const uint16_t*>(buf);
  int sum = 0;
  for (int i = 0; i < len; i+=2)
  {
    sum += *data++;
  }
  while (sum >> 16)
    sum = (sum & 0xFFFF) + (sum >> 16);
  assert(sum <= 0xFFFF);
  return ~sum;
}

void icmp_input(int fd, const void* input, const void* payload, int len)
{
  const struct iphdr* iphdr = static_cast<const struct iphdr*>(input); // ip头部
  const struct icmphdr* icmphdr = static_cast<const struct icmphdr*>(payload); // icmp头部
  // const int icmphdr_size = sizeof(*icmphdr);
  const int iphdr_len = iphdr->ihl*4;

  if (icmphdr->type == ICMP_ECHO)
  {
    char source[INET_ADDRSTRLEN];
    char dest[INET_ADDRSTRLEN];
    inet_ntop(AF_INET, &iphdr->saddr, source, INET_ADDRSTRLEN);
    inet_ntop(AF_INET, &iphdr->daddr, dest, INET_ADDRSTRLEN);
    printf("%s > %s: ", source, dest);
    printf("ICMP echo request, id %d, seq %d, length %d\n",
           ntohs(icmphdr->un.echo.id),
           ntohs(icmphdr->un.echo.sequence),
           len - iphdr_len);

    union
    {
      unsigned char output[ETH_FRAME_LEN]; // 以太网头部
      struct
      {
        struct iphdr iphdr;
        struct icmphdr icmphdr;
      } out;
    };

    memcpy(output, input, len);
    out.icmphdr.type = ICMP_ECHOREPLY;
    out.icmphdr.checksum += ICMP_ECHO; // FIXME: not portable
    std::swap(out.iphdr.saddr, out.iphdr.daddr);
    write(fd, output, len);
  }
}
```

`icmpecho.cc`
```c++
#include "faketcp.h"

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <netinet/ip.h>
#include <linux/if_ether.h>

int main()
{
  char ifname[IFNAMSIZ] = "tun%d";
  int fd = tun_alloc(ifname); // tun_alloc函数主要用于开启

  if (fd < 0)
  {
    fprintf(stderr, "tunnel interface allocation failed\n");
    exit(1);
  }

  printf("allocted tunnel interface %s\n", ifname);
  sleep(1);

  for (;;)
  {
    union
    {
      unsigned char buf[ETH_FRAME_LEN]; // 以太网头部
      struct iphdr iphdr;   // ip头部
    };

    const int iphdr_size = sizeof iphdr; // ip头部默认是20字节

    int nread = read(fd, buf, sizeof(buf));
    if (nread < 0)
    {
      perror("read");
      close(fd);
      exit(1);
    }
    printf("read %d bytes from tunnel interface %s.\n", nread, ifname);

    const int iphdr_len = iphdr.ihl*4;
    if (nread >= iphdr_size
        && iphdr.version == 4
        && iphdr_len >= iphdr_size
        && iphdr_len <= nread
        && iphdr.tot_len == htons(nread)
        && in_checksum(buf, iphdr_len) == 0)
    {
      const void* payload = buf + iphdr_len;
      if (iphdr.protocol == IPPROTO_ICMP)  // icmp协议
      {
        icmp_input(fd, buf, payload, nread);
      }
    }
    else
    {
      printf("bad packet\n");
      for (int i = 0; i < nread; ++i)
      {
        if (i % 4 == 0) printf("\n");
        printf("%02x ", buf[i]);
      }
      printf("\n");
    }
  }
  return 0;
}
```

第二步，编译并执行

```sh
g++ icmpecho.cc faketcp.cc -o icmpecho
./icmpecho
```

第三步，打开第二个会话，监听`tun0`的数据包

```sh
tcpdump -i tun0
```

第四步，打开第三个会话，测试`ping`

```sh
ping 192.168.0.3
ping 192.168.0.188
ping 192.168.0.254
```

看到`icmpecho`打印结果：

```
allocted tunnel interface tun0
read 84 bytes from tunnel interface tun0.
192.168.0.1 > 192.168.0.3: ICMP echo request, id 14336, seq 3, length 64
read 84 bytes from tunnel interface tun0.
...
```

看到`tcpdump`打印结果：

```
06:31:13.685231 IP 192.168.0.1 > 192.168.0.2: ICMP echo request, id 19712, seq 0, length 64
06:31:13.686110 IP 192.168.0.2 > 192.168.0.1: ICMP echo reply, id 19712, seq 0, length 64
...
```

至此，大概清楚了TUN的原理和实现。后面继续讲`tun2socks`的SOCKS部分。


# 参考

参考资料：

1. [TUN/TAP - Wikipedia](https://en.wikipedia.org/wiki/TUN/TAP)
2. [TUN/TAP设备浅析(一) -- 原理浅析](https://www.jianshu.com/p/09f9375b7fa7)
3. 以上代码来源 [chenshuo/recipes](https://github.com/chenshuo/recipes/tree/master/faketcp)

延伸阅读：
1. [陈硕：关于 TCP 并发连接的几个思考题与试验](https://blog.csdn.net/solstice/article/details/6579232)
2. [Linux 网络工具详解之 ip tuntap 和 tunctl 创建 tap/tun 设备](https://segmentfault.com/a/1190000018351915)
