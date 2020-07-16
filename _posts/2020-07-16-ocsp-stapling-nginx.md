---
layout: post
title: "解决Let's Encrypt证书的OCSP问题"
subtitle: "从Flutter for iOS的UI卡顿到OCSP Stapling"
background: '/assets/bg/puket.jpg'
date:   2020-07-16 23:22:00 +0800
categories: scinet
---

iOS版发版在即，但却在一位同事的设备上出现发送HTTP请求时的严重卡顿，整个UI都被冻结住了，但Android一切正常。

Shawn查了很久，终于发现是Flutter for iOS的TLS握手时要做OCSP，但Let's Encrypt签发的证书的OCSP地址被墙了导致的。参见：

- [dart-lang/sdk/issues/41519](https://github.com/dart-lang/sdk/issues/41519)
- [flutterchina/dio/issues/703](https://github.com/flutterchina/dio/issues/703)

问题定位到了，怎么解决？网上没有一个讲的很完整的，下面我们抛开Flutter，从服务端着手解决此事：

1. 简单原理
2. 确认证书是否可用
3. 三个解决方案
4. 外网服务器配置OCSP Stapling
5. 内网服务器配置OCSP Responder
6. 验证

# 1 - 简单原理

`OCSP` 全称 [Online Certificate Status Protocol](https://en.wikipedia.org/wiki/Online_Certificate_Status_Protocol)，译为 `在线证书状态协议`。简单来说，当我们访问一个带CA证书的网站时，会访问相应的验证URL来验证此证书的状态。

所以当浏览器或其他客户端访问一个HTTPS网站，拿到证书之后，就会去指定URL验证一下，并将结果缓存一段时间。

看起来一切和谐，这个时候有人跳出来说了，每个客户端都请求验证一下效率太低了，我们能不能在服务端去请求好，然后和TLS握手一起下发？于是就有了`OCSP Stapling`。

然后又有人跳出来说了，你们这个OCSP并不能增强安全性，会导致HTTPS请求时间变长，还不如我分发一个列表到本地来解决证书问题，这个人叫Google。于是从2012年开始Google旗下产品逐渐的去OCSP化，这也是为什么Flutter在Android上没有问题但iOS有。

最后，如果验证URL被防火墙屏蔽了，就会导致TLS握手过程很慢并且失败。

# 2 - 确认证书是否可用

第一步首先要做的，是确认证书是否可用，如果可用，大概率不是OCSP的问题。

准备证书

```sh
# Get server cert
openssl s_client -connect tc.cen2.pw:443 < /dev/null 2>&1 | sed -n '/-----BEGIN/,/-----END/p' > certificate.pem
# Get intermediate cert
openssl s_client -showcerts -connect tc.cen2.pw:443 < /dev/null 2>&1 | sed -n '/-----BEGIN/,/-----END/p' | awk 'BEGIN { n=0 } { if ($0=="-----BEGIN CERTIFICATE-----") { n+=1 } if (n>=2) { print $0 } }' > chain.pem
```

获取OCSP验证URL

```sh
# Get the OCSP responder for server cert
openssl x509 -noout -ocsp_uri -in certificate.pem
# http://ocsp.int-x3.letsencrypt.org

# 或者
# openssl x509 -in certificate.crt -noout -text | grep OCSP
```

发起一个OCSP验证请求

```sh
openssl ocsp -issuer chain.pem -cert certificate.pem \
        -verify_other chain.pem \
        -header "Host" "ocsp.int-x3.letsencrypt.org" -text \
        -url http://ocsp.int-x3.letsencrypt.org
```

正常输出结果

```
OCSP Request Data:
    Version: 1 (0x0)
    Requestor List:
        Certificate ID:
          Hash Algorithm: sha1
          Issuer Name Hash: 7EE66AE7729AB3FCF8A220646C16A12D6071085D
          Issuer Key Hash: A84A6A63047DDDBAE6D139B7A64565EFF3A8ECA1
          Serial Number: 0353F3B3D1D03160B982105841C733978C28
    Request Extensions:
        OCSP Nonce: 
            0410104F5B81F58C45149ACD6EF72B64A333
OCSP Response Data:
    OCSP Response Status: successful (0x0)
    Response Type: Basic OCSP Response
    Version: 1 (0x0)
    Responder Id: C = US, O = Let's Encrypt, CN = Let's Encrypt Authority X3
    Produced At: Jul 15 00:19:00 2020 GMT
    Responses:
    Certificate ID:
      Hash Algorithm: sha1
      Issuer Name Hash: 7EE66AE7729AB3FCF8A220646C16A12D6071085D
      Issuer Key Hash: A84A6A63047DDDBAE6D139B7A64565EFF3A8ECA1
      Serial Number: 0353F3B3D1D03160B982105841C733978C28
    Cert Status: good
    This Update: Jul 15 00:00:00 2020 GMT
    Next Update: Jul 22 00:00:00 2020 GMT

    Signature Algorithm: sha256WithRSAEncryption
         1d:a8:35:ba:14:83:fe:1a:0b:95:e8:b8:9f:5a:18:3f:fa:ca:
         3b:db:74:10:68:9c:dd:aa:e2:c3:af:2e:c7:c6:80:02:49:84:
         e1:4b:98:0f:b4:e1:88:4d:14:7d:ae:18:12:ee:0c:21:6d:c0:
         7a:00:48:17:a2:b9:0b:80:34:34:cb:00:a0:cf:ee:86:c0:ea:
         6d:66:0e:eb:af:0c:30:93:4f:c1:86:46:15:e1:5f:60:3d:5f:
         33:dc:3e:97:a5:8d:94:52:b9:b1:fe:1a:0a:b1:59:4b:a2:d2:
         11:fe:09:87:9e:ce:5f:c7:8b:b5:3c:c0:a2:61:a8:37:0b:93:
         3c:0b:82:2e:da:49:76:4a:23:e2:4d:45:4b:81:34:90:8d:0c:
         a0:65:76:8a:de:0f:32:bb:1f:da:fa:91:32:d2:c3:4a:d5:d8:
         04:66:ec:1d:d3:12:12:a6:6a:23:93:6e:d1:45:c7:12:ce:7a:
         0a:c8:47:31:fc:1f:e3:19:a2:c0:02:2a:26:55:a6:58:7b:41:
         31:1c:6e:55:cf:68:08:b3:05:dd:96:31:15:bb:14:9b:7c:65:
         e6:18:de:fa:1a:9d:59:7a:b1:41:fc:d7:88:8c:5e:56:9f:c7:
         69:f8:2f:be:6c:ae:0c:7f:9a:58:d1:39:c3:55:1a:5f:2c:42:
         c8:3b:20:14
WARNING: no nonce in response
Response verify OK
certificate.pem: good
    This Update: Jul 15 00:00:00 2020 GMT
    Next Update: Jul 22 00:00:00 2020 GMT
```

如果OCSP验证URL不可访问，就会被卡在`OCSP Request Data`输出之后，一直等待`OCSP Response Data`

由于我们使用的`Let's Encrypt`的证书，验证URL`http://ocsp.int-x3.letsencrypt.org`后面的实际地址，大部分已经被和谐了。因此必须用其他方式来解决。

# 3 - 三个解决方案

1. 更换证书（直接关闭此网页）
2. 避免使用HTTPS链接（不推荐）
3. 配置服务端OCSP Stapling（往下看）

# 4 - 外网服务器配置OCSP Stapling

如果我们的服务器在外网的，那就比较好办了，直接启用`OCSP Stapling`，在服务端提前做OCSP验证，然后再把验证信息随TLS握手下发。编辑服务端`nginx.conf`

```nginx
http {
    # ...

    # DNS解析器
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 10s;

    server {
        # ...
        ssl_stapling on;
        ssl_stapling_verify on;
        ssl_trusted_certificate /certs/cen2.pw.crt;  # 和ssl_certificate保持一致
    }
}
```

重启nginx即可

# 5 - 内网服务器配置OCSP responder

如果我们的服务器在内网，本身无法访问OCSP验证URL，那么你需要：

1. 一个HTTP代理服务器允许访问外网
2. 配置一个`stapling responder`

假设HTTP代理服务器已经有了，下面讲一下`stapling responder`的配置。

首先得到验证用URL

```sh
openssl x509 -in certificate.crt -noout -text | grep OCSP
# eg. OCSP - URI:http://ocsp.int-x3.letsencrypt.org
```

我封装了一个responder的简单docker镜像 [cooolin/ocsp-proxy](https://hub.docker.com/r/cooolin/ocsp-proxy)，可以直接使用（[源码及文档](https://github.com/fallenlord/ocsp-proxy)），将其在服务端启动起来

```sh
docker run -d --rm --name ocsp-proxy -p 8080:8080 \
           -e HTTP_PROXY=http://YOUR_PROXY:8888 \
           -e ocsphost='http://ocsp.int-x3.letsencrypt.org' \
           -e http=':8080' \
           cooolin/ocsp-proxy
```

注意`YOUR_PROXY`需指向已存在的HTTP代理服务器，然后它就开始监听8080端口了

然后在`nginx.conf`中配置：

```nginx
ssl_stapling on;
ssl_stapling_verify on;
ssl_stapling_responder http://127.0.0.1:8080/; 
ssl_trusted_certificate /etc/ssl/ca-certs.pem;  # as the same as `ssl_certificate`
```

注意若nginx在docker容器中，`127.0.0.1`需改为宿主机IP，然后重启nginx即可。

# 6 - 验证

验证OCSP Stapling是否配置成功

```sh
openssl s_client -connect tc.cen2.pw:443 -tls1 -tlsextdebug -status < /dev/null 2>&1 | awk '{ if ($0 ~ /OCSP response: no response sent/) { print "disabled" } else if ($0 ~ /OCSP Response Status: successful/) { print "enabled" } }'
```

`enabled`表示配置成功，`disabled`表示配置失败。


> 参考<br />
> https://akshayranganath.github.io/OCSP-Validation-With-Openssl/<br />
> https://jhuo.ca/post/ocsp-stapling-letsencrypt/

