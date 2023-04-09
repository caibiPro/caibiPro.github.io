---
title: MySQL中EXPLAIN输出type列解析
author: mingqing
date: 2023-04-09 14:01:00 +0800
categories: [DataBase, MySQL]
tags: [mysql, explain, type, mysql性能分析]
math: true
mermaid: true
---
众所周知SQL性能调优的依据就是`EXPLAIN`，其中`type`对结果影响最大。本文结合MySQL8.0官方文档中对[EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html)的说明以及其他网上其他文章，试图对该工具进行简明扼要的阐述。(本文实例环境为MySQL8.0.32，不再赘述)


## `type` 概述

> The type column of EXPLAIN output describes ***how tables are joined***.

MySQL的官网解释非常简洁，只用了3个单词：连接类型(the join type)。它描述了找到所需数据使用的扫描方式。

最为常见的扫描方式有：

- **system**：系统表，少量数据，往往不需要进行磁盘IO；
- **const**：常量连接；
- **eq_ref**：主键索引(primary key)或者非空唯一索引(unique not null)等值扫描；
- **ref**：非主键非唯一索引扫描；
- **fulltext**：全文索引
- **ref_or_null**：带NULL的非主键非唯一索引扫描；
- **range**：范围扫描；
- **index**：索引树扫描；
- **ALL**：全表扫描(full table scan)；

上面各类扫描方式**由快到慢**： system > const > eq_ref > ref > fulltext > raf_or_null > range > index > ALL

## `system`

> The table has <mark>***only one row***</mark>(= system table). This is a <mark>special case of the `const`</mark> join type.

特点简单概括如下：

- 表中只有**一行记录**（系统表）
- 是const类型的一个特殊情况
- 需要存储引擎具有统计精确性，即执行`const(*)`无需遍历（目前InnoDB已经没有，在MyISAM可以）

``` SQL
> CREATE TABLE test_innodb ( `uid` INT ) ENGINE=InnoDB;
> CREATE TABLE test_myisam ( `uid` INT ) ENGINE=MyISAM;

> INSERT INTO `test_innodb` VALUES(1);
> INSERT INTO `test_myisam` VALUES(1);

> EXPLAIN SELECT * FROM `test_innodb`\G
           id: 1
  select_type: SIMPLE
        table: test_innodb
   partitions: NULL
         type: ALL

> EXPLAIN SELECT * FROM `test_myisam`\G
           id: 1
  select_type: SIMPLE
        table: test_myisam
   partitions: NULL
         type: system

> INSERT INTO `test_innodb` VALUES(2);
> INSERT INTO `test_myisam` VALUES(2);

> EXPLAIN SELECT * FROM `test_innodb`\G
          id: 1
  select_type: SIMPLE
        table: test_innodb
   partitions: NULL
         type: ALL

> EXPLAIN SELECT * FROM `test_myisam`\G
           id: 1
  select_type: SIMPLE
        table: test_myisam
   partitions: NULL
         type: ALL
```

## `const`

> The table has <mark>at most one matching row</mark>, which is read at the start of the query. Because there is only one row, values from the column in this row can be regarded as constants by the rest of the optimizer. const tables are very fast because they are read only once.

`const`扫描方法的关键在于**at most one matching row**，即最多一条匹配行，通常应用在`WHERE`语句中查询常量，而一条匹配行的限定自然而然使得出现在`WHERE`语句中的Colmn满足唯一性，也就是使用主键索引（Primary Key）或唯一索引（Unique Key）。

而查询迅速的原因在于**这一条查询结果只会读取一次**，被优化器视为常量。因此官方文档给出了如下建议：

> const is used when you compare all parts of a PRIMARY KEY or UNIQUE index to constant values. 

总结下运用条件：
- <mark>**查询结果最多一条匹配行**</mark>
    - 等值查询
    - 主键约束或唯一约束
    - 对于联合索引则需全匹配来保证一条匹配行（即各索引字段都需有单值查询）

具体参考如下示例：

```SQL
> SELECT * FROM tbl_name WHERE primary_key=1;
> SELECT * FROM tbl_name WHERE primary_key_part1=1 AND primary_key_part2=2;
```

## `eq_ref`
> <mark>One row is read from this table for each combination of rows from the previous tables</mark>. Other than the system and const types, this is the best possible join type. It is used when all parts of an index are used by the join and the index is a PRIMARY KEY or UNIQUE NOT NULL index.

该方法主要针对**多表等值连接**的情况下的**被驱动表**，多表连接的笛卡尔积操作可以理解为一个嵌套循环，<mark>`eq_ref`的关键就在于对于外层的每一次循环，内层循环类似与`const`扫描方法：只匹配一行。这就要求被驱动表的约束条件为常数，且约束字段为主键或唯一约束（索引）</mark>。

另外值得一提的是，这里的常数，即`comparison value = const`不一定要求显式为常数，因为外层循环每次遍历传入内层循环的变量也为常数，示例如下：

```SQL
SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
  AND ref_table.key_column_part2=1;
```

小结一下`eq_ref`的条件：

- **多表等值连接**
- **被驱动表（内循环）只匹配一行**
  - 约束条件为主键或唯一索引
  - 约束条件为常量
  - 联合索引要求全匹配

## `ref`

> All rows with matching index values are read from this table for each combination of rows from the previous tables. ref is used if the join uses only a leftmost prefix of the key or if the key is not a PRIMARY KEY or UNIQUE index <mark>(in other words, if the join cannot select a single row based on the key value)</mark>. If the key that is used matches only a few rows, this is a good join type.

划重点：`ref`与`eq_ref`的唯一区别（见上高亮文本）**在于不能匹配单行**， 详细来说对于单表查询或多表连接来说，使用非主键/唯一索引使得失去了UNIQUE的特性，使得可能匹配多行，但由于仍然是较小的数字，所以性能不至于差太多。

`ref`扫描，可能出现在join里，也可能出现在单表普通索引里，每一次匹配可能有多行数据返回，虽然它比`eq_ref`要慢，但它仍然是一个很快的join类型。

```SQL
SELECT * FROM ref_table WHERE key_column=expr;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
  AND ref_table.key_column_part2=1;
```
## `fulltext`
> The join is performed using a FULLTEXT index.

没什么好说的。

## `ref_or_null`

> This join type is like `ref`, but with the addition that MySQL does an <mark>extra search for rows that contain NULL values</mark>. This join type optimization is used most often in resolving subqueries. 

类似`ref`，差别在于：

- 额外搜索包含NULL值的行
- 常运用在子查询中

```SQL
SELECT * FROM ref_table
  WHERE key_column=expr OR key_column IS NULL;
```

## index_merge

> This join type indicates that the Index Merge optimization is used. In this case, the key column in the output row contains a list of indexes used, and key_len contains a list of the longest key parts for the indexes used.
