---
title: "对Mysql索引类型的一次重新认识"
layout: single
categories: Mysql
tags: index btree hash
---

今天下午突然收到报警邮件，投票系统的Mysql数据库有大量慢查询，导致前台站点访问出现异常。按图索骥开始进行分析，先看后端站点日志，发现出问题时一个没有启用验证码的投票被刷了，出问题的表是实现防刷功能的内存表，数据量大概有十几万行。先在后台开启验证码，恢复前台站点访问。

按道理内存表再慢也比MyISAM要快，除了数据不能有效落地之外，没有其它明显的缺点了，应用在防刷功能上最合适了，怎么会出现大面积锁表的情况呢？随着分析的深入，发现自己忽略了一个重要的问题，索引类型对内存表也有至关重要的影响，特别是在QPS比较高的情况下。

内存表使用的索引默认为HASH，而一般的MyISAM默认则为BTREE。

由于类型为HASH的索引是以hash以后的值作为键，hash之前的大小关系和hash之后的结果没有必然联系，所以HASH索引仅仅能满足"="，"IN"和"<=>"查询，不能使用范围查询。简而言之就是与memcache一样的kv存储，不能通过范围查询也是理所当然的了。

进而可以想到，也有如下问题

* 无法对排序进行优化
* 不能利用部分索引查询
* 不能避免全表扫描

## 建表测试

```sql
CREATE TABLE `memory_test` (
  `id` int(10) unsigned NOT NULL DEFAULT '0',
  `firstname` varchar(10) NOT NULL DEFAULT '',
  `lastname` varchar(10) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  KEY `index` (`firstname`,`lastname`)
) ENGINE=MEMORY;

CREATE TABLE `myisam_test` (
  `id` int(10) unsigned NOT NULL DEFAULT '0',
  `firstname` varchar(10) NOT NULL DEFAULT '',
  `lastname` varchar(10) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  KEY `index` (`firstname`,`lastname`)
) ENGINE=MyISAM;
```

### 范围查询测试

```sql
EXPLAIN SELECT * FROM `memory_test` WHERE `id` > 5;
+----+-------------+-------------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table       | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-------------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | memory_test | ALL  | PRIMARY       | NULL | NULL    | NULL |    6 | Using where |
+----+-------------+-------------+------+---------------+------+---------+------+------+-------------+
#无法使用hash索引，对 memory_test 表进行了一次扫描行数为6的全表扫描

EXPLAIN SELECT * FROM `myisam_test` WHERE `id` > 5;
+----+-------------+-------------+-------+---------------+---------+---------+------+------+-------------+
| id | select_type | table       | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+----+-------------+-------------+-------+---------------+---------+---------+------+------+-------------+
|  1 | SIMPLE      | myisam_test | range | PRIMARY       | PRIMARY | 4       | NULL |    2 | Using where |
+----+-------------+-------------+-------+---------------+---------+---------+------+------+-------------+
#使用主键索引，对 myisam_test 表进行了一次扫描行数为2的范围查询
```

### 排序测试

```sql
EXPLAIN SELECT * FROM `memory_test` ORDER BY `id` DESC;
+----+-------------+-------------+------+---------------+------+---------+------+------+----------------+
| id | select_type | table       | type | possible_keys | key  | key_len | ref  | rows | Extra          |
+----+-------------+-------------+------+---------------+------+---------+------+------+----------------+
|  1 | SIMPLE      | memory_test | ALL  | NULL          | NULL | NULL    | NULL |    6 | Using filesort |
+----+-------------+-------------+------+---------------+------+---------+------+------+----------------+
#仍然无法使用hash索引，产生一次全表扫描

EXPLAIN SELECT * FROM `myisam_test` ORDER BY `id` DESC;
+----+-------------+-------------+-------+---------------+---------+---------+------+------+-------+
| id | select_type | table       | type  | possible_keys | key     | key_len | ref  | rows | Extra |
+----+-------------+-------------+-------+---------------+---------+---------+------+------+-------+
|  1 | SIMPLE      | myisam_test | index | NULL          | PRIMARY | 4       | NULL |    6 |       |
+----+-------------+-------------+-------+---------------+---------+---------+------+------+-------+
#使用主键索引
```

### 联合索引中左前缀测试

```sql
EXPLAIN SELECT * FROM `memory_test` WHERE `firstname` = 'firstname1';
+----+-------------+-------------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table       | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+-------------+------+---------------+------+---------+------+------+-------------+
|  1 | SIMPLE      | memory_test | ALL  | index         | NULL | NULL    | NULL |    6 | Using where |
+----+-------------+-------------+------+---------------+------+---------+------+------+-------------+
#无法使用联合索引

EXPLAIN SELECT * FROM `myisam_test` WHERE `firstname` = 'firstname1';
+----+-------------+-------------+------+---------------+-------+---------+-------+------+-------------+
| id | select_type | table       | type | possible_keys | key   | key_len | ref   | rows | Extra       |
+----+-------------+-------------+------+---------------+-------+---------+-------+------+-------------+
|  1 | SIMPLE      | myisam_test | ref  | index         | index | 32      | const |    1 | Using where |
+----+-------------+-------------+------+---------------+-------+---------+-------+------+-------------+
#可以使用到 index 索引
```

如果非要对内存表进行范围查询怎么办？当然也不是没有解决办法，Mysql在建索引的时候是可以指定索引类型的。

## 对内存表建立BTREE索引

```sql
CREATE TABLE `memory_btree_test` (
  `id` int(10) unsigned NOT NULL DEFAULT '0',
  `firstname` varchar(10) NOT NULL DEFAULT '',
  `lastname` varchar(10) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  KEY `index` (`firstname`,`lastname`) USING BTREE
) ENGINE=MEMORY
```

### 联合索引左前缀

```sql
EXPLAIN SELECT * FROM `memory_btree_test` WHERE `firstname` = 'firstname1';
+----+-------------+-------------------+------+---------------+-------+---------+-------+------+-------------+
| id | select_type | table             | type | possible_keys | key   | key_len | ref   | rows | Extra       |
+----+-------------+-------------------+------+---------------+-------+---------+-------+------+-------------+
|  1 | SIMPLE      | memory_btree_test | ref  | index         | index | 32      | const |    1 | Using where |
+----+-------------+-------------------+------+---------------+-------+---------+-------+------+-------------+
```

总结：

Mysql的每个存储引擎（MyISAM、InnoDB、Memory）都有它的适用场景，索引类型也不例外，当然更不要说各种产品了（Memcache、Redis等等）。了解了每个产品的应用场景才能更好的帮我们解决实际困难，而不是时不时的给自己添堵。在这次故障上，还是体现了自己的无知，仍需要加强学习。