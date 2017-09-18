---
title: "IO密集型应用优化策略"
layout: single
categories: Network
tags: Linux Resin Nginx TCP
---
最近刚刚推广的一个应用，在使用 JMeter 压测过程中出现了性能问题，具体表现为3000并发下 Load\-average 和 CPU\-Utilization 都不高，JVM内存也没有明显异常，压测结果中有2\-5%的请求超过10秒。

通过对业务的了解，发现这是一个典型的IO密集型应用（对Redis的只读操作）。前端为Nginx，后端用Resin作为WebServer，代码中配置了连接池（最大连接数为1000，最大空闲连接为500），Redis服务器中查看的来自这个服务的连接数与连接池配置相同，且远没有达到Redis连接数的上限。netstat -s | grep -i listen 观察到压测期间，队列长度一直在大量增常，很多请求没有被正常处理。

从 Nginx、Resin、JVM、Linux 4个方面下手，对涉及到网络和进程数量的部分参数进行调整，以提高处理能力增加队列长度，降低超时率。

## Nginx

```ini
# 设置为与CPU核心数量一致，before 2
worker_processes 8;
# 增加单进程最大并发连接，before 1024
worker_connections 65535;
```

## Resin

```ini
# before 4000
accept-listen-backlog=8192
# before no set
accept-thread-min=256
# before 10
accept-thread-max=512
# defore 100
keepalive-max=256
# before 1024
thread-max=2048
# defore 20
thread-idle-max=512
# defore 10
thread-idle-min=256
```

## JVM

```ini
# 没有复杂的递归操作，留出更多的内存创建子线程 defore 512k
xss=128k
```

## Linux

```ini
# /etc/sysctl.conf
# 未完成三次握手的连接队列长度
net.ipv4.tcp_max_syn_backlog=8192
# TIME-WAIT 连接重用
net.ipv4.tcp_tw_reuse=1
# 已完成三次握手等待的队列长度
net.core.somaxconn=20480
# 网卡队列长度
net.core.netdev_max_backlog=51200
```

复测超时问题消失，负载和cpu使用率明显上升。