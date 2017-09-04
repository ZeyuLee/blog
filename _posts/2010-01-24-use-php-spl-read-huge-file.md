---
title: "使用SPL读取大文件"
layout: single
categories: PHP
tags: SPL
---
开发中需要读取一个大文件的指定行，试了很多种方法效果都不太理想，想起前几天翻看PHP手册中提到的SPL好像有能解决问题的办法，尝试了一下成功解决问题。

```php
/**
 * @param $file_name string 文件路径
 * @param $start integer 起始行
 * @param $limit integer 向后读取行数
 * @return array 内容
 */
function get_line($file_name, $start, $limit)
{
    $f = new SplFileObject($file_name, ’r’);
    $f->seek($start);
    $ret = Array();

    for ($i = 0; $i < $limit; $i++) {
        $ret[] = trim($f->current());
        $f->next();
    }

    return $ret;
}

#测试的file.log文件700MB
$time_start = microtime(true);
print_r(get_line('/path/file.log', 6789, 5));
echo 'in '.(microtime(true) - $time_start).' seconds'.PHP_EOL;
#in 0.003400924346561 seconds
```