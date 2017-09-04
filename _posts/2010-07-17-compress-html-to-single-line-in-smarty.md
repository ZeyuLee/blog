---
title: "在Smarty里把页面输出压缩成1行"
layout: single
categories: PHP
tags: Smarty filter
---
为了提高页面的加载速度，尝试对输出的html响应内容进行压缩。项目使用的是Smarty作为模版引擎，查了[官方手册](http://www.smarty.net/docs/en/api.functions.tpl)是支持过滤器的，三种过滤器区别如下

* prefilter：编译成php脚本前
* postfilter：编译成php脚本后
* outputfilter：显示前

考虑到Smarty可能会升级，没有采用loadFilter这种标准扩展方式，而是在工厂类使用注册函数的方式实现。

```php
$this->_smarty = new Smarty;
#Smarty3版本
$this->_smarty->registerFilter('output', 'htmlCompress');
#Smarty2版本
$this->_smarty->register_outputfilter('htmlCompress');

/**
 * Html代码压缩
 * @param string $tpl_source 模版资源路径
 * @param obj    $smarty     smarty对象
 * @return string
 */
function htmlCompress($tpl_source, $smarty)
{
    //这4类情况进行转换：
    //HTML注释（非贪婪匹配）
    //连续多个空白
    //HTML标记之间的空白，如 </p> <p> 之间的空白
    //js代码分号结尾后的空白
    $patterns = array('/<!--.*-->/U', '/\s+/m', '/>\s+</m', '/;\s+/m');
    $replacements = array('', ' ', '><', ';');
    return preg_replace($patterns, $replacements, $tpl_source);
}
```

写好了本地一跑js报错一大堆，又改了1个小时的内连js脚本才好。总结一下报错的原因无外乎以下2个，算是一个小坑吧。

```javascript
//修改单行注释为块注释
/* 这个ok */

/* 结尾一定要写分号 */
$("p.intro").show();
```