---
layout: post
title:  "Squid极简搭建HTTP/HTTPS代理服务器"
background: '/assets/bg/puket.jpg'
date:   2020-06-22 00:34:00 +0800
categories: scinet
---

最近搞科学上网，需要一个测试用HTTPS代理服务器，选中了 [Squid](http://www.squid-cache.org/)。

搭建分为3步：

1. 启动HTTP代理服务器
2. 添加HTTPS支持
3. 添加Basic认证

# 准备

首先准备以下几项：

1. 服务器一台，最好在外网
2. 装好Docker

# 启动HTTP代理服务器

在服务器上创建`docker-compose.yml`，内容如下：

```yaml
version: '3.4'

services:
  squid:
    image: b4tman/squid
    container_name: squid
    ports:
      - "6666:3128"  # 注意此处端口6666
    volumes:
      - ./cache:/var/spool/squid
      - ./squid.conf:/etc/squid/squid.conf
```

然后建立`squid.conf`文件：

```
http_access allow all
http_port 3128
```

启动squid：

```sh
docker-compose up -d
```

在本机测试（假设服务器域名为`cent.net`）：

```sh
curl -x http://cent.net:6666 -I http://www.baidu.com
```

出现类似如下内容即成功：

```
HTTP/1.1 200 OK
Accept-Ranges: bytes
Cache-Control: private, no-cache, no-store, proxy-revalidate, no-transform
Content-Length: 277
Content-Type: text/html
...
```

> 注意：此时如果用这个代理来访问`http://google.com`，可能会出现`Connection reset by peer`的错误，这是因为HTTP代理是明文传输的，大墙会把它拦截掉。
> 如果改为HTTPS代理，就没有问题，请看下一节。

# 添加HTTPS支持

首先需要准备证书，有三种方式：

1. 自己签名
2. 找机构购买（如阿里云）
3. 使用[acme.sh](https://github.com/acmesh-official/acme.sh)免费生成

我用的第三种，操作官网上写着，这里不多说了。

生成出来的证书文件有两个，分别是：

- 公钥文件 `cent.net.crt`
- 私钥文件 `cent.net.key`

但squid只认`pem`标准格式，所以我们做一个pem证书出来：

```sh
cat cent.net.crt cent.net.key > cent.net.pem
```

然后修改`docker-compose.yaml`，将证书映射进容器，同时将HTTPS端口暴露出来：

```yaml
version: '3.4'

services:
  squid:
    image: b4tman/squid
    container_name: squid
    ports:
      - "6666:3128"
      - "3333:3127"  # 将宿主机的3333映射到容器内部的HTTPS端口3127
    volumes:
      - ./cache:/var/spool/squid
      - ./squid.conf:/etc/squid/squid.conf
      - /home/work/.certs:/certs:ro  # 将证书存放目录~/.certs映射到容器中的/certs目录
```

修改`squid.conf`：

```
http_access allow all
http_port 3128
https_port 3127 \  # HTTPS端口
cert=/certs/cent.net.crt \  # 公钥，注意要填写容器内部路径，而非宿主机路径
key=/certs/cent.net.pem  # 私钥
```

重启容器：

```sh
docker-compose down
docker-compose up -d
```

测试：

```sh
curl -x https://cent.net:3333 -I https://www.google.com
```

看到类似如下内容即为成功：

```
HTTP/1.1 200 Connection established

HTTP/2 200 
content-type: text/html; charset=ISO-8859-1
...
```

> 注意：此时我们的代理服务器是无需认证的，任何人都可以直接连上，很容易被别人扫来做坏事。
> 接下来加上认证，确保只有自己能用。

# 添加Basic认证

首先生成一个账号密码文件（假设`cooolin`是用户名，`c000lin`是密码）：

```sh
docker run --rm xmartlabs/htpasswd cooolin c000lin > htpasswd
```

修改`docker-compose.yml`，将密码文件映射到容器中：

```yaml
version: '3.4'

services:
  squid:
    image: b4tman/squid
    container_name: squid
    ports:
      - "3332:3128"
      - "3333:3127"
    volumes:
      - ./cache:/var/spool/squid
      - ./squid.conf:/etc/squid/squid.conf
      - /home/work/apps/trojan/certs:/certs:ro
      - ./htpasswd:/etc/squid/passwords  # 将密码文件映射到容器内部
```

修改`squid.conf`：

```
auth_param basic program /usr/lib/squid/basic_ncsa_auth /etc/squid/passwords  # 映射进来的密码文件
auth_param basic realm proxy
acl authenticated proxy_auth REQUIRED
http_access allow authenticated
# http_access allow all  # 这句要删除，改为上面那句，即认证后方可访问
http_port 3128
https_port 3127 cert=/certs/cent.net.crt key=/certs/cent.net.pem
```

重启容器

```sh
docker-compose down
docker-compose up -d
```

测试

```sh
curl -x https://cent.net:3333 -I https://www.google.com
```

会收到`407 Proxy Authentication Require`响应：

```
HTTP/1.1 407 Proxy Authentication Required
Server: squid/4.12
Content-Type: text/html;charset=utf-8
Content-Length: 3524
X-Squid-Error: ERR_CACHE_ACCESS_DENIED 0
Proxy-Authenticate: Basic realm="proxy"
Connection: keep-alive
...
```

此时我们添加认证信息再访问：

```sh
curl -x https://cooolin:c000lin@cent.net:3333 -I https://www.google.com
```

得到200响应，认证完成。


全篇完成。

