layout: post
title: OpenSSL自签证书
author: Pyker
categories: TLS
tags:
  - openssl
  - ssl
date: 2018-1-10 19:15:00
---

# 概念
`证书`: 英文叫`digital certificate` 或`public key certificate`。 用于在客户端和服务器端做通信时的一种验证证明。
`CA`: CA是Certificate Authority的缩写,也叫“证书授权中心”。如果要被他人安全访问，该证书授权中心为第三方公司（要钱）
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
 genrsa          产生RSA密钥命令
 -new            表示生成新的证书请求。
 -key            指定证书私钥
 -out            输出路径
 -in             输入文件
 req             产生证书签发申请命令
 x509            签发X.509格式证书命令。 
 -signkey        表示自签名密钥
 -req            表示证书输入请求。
 -days           表示有效天数
 -CAcreateserial 表示创建CA证书序列号
 -CAkey          表示CA证书密钥
 -CA             表示CA证书
 -CAserial       表示CA证书序列号文件
 
 ca              签发证书命令
 -cert           表示证书文件
 -keyfile        表示根证书密钥文件
 -subj           指定用户信息 
 -clcerts        表示仅导出客户证书。
 -inkey          表示输入文件
 -export         表示导出证书。
 -extensions     表示按OpenSSL配置文件v3_ca项添加扩展。
 -extensions     表示按OpenSSL配置文件v3_req项添加扩展。
 -sha1           表示证书摘要算法,这里为SHA1算法。
 pkcs12          PKCS#12编码格式证书命令。
 rand            随机数命令 
 -aes256         使用AES算法（256为密钥）对产生的私钥加密。可选算法包括DES,DESede,IDEA和AES。
```

# 生成CA证书步骤
生成CA私钥（.key）--> 生成CA证书请求（.csr）--> 自签名得到CA根证书（.crt）（CA给自已颁发的证书）。该CA证书可用于签名其他服务证书。

## 生成CA密钥
```bash
$ openssl genrsa -out cakey.pem 2048
$ cat ca.key
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEAtTAp8GmUsWJ7q3UPcaQUCYx7ue32Ki0nzI6X8BA2zNT0lMkD
HhxdbWPWXaSEiaoiRPtMoCOtemLMzX8MRjCtA31yf3RRsXKIIg6OFfhesgBDdGcf
7Zkhk9zKq1evs8y4G7wNduOg76lKLQg3SuWSS8OlRBS4t17jPY7Z4Kr6UZtn40Fc
19I1FEEjZaOCTUk/WN9G2E/pO+NWN23+2m6O/STiESyFSq5wSx1Vqio2BAoidiCh
dJgxT4lL8tv3YBHCO4oWkex6Nb3yhZq86yfJIwXXj+feJZb0hFJQ2MpECMhk95HH
kiwLlQc+y3Nj0fOrqbBMFdnmyCv2wzC6xx1/HQIDAQABAoIBAC586BXOER+eJBru
0wKWVanJiKlA2+sgYNjEMUmf71+IuCRAmvMr1fDOL98g6fykUVyfmZ5w6P7AwMls
8opDzPBbTHhVMOy1dSY/08bhTfKfzK7eErwUkR/uA3YI7oTUXtyG2HGLn+w95FE/
jWhDFNEppoqcQnSR/P37W/2gAM/VADSAwF44mBeJMwWEH51FbTQVDWtnaXnzEtnG
qXBe7Jrfy01EvNp9auQ6arGgdBf0YcmGEd/5d0numkvCrmm0YgkNzhG5abcl75W6
bwkY8ubGUhbe/JXxyXfr2ryKcin1LkiERgmwSUwjcAuskuIJMJmAk4weIos5RXLp
Igx2r8ECgYEA3mv7dSAOtEs3ztD5D5VqirEs/QLb6hKtKSDJhwn66FAl3A6NWvTU
p/nhZ/RznP2VGPRCAJeBg9SFj7wkMP77bz5dPyJBWfX8CK92PMWreSfzYrs32xah
Zqs6+7rML94MSPuSwNE7rY4tyUwyFj1wA7UqwheRe6rfkEej7LveATECgYEA0Iqb
3VmxKwIraNlPlhgOxwX2cNzLPCUT0UWSYMdy1E6BwOuluauqx1VHdGZ7k51Di9Sa
DRgMQpVPF9Bk83P/x9h4qF6EsVCV+sVw5pWB+dhAEjA3UgOFGYG+RjyfFRboQbcc
sMQpXjfcdJT4lRSRA3WICPxLFn2/m+7d5CQNga0CgYEA2crovma2n0qsCgLMbrsD
SW12PQWIq6rADm7Bh055dvPMLq+9MJxeg2EGm9FdSBNy5K2A162DL8BxTC6RTbzQ
Hbz2d7SmQ12//g05/QYeAxPgmgPzDMAbKTpwFkByYkjOxMQ6jj4Tbr2zDdJjlS1x
ut+yT73eQjculMvhsxS+rXECgYAYp4ptzODJOORw7OAf2pBEr0vHZBMS9T82iocX
sfy9ZNqqODHLlaQHFOnxtPv/I6SMr4HW8nTgmk5Tfmuw7JHcypbZMPN3ExPoJdeH
Kz3Gj+5jOBgSNiBSN6iLHTehgqfKvR9DNq29WdVSYxpQZbIPOqHujgVCj3NLuB27
jxeZsQKBgQCIkOrz8Ly2heIhhkBLcP5D90RW4lG4nbWpHSmKU06t8lKozB1YppUH
2JsKJm8X4kh+VfzcpheY3Hju3Nw/aL/xQDmoh74kE8/VvTfB4HuAVbV9cQArL+6C
UANsu9K+NtnME1cbluQtLpe0XhElcix5mXWnGYX4ZojBzVxt/o+mww==
-----END RSA PRIVATE KEY-----
```

## 生成CA证书请求
```bash
$ openssl req -new -key cakey.pem -out ca.csr

You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN               # ----- 证书持有者所在国家
State or Province Name (full name) []:GuangDong    # ----- 证书持有者所在州或省份
Locality Name (eg, city) [Default City]:ShenZhen   # ----- 证书持有者所在城市
Organization Name (eg, company) [Default Company Ltd]:派有限公司 # -----  证书持有者公司
Organizational Unit Name (eg, section) []:IT # ----- 证书持有者所属部门
Common Name (eg, your name or your server's hostname) []:ssl.ipyker.com # ---- 域名
Email Address []:pyker@qq.com # ----- 邮箱 （可不填）

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:   # ----- 证书密码
An optional company name []:  # ----- 确认证书密码

$ cat ca.csr
-----BEGIN CERTIFICATE REQUEST-----
MIIC2DCCAcACAQAwgZIxCzAJBgNVBAYTAkNOMRIwEAYDVQQIDAlHdWFuZ0Rvbmcx
ETAPBgNVBAcMCFNoZW5aaGVuMQ8wDQYDVQQKDAZpcHlrZXIxFTATBgNVBAsMDHB5
a2VyJ3MgYmxvZzEXMBUGA1UEAwwOc3NsLmlweWtlci5jb20xGzAZBgkqhkiG9w0B
CQEWDHB5a2VyQHFxLmNvbTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEB
ALUwKfBplLFie6t1D3GkFAmMe7nt9iotJ8yOl/AQNszU9JTJAx4cXW1j1l2khImq
IkT7TKAjrXpizM1/DEYwrQN9cn90UbFyiCIOjhX4XrIAQ3RnH+2ZIZPcyqtXr7PM
uBu8DXbjoO+pSi0IN0rlkkvDpUQUuLde4z2O2eCq+lGbZ+NBXNfSNRRBI2Wjgk1J
P1jfRthP6TvjVjdt/tpujv0k4hEshUqucEsdVaoqNgQKInYgoXSYMU+JS/Lb92AR
wjuKFpHsejW98oWavOsnySMF14/n3iWW9IRSUNjKRAjIZPeRx5IsC5UHPstzY9Hz
q6mwTBXZ5sgr9sMwuscdfx0CAwEAAaAAMA0GCSqGSIb3DQEBCwUAA4IBAQA4UgXf
pamRmrL74i7EpzaLw1RQ0T4dF/VCbNuVU8ajvR8ajm6pe1XkoYjKarnc8mQo0Wwq
ukWZhyGoG0ggn0HpgfEQA4AQSjQTjp7Kh8B0N5i+KnfygzQzRyDr+MXdAvvwd9aq
1Hkyb2dqQhamyK62gQ4A2AG9xraxVsJB4KLEGY/wCLzIjaqToPv1LlyWpejn6jeJ
XbfC+pI52yRdWYCvEBXzp8pNp0EdCVeUXJ05c6KZHEx+GuhiRMxWqLRqiqrgdi2u
OX5v4HxMuvHrd3l7eb2qt/82gnmIlNODfFv3ogIAZz/1ut4d+XUle6gWlTtDjCLl
BKGSxBd2ELFmSTCc
-----END CERTIFICATE REQUEST-----
```

## 自签名得到CA根证书
```bash
$ openssl x509 -req -in ca.csr -out ca.pem -signkey cakey.pem -days 3650
$ cat ca.pem 
-----BEGIN CERTIFICATE-----
MIIDojCCAooCCQD1vqZ2J4Kr9TANBgkqhkiG9w0BAQsFADCBkjELMAkGA1UEBhMC
Q04xEjAQBgNVBAgMCUd1YW5nRG9uZzERMA8GA1UEBwwIU2hlblpoZW4xDzANBgNV
BAoMBmlweWtlcjEVMBMGA1UECwwMcHlrZXIncyBibG9nMRcwFQYDVQQDDA5zc2wu
aXB5a2VyLmNvbTEbMBkGCSqGSIb3DQEJARYMcHlrZXJAcXEuY29tMB4XDTE5MDYy
NjAyMDcxOVoXDTI5MDYyMzAyMDcxOVowgZIxCzAJBgNVBAYTAkNOMRIwEAYDVQQI
DAlHdWFuZ0RvbmcxETAPBgNVBAcMCFNoZW5aaGVuMQ8wDQYDVQQKDAZpcHlrZXIx
FTATBgNVBAsMDHB5a2VyJ3MgYmxvZzEXMBUGA1UEAwwOc3NsLmlweWtlci5jb20x
GzAZBgkqhkiG9w0BCQEWDHB5a2VyQHFxLmNvbTCCASIwDQYJKoZIhvcNAQEBBQAD
ggEPADCCAQoCggEBALUwKfBplLFie6t1D3GkFAmMe7nt9iotJ8yOl/AQNszU9JTJ
Ax4cXW1j1l2khImqIkT7TKAjrXpizM1/DEYwrQN9cn90UbFyiCIOjhX4XrIAQ3Rn
H+2ZIZPcyqtXr7PMuBu8DXbjoO+pSi0IN0rlkkvDpUQUuLde4z2O2eCq+lGbZ+NB
XNfSNRRBI2Wjgk1JP1jfRthP6TvjVjdt/tpujv0k4hEshUqucEsdVaoqNgQKInYg
oXSYMU+JS/Lb92ARwjuKFpHsejW98oWavOsnySMF14/n3iWW9IRSUNjKRAjIZPeR
x5IsC5UHPstzY9Hzq6mwTBXZ5sgr9sMwuscdfx0CAwEAATANBgkqhkiG9w0BAQsF
AAOCAQEAOtAJpnww2LLO1pMVb/tBChgAYloCj0GfykGUryLiWqDCeFXYMO5KLonW
0T87aDxvYDEKgo1ITAK26EXlcvi03Pq1DfUfLWrLK+B1+IXXFYGuAcWyrxqM1hp8
TN/g9CB5GXnY8AiEflLw7IpajYRJ0f9sSaIcH8o0sUVTZzBJ9kExv+m0w6GW8+oX
+2Olh8KKavw7ctNze2/A9dTRucIfWH+a+A0dtPX5rnSnkiiBG8xHfMxKsgTl83l5
CunrFcarJMeI/f/49rdiaUr68thNTVx/eUWpAKdgjO2t5JLPGx1iqs9bJCiaWbpg
M97z4KTuB6BizyQBicILVDULhMCtag==
-----END CERTIFICATE-----
```
以上步骤即是自签根证书了。

# 用户证书生成步骤
一般说的生成用户证书分两种，客户端证书和服务端证书，除了名称不一样步骤命令完全一样一样的。
`生成私钥（.key）-->生成证书请求（.csr）-->用CA根证书签名得到证书（.crt）`

## 生成密钥
```bash
$ openssl genrsa -out serverkey.pem 2048
$ cat server.key 
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAvC2I47h2lMYsZalJ16Uy9HYqcoecTep2gT/RRMO3wnHF8e1p
I8iU+GQXkIsMhTmjjcyhjfxxtooxnkp+U6mVdhkDkufMm+ER3z5mCXlhbA8T2Ia6
GGcUScX+4qA834H/Ams8r0UoOc6zi9Tnku+AnK6eq7apYc3KoIc2Uj6qyS9jAkGN
JFplEdjUgSHBsaB78wi8ukN2nGeZR9c4WVoXm8Cz63RR8a9FB/JjVrCe8Ey1Xk55
uTiU/NJa2VWLi/92MKwvnyulr004yIS1DsN06X3fneffV8HVux+LPgL6QHH55UJj
Kp1LBqvmNFrrS4AReI73JVQR5j6vVvsyeSXbuwIDAQABAoIBADpkJsYCx0kC9WPW
VAOGT3lr8V/4lJfY2Uzh8J3V3X+IrlOTx7xC0XcCGA3SF+B/MjEd/kOAwghSeXMU
yn5LcQVkXaeIJgV4oYMUabUm5QQS6aWWqMhJtBHwTlckQb9ZJzgo7nu0ifbmHPCW
8AS4LMBxruq5k3W11dpaGpEKwRQMB97ZvbgaIJxgIp/9Oevjq6OYX9UXa0ztxY8k
itvLpMPaNlSc39T5XYumq52MvOccZGzDO3Njv9MG6rWt/Fw1NWsB/6RshUr4yeVe
QD66Wp+HaD9qaycb9jIR3SeJQoKaEgq0mmA/y5np3SKP/HzcmYK/ubcapF/7cilE
XR9MwYECgYEA5i7d4qSmX1BmphrTJ1yWcTESx4aXsjAuDY+HHMe1nLyozek3aXuc
IkTx4JE/HGYqjOeminYhuaZ5jJLRf07s1VT+8lk/XobDplmQmH8kAHtHFuIWY5l/
EW2L+QV31AFv0Dt5OEhUuTE6F1+6xG8dDPoIGIWIg4luNtylslhQO30CgYEA0UiX
4ED2yHF5cd9cLKWi0niYQZILO87N9U0IRxIQbpdm7UZGuLPLMs/W7Xf+kjOb3qFC
hNS5KGT3JbuhbcwO+34gULrK7AYejwhUhO7CYi28OaRropjoxId7nMcQw2ciISTE
eJHKVXz0PKW7Wt2uq3n6unwW8VNZm6qXfA3G6ZcCgYA78ZqRCkXVbo+81CGHD6KS
CbCVS2S337ouh+EsyoluLuda8FAg5TLs7b17uPeRgr20AiOpzUfNHCBtTlLGb5xX
lhHqtPk+uaO773krbXjHs1L5D5m7CF9B/6BDEnx5NoKS3NodoSCHNd2l9qUhwLn1
BiwTjrrVXnXYTa/M+Riz1QKBgEIdjN1rqIrqTlOLHLN+IFIdhvwwBxx92NMF4veQ
3WAStJGBAhaXtjn3Lw8WOXY2l6ddioYsLdJ1Ex74h6cIMDODRPI8EJ8/z6egGhNk
2kPp7uzG5LoZVG/B3WtJ+CHDEyUlWGw+oo0fTIlcUjQClIvXnT4MtbLHgieLXQ/z
ykNBAoGADlz53txUajLxhBPdEs6f7rIOux0e02AaYff9oJlvikjmIAdvUDQ2HI6i
118pZpTwyi8wDDw6QlhpHssEIUEeZ4Umu1k/tvKqGg5VWfisUU9gch5lbxtZ7tdp
RpsBDBtqfjUbWhnApZCxfA45mfB/xGwPPFLdPUKX9578wsDQQlU=
-----END RSA PRIVATE KEY-----
```

## 生成证书请求文件
```bash
$ openssl req -new -key serverkey.pem -out server.csr -subj "/C=CN/ST=GuangDong/L=ShenZhen/O=公司名/OU=it/CN=tls.ipyker.com/emailAddress=pyker@qq.com"
$ cat server.csr 
-----BEGIN CERTIFICATE REQUEST-----
MIIC2jCCAcICAQAwgZQxCzAJBgNVBAYTAkNOMRIwEAYDVQQIDAlHdWFuZ0Rvbmcx
ETAPBgNVBAcMCFNoZW5aaGVuMRswGQYDVQQKDBLDpcKFwqzDpcKPwrjDpcKQwo0x
CzAJBgNVBAsMAml0MRcwFQYDVQQDDA50bHMuaXB5a2VyLmNvbTEbMBkGCSqGSIb3
DQEJARYMcHlrZXJAcXEuY29tMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKC
AQEAvC2I47h2lMYsZalJ16Uy9HYqcoecTep2gT/RRMO3wnHF8e1pI8iU+GQXkIsM
hTmjjcyhjfxxtooxnkp+U6mVdhkDkufMm+ER3z5mCXlhbA8T2Ia6GGcUScX+4qA8
34H/Ams8r0UoOc6zi9Tnku+AnK6eq7apYc3KoIc2Uj6qyS9jAkGNJFplEdjUgSHB
saB78wi8ukN2nGeZR9c4WVoXm8Cz63RR8a9FB/JjVrCe8Ey1Xk55uTiU/NJa2VWL
i/92MKwvnyulr004yIS1DsN06X3fneffV8HVux+LPgL6QHH55UJjKp1LBqvmNFrr
S4AReI73JVQR5j6vVvsyeSXbuwIDAQABoAAwDQYJKoZIhvcNAQELBQADggEBACIL
Q8Exa81xOEPliER0b86h5xKqxBXInqETn0vKYXjU+D5hjePtNB0NTP/gleYMNrjK
V14XMj3iI9Dq3lMAiWa06Z4Mx2CowqbQY9dvkm/MoDOIoA2q+fWr4H+NACsALYWh
3/sALDdcry9FcOrC6nxD9sBcK8n65z3hBjWudvBFtFYTQ+zRuWHeD4aBxmbpfaDC
W/rl644AJ6varcAQ29yqxiWFvyD+C3lKycCmFY2D52pJLcw4IoVvGx5VzG7KrTUD
W+qPclyM6vj0SeDaYKZKmUDRLzPTroxNganc+TDsaz6CmgXQ5sJGaIiDmGdpbGzP
evDa5t3GovDNWiyZDog=
-----END CERTIFICATE REQUEST-----
```

## 使用CA证书及CA密钥 对请求签发证书进行签发，生成 x509证书
```bash
$ openssl x509 -req -in server.csr -CA ca.pem -CAkey cakey.pem -CAcreateserial -out server.pem -days 3650
$ cat server.pem
-----BEGIN CERTIFICATE-----
MIIDmjCCAoICCQCO6Ai3pDkDkjANBgkqhkiG9w0BAQsFADCBiDELMAkGA1UEBhMC
Q04xEjAQBgNVBAgMCUdVQU5HRE9ORzERMA8GA1UEBwwIU0hFTlpIRU4xDzANBgNV
BAoMBklQWUtFUjELMAkGA1UECwwCSVQxFzAVBgNVBAMMDnNzbC5pcHlrZXIuY29t
MRswGQYJKoZIhvcNAQkBFgxweWtlckBxcS5jb20wHhcNMTkwNjI2MDIyNDM0WhcN
MjkwNjIzMDIyNDM0WjCBlDELMAkGA1UEBhMCQ04xEjAQBgNVBAgMCUd1YW5nRG9u
ZzERMA8GA1UEBwwIU2hlblpoZW4xGzAZBgNVBAoMEsOlwoXCrMOlwo/CuMOlwpDC
jTELMAkGA1UECwwCaXQxFzAVBgNVBAMMDnRscy5pcHlrZXIuY29tMRswGQYJKoZI
hvcNAQkBFgxweWtlckBxcS5jb20wggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEK
AoIBAQDGbvCmdHFwF5fEI49P0CsFcIjvCFX8esLNQyX7ldEAVXBaW/JHfeNZv2ko
C6TsXLrywaKhqzK4IOf15tLd3pQGEA+9N9JtpZIjNNztzby2shWZqCWxDa+k1qZF
tQ7TgX4LrytSDtjJHZgHR0zPZkjBTI8SBETfhTxIFWE75dGQrejhaSXeqL32Rs1x
OJmBlcJq0eNvljeCnZ+VPU9Ujnge8WwEZ8bMgd35XuvmPZyZiP0HAjg6+8BI1icc
Pgu9utPoF8sZ4+npYkkWUht7hmw03pWGv3xu5XE2nwdnuUTlpKwcOYv90YD3kVvP
5sHlMvimErLSV7jRWF394GyRBhItAgMBAAEwDQYJKoZIhvcNAQELBQADggEBAJkl
54O5RpxaIfDtdT+LOxB2iwLe36XybJV0EvBffZH9vyK0Xg/OBLrMHAnZ8B5Y1u+t
P9nn5QtM+pnBXBt+qtO45nUG1bjGZQPHpvC68tG6eB+iw98u72cuMD6wbPE+kfaQ
Q3VS+xbyDvdRwscmOQjDLMfR0mLVM1WTWOW7IaII75IIJuHBTx7eSvhERtyVFP1g
j7FaJMmMhK0vCrG4awofpzdlV2bTGp7oD0XaEAxYb5pF+x3e5GzOZXFR/1gGezC8
ubyFuyUkifpg4015e+Rnv7KSIRWaegT4fGSZ0eMMNaSn+8WUIhOvnZtptZCeL9Rp
f0vimC2Ad3IyRt8GmsQ=
-----END CERTIFICATE-----
```

查看当前生成的CA、客户端证书私钥、证书请求以及CA证书。
```bash
$ ls -l
-rw-r--r-- 1 root root 1050 Jun 26 10:21 ca.csr
-rw-r--r-- 1 root root 1679 Jun 26 10:21 cakey.pem
-rw-r--r-- 1 root root 1294 Jun 26 10:21 ca.pem
-rw-r--r-- 1 root root   17 Jun 26 10:24 ca.srl
-rw-r--r-- 1 root root 1066 Jun 26 10:22 server.csr
-rw-r--r-- 1 root root 1679 Jun 26 10:22 serverkey.pem
-rw-r--r-- 1 root root 1310 Jun 26 10:24 server.pem
```