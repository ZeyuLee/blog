---
title: "Azkaban 3的安装与配置"
layout: single
categories: Bigdata
tags: azkaban schedule workflow
---
本次Azkaban以3.34源代码为基础进行编译安装 [Github地址](https://github.com/azkaban/azkaban/tree/3.34.0)，编译时需要使用gradle，需要Java 8及以上版本。

# 编译

```bash
git clone https://github.com/azkaban/azkaban.git

# 编译
./gradlew build

# 清理并编译（上次编译失败时执行）
./gradlew clean

# 跳过测试进行编译（会快一些）
./gradlew build -x test
```

gradlew 下载比较慢，可以从 gradle/wrapper/gradle-wrapper.properties 里面找到下载地址，提前下载好，执行一次编译命令并中断后，放在 $HOME/.gradle/wrapper/dists/gradle-{version}-all/{folder}

```bash
cp ./*/build/distributions/*.tar.gz /install/path
```

# 配置

## 单机模式

```bash
# 解压
tar -zxvf azkaban-solo-server-*-*-*.tar.gz

# 目录结构如下
#
# bin      执行脚本
# conf     配置文件
# lib      依赖库，自带了mysql-connector不需要下载了
# plugins  插件
# projects 执行job时，任务存放目录（系统会自动创建，请无视）
# web      前台页面
# logs     日志（需要新建，为了方便管理日志）
```

### 主配置文件 azkaban.properties

```ini
# Azkaban 个性化设置，除了时区和web资源路径需要注意，其它随意
azkaban.name=Test
azkaban.label=My Local Azkaban
azkaban.color=#FF3601
azkaban.default.servlet.path=/index
web.resource.dir=web/
default.timezone.id=Asia/Shanghai

# Azkaban 用户管理设置，默认值
user.manager.class=azkaban.user.XmlUserManager
user.manager.xml.file=conf/azkaban-users.xml

# Azkaban 项目的设置，默认值
# 所有job参数的父类，覆盖关系见下表
executor.global.properties=conf/global.properties
# 执行job时任务存放目录
azkaban.project.dir=projects
# 数据库配置
database.type=h2
h2.path=./h2
h2.create.tables=true

# Velocity dev mode，官网里没介绍这个参数，也没去研究，默认值
velocity.dev.mode=false

# Azkaban 服务器设置，默认值
jetty.use.ssl=false
jetty.maxThreads=25
jetty.port=8081

# Azkaban 执行服务器端口，默认值
executor.port=12321

# 邮件设置（【坑1】由于需要支持tls并修改port，这里折腾了很久）
# port和tls官方文档里虽然没说，源码里面是支持的，25和false是默认值
mail.sender=
mail.host=
mail.port=25
mail.tls=false
mail.user=
mail.password=

# 任务通知邮件（【坑2】官方文档没有一点儿说明，不看源码肯定不知道）
# 这里配置了也没用，看了源码，至少在job里面设置了才能收到邮件
# job.failure.email=
# job.success.email=
# job.notify.email=

# 非 admin 账户对项目的创建及job上传权限，默认值
lockdown.create.projects=false
lockdown.upload.projects=false

# 缓存目录
cache.directory=cache

# JMX stats，默认值
jetty.connector.stats=true
executor.connector.stats=true

# Azkaban jobtypes 插件设置，默认值
# 涉及 executor 的插件配置起来都比较麻烦，后面有一篇博客专门说插件配置
azkaban.jobtype.plugin.dir=plugins/jobtypes
```

### 参数覆盖关系

| 参数来源 | 描述 | 优先级 |
| ---- | ---- | ---- |
| global.properties（conf） | 所有job类型 | 0（最低） |
| common.properties（jobtype） | 所有job类型 | 1 |
| plugin.properties（jobtype/{jobtype-name}） | 某一种job类型 | 2 |
| common.properties（job zip file） | 某一个job所有环节 | 3 |
| 工作流属性 | 从UI或Ajax执行的job | 4 |
| {job-name}.job | 某一个job内的配置 | 5 |

### 用户配置文件 azkaban-users.xml

```xml
<!--权限分 READ,WRITE,EXECUTE,SCHEDULE,CREATEPROJECTS,ADMIN 一级比一级高-->
<!--某个人的权限 = 角色 + 组，下面是我测试时配置的-->
<azkaban-users>
  <user groups="azkaban" password="azkaban" username="azkaban"/>
  <user groups="metrics" roles="admin" password="metrics" username="metrics"/>
  <user groups="group_leader" password="group_leader" username="group_leader"/>
  <user groups="group_inspector" password="group_inspector" username="group_inspector"/>

  <group name="azkaban" roles="admin"/>
  <group name="metrics" roles="metrics"/>
  <group name="group_leader" roles="leader"/>
  <group name="group_inspector" roles="inspector"/>
  <group name="group_schedule" roles="schedule"/>

  <role name="admin" permissions="ADMIN"/>
  <role name="metrics" permissions="METRICS"/>
  <role name="leader" permissions="READ,WRITE,EXECUTE,SCHEDULE,CREATEPROJECTS"/>
  <role name="inspector" permissions="READ"/>
  <role name="write" permissions="WRITE"/>
  <role name="execute" permissions="EXECUTE"/>
  <role name="schedule" permissions="SCHEDULE"/>
  <role name="createprojects" permissions="CREATEPROJECTS"/>
</azkaban-users>
```

./conf/global.properties 和 ./plugins/jobtypes/commonprivate.properties 使用默认值即可。需要对 execute.as.user 进行修改的请参照官网 [Plugin Configurations](http://azkaban.github.io/azkaban/docs/latest/#azkaban-plugin-configuration)

### 启动

```bash
# 启动
./bin/azkaban-solo-start.sh
# 关闭
./bin/azkaban-solo-shutdown.sh
```

看不到 [UI界面](http://localhost:8081)，请看logs里的报错

## 多机模式

```bash
# web-server
tar -zxvf azkaban-web-server-*-*-*.tar.gz
# executor-server
tar -zxvf azkaban-executor-server-*-*-*.tar.gz
# 建表语句
tar -zxvf azkaban-db-*-*-*.tar.gz

# 目录结构同单机基本相同，executor不需要web目录
```

### Mysql数据库

```sql
# Mysql数据库初始化
source create-all-sql.sql

# executor 在启动时会向executors表插入记录，但默认是不启用，需要修改
# 【坑3】如果这里不改，每次 executor 启动都要手动设置为1，非常不便
ALTER TABLE `executors` CHANGE `active` `active` TINYINT DEFAULT 1;
```

### web-server 主配置文件 azkaban.properties

```ini
# 仅列出于单机模式的不同
# 数据库配置
database.type=mysql
mysql.port=3306
mysql.host=localhost
mysql.database=azkaban
mysql.user=
mysql.password=
mysql.numconnections=100

# 多executor调度规则设置
azkaban.use.multiple.executors=true
azkaban.executorselector.filters=StaticRemainingFlowSize,MinimumFreeMemory,CpuStatus
azkaban.executorselector.comparator.NumberOfAssignedFlowComparator=1
azkaban.executorselector.comparator.Memory=1
azkaban.executorselector.comparator.LastDispatched=1
azkaban.executorselector.comparator.CpuUsage=1
```

### executor-server 主配置文件 azkaban.properties

```ini
# 相比web-server配置很简单
# 数据库配置
database.type=mysql
mysql.port=3306
mysql.host=localhost
mysql.database=azkaban
mysql.user=
mysql.password=
mysql.numconnections=100

# 下面都是默认值
# Azkaban Executor settings
executor.maxThreads=50
executor.port=12321
executor.flow.threads=30

# Azkaban JobTypes Plugins
azkaban.jobtype.plugin.dir=plugins/jobtypes

# Azkaban the parent for all jobs
executor.global.properties=conf/global.properties
```

### 启动

```bash
# 必须先启动executor-server，否则web-server会报找不到executor的错误
./bin/start-exec.sh
./bin/start-web.sh
```

### job demo

```ini
# foo.job
type=command
command=echo "foo"

# bar.job
type=command
command=echo "bar"
dependencies=foo
```

打包成zip格式文件，新建项目后上传进行测试