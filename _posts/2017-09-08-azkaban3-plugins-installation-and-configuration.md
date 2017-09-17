---
title: "Azkaban 3插件的安装与配置"
layout: single
categories: Bigdata
tags: azkaban plugins schedule workflow
---
Azkaban的插件分为2类，一种是web-server用的报表、展示类插件（viwer），一种是exec-server用的工作流类型插件。

# 编译

```bash
git clone https://github.com/azkaban/azkaban-plugins.git

# 全部编译
cd azkaban-plugins
ant
cp ./dist/*/packages/*.tar.gz /plugins/path

# 单独编译
cd azkaban-plugins/jobtype
ant
```

# 安装

## web-server 插件

目前有以下几个：

* hdfsviewer
* javaviewer
* jobsummary
* reportal
* pigvisualizer

安装比较简单，其它解压到plugins/viewer既可（hdfsviewer需要改名为hdfs，reportal需要用viewer目录内的内容），最终结构如下

```base
tree -L 2 ./plugins/viewer
./plugins/viewer
├── hdfs
│   ├── conf
│   ├── extlib
│   ├── lib
│   └── package.version
├── javaviewer
│   ├── conf
│   ├── extlib
│   ├── lib
│   ├── package.version
│   └── web
├── jobsummary
│   ├── conf
│   ├── extlib
│   ├── lib
│   ├── package.version
│   └── web
├── pigvisualizer
│   ├── conf
│   ├── extlib
│   ├── lib
│   ├── package.version
│   └── web
└── reportal
    ├── conf
    ├── lib
    └── web
```

如果Hadoop启用了安全验证，注意设置插件中conf目录下的配置文件到对应的版本，并复制对应的jar包到对应插件的lib里面，找不到就去其它插件里面找找。

```ini
hadoop.security.manager.class=azkaban.security.HadoopSecurityManager_H_1_0

# hadoop.security.manager.class=azkaban.security.HadoopSecurityManager_H_2_0
# azkaban-hadoopsecuritymanager-2.2.jar

# hadoop.security.manager.class=azkaban.security.HadoopSecurityManager_H_3_0
# azkaban-hadoopsecuritymanager-3.0.0.jar
```

HDFS 和 Reportal 在头部可见，其它几个在几种特定的jobtypes结果中可见。

![azkaban-plugins-install](http://ot41apokn.bkt.clouddn.com/azkaban-plugins-install.png)

## exec-server 插件

主要配置的是jobtypes，具体配置情况如下

```ini
# jobtypes/common.properties
# jobtypes/commonprivate.properties
hadoop.home=/hadoop/home/path
hive.home=/hadoop/hive/path
pig.home=/hadoop/pig/path
spark.home=/hadoop/spark/path

# jobtypes/hadoopJava 默认值

# jobtypes/hive/private.properties
jobtype.class=azkaban.jobtype.HadoopHiveJob
hive.aux.jar.path=${hive.home}/aux/lib
jobtype.classpath=${hadoop.home}/conf,${hadoop.home}/lib/*,${hive.home}/lib/*,${hive.home}/conf,${hive.aux.jar.path}

# jobtypes/java 默认值

# pig使用对应版本，高于0.12使用0.12
# jobtypes/pig-0.12.0/plugin.properties
pig.listener.visualizer=false
jobtype.classpath=${pig.home}/lib/*,${pig.home}/*

# jobtypes/pig-0.12.0/private.properties
jobtype.class=azkaban.jobtype.HadoopPigJob
jobtype.classpath=${hadoop.home}/conf,${hadoop.home}/lib/*,lib/*

# jobtypes/saprk 里少azkaban-jobtype-3.0.0.jar，从别的插件里copy过来
# jobtypes/spark/private.properties
jobtype.class=azkaban.jobtype.HadoopSparkJob
hadoop.classpath=${hadoop.home}/lib
jobtype.classpath=${hadoop.classpath}:${spark.home}/conf:${spark.home}/jars/*
```

reportal 插件里面目录和配置文件，请合并到 jobtypes 内，具体配置文件参照 jobtypes 里对应的插件。