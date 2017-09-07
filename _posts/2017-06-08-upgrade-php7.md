---
title: "PHP7升级全过程记录"
layout: single
categories: PHP
tags: linux php7 upgrade
---

关于PHPNG的计划，在2014年初社区就有讨论，自己也是从那个时候开始，关注着这个里程碑版本的一举一动，毕竟上一次主版本号级的发布还是在遥远的2004年。14年底鸟哥的一篇博文["PHP7 VS HHVM"](http://www.laruence.com/2014/12/18/2976.html)，为开发者展示了PHP7的巨大性能提升。又经过一年漫长的等待，终于在2015年的圣诞节前，迎来了PHP7的第一个正式发布版。

作为一个极端业务导向的二线公司，受限于技术积累和人力资源，我们把升级的时间初步定在正式发布的1年后。由于还有一些历史问题需要处理，所以分为2个大的阶段：

* 第一阶段（2016.4 ～ 2016.8）
  * 升级PHP版本到5.6，处理历史遗留问题（Mysql、Mssql向PDO迁移，更换mysqlnd）
  * 统一生产环境（RedHat 5.8 => CentOS 6.5，制作RPM标准包，统一发布方式）
  * 在统一后的生产环境进行5.6的灰度发布，逐步安排上线
* 第二阶段（2016.12 ～ 2017.3）
  * 了解PHP7原理
  * 运行环境搭建，benchmark 测试
  * 处理语法兼容工作，测试并修复bug
  * 灰度发布
  * 上线

业务的选择上，我们选取了移动站点作为试点，主要从这几个因素进行的考量：

* 项目新：2013年开坑，2015年经历了一次从5.3到5.5的升级
* 了解深：前后跟了这个项目2年多
* 流量大：每天处理大几千万请求，高峰时勉强过亿
* 有发展：移动化的红利还在

# 实践

## Benchmark

测试使用的是官方提供的脚本 [bench.php](https://github.com/php/php-src/blob/master/Zend/bench.php)，简单的从执行时间上比较了一下（执行10次），效果惊人。

![php56 to php70 benchtime](http://ot41apokn.bkt.clouddn.com/php56-php70-benchtime.png)

2015年从 PHP5.3 升级到 PHP5.5 也测试过一次，速度提升比这个小多了，不过内存占用下降了很多。

![php53 to php55 benchtime](http://ot41apokn.bkt.clouddn.com/php53-php55-benchtime.png)

![php53 to php55 benchmemory](http://ot41apokn.bkt.clouddn.com/php53-php55-benchmemory.png)

## 源码部分

我们在运行环境准备的时候，PHP已经发布到了7.0.14，使用的扩展（memcached、redis、xdebug、pthreads等）基本都有了至少一个稳定版本，除了第一次忘了用GCC 4.8+编译，在整个编译过程中也没有遇到问题。开发的扩展代码量很少，只是对几个常用处理过程用C进行了封装。参照[wiki](https://wiki.php.net/phpng-upgrading)给出的指引，再结合编译过程中的错误提示，很快就解决了问题。

通过对比阅读 PHP7 和 PHP5 的源码，发现其之所以在保证兼容性的情况下还能达到很好的性能，主要的优化点在以下几个方面：

1. MAKE\_STD\_ZVAL 从栈上分配 zval，减少一次内存读取
2. Hashtable 中使用 arData 直接存储数据，内存使用更加高效
3. 字符串结构体 \_zend\_string 存储了 hash 值，在数组中对字符串 key 的查找时减少一次 hash 计算

### zval

PHP7 对 zval 结构体改变很大，之前的处理方式是type表示值的类型，变量的值存储在 zvalue\_value 联合体中，is\_ref\_\_gc 和 refcount\_\_gc 表示是否引用和引用计数器。这种方式有2个比较大的缺点：

1. 占用内存多，实际使用中对象类型的变量少，zend\_object\_value 可以换成指针
2. 结构体没有预留自定义的字段，扩展起来不方便

```c
struct _zval_struct {
  zvalue_value  value;
  zend_uint     refcount__gc;
  zend_uchar    type;
  zend_uchar    is_ref__gc;
};

typedef union _zvalue_value {
  long   lval;
  double dval;
  struct {
    char *val;
    int   len;
  } str;
  HashTable *ht;
  zend_object_value obj;
} zvalue_value;
```

PHP7 的 zval 结构体包含 zend\_value 用于存储变量的值（long or double）或者指针，另外还有 u1 和 u2 两个联合体，u1 是 zval 类型信息（共17种类型，见下zval.u1.type），u2 是辅助信息。另外，引用计数是在 zend\_value 而不是 zval 上，变量之间的传递、赋值主要也是对 zend\_value，新的做法显然更好一些。

```c
struct _zval_struct {
  zend_value        value;          /* value */
  union {
    struct {
      ZEND_ENDIAN_LOHI_4(
        zend_uchar    type,         /* active type */
        zend_uchar    type_flags,
        zend_uchar    const_flags,
        zend_uchar    reserved)     /* call info for EX(This) */
    } v;
    uint32_t type_info;
  } u1;
  union {
    uint32_t     var_flags;
    uint32_t     next;            /* hash collision chain */
    uint32_t     cache_slot;      /* literal cache slot */
    uint32_t     lineno;          /* line number (for ast nodes) */
    uint32_t     num_args;        /* arguments number for EX(This) */
    uint32_t     fe_pos;          /* foreach position */
    uint32_t     fe_iter_idx;     /* foreach iterator index */
    uint32_t     access_flags;    /* class constant access flags */
    uint32_t     property_guard;  /* single property guard */
    uint32_t     extra;           /* not further specified */
  } u2;
};

typedef union _zend_value {
  zend_long         lval;           /* long value */
  double            dval;           /* double value */
  zend_refcounted  *counted;
  zend_string      *str;
  zend_array       *arr;
  zend_object      *obj;
  zend_resource    *res;
  zend_reference   *ref;
  zend_ast_ref     *ast;
  zval             *zv;
  void             *ptr;
  zend_class_entry *ce;
  zend_function    *func;
  struct {
    uint32_t w1;
    uint32_t w2;
  } ww;
} zend_value;

/* zval.u1.type */
/* regular data types */
#define IS_UNDEF                    0
#define IS_NULL                     1
#define IS_FALSE                    2
#define IS_TRUE                     3
#define IS_LONG                     4
#define IS_DOUBLE                   5
#define IS_STRING                   6
#define IS_ARRAY                    7
#define IS_OBJECT                   8
#define IS_RESOURCE                 9
#define IS_REFERENCE                10

/* constant expressions */
#define IS_CONSTANT                 11
#define IS_CONSTANT_AST             12

/* fake types */
#define _IS_BOOL                    13
#define IS_CALLABLE                 14

/* internal types */
#define IS_INDIRECT                 15
#define IS_PTR                      17

```

### HashTable

PHP5 的 HashTable 结构体中，数据存储在 arBuckets 指针数组中，它是由 bucket 组成的双向链表，每个元素的值（zval结构）也存在这些 bucket 中，每个 bucket 中保存一个指向 zval 结构的指针，由于老的实现过于考虑通用性，所以不止需要一个指针，而是两个指针。这种结构下 bucket 和 zval 都需要分开分配，分配效率比较低。

```c
typedef struct _hashtable {
  uint nTableSize;
  uint nTableMask;
  uint nNumOfElements;
  ulong nNextFreeElement;
  Bucket *pInternalPointer;
  Bucket *pListHead;
  Bucket *pListTail;
  Bucket **arBuckets;
  dtor_func_t pDestructor;
  zend_bool persistent;
  unsigned char nApplyCount;
  zend_bool bApplyProtection;
  #if ZEND_DEBUG
    int inconsistent;
  #endif
} HashTable;

typedef struct bucket {
  ulong h;
  uint nKeyLength;
  void *pData;
  void *pDataPtr;
  struct bucket *pListNext;
  struct bucket *pListLast;
  struct bucket *pNext;
  struct bucket *pLast;
  char *arKey;
} Bucket;
```

![HashTable](http://ot41apokn.bkt.clouddn.com/basic_hashtable.png)

保证插入顺序的 bucket 实现

![Ordered_HashTable](http://ot41apokn.bkt.clouddn.com/ordered_hashtable.png)

PHP7 的 HashTable 结构体中，bucket 是一个条目，zval是直接嵌入bucket结构体中，没有必要单独为他分配内存，也不会产生因内存分配引起的冗余信息，减少了空间浪费。

```c
typedef struct _zend_array HashTable;

struct _zend_array {
  zend_refcounted_h gc;
  union {
    struct {
      ZEND_ENDIAN_LOHI_4(
        zend_uchar    flags,
        zend_uchar    nApplyCount,
        zend_uchar    nIteratorsCount,
        zend_uchar    consistency)
    } v;
    uint32_t flags;
  } u;
  uint32_t          nTableMask;
  Bucket           *arData;
  uint32_t          nNumUsed;
  uint32_t          nNumOfElements;
  uint32_t          nTableSize;
  uint32_t          nInternalPointer;
  zend_long         nNextFreeElement;
  dtor_func_t       pDestructor;
};

typedef struct _Bucket {
	zval              val;
	zend_ulong        h;        /* hash value (or numeric index)   */
	zend_string      *key;      /* string key or NULL for numerics */
} Bucket;

/*
 * HashTable Data Layout
 * =====================
 *
 *                 +=============================+
 *                 | HT_HASH(ht, ht->nTableMask) |
 *                 | ...                         |
 *                 | HT_HASH(ht, -1)             |
 *                 +-----------------------------+
 * ht->arData ---> | Bucket[0]                   |
 *                 | ...                         |
 *                 | Bucket[ht->nTableSize-1]    |
 *                 +=============================+
 */
```

## PHP代码部分

### 引入Composer

以前的程序中使用了很多第三方库（smarty2、phpexcel、phpmailer、qrcode等），有一些用的版本比较老，升级PHP7的过程中也需要同步进行升级。为了将来的考虑，这次做的彻底一些，用[Composer](http://www.phpcomposer.com/)来管理第三方库，一些实在找不到替代品的只能自己处理了。现在再看这一步，真应该在升级5.6解决历史包袱的时候就应该处理掉，测试的工作量会少很多。

### 使用openssl替代mcrypt

吃过上面亏，后面再动手前先看了看刚发布的 PHP7.1 迁移手册中[废弃的特性](http://php.net/manual/zh/migration71.deprecated.php)一节，准备比较一下 openssl 和 mcrypt。

* [OpenSSL or Mcrypt](https://stackoverflow.com/questions/36571743/openssl-or-mcrypt-openssl-encrypt-or-mcrypt-encrypt) 
* [PHP 中 OpenSSL 与 Mcrypt 加密算法效率比较](http://www.cnblogs.com/jgosling/articles/4733417.html)

通过上面两篇文章的介绍，openssl 作为 mcrypt 替代者不论从支持的加密种类还是加密速度上都完爆后者。自己经过测试验证，确认无误。替换方式也很简单，这里以 DES-CBC 为例。

```php
$pad = 8 - (strlen($text) % 8);
$text .= str_repeat(chr($pad), $pad);

base64_encode(mcrypt_encrypt(MCRYPT_DES, $key, $text, MCRYPT_MODE_CBC, $iv));
openssl_encrypt($text, 'DES-CBC', $key, OPENSSL_ZERO_PADDING, $iv);
```

另外最后提一下，特别是在跨平台时，填充方式可能有的区别，有兴趣的可以看看[这里](https://crypto.stackexchange.com/questions/9043/what-is-the-difference-between-pkcs5-padding-and-pkcs7-padding)

### 错误和异常处理

set\_exception\_handler() 不再保证收到的一定是 Exception 对象

抛出 Error 对象时，如果 set\_exception\_handler() 里的异常处理代码声明了类型 Exception ，将会导致 fatal error。

想要异常处理器同时支持 PHP5 和 PHP7，应该删掉异常处理器里的类型声明。如果代码仅仅是升级到 PHP7，则可以把类型 Exception 替换成 Throwable。

```php
// PHP 5 时代的代码将会出现问题
function handler(Exception $e) { ... }
set_exception_handler('handler');

// 兼容 PHP 5 和 7
function handler($e) { ... }

// 仅支持 PHP 7
function handler(Throwable $e) { ... }
```

![警告级别变更](http://ot41apokn.bkt.clouddn.com/php_error_reporting_changes.png)

### 其它细碎变更

```php
//有变化
list()
//移除
call_user_method()
call_user_method_array()
ereg_replace() //ereg整个系列函数
$HTTP_RAW_POST_DATA
```

## 参数优化

### Opcache

```php
opcache.enable = 1
opcache.enable_cli = 1
opcache.interned_strings_buffer = 8
//php文件多可以设置的大一些
opcache.max_accelerated_files = 8000
opcache.memory_consumption = 256
//文件检查周期，追求性能的在生产环境可以关闭，关闭后修改文件必须重启php-fpm才能生效
opcache.revalidate_freq = 600
opcache.fast_shutdown = 1
//开启hugepages支持
opcache.huge_code_pages = 1
```

opcache.file_cache这个还属于实验性质，在生产环境没有启用。

### HugePages

开启系统的HugePages

```bash
sysctl vm.nr_hugepages=512

grep Huge /proc/meminfo
AnonHugePages:    391168 kB
HugePages_Total:     512
HugePages_Free:      477
HugePages_Rsvd:      103
HugePages_Surp:        0
Hugepagesize:       2048 kB
```

在第一次部署生产环境的时候还遇到了命令行下报错的情况。

```bash
/usr/local/php/bin/php -v
PHP Warning: Zend OPcache huge_code_pages: mmap(HUGETLB) failed: Cannot allocate memory (12) in Unknown on line 0
```

看了一下 text 段的占用，需要5个 HugePages，再看 HugePages\_Free 发现只剩下4个了。调整参数到1024并重启 php-fpm，问题解决。出现这个问题主要是生产环境的 php-fpm 进程开的很多，导致 HugePages 不够用了。

```bash
size /usr/local/php/bin/php
   text	   data	    bss	    dec	    hex	filename
11092574	 778968	 149160	12020702	 b76bde	/usr/local/php/bin/php
```

分享一个查看进程 HugePages 占用的脚本

```bash
#进程hugepage占用，Redhat&CentOS首选
sudo grep -B 11 'KernelPageSize:     2048 kB' /proc/[pid]/smaps | grep "^Size:" | awk '{sum+=$2} END{print sum/1024}'
```

```perl
#!/usr/bin/perl
#查看huge_pages占用
sub counthugepages {
    my $pid=$_[0];
    open (NUMAMAPS, "/proc/$pid/numa_maps") || die "can't open numa_maps";
    my $HUGEPAGECOUNT=0;
    while (<NUMAMAPS>) {
        if (/huge.*dirty=(\d+)/) {
          $HUGEPAGECOUNT+=$1;
        }
    }
    close NUMAMAPS;
    return ($HUGEPAGECOUNT);
}

printf "%d huge pages\n",counthugepages($ARGV[0]);
```

## 结果

从3月中旬第一次灰度上线，到7月12日下午最后一台服务器升级完成。整个集群CPU使用率平均下降15% \- 20%，负载也有很大程度的下降，完全达到预期。

![php7升级后cpu使用率情况](http://ot41apokn.bkt.clouddn.com/after-upgrade-cpu.png)

![php7升级后负载情况](http://ot41apokn.bkt.clouddn.com/after-upgrade-load.png)

# 总结

首先要感谢PHP社区各位成员的努力，让PHP的性能有了一次巨大提升，并且这次提升对于大多数应用领域的开发者来说，几乎可以算是透明的。

在这次漫长的升级过程中，各位小伙伴也为这次里程碑式的升级贡献了太多的智慧，付出了辛勤的劳动，感谢各位。

接下来还要推动公司内其它老项目的升级工作，希望其他小伙伴也能尽快享受到升级带来的实惠。

# 参考资料

[PHP Internals Book](http://www.phpinternalsbook.com/index.html)

[laruence/php7-internal](https://github.com/laruence/php7-internal)

[PHP's new hashtable implementation](http://nikic.github.io/2014/12/22/PHPs-new-hashtable-implementation.html)

[日请求亿级的 QQ 会员 AMS 平台 PHP7 升级实践](http://blog.jobbole.com/103322/)

[hugetlbpage](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)

[Linux HugePages](http://yong321.freeshell.org/oranotes/HugePages.txt)

[check for hugepages usage](https://access.redhat.com/solutions/320303)

[让PHP7达到最高性能的几个Tips](http://www.laruence.com/2015/12/04/3086.html)
