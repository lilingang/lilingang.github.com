---
title: 使用acme-tiny获取Let's Encrypt免费HTTPS证书
date: 2017-05-15 17:54:25
categories:
tags:
    - acme-tiny
    - Let's Encrypt
    - letsencrypt
    - https证书
description: 在搭建owncloud私有云的时候，申请了一个免费ddns的二级域名，顺便做下https
---
&emsp;&emsp;证书签发服务[Let's Encrypt](https://letsencrypt.org/)提供免费的https证书，但是证书有效期只有90天，因此获取证书后，每90天刷新，获取新证书，官方有提供工具，但是并不好用。第三方的工具也有一大把，个人认为最好用的就是[acme-tiny](https://github.com/diafygi/acme-tiny)，基于python，几百行的脚本就可以完成获取证书、刷新证书。

## 环境
 - linux系统
 - 安装有python
 - 一个外网可访问的web服务器，apache、nginx，端口必须是80。自己用node或python也可以创建一个简单容器（参考其他文档）

## 获取证书
 - **Stp 1：创建一个账户key**
&emsp;&emsp;先创建一个文件夹，用来放置生成过程中的各种文件
 ```
 mkdir letsencrypt
 cd letsencrypt
 ```
 &emsp;&emsp;如果以前使用过Let's Encrypt，保留着`account.key`，`domain.key`，`domain.csr`，只是想刷新域名，请从“Step 3：获取签名证书”开始
 ```
 $ openssl genrsa 4096 > account.key
 ```
&emsp;&emsp;账户key是用来让Let's Encrypt识别你的身份的，在以后的刷新证书中需要，代表证书签约者身份


 - **Step 2：生成证书签名请求（certificate signing request － CSR）**

&emsp;&emsp;Let's Encrypt使用的ACME协议，需要提交一个CSR文件，包括刷新证书，可以使用相同的CSR刷新多次
 1. 创建一个domain-key，不能将账户key用作域名key
```
$ openssl genrsa 4096 > domain.key
```
 2. 生成CSR
```
#单个域名
$ openssl req -new -sha256 -key domain.key -subj "/CN=yoursite.com" > domain.csr
#多个域名
$ openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/ssl/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:yoursite.com,DNS:www.yoursite.com")) > domain.csr
```
 &emsp;&emsp;根据实际情况以上命令任选其一，Let's Encrypt的限制，不能使用域名泛解析，并且一个证书最多100个域名。多个域名时，添加的每个域名需要指向同一ip，在后面的验证域名身份时，每个域名都会验证一次


 - **Step 3：获取签名证书**
 1. 搭建临时web服务器
 &emsp;&emsp;Let's Encrypt要验证你是不是域名的真正拥有者，在获取证书的过程中，Let's Encrypt会请求一个类似这样的链接
 `http://yoursite.com/.well-known/acme-challenge/xxxxxxxxxxx`
 如果链接所指向的文件可访问，则认为域名是你的，当然这个文件内容不是固定的，也是在获取证书的过程中生成的。
 &emsp;&emsp;因此需要你将域名解析到外网可访问的web服务器上，准确的说是要国外都可以访问，因为Let’s Encrypt服务器是上述地址的访问者，所以国内的主机或者你家里的主机，要确保国外可以访问（如果你的CSR中含有多个域名，每个域名都要解析到该web服务器，因为Let’s Encrypt会把每个域名相应的地址都会访问验证一次，确保你都是域名拥有者）。
 &emsp;&emsp;假如现有web服务器，最简单的做法就是在web服务器根下创建文件夹 `.well-known/acme-challenge/`，记住此文件夹的绝对路径，假如是`/var/www/html/.well-known/acme-challenge/`。
 &emsp;&emsp;也可以在nginx中配置一个规则，例如，此时的绝对路径为`/var/www/challenges/`
 ```
 #example for nginx
    server {
        listen 80;
        server_name yoursite.com www.yoursite.com;

        location /.well-known/acme-challenge/ {
            alias /var/www/challenges/;
            try_files $uri =404;
        }

        ...the rest of your config
    }
 ```
 2. 获取证书 
 从gitbub下载acme_tiny，其实就是一个python脚本，单文件，放置在当前目录
 ```
 python acme_tiny.py --account-key ./account.key --csr ./domain.csr --acme-dir /var/www/html/.well-known/acme-challenge/ > ./signed.crt
 ```
 &emsp;&emsp;命令中需要指定`account.key`，`domain.csr`，和验证域名文件存放路径`--acme-dir`设置为上一步创建的文件夹，需要绝对路径。生成的证书文件保存在当前文件夹下，文件名 `signed.crt`
 &emsp;&emsp;创建域名验证文件和访问域名验证都是在这个命令中一气呵成，所以执行命令的机器和web服务器需在同一主机

 3. 合并中间证书
 ```
 wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > intermediate.pem
 cat signed.crt intermediate.pem > chained.pem
 ```
 &emsp;&emsp;chained.pem即是最终的https证书，是最终在应用web容器中配置的证书。


 - **Step 4：安装证书**

 &emsp;&emsp;不同的web容器安装方式不同，依nginx为例
 ```
 server {
    listen 443;
    server_name yoursite.com, www.yoursite.com;

    ssl on;
    ssl_certificate /path/to/chained.pem;
    ssl_certificate_key /path/to/domain.key;
    ssl_session_timeout 5m;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA;
    ssl_session_cache shared:SSL:50m;
    ssl_dhparam /path/to/server.dhparam;
    ssl_prefer_server_ciphers on;

    ...the rest of your config
}

server {
    listen 80;
    server_name yoursite.com, www.yoursite.com;

    location /.well-known/acme-challenge/ {
        alias /var/www/challenges/;
        try_files $uri =404;
    }

    ...the rest of your config
}
 ```
 &emsp;&emsp;刚才的过程中会生成很多文件，需要配置在应用web容器上的其实就`chained.pem`和`domain.key`，但是其他文件都有用，**需要保留和备份**


 - **Step 5：刷新证书**
 &emsp;&emsp;证书有效期只有90天，所以在证书快要过期的时候，需要刷新证书，给证书续期。可以写成shell脚本，使用linux的crontab定时执行。
 &emsp;&emsp;创建shell脚本`renew_cert.sh`
 ```
 #!/usr/bin/sh
python /path/to/acme_tiny.py --account-key /path/to/account.key --csr /path/to/domain.csr --acme-dir /var/www/html/.well-known/acme-challenge/ > /tmp/signed.crt || exit
wget -O - https://letsencrypt.org/certs/lets-encrypt-x3-cross-signed.pem > intermediate.pem
cat /tmp/signed.crt intermediate.pem > /path/to/chained.pem
service nginx reload
 ```
 &emsp;&emsp;可以看到以上命令，刷新证书其实就是从获取证书步骤开始再来一遍，需要之前生成的账户key **`account.key`**，证书签名请求CSR **`domain.csr`**，以及用来验证域名属于你的临时web服务器。当然你可以将你的应用web服务器用作验证域名用的临时服务器。

 &emsp;&emsp;创建crontab，例如每月执行一次
 ```
 0 0 1 * * /path/to/renew_cert.sh 2>> /var/log/acme_tiny.log
 ```
 &emsp;&emsp;需要注意的地方:
 &emsp;&emsp;  1.验证域名的challenge文件夹的安全性，不知道和应用服务器放置在一起会有什么问题，期望高人来解答。这也是Let’s Encrypt让人比较头疼的地方，刷新域名仍需验证，而且还要占用80端口。
 &emsp;&emsp;  2.注意`account.key`，`domain.key`的保密和备份
 &emsp;&emsp;  3.acme-tiny官方给的建议是，刷新脚本用一个特殊的用户，最大的权限是读取`account.key`、读写challenge文件夹、读写web服务器证书文件的权限（例如/path/to/chained.pem），以及reload web服务器的权限

> 本文参考链接[https://github.com/diafygi/acme-tiny](https://github.com/diafygi/acme-tiny)


 