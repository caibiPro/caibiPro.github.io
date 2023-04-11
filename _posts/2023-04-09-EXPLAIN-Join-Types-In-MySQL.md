---
title: MySQL中EXPLAIN输出type列解析
author: mingqing
date: 2023-04-09 14:01:00 +0800
categories: [DataBase, MySQL]
tags: [mysql, explain, type, mysql性能分析]
math: true
mermaid: true
---
众所周知SQL性能调优的依据就是`EXPLAIN`，其中`type`对结果影响最大。本文结合MySQL8.0官方文档中对 [EXPLAIN Output Format](https://dev.mysql.com/doc/refman/8.0/en/explain-output.html) 的说明以及其他网上其他文章，试图对该工具进行简明扼要的阐述。(本文实例环境为MySQL8.0.32，不再赘述)

## **type** 概述

> The type column of EXPLAIN output describes **how tables are joined**.
{: .prompt-tip }

MySQL的官网解释非常简洁，只用了3个单词：连接类型(the join type)。它描述了找到所需数据使用的扫描方式。

最为常见的扫描方式有：

- **system**：系统表，少量数据，往往不需要进行磁盘IO；
- **const**：常量连接；
- **eq_ref**：主键索引（primary key）或者非空唯一索引（unique not null）等值扫描；
- **ref**：非主键非唯一索引扫描；
- **fulltext**：全文索引；
- **ref_or_null**：带NULL的非主键非唯一索引扫描；
- **index_merge**：合并索引；
- **unique_subquery**：带主键/唯一索引扫描且包含IN语句的子查询；
- **index_subquery**：带非唯一索引且包含IN语句的子查询；
- **range**：范围扫描；
- **index**：索引树扫描；
- **ALL**：全表扫描（full table scan）；

上面各类扫描方式**由快到慢**： system > const > eq_ref > ref > fulltext > raf_or_null > index_merge > unique_query > index_query > range > index > ALL

## **system**

> The table has **only one row**(= system table). This is a **special case of the const** join type.
{: .prompt-tip }

特点简单概括如下：

- 表中只有**一行记录**（系统表）
- 是const类型的一个特殊情况
- 需要存储引擎具有统计精确性，即执行`COUNT(*)`无需遍历（目前InnoDB已经没有，在MyISAM可以）

## **const**

> The table has **at most one matching row**, which is read at the start of the query. Because there is only one row, values from the column in this row can be regarded as constants by the rest of the optimizer. const tables are very fast because they are read only once.
{: .prompt-tip }

`const`扫描方法的关键在于**at most one matching row**，即最多一条匹配行，通常应用在`WHERE`语句中查询常量，而一条匹配行的限定自然而然使得出现在`WHERE`语句中的Column满足唯一性，也就是使用主键索引（Primary Key）或唯一索引（Unique Key）。

而查询迅速的原因在于**这一条查询结果只会读取一次**，被优化器视为常量。因此官方文档给出了如下建议：

> `const` is used when you compare all parts of a PRIMARY KEY or UNIQUE index to constant values.
{: .prompt-tip }

总结下运用条件：

- **查询结果最多一条匹配行**
  - 等值查询
  - 主键约束或唯一约束
  - 对于联合索引则需全匹配来保证一条匹配行（即各索引字段都需有单值查询）

具体参考如下示例：

```sql
SELECT * FROM tbl_name WHERE primary_key=1;

SELECT * FROM tbl_name WHERE primary_key_part1=1 AND primary_key_part2=2;
```

## **eq_ref**
>
> **One row is read from this table for each combination of rows from the previous tables**. Other than the system and const types, this is the best possible join type. It is used when all parts of an index are used by the join and the index is a PRIMARY KEY or UNIQUE NOT NULL index.
{: .prompt-tip }

该方法主要针对**多表等值连接**的情况下的**被驱动表**，多表连接的笛卡尔积操作可以理解为一个嵌套循环，**`eq_ref`的关键就在于对于外层的每一次循环，内层循环类似与`const`扫描方法：只匹配一行。这就要求被驱动表的约束条件为常数，且约束字段为主键或唯一约束（索引）**。

另外值得一提的是，这里的常数，即`comparison value = const`不一定要求显式为常数，因为外层循环每次遍历传入内层循环的变量也为常数，示例如下：

```sql
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

## **ref**
>
> All rows with matching index values are read from this table for each combination of rows from the previous tables. ref is used if the join uses only a leftmost prefix of the key or if the key is not a PRIMARY KEY or UNIQUE index (**in other words, if the join cannot select a single row based on the key value**). If the key that is used matches only a few rows, this is a good join type.
{: .prompt-tip }

划重点：`ref`与`eq_ref`的唯一区别**在于不能匹配单行**， 详细来说对于单表查询或多表连接来说，使用非主键/唯一索引使得失去了UNIQUE的特性，使得可能匹配多行，但由于仍然是较小的数字，所以性能不至于差太多。

`ref`扫描，可能出现在join里，也可能出现在单表普通索引里，每一次匹配可能有多行数据返回，虽然它比`eq_ref`要慢，但它仍然是一个很快的扫描类型。

```sql
SELECT * FROM ref_table WHERE key_column=expr;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column=other_table.column;

SELECT * FROM ref_table,other_table
  WHERE ref_table.key_column_part1=other_table.column
  AND ref_table.key_column_part2=1;
```

## **fulltext**
>
> The join is performed using a FULLTEXT index.
{: .prompt-tip }

没什么好说的。

## **ref_or_null**
>
> This join type is like `ref`, but with the addition that MySQL does an **extra search for rows that contain NULL values**. This join type optimization is used most often in resolving subqueries.
{: .prompt-tip }

类似`ref`，差别在于：

- 额外搜索包含NULL值的行
- 常运用在子查询中

```sql
SELECT * FROM ref_table
  WHERE key_column=expr OR key_column IS NULL;
```

## **index_merge**

> This join type indicates that the Index Merge optimization is used. In this case, the key column in the output row contains a list of indexes used, and key_len contains a list of the longest key parts for the indexes used.
>
> The Index Merge access method retrieves rows with multiple range scans and merges their results into one. This access method merges index **scans from a single table only**, not scans across multiple tables. The merge can produce **unions**, **intersections**, or **unions-of-intersections** of its underlying scans.
{: .prompt-tip }

简单概括：

- 基于索引合并优化
- 仅仅针对单表查询
- 根据合并算法的不同分为`intersect(...)`, `union(...)`, `sort_union(...)`

```sql
SELECT * FROM tbl_name WHERE key1 = 10 OR key2 = 20;

SELECT * FROM tbl_name
  WHERE (key1 = 10 OR key2 = 20) AND non_key = 30;

SELECT * FROM t1, t2
  WHERE (t1.key1 IN (1,2) OR t1.key2 LIKE 'value%')
  AND t2.key1 = t1.some_col;

SELECT * FROM t1, t2
  WHERE t1.key1 = 1
  AND (t2.key1 = t1.some_col OR t2.key2 = t1.some_col2);
```

## **unique_subquery**

> This type replaces eq_ref for some IN subqueries of the following form:
>
> `value IN (SELECT primary_key FROM single_table WHERE some_expr)`
>
> unique_subquery is just an index lookup function that replaces the subquery completely for better efficiency.
{: .prompt-tip }

unique_query针对包含IN语句的子查询且子查询中使用了主键索引进行等值匹配的情况，其原理为优化器将IN语句转换为EXISTS语句，如对于

`value IN (SELECT primary_key FROM single_table WHERE some_expr)`

转换为：

`EXISTS SELECT 1 FROM single_table WHERE primary_key = value AND some_expr`

从而达到利用主键索引提高查询效率的目的。另外经测试子查询中如果是唯一索引也能运用改扫描方法

## **index_subquery**

> This join type is similar to unique_subquery. It replaces IN subqueries, but it works for nonunique indexes in subqueries of the following form:
>
>`value IN (SELECT key_column FROM single_table WHERE some_expr)`
{: .prompt-tip }

简单来说，基本同unique_subquery，除了子查询中使用的是非唯一性索引，差别可参考`eq_ref`与`ref`的差别

## **range**

>Only rows that are in a given range are retrieved, using an index to select the rows. The key column in the output row indicates which index is used. The key_len contains the longest key part that was used. The ref column is NULL for this type.
>
>range can be used when a key column is compared to a constant using any of the =, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, LIKE, or IN() operators
{: .prompt-tip }

范围查询，使用了诸如`=, <>, >, >=, <, <=, IS NULL, <=>, BETWEEN, LIKE, or IN()`的运算符：

```Sql
SELECT * FROM tbl_name
  WHERE key_column = 10;

SELECT * FROM tbl_name
  WHERE key_column BETWEEN 10 and 20;

SELECT * FROM tbl_name
  WHERE key_column IN (10,20,30);

SELECT * FROM tbl_name
  WHERE key_part1 = 10 AND key_part2 IN (10,20,30);
```

## **index**

> The index join type is the same as `ALL`, <mark>except that the index tree is scanned</mark>. This occurs two ways:
>
> - If the index is a covering index for the queries and can be used to satisfy all data required from the table, only the index tree is scanned. In this case, the Extra column says Using index. An index-only scan usually is faster than ALL because the size of the index usually is smaller than the table data.
>
> - A full table scan is performed using reads from the index to look up data rows in index order. Uses index does not appear in the Extra column.
>
>MySQL can use this join type when the query uses only columns that are part of a single index.
{: .prompt-tip }

这里提到`index`扫描方法相较与`ALL`全表扫描来说，区别关键在于**索引覆盖**这个概念使得只需扫描整个索引树而非全表。可以简单理解为所查询的字段位于联合索引所在的BTREE中而不再需要进行回表的操作。

扫描整个索引，效率很低，**仅仅因为辅助索引的空间比主键索引小**，所以比`ALL`效率高一点。最常用的有`SELECT COUNT(*)`，其余举例如下：

```sql
-- INDEX idx_key_part(key_part1, key_part2, key_part3)

EXPLAIN SELECT `key_part1` FROM `s1` WHERE `key_part3` = 'a' AND `key_part2` = 'a';
```

根据最左前缀原则，`WHERE`语句中的`key_part2`和`key_part3`无法有效利用该联合索引，只能以近乎全表扫描的方式进行搜索，但`SELECT`中的`key_part1`字段也位于该索引中，故不需要再回表

## **ALL**

> A full table scan is done for each combination of rows from the previous tables. This is normally not good if the table is the first table not marked const, and usually very bad in all other cases. Normally, you can avoid ALL by adding indexes that enable row retrieval from the table based on constant values or column values from earlier tables.
{: .prompt-tip }

没什么好说的，比较糟糕的扫描方法。
