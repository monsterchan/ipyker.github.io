layout: post
title: Shell 特殊变量和字符串截取
author: Pyker
categories: shell
tags:
  - shell
date: 2018-03-11 12:01:00
---

# Shell 特殊变量

| ID   |         表达式          |                    含义                    |          举例          |          结果          |
| :--: | :--------------------: | :----------------------------------------: | :--------------------: | :--------------------: |
| 1    |          $@             |           依次列出脚本参数的值              |    sh test.sh a b c d  |     "a" "b" "c" "d"    |
| 2    |          $#             |           传递到脚本的参数个数              |    sh test.sh a b c d  |           4            |
| 3    |          $*             |     以一个单字符串显示所有向脚本传递的参数   |    sh test.sh a b c d   |        "a b c d"       |
| 4    |          $!             |            后台运行的最后一个进程的ID号     |    sh t.sh $!          |          15797         |
| 5    |          $-             |   显示Shell使用的当前选项，与set命令功能相同 |    sh t.sh && echo $-   |        himBH          |
| 6    |          $?             | 显示命令的退出状态。0表示没有错误，其他为有错误 |   sh t.sh && echo $?  |           0            |
| 7    |          $$             |         脚本运行的当前进程ID号              |       sh t.sh $$       |        12738           |

# Shell字符串截取
以下举例varible的值等于my_name_is_pyker。 varible=my_name_is_pyker

|  ID  |         表达式          |                    含义                    |          举例          |          结果          |
| :--: | :--------------------: | :----------------------------------------: | :--------------------: | :--------------------: |
| 1 | ${varible##*string} | 从左向右截取最后一个string后的字符串 | echo ${varible##*_} |  pyker |
| 2 | ${varible#*string} | 从左向右截取第一个string后的字符串 | echo ${varible#*_} | name_is_pyker |
| 3 | ${varible%%string*} | 从右向左截取最后一个string后的字符串 | echo ${varible%%_*} | my |
| 4 | ${varible%string*} | 从右向左截取第一个string后的字符串 | echo ${varible%_*} | my_name_is |
| 5 | ${varibleg:position} | 在$varible中, 从位置$position开始提取子串 | echo ${varible:5} | me_is_pyker |
| 6 | ${varible:position:length} | 在$varible中, 从位置$position开始提取长度为$length的子串 | echo ${varible:5:4} | me_i |
| 7 | ${varible#substring} | 从变量$varible的开头, 删除最短匹配$substring的子串 | echo ${varible#my_} | name_is_pyker |
| 8 | ${varible##substring} | 从变量$varible的开头, 删除最长匹配$substring的子串 | echo ${varible##my_} | name_is_pyker |
| 9 | ${varible%substring} | 从变量$varible的结尾, 删除最短匹配$substring的子串 | echo ${varible%pyker} | my_name_is_ |
| 10 | ${varible%%substring} | 从变量$varible的结尾, 删除最长匹配$substring的子串 | echo ${varible%%pyker} | my_name_is_ |
| 11 | ${varible/substring/replacement} | 使用$replacement, 来代替第一个匹配的$substring | echo ${varible/_/-} | my-name_is_pyker |
| 12 | ${varible//substring/replacement} | 使用$replacement, 代替所有匹配的$substring | echo ${varible//_/-} | my-name-is-pyker |
| 13 | ${varible/#substring/replacement} | 如果$varible的前缀匹配$substring, 则$replacement替换$substring | echo ${varible/#my_/the-} | the-name_is_pyker |
| 14 | ${varible/%substring/replacement} | 如果$varible的后缀匹配$substring, 则$replacement替换$substring | echo ${varible/%pyker/leo} | my_name_is_leo |

# Shell 特殊变量赋值
以下举例var的值等于demo，var=demo。 var1没有定义。

|  ID  |         表达式          |                    含义                    |          举例          |          结果          |
| :--: | :--------------------: | :----------------------------------------: | :--------------------: | :--------------------: |
| 1 | ${var:-DEFAULT} | 如果var没有被声明, 或者其值为空, 那么就以$DEFAULT作为其值 | echo ${var:-value}<Br>echo ${var1:-value} | demo<Br>value |
| 2 | ${var:+OTHER} | 如果var被设置了, 那么其值就是$OTHER, 否则就为null字符串 | echo ${var:+value}<Br>echo ${var1:+value} | value<Br> " " |
| 3 | ${var:?ERR_MSG} | 如果var没被声明, 那么就打印$ERR_MSG | echo ${var:?ERR_MSG}<Br>echo ${var1:?ERR_MSG} | demo<Br>-bash: var1: ERR_MSG |
| 4 | ${!var*} | 匹配之前所有以var开头进行声明的变量 | echo ${!var*} | var varible |
