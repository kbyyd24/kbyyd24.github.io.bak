---
title: 从http到https
date: 2016-05-27
updated: 2016-05-27
---


这个五一折腾了下`https`，看了加密的建立过程和原理，然后动手实践，把博客从不支持`https`的阿里云虚机上搬到了新买的腾讯云的主机上，配好了`https`，这里记录一下。


# 加密连接建立过程与原理

这个部分不想自己写了，参见 sf 上的[这篇文章](https://segmentfault.com/a/1190000004985253)就很容易理解。

## 我的理解

`https`并不是一个全新的协议，而是一个组合的协议，是`ssl`与`http`组合而来的，其模型如下图
![alt text](https://cattail.me/assets/how-https-works/tcp-ip-model.png)

不难发现其实`https`就是在`ssl`连接的基础上对`http`报文进行加密后发送。通过抓包，我们不难发现这一点：

> 以`segmentfault.com`为例，用`wireshark`抓取发送的数据包
> 用`http`关键字进行筛选：
> ![alt text](/images/http.jpg)
> 用`ip.addr`关键字进行筛选
> ![alt text](/images/ipaddr.jpg)

从截图可以看出：

* 数据包的抓取并不能获取到`http`协议的任何信息，哪怕是`URL`也不行
* 通过`ip`地址获取到的数据包，应用层协议是`SSL`，在协议的报文中包含加密数据包的协议，加密协议的版本等信息，但无法获取原始报文

按照这个逻辑，我们就可以对所有应用层的协议进行加密工作，比如`sftp`。另外，最让码农感觉幸福的是，只要服务器做好配置，我们不需要修改代码，只需要修改接口的协议，其加密过程对于程序来说是透明的。

## letsencrypt

Let's Encrypt 是一个提供免费`SSL`证书的项目，托管在`github`上：
[https://github.com/letsencrypt/letsencrypt](https://github.com/letsencrypt/letsencrypt)

有了这个项目，我们可以方便的把自己的站点升级成为`https`

> 然而，这个项目提供的证书默认有效期是 90 天，我们需要定期的更新证书，或者写个脚本，定时执行。

# 实战

## 准备工作

在获取证书的时候，需要使用 80 和 443 端口，所以需要先关闭占用这两个端口的程序

```shell
# systemctl stop nginx.service
```

## 获取`letsencrypt`

```shell
# git clone https://github.com/letsencrypt/letsencrypt
# cd letsencrypt
# ./letsencrypt-auto -h
```
> 最后一个命令会帮助解决项目的依赖问题

## 获取证书

上一个命令结束后会打印出一些使用的帮助信息，可以根据帮助信息选择需要的参数。

因为我的站点部署在`nginx`上，所以使用如下命令：

```shell
# ./letsencrypt-auto certonly --standalone --email melo@gaoyuexiang.cn -d gaoyuexiang.cn -d www.gaoyuexiang.cn
```

如果得到如下信息则说明证书已经正确获得了：

```shell
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/gaoyuexiang.cn/fullchain.pem. Your cert will
   expire on 2016-08-01. To obtain a new version of the certificate in
   the future, simply run Let's Encrypt again.
 - If you like Let's Encrypt, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

## 配置nginx

`nginx`的配置能够在网上找到很多资料，这里我就把我的与`ssl`相关的配置贴出来

```shell
#https配置
server{
  listen                     443 ssl;
  root                       /opt/wordpress;

  ssl_certificate            /etc/letsencrypt/live/gaoyuexiang.cn/fullchain.pem;
  ssl_certificate_key        /etc/letsencrypt/live/gaoyuexiang.cn/privkey.pem;
  ssl_protocols              TLSv1 TLSv1.1 TLSv1.2;
  ssl_prefer_server_ciphers  on;
  ssl_ciphers                AES256+EECDH:AES256+EDH:!aNULL;
  
  ...
}
# 强制使用https
server {
  listen           80;
  server_name      gaoyuexiang.cn;

  #强制将 http 访问转发到 https
  rewrite ^/(.*) https://$server_name$1 permanent;
}
```

配置完成，启动`nginx`和`php-fpm`

```shell
# systemctl start nginx.service
# systemctl start php-fpm.service
```

然后访问自己的站点，发现已经能够强制使用`https`了。

> `wordpress`还有一些设置需要调整，把以前使用`http`链接本站资源的地方给改过来，如站点URL、主题使用的图片等信息

