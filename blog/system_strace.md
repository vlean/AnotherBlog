<!--
author: vlean
date: 2016-07-11
title: strace等进程执行监测工具
tags: strace,系统工具
category: tools
status: publish
summary: 进程执行堆栈监测
-->

## 0x00  strace
strace是Linux环境下的一款程序调试工具，用来监察一个应用程序所使用的系统调用及它所接收的系统信息。追踪程序运行时的整个生命周期，输出每一个系统调用的名字，参数，返回值和执行消耗的时间等。
### 常用参数：
-p 跟踪指定的进程
-f 跟踪由fork子进程系统调用
-F 尝试跟踪vfork子进程系统调吸入，与-f同时出现时, vfork不被跟踪
-o filename 默认strace将结果输出到stdout。通过-o可以将输出写入到filename文件中
-ff 常与-o选项一起使用，不同进程(子进程)产生的系统调用输出到filename.PID文件
-r 打印每一个系统调用的相对时间
-t 在输出中的每一行前加上时间信息。 -tt 时间确定到微秒级。还可以使用-ttt打印相对时间
-v 输出所有系统调用。默认情况下，一些频繁调用的系统调用不会输出
-s 指定每一行输出字符串的长度,默认是32。文件名一直全部输出
-c 统计每种系统调用所执行的时间，调用次数，出错次数。
-e expr 输出过滤器，通过表达式，可以过滤出掉你不想要输出

### 追踪多个进程方法
当有多个子进程的情况下，比如php-fpm、nginx等，用strace追踪显得很不方便。可以使用下面的方法来追踪所有的子进程。
```bash
# vim /root/.bashrc //添加以下内容
function straceall {
    strace $(pidof "${1}" | sed 's/\([0-9]*\)/-p \1/g')
}
# source /root/.bashrc
# traceall php-fpm //监控phpfpm
OR
# strace -tt -T $(pidof 'php-fpm: pool www' | sed 's/\([0-9]*\)/\-p \1/g')
```
### 追踪web服务
```bash
# strace -f -F -s 1024 -o nginx-strace /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
# strace -f -F -o php-fpm-strace /usr/local/php/sbin/php-fpm -y /usr/local/php/etc/php-fpm.conf
```
### 追踪mysql
```bash
# strace -f -F -ff -o mysqld-strace -s 1024 -p mysql_pid
# find ./ -name "mysqld-strace*" -type f -print |xargs grep -n "SELECT.*FROM"
```
### 查看程序做了什么
```bash
#!/bin/bash
# This script is from http://poormansprofiler.org/
nsamples=1
sleeptime=0
pid=$(pidof $1)
 
for x in $(seq 1 $nsamples)
  do
    gdb -ex "set pagination 0" -ex "thread apply all bt" -batch -p $pid
    sleep $sleeptime
  done | \
awk '
  BEGIN { s = ""; } 
  /^Thread/ { print s; s = ""; } 
  /^\#/ { if (s != "" ) { s = s "," $4} else { s = $4 } } 
  END { print s }' | \
sort | uniq -c | sort -r -n -k 1,1
```

## 0x01 ltrace

-a 对齐具体某个列的返回值
-c 计算时间和调用，并在程序退出时打印摘要
-C 解码低级别名称（内核级）为用户级名称 
-d 打印调试信息
-e 改变跟踪的事件
-f 跟踪子进 
-h 打印帮助信息
-i 打印指令指针，当库调用时。 
-l 只打印某个库中的调用。 
-L 不打印库调用。 
-n, --indent=NR 对每个调用级别嵌套以NR个空格进行缩进输出。 
-o, --output=file 把输出定向到文件。 
-p PID 附着在值为PID的进程号上进行ltrace。 
-r 打印相对时间戳。 
-s STRLEN 设置打印的字符串最大长度。 
-S 显示系统调用。 
-t, -tt, -ttt 打印绝对时间戳。 
-T 输出每个调用过程的时间开销。 
-u USERNAME 使用某个用户id或组ID来运行命令。 
-V, --version 打印版本信息，然后退出。 
-x NAME treat the global NAME like a library subroutine.

## 0x02 phpstrace
[phpstrace](https://github.com/markus-perl/php-strace)追踪php进程

```
Usage: ./php-strace [ options ]
-h|--help               show this help
-l|--lines <integer>    output the last N lines of a stacktrace. Default: 100
--process-name <string> name of running php processes. Default: autodetect
--live                  search while running for new upcoming pid's
```

## 0xff 参考
> [使用truss/strace/ltrace跟踪进程](https://www.ibm.com/developerworks/cn/linux/l-tsl/)

