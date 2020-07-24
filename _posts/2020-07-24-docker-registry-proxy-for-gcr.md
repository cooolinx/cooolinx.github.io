---
layout: post
title: "架设GCR的代理Docker Registry"
subtitle: "谷歌云GCR被阿里云屏蔽，如果要从中取出Docker镜像，只有两个办法"
background: '/assets/bg/puket.jpg'
date:   2020-07-24 20:52:00 +0800
categories: scinet
---

本来服务跑的挺好的，下午突然发现用户收不到短信验证码了，阿里云返回消息：

```json
{"Message": "源IP地址所在的地区被禁用", "RequestId": "xxx", "Code": "isv.DENY_IP_RANGE"}
```

原来是阿里云开始限制境外服务器IP请求短信服务。

由于我们系统是微服务架构，想得比较简单拆分迁移一下就行了，于是吭哧吭哧经过2小时的代码拆分，将短信、邮件和验证码连带一个redis缓存一起拆出来成为了独立的消息推送微服务，准备部署到国内阿里云服务器上。

结果部署的时候傻眼了，阿里云把GCR（Google Cloud Registry，下同）给屏蔽了，好好的镜像取不出来，离晚高峰只有2小时了，此时有两个办法：

1. 用一个外网服务器做HTTPS/SOCKS代理，然后配置docker客户端走代理
2. 用外网服务器做docker registry，并将registry开启代理模式

前者简便，后者合理，我选了后者。

# 1 - 启动Docker Registry

先启动一个[Docker Registry](https://hub.docker.com/_/registry)，直接上`docker-compose.yml`：

```yaml
version: '3.4'

services:

  registry:
    image: registry:2.7
    container_name: registry
    ports:
      - "80:5000"
    restart: always
```

> 因为机器上没有起nginx一类的，就直接映射到80了

启动，一个基本的Docker Registry就好了

```sh
docker-compose up -d
```

# 2 - 挂接代理

启动之后第一件事儿就是关联到GCR，这也是本次的主要目标。

### 2.1 - 配置文件

先将启动后的容器内配置文件拷贝出来：

```sh
cp registry:/etc/docker/registry/config.yml .  # registry是container name
```

然后编辑加入代理配置

```yaml
proxy:
  remoteurl: https://gcr.io
  username: [username]
  password: [password]
```

### 2.2 - GCR账号密码配置（非GCR请跳过）

敲黑板，这里是重点，GCR的`username`和`password`是什么？

- `username`固定值是`_json_key`
- `password`是我们通过Google服务账号下载下来的JSON文件全文（对没有看错，是整个JSON）

> 如果不知道 [Google服务账号JSON文件](https://cloud.google.com/container-registry/docs/advanced-authentication#json-key) 以及如何下载的，点进去看下。

下载好了`Google服务账号JSON文件`后，获取`password`：

```sh
pwd=$(~/.gcloud-auth/your-gcloud-service-account.json); echo $pwd  # 移除换行
```

将出来的结果复制粘贴到`config.yml`配置中

```yaml
proxy:
  remoteurl: https://gcr.io
  username: _json_key
  password: '{ "type": "service_account", "project_id": "cen2", "private_key_id":... }'
```

### 2.3 - 配置文件映射到容器

最后将`config.yml`映射回容器

```yaml
version: '3.4'

services:

  registry:
    image: registry:2.7
    container_name: registry
    ports:
      - "80:5000"
    restart: always
    volumes:
      - ./config.yml:/etc/docker/registry/config.yml  # 映射到容器
```

这样我们的GCR代理（Proxy）或叫镜像（Mirror）Registry就做好了

### 2.4 - 测试

```sh
docker run --rm localhost:80/my-project/pns:latest
```

但这样就完了吗？显然没有，往下看。

# 3 - 自定义存储

之前的容器，缓存的内容在docker自己管理的`volume`里，如果想更加可控，或者是你有个SSD，就需要将存储指向容器外部磁盘。

```yaml
version: '3.4'

services:

  registry:
    image: registry:2.7
    container_name: registry
    ports:
      - "80:5000"
    restart: always
    volumes:
      - ./config.yml:/etc/docker/registry/config.yml
      - ./registry:/var/lib/registry  # 将容器内的`/var/lib/registry`映射出来
```

# 4 - 远程访问

如果你拿另外一台机器测试，就会发现此时的registry仅能本地访问，显然不符合要求

```yaml
version: '3.4'

services:

  registry:
    image: registry:2.7
    container_name: registry
    ports:
      - "80:5000"
    restart: always
    volumes:
      - ./config.yml:/etc/docker/registry/config.yml
      - ./registry:/var/lib/registry
    environment:
      - REGISTRY_HTTP_ADDR=0.0.0.0:5000  # 加上0.0.0.0就好了
```

# 5 - HTTPS

加了TLS对docker客户端友好一些，也安全一些。

### 5.1 - 域名解析

TLS必须有域名，没有先注册一个。然后到后台配置下DNS解析，A记录指向当前服务器。

> 阿里云的域名绑定比较麻烦，还要备案什么的，建议大家自己想想办法

### 5.2 - 生成证书

还是用免费的`Let's Encrypt`证书，如果有自有证书可以省略此步骤

```sh
# 安装acme.sh
curl https://get.acme.sh | sh
source ~/.bashrc
# 用DNS方式生成证书（for GoDaddy）
export GD_Key="GoDaddyKey"
export GD_Secret="GoDaddySecrect"
acme.sh --issue --dns dns_gd -d cen2.com -d '*.cen2.com'
# 证书导出
mkdir ~/.certs
acme.sh --install-cert -d cen2.com \
--key-file ~/.certs/cen2.com.key  \
--fullchain-file ~/.certs/cen2.com.crt \
--reloadcmd "docker-compose -f /home/work/apps/registry/docker-compose.yml restart"  # 证书更新时，自动重启registry服务
```

### 5.3 - 配置证书

```yaml
version: '3.4'

services:

  registry:
    image: registry:2.7
    container_name: registry
    ports:
      - "443:443"
    restart: always
    volumes:
      - ./config.yml:/etc/docker/registry/config.yml
      - ./registry:/var/lib/registry
      - ~/.certs:/certs
    environment:
      - REGISTRY_HTTP_ADDR=0.0.0.0:443
      - REGISTRY_HTTP_TLS_CERTIFICATE=/certs/cen2.com.crt
      - REGISTRY_HTTP_TLS_KEY=/certs/cen2.com.key
```

> 注意，证书文件配置的是容器内的路径，是由我们在`volumes`里映射进去的。

# 6 - 认证

看起来挺完备的，但还有个致命问题，就是这个registry是不需要认证的！

### 6.1 - 生成认证账号密码

```sh
docker run --rm -it xmartlabs/htpasswd myname mypass > auth
```

其中`myname`和`mypass`是自定义的账号密码

### 6.2 - 配置认证信息

下面`REGISTRY_AUTH`开头的，都是认证相关的，其中`REGISTRY_AUTH_HTPASSWD_PATH`需要指向我们刚刚生成的文件

```yaml
version: '3.4'

services:

  registry:
    image: registry:2.7
    container_name: registry
    ports:
      - "443:443"
    restart: always
    volumes:
      - ./config.yml:/etc/docker/registry/config.yml
      - ./registry:/var/lib/registry
      - ~/.certs:/certs
      - ./auth:/auth/htpasswd
    environment:
      - REGISTRY_AUTH=htpasswd
      - "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm"
      - REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd
      - REGISTRY_HTTP_ADDR=0.0.0.0:443
      - REGISTRY_HTTP_TLS_CERTIFICATE=/certs/centaur.cloud.crt
      - REGISTRY_HTTP_TLS_KEY=/certs/centaur.cloud.key
```

现在在本地使用时，要先登录

```sh
docker login cen2.com
```

输入刚刚的账号密码，再拉取

```sh
docker pull cen2.com/my-project/pns
```

至此一个完整的`Docker Proxy Registry`搭建完毕。

# 后话

虽然在1小时内解决了Registry，但后来又遇到了很坑爹的事儿，最后还是争分夺秒在晚高峰前完成了短信服务迁移。

# 参考

- [Docker Registry](https://hub.docker.com/_/registry)
- [Registry as a pull through cache](https://docs.docker.com/registry/recipes/mirror/)
- [Docker Registry Deploying](https://docs.docker.com/registry/deploying/)
- [GCP Advanced Authentication](https://cloud.google.com/container-registry/docs/advanced-authentication#json-key)
- [xmartlabs/htpasswd](https://hub.docker.com/r/xmartlabs/htpasswd)
- [tiangolo/docker-registry-proxy](https://hub.docker.com/r/tiangolo/docker-registry-proxy)
- [ahmetb/serverless-registry-proxy](https://github.com/ahmetb/serverless-registry-proxy)

