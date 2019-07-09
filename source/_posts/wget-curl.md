layout: post
title: 常用的curl和wget命令
author: Pyker
categories: shell
tags:
  - shell
date: 2018-03-21 09:18:00
---

# 常用的curl命令
```
 -o 将请求地址的输出保持到文件
 -i 显示请求头和响应内容
 -I 只显示请求头
 -L 响应显示重定向跳转后的内容
 -w 格式输出，如%{time_total}, %{http_code}等
 -s 静默输出，不显示进度
 -# 显示下载进度
 -X 指定GET、PUT、DELETE、POST等请求指令
 -d 传入数据，默认配合-X POST指令
 -u 输入服务的用户名和密码，冒号隔开
 -U 使用http反向代理的地址请求网页
 -v 显示请求头、响应头以及响应内容信息
 -c 访问页面后保存页面的cookies
 -b 从文件中读取cookies访问页面
 -T 上传文件到目标地址，比如FTP
 -r 指定范围分块下载，如 -r 0-100 -o ，-r 101-200 -o，-r 200 - -o
 -k 允许在没有证书的情况下连接SSL站点，不对服务器的证书进行检查
 -E/--cert 使用客户端证书来发起请求
```

# 常用的wget命令
```
 -O 指定下载的文件名
 --limit-rate 限制下载速度，如--limit-rate=300K
 -c 下载断点续传
 -b 后台下载，使用tail -f wget-log 查看下载进度
 -q 安静模式
 -v 显示详细的信息
 -nv 关闭详尽输出，但不进入安静模式
 -nc 不要重复下载已经存在的文件
 --spider 测试远程文件是否存在
 -t/--tries 设置下载重试的次数，默认0无限次
 --http-user=USER 设置http用户名为USER
 --http-password=PASS    设置tttp密码为PASS
 --proxy-user=USER       使用USER作为代理用户名
 --proxy-password=PASS   使用PASS作为代理密码
 --post-data=STRING      使用POST方式把STRING作为数据发送
 --post-file=FILE        使用POST方式发送FILE里的内容
 -O - -q 标准输出请求内容
 -O- 不保存标准输出内容，直接用管道符运行标准输出
```