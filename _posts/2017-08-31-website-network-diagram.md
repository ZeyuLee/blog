---
title: "站点网络架构示意图"
layout: single
categories: Training
tags: network
---
日常工作中经常有开发的小伙伴不清楚运维嘴里的常用术语，导致反馈问题表述不准确，判断网络故障也没有一个清晰的思路，有事儿没事儿就甩给运维。作为一个经常跟运维混迹在一起的coder，实在忍受不了运维的吐槽，今天给公司小伙伴从运维的视角讲了讲网站的络架构，下面是用到的示意图（Drew By [ProcessOn](https://www.processon.com)）

主要的知识点：

* 基础：[OSI模型](https://zh.wikipedia.org/wiki/OSI%E6%A8%A1%E5%9E%8B)
* 数据链接层：[ARP](https://zh.wikipedia.org/wiki/%E5%9C%B0%E5%9D%80%E8%A7%A3%E6%9E%90%E5%8D%8F%E8%AE%AE)
* 网络层：[IP](https://zh.wikipedia.org/wiki/%E7%BD%91%E9%99%85%E5%8D%8F%E8%AE%AE)
* 传输层：[TCP](https://zh.wikipedia.org/wiki/%E4%BC%A0%E8%BE%93%E6%8E%A7%E5%88%B6%E5%8D%8F%E8%AE%AE) & [UDP](https://zh.wikipedia.org/wiki/%E7%94%A8%E6%88%B7%E6%95%B0%E6%8D%AE%E6%8A%A5%E5%8D%8F%E8%AE%AE)
* 应用层：[HTTP](https://zh.wikipedia.org/wiki/%E8%B6%85%E6%96%87%E6%9C%AC%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE) & [DNS](https://zh.wikipedia.org/wiki/%E5%9F%9F%E5%90%8D%E7%B3%BB%E7%BB%9F)

![网络架构图](http://ot41apokn.bkt.clouddn.com/network.png)
