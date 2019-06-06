layout: post
title: shell脚本echo颜色
author: Pyker
categories: shell
tags:
  - shell
  - color
date: 2018-01-21 13:21:00
---

# echo显示颜色
## 格式
echo 显示内容颜色，需要使用 -e 参数
`-e` :打开反斜杠转义 (默认不打开) ,可以转义 “\n, \t” 等
`-n` :在最后不自动换行

## 字体颜色列表
```
#字体颜色：30m-37m 黑、红、绿、黄、蓝、紫、青、白
str="=====Hello Word====="
echo -e "\033[30m ${str}\033[0m"      ## 黑色字体
echo -e "\033[31m ${str}\033[0m"      ## 红色
echo -e "\033[32m ${str}\033[0m"      ## 绿色
echo -e "\033[33m ${str}\033[0m"      ## 黄色
echo -e "\033[34m ${str}\033[0m"      ## 蓝色
echo -e "\033[35m ${str}\033[0m"      ## 紫色
echo -e "\033[36m ${str}\033[0m"      ## 青色
echo -e "\033[37m ${str}\033[0m"      ## 白色
```

## 背景颜色列表
```
#背景颜色：40-47 黑、红、绿、黄、蓝、紫、青、白
str="=====Hello Word====="
echo -e "\033[41;37m ${str} \033[0m"     ## 红色背景色，白色字体
echo -e "\033[41;33m ${str} \033[0m"     ## 红底黄字
echo -e "\033[1;41;33m ${str} \033[0m"   ## 红底黄字 高亮加粗显示
echo -e "\033[5;41;33m ${str} \033[0m"   ## 红底黄字 字体闪烁显示
echo -e "\033[47;30m ${str} \033[0m"     ## 白底黑字
echo -e "\033[40;37m ${str} \033[0m"     ## 黑底白字
```
* 其他参数说明
`\033[1;m `: 设置高亮加粗
`\033[3;m`:  斜体 
`\033[4;m`:  下划线 
`\033[5;m`:  闪烁

## 颜色在脚本中使用
```bash
#!/bin/sh
STR='=====Hello Word====='
RES='\033[0m'

# 正常红黄绿
Normal_Color() {
	local RED="\e[31m"
	local YELLOW="\e[33m"
	local GREEN="\e[32m"
	echo -e "${RED}$STR${RES}"
	echo -e "${YELLOW}${STR}${RES}"
	echo -e "${GREEN}${STR}${RES}"
}
Normal_Color
echo 
# 高亮红黄绿
Light_Color() {
	local RED="\e[1;31m"
	local YELLOW="\e[41;1;33m"
	local GREEN="\e[1;32m"
	echo -e "${RED}$STR${RES}"
	echo -e "${YELLOW}${STR}${RES}"
	echo -e "${GREEN}${STR}${RES}"
}
Light_Color
echo 
# 高亮闪烁红黄绿
Light_Blink_Color() {
	local RED="\e[5;1;31m"
	local YELLOW="\e[5;1;33m"
	local GREEN="\e[5;1;32m"
	echo -e "${RED}$STR${RES}"
	echo -e "${YELLOW}${STR}${RES}"
	echo -e "${GREEN}${STR}${RES}"
}
Light_Blink_Color
echo 
# 有底色闪烁红黄绿
Under_Blink_Color() {
	local RED="\e[5;47;31m"
	local YELLOW="\e[5;41;33m"
	local GREEN="\e[5;45;32m"
	echo -e "${RED}$STR${RES}"
	echo -e "${YELLOW}${STR}${RES}"
	echo -e "${GREEN}${STR}${RES}"
}
Under_Blink_Color
```
效果如下：
![](/images/pic/echo.gif)