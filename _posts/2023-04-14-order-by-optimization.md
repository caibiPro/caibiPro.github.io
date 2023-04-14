---
title: ORDER BY Optimization in MySQL8
author: mingqing
date: 2023-04-14 10:31:00 +0800
categories: [DataBase, MySQL]
tags: [mysql, order by, optimization]
math: true
mermaid: true
---
本文结合[官方文档](https://dev.mysql.com/doc/refman/8.0/en/order-by-optimization.html)对MySQL优化器使用索引或`filesort`等方式对`ORDER BY`语句进行查询优化对原理简单剖析。

## 使用索引

> The index may also be used even if the `ORDER BY` does not match the index exactly, as long as all unused portions of the index and all extra ORDER BY columns are constants in the WHERE clause. If the index does not contain all columns accessed by the query, the index is used only if index access is cheaper than other access methods.
{: .prompt-tip }

`ORDER BY`语句是否使用索引，其不确定性主要发生在`ORDER BY`字段与索引并未完美匹配的情况。

>  Whether the optimizer actually does so depends on whether reading the index is more efficient than a table scan if columns not in the index must also be read.
{: .prompt-tip }

核心判断原则在于：优化器在使用索引（可能造成查询了过多索引外的字段导致了回表后多次读取）和不使用索引（全表扫描直接排序）两种方案间来选择开销较小的方案。

### 使用索引案例分析

假设建立如下索引：

```sql
CREATE INDEX idx_key1_key2 ON t1(key_part1, key_part2)
```

1. `SELECT * ... ORDER BY`

```sql
SELECT * FROM t1
  ORDER BY key_part1, key_part2;
```
若`SELECT *`中包含的非索引字段的大小过多（字段数*行数），使得扫描索引树后回表读取的数据过多，那么可能不会使用索引。

若`SELECT *`只包含了索引字段，那么会使用索引而非`filesort`。如下所示：

```sql
SELECT pk, key_part1, key_part2 FROM t1
  ORDER BY key_part1, key_part2;
```

2. `WHERE key_part1 = const`

```sql
SELECT * FROM t1
  WHERE key_part1 = constant
  ORDER BY key_part2;
```
由于`ORDER BY`语句的执行顺序比较靠后，如果`WHERE`语句筛选后的行数比较小，那么使用索引筛选后的页中`key_part2`本身就是有序的。此时即使回表开销也很可能比全表扫描开销小。

3. `DESC`

如果是`DESC`的条件如果在索引树中可以反向扫描的话那么与前述分析没有本质的区别，总之`ORDER BY`中无论是否`DESC`，只需与`INDEX`中保持一致性即可：

```sql
SELECT * FROM t1
  ORDER BY key_part1 DESC, key_part2 DESC;

SELECT * FROM t1
  WHERE key_part1 = constant
  ORDER BY key_part2 DESC;
```

4. `WHERE key_part1 > const`

>  The index is used if the WHERE clause is selective enough to make an index range scan cheaper than a table scan
{: .prompt-tip }

```sql
SELECT * FROM t1
  WHERE key_part1 > constant
  ORDER BY key_part1 ASC;

SELECT * FROM t1
  WHERE key_part1 < constant
  ORDER BY key_part1 DESC;
```

与`WHERE`中的等值筛选相同，使用索引的前提条件在于`WHERE`筛选后较小的结果集使得即使回表查询的开销仍然少于全表扫描。

### 索引实效案例分析

在某些情况下，MySQL不能使用索引来解析`ORDER BY`，即使可能会使用这些索引来查找`WHERE`子句中的匹配行。

1. `ORDER BY`中字段属于不同的索引

```sql
SELECT * FROM t1 ORDER BY key1, key2;
```

2. `ORDER BY`中字段在联合索引中不连续

```sql
SELECT * FROM t1 WHERE key2=constant ORDER BY key1_part1, key1_part3;
```

3. `ORDER BY`中字段的索引与`WHERE`中筛选字段的索引不同

```sql
-- Using Index idx_key2, Using filesort
SELECT * FROM t1 WHERE key2=constant ORDER BY key1;
```

4. `ORDER BY`中包含列名以外的表达式

```sql
SELECT * FROM t1 ORDER BY ABS(key);
SELECT * FROM t1 ORDER BY -key;
```

5. `ORDER BY`与`GROUP BY`中查询不同的字段

6. `ORDER BY`中字段使用的索引为前缀索引

> There is an index on only a prefix of a column named in the ORDER BY clause. In this case, the index cannot be used to fully resolve the sort order. For example, if only the first 10 bytes of a CHAR(20) column are indexed, the index cannot distinguish values past the 10th byte and a filesort is needed.
{: .prompt-tip }

如某字段属性为`CHAR(20)`， 但以该字段的前缀建立索引如`CHAR(10)`的情况下，`ORDER BY`中出现该字段则无法使用该字段对应的索引。

7. 未按序存储的索引如`HASH`索引

### 列别名使得索引实效的情况

索引用于排序的可用性可能会收到列别名的使用情况。现假设列`a`为`t1.a`的别名，那么下述表达式中可以使用以列`a`建立的索引：

```sql
SELECT a FROM t1 ORDER BY a;
```

但若在选择集中出现了关联其他列或表达式且重复的别名，则`ORDER BY`中对该别名的排序无法应用上述索引：

```sql
-- CAN NOT USE THE INDEX(a)
SELECT ABS(a) AS a FROM t1 ORDER BY a;

-- CAN USR THE INDEX(a)
SELECT ABS(a) AS b FROM t1 ORDER BY a;
```

## 使用`filesort`

> If an index cannot be used to satisfy an ORDER BY clause, MySQL performs a filesort operation that reads table rows and sorts them. A filesort constitutes an extra sorting phase in query execution.
{: .prompt-tip }

无法使用索引排序的情况下，MySQL需要额外的内存空间来在查询过程中进行额外的排序。关于内存分配的方式，简单概括下：

- MySQL 8.0.12起：优化器根据所需递增地分配内存空间，直至Buffers的空间大小达到`sort_buffer_size`指定的大小

- MySQL 8.0.12之前：优化器直接分配`sort_buffer_size`固定大小的内存空间

这使得用户可以设置较大的`sort_buffer_size`来加速大规模的排序，而无需担心其他场景下资源的浪费。

### 完全内存排序（In-memory filesort operations）

> A filesort operation uses temporary disk files as necessary if the result set is too large to fit in memory. Some types of queries are particularly suited to completely in-memory filesort operations.
{: .prompt-tip }

如下格式的查询语句会使得优化器使用完全的内存中排序，而无需磁盘上的临时文件：

```sql
SELECT ... FROM single_table ... ORDER BY non_index_column [DESC] LIMIT [M,]N;
```

这种查询常见于各种web应用，即从一个大的结果集中仅显示少量数据：

```sql
SELECT col1, ... FROM t1 ... ORDER BY name LIMIT 10;
SELECT col1, ... FROM t1 ... ORDER BY RAND() LIMIT 15;
```

## 影响`ORDER BY`优化的因素

自MySQL 8.0.20之后，对于未使用`filesort`且包含`ORDER BY`的慢查询，已不再使用降低`max_length_for_sort_data`的值来达到触发`filesort`的方法（过高的`max_length_for_sort_data`值会导致比较高的磁盘I/O和较低的CPU利用率）。

如果不能使用索引，那么可以考虑如下策略：

- 增大`sort_buffer_size`：使得内存能够容纳排序的整个结果集，避免多余的磁盘读写

- 