layout: post
title: OpenSSL自签证书
author: Pyker
categories: TLS
tags:
  - tls
  - ssl
date: 2018-1-10 19:15:00
---

# 概念
`证书`: 英文叫`digital certificate` 或`public key certificate`。 用于在客户端和服务器端做通信时的一种验证证明。
`CA`: CA是Certificate Authority的缩写,也叫“证书授权中心”。
`CA证书`: 顾名思义，就是CA(证书授权中心)颁发的证书。

# 安装
想要使用证书，首先需要安装openssl。利用openssl生成根证书，以后的服务器端证书或者客户端证书都用他来签发，可以建立多个根证书。
```bash
# 安装
$ yum install -y openssl-devel

# 查看版本
$ openssl version
OpenSSL 1.0.2k-fips  26 Jan 2017
```

# X509证书链
x509证书一般会用到三类文件，它们分别是`key`，`csr`，`crt`。 key是私用密钥，openssl格式，通常是rsa算法。`csr`是证书请求文件，用于申请证书。在制作csr文件的时候，必须使用自己的私钥来签署申请，还可以设定一个密钥。`crt`是CA认证后的证书文件，签署人用自己的key给你签署的凭证。

## 证书格式
`.key格式`：私有的密钥
`.csr格式`：证书签名请求（证书请求文件），含有公钥信息，certificate signing request的缩写
`.crt格式`：证书文件，certificate的缩写
`.crl格式`：证书吊销列表，Certificate Revocation List的缩写
`.pem格式`：用于导出，导入证书时候的证书的格式，有证书开头，结尾的格式

# openssl常用命令
```
 -new            表示新的请求。
 -key            指定证书密钥
 -out            输出路径
 -in             输入文件
 req             产生证书签发申请命令
 x509            签发X.509格式证书命令。
 -subj           指定用户信息
 ca              签发证书命令
 genrsa          产生RSA密钥命令
 -CAcreateserial 表示创建CA证书序列号
 -CAkey          表示CA证书密钥
 -CAserial       表示CA证书序列号文件
 -CA             表示CA证书
 -cert           表示证书文件
 -clcerts        表示仅导出客户证书。
 -days           表示有效天数
 -export         表示导出证书。
 -extensions     表示按OpenSSL配置文件v3_ca项添加扩展。
 -extensions     表示按OpenSSL配置文件v3_req项添加扩展。
 -inkey          表示输入文件
 -keyfile        表示根证书密钥文件
 -req            表示证书输入请求。
 -sha1           表示证书摘要算法,这里为SHA1算法。
 -signkey        表示自签名密钥
 pkcs12          PKCS#12编码格式证书命令。
 rand            随机数命令 
 -aes256         使用AES算法（256为密钥）对产生的私钥加密。可选算法包括DES,DESede,IDEA和AES。
```