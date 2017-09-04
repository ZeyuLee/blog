---
title: "Linux常用命令"
layout: single
categories: Linux
tags: Bash Shell
---

> Linux中非常常用的命令，工作中99%会用到，熟练使用事半功倍
> 
> 参考文献里的几个文章，对于想深入学习Linux的开发人员推荐阅读

## grep ##

```bash
#忽略大小写，显示匹配到内容的及前后各5行内容，并附带行号
grep -in 'findme' --context=5 file

#显示没有匹配到内容的行
grep -v 'findme' file

#在当前目录中递归查询，没有匹配到内容的文件名
grep -r -L 'findme' ./*

#支持gzip文件的匹配
grep -Z -C 5 -i 'findme' file.gz
```

## find ##

```bash
#在给定目录中，寻找访问时间在最近30分钟内的文件，并查看其文件信息
find path -atime -30m -type f -ls

#在给定目录的最大深度为2的目录结构中，寻找创建时间在1天以前的文件
find path -maxdepth 2 -ctime +1d -type f

#当前目录下，寻找空文件并删除
find ./ -empty -delete

#从给定目录中，按文件名寻找，并执行给定命令
find path -name 'file' -exec md5 {} \;

#寻找比给定文件修改时间新的链接
find path -newer file -type l

#寻找1MB以下文件
find path -size -1M -type f

#删除所有失效的链接
find -L path -type l -exec rm -- {} +

```

## sed ##

```bash
#打印被替换的行
sed -n 's/test/TEST/p' file

#显示从第一个表达式到第二个表达式之间的行
sed -n '/^fls/,/^dd/p' file

#删除空行
sed '/^$/d' file

#删除第7行到最后一行
sed '7,$d' file

#替换以fml开头的行为fmlgo
sed 's/^fml/&go/' file
sed 's/^\(fml\)/\1go/' file

#删除第1到第3行，并进行替换，在结尾附加file2
sed -e '1,3d' -e 's/^fml/lmf/' -e '$r file2' file1

#输出奇数行
sed -n 'p;n' file

#输出偶数行
sed -n 'n;p' file
```

## awk ##

```bash
#平均QPS
awk '{sec=substr($4,2,20);reqs++;reqsBySec[sec]++;} END{print reqs/length(reqsBySec)}' nginx.log

#最大QPS
awk '{sec=substr($4,2,20);reqsBySec[sec]++;} END{for(s in reqsBySec){printf("%s %s\n", reqsBySec[s], s)}}' nginx.log | sort -nr | head -n 3

#单个url最大体积
awk '{url=$7;size[url]+=$10} END{for(s in size){printf("%s %s\n",size[s],s)}}' nginx.log | sort -nr | head -n 5

#带宽
awk '{sec=substr($4,2,20);size[sec]+=$10} END{for(s in size){printf("%sKB %s\n",size[s]/1024,s)}}' nginx.log | sort -nr | head -n 5
```

## xargs ##

```bash
#批量杀进程
ps aux | grep php-fpm | grep -v 'grep' | awk '{print $2}' | xargs kill -9

#替换指定位置
less filenames.txt | axrgs -I {} cp {} path
```

## 参考文献 ##

[Linux命令大全](http://man.linuxde.net/)

[linuxcommand](http://linuxcommand.org/)

[50 Most Frequently Used UNIX / Linux Commands](http://www.thegeekstuff.com/2010/11/50-linux-commands/)