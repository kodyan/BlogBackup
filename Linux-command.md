title: Linux命令笔记
date: 2015/9/07 17:02:16 
categories: 
tags: Linux
---
## 打印文件的第n行 ##
    sed -n -e 'np' filename
## 对含有形如key,value的数据按value排序 ##
将排序结果写回原文件：

    sort -t "," -n -r -k2 filename -o filename
## 统计当前目录下Java文件行数 ##
    find . -name "*.java" |xargs cat|grep -v ^$|wc -l
## 打印内容中包含某字符串的文件名 ##
例如打印包含`bitmap size exceeds VM budget`的文件的文件名

    find .|xargs grep -ri "bitmap size exceeds VM budget" -l

<!--more-->
## 提取逗号分隔的某一列 ##
例如*file*的内容如下：
> a,2
> b,1
> c,3
> d,4

提取出第一列：
>a
>b
>c
>d

则执行：

    cut -d',' -f1 file

## 按每行字符串长度排序 ##
    cat file | while read i;do echo ${#i},$i;done | sort -nr|cut -f2 -d','