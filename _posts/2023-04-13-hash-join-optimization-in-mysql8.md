---
title: Hash Join Optimization in MySQL8
author: mingqing
date: 2023-04-13 16:13:00 +0800
categories: [DataBase, MySQL]
tags: [mysql, hash join, mysql性能分析]
math: true
mermaid: true
---
Hash Join 就是 当两个或者多个表Join 查询时，基于其中一个表(驱动表)在内存构建一个哈希表，然后一行一行读另一个表(被驱动表)，计算其哈希值到内存哈希表中进行查找。

## [Employ Conditions](https://dev.mysql.com/doc/refman/8.0/en/hash-joins.html)

- **等值连接**（Each join has an **equi-join condition**）
   - 多表连接同样适用，只要**每对表的至少一个连接条件为等值连接**（as long as **at least one** join condition for each pair of tables **is an equi-join**）
        > 原因显然在于散列后的无序性，要充分利用Hash表O(1)的查找特性必须是等值查找
        {: .prompt-tip }

    - **但在MySQL 8.0.20版本后，不再要求必须包含有等值连接了**。因此诸如Inner non-equi-join, Semijoin, Antijoin, Left outer join, Right outer join之类的多表连接均可采用Hash Join Optimization（In MySQL 8.0.20 and later, it is no longer necessary for the join to contain at least one equi-join condition in order for a hash join to be used.）

        > 个人认为原因见下点：非等值连接条件作为Filter应用在Hash之后的多表笛卡尔积结果上，即使这样也仍能有较优的性能
        {: .prompt-tip }

- **非等值连接条件**：作为筛选器被运用在多表连接后的表上(any extra conditions which are not equi-joins are applied as filters after the join is executed)

    ```sql
    > EXPLAIN FORMAT=TREE SELECT * FROM t1
        JOIN t2 ON (t1.c1 = t2.c1 AND t1.c2 < t2.c2)
        JOIN t3 ON (t2.c1 = t3.c1);
        
    ************************* 1. row *************************
    EXPLAIN: -> Inner hash join (t3.c1 = t1.c1)  (cost=1.05 rows=1)
        -> Table scan on t3  (cost=0.35 rows=1)
        -> Hash
            -> Filter: (t1.c2 < t2.c2)  (cost=0.70 rows=1)
                -> Inner hash join (t2.c1 = t1.c1)  (cost=0.70 rows=1)
                    -> Table scan on t2  (cost=0.35 rows=1)
                    -> Hash
                        -> Table scan on t1  (cost=0.35 rows=1)
    ```

- **连接条件中无索引**（**No indexes can be applied** to any join condition）

    > 有索引的情况下优化器仍偏向使用带索引表作为被驱动表的Index Nested Loop Join算法
    {: .prompt-tip }

- 若有索引的情况下使用Hash Join（A hash join can also be used when there are one or more indexes that can be used for **single-table predicates**）

## Hash Join原理

此部分参考[官方文档](https://dev.mysql.com/blog-archive/hash-join-in-mysql-8/)的解析简要剖析，以如下的多表连接来举例：

```sql
SELECT
  given_name, country_name
FROM
  persons JOIN countries ON persons.country_id = countries.country_id;
```

Hash Join 包含两个部分：

- build 构建阶段：对应build input，即驱动表，小表
- probe 探测阶段：对应probe input，即被驱动表，大表

### build阶段

> In the build phase, the server builds an **in-memory hash table** where rows from one of the inputs are stored, using the **join attribute(s) as the hash table key.**
{: .prompt-tip }

build阶段关键操作：

- 对**驱动表的连接条件**（此案例中假定驱动表为countries）作为key进行散列运算（`hash(countries.country_id)`）建立散列表
- **散列表存储在内存中**，建立过程中将查询需要的列作为value放入散列表对应key的位置（在图中示例为country_name），直到驱动表中所有rows存入散列表

![Image.tiff](https://res.craft.do/user/full/5dbbba6d-7cd5-7f7a-23e0-93b375d4df25/doc/0421481B-0DA2-4C1E-A28D-8EBD158B4337/64FAACF0-6FFD-4604-BDEF-3E22C97DD4D5_2/5Y2Pa23RVjqu3MJoFFBcggxabWyxKYO2dyKQZXYDutAz/Image.tiff)

> This input is also known as the build input, and let us assume that *‘countries’* is designated as the build input. Ideally, the server will **choose the smaller of the two inputs as the build input** (measured **in bytes**, not number of rows). Since *‘countries.country_id’* is the join condition that belongs to the build input, this is used as the key in the hash table. Once all the rows has been stored in the hash table, the build phase is done.
{: .prompt-tip }

此处再次强调了build input选择驱动表的原则，即**结果集字节数较小的表**

- **in bytes:** 即筛选后的表行数✖️所需要的行记录大小

### probe阶段

> During the probe phase, the server starts reading rows from the probe input (*‘persons’* in our example). For each row, the server probes the hash table for matching rows using the value from *‘persons.country_id’* as the lookup key. For each match, a joined row is sent to the client. In the end, the server has scanned each input only once, using constant time lookup to find matching rows between the two inputs.
{: .prompt-tip }

build阶段完成后，MySQL逐行遍历被驱动表，然后计算 join条件的hash值，并在hash表中查找，如果匹配，则输出给客户端，否则跳过。所有内表记录遍历完，则整个过程就结束了。

如图所示，MySQL 对 persons 表中每行中的 join 字段的值进行 hash 计算；`hash(persons.country_id)`，拿着计算结果到内存 hash table 中进行查找匹配，找到记录就发送给 client。

关于最终开销：**每个连接表只需要扫描一次**，并且连接条件的匹配得益于Hash算法只需O(1)的时间复杂度。所以对于内存足够存放整个散列的情况Hash Join具有非常好的效率。

![Image.tiff](https://res.craft.do/user/full/5dbbba6d-7cd5-7f7a-23e0-93b375d4df25/doc/0421481B-0DA2-4C1E-A28D-8EBD158B4337/1C9F472E-4D27-4EE8-BC29-2EC901E405BF_2/9hezptv677rlvIqoPy5bM1xcmb7xCj1jC77fkCGAgN4z/Image.tiff)

### 溢出至磁盘情况（Spill to disk）

#### build phase

hash join 构建hash表的大小是由参数`join_buffer_size` 控制的，实际生产环境中，如果驱动表的数据记录在内存中存不下怎么办？当然只能利用磁盘文件了。此时MySQL 要构建临时文件做hash join。此时的过程如下：

> When the memory goes full during the build phase, the server **writes the rest of the build input out to a number of chunk files on disk**. The server tries to **set the number of chunks so that the largest chunk fits exactly in memory** (we will see why soon), but MySQL has a hard upper limit of 128 chunk files per input. Which chunk file a row is written to is determined by calculating a hash of the join attribute. Note that in the illustration, there is a different hash function used than the one used in the in-memory build phase.
{: .prompt-tip }

溢出操作特点如下：

- 内存满时，驱动表剩余部分写入磁盘上的一系列块中
- 服务器会尝试调整块数量使得块的大小与内存大小精确匹配
- 但MySQL对每个连接表有着128的块数目上限
- 每一行写入哪一块取决于连接属性的散列值
- 注意到此处的散列函数与非溢出阶段的散列函数不同

![Image.tiff](https://res.craft.do/user/full/5dbbba6d-7cd5-7f7a-23e0-93b375d4df25/doc/0421481B-0DA2-4C1E-A28D-8EBD158B4337/B56627C3-9AD8-4F47-85D0-55AC04F63A8F_2/6kWOurM9ODBikGIeEc2enWCyswEypyFkosVvRVYlozMz/Image.tiff)

#### probe phase

> During the probe phase, the server probes for matching rows in the hash table, just like when everything fits in memory. But in addition, a row may also match one of the rows from the build input that is written to disk. **Thus, each row from the probe input is also written to a set of chunk files.** Which chunk file a row is written to is determined using the same hash function and formula that is used when the build input is written to disk. **This way, we know for sure that matching rows between the two inputs will be located in the same pair of chunk files**.
{: .prompt-tip }

探测阶段对磁盘上的块写入操作关键点：

- 被驱动表中的每一行记录都使用相同的散列函数写入到磁盘块中（这里对应的为`hash2()`函数），因为被驱动表的记录既可能与内存中的散列表匹配，也可能与磁盘块上的key匹配
- 由于散列函数相同，所以build阶段与probe阶段满足连接条件的记录必定位于相同的块编号中

![Image.tiff](https://res.craft.do/user/full/5dbbba6d-7cd5-7f7a-23e0-93b375d4df25/doc/0421481B-0DA2-4C1E-A28D-8EBD158B4337/9D4070E3-9906-4B74-8E17-90E1B2663C72_2/4p52Wbx6g2nH5ePPkb7ePZuTVaQLcTzMwyxe5oyZIU8z/Image.tiff)

#### 读取块文件

> When the probe phase is done, we start reading chunk files from disk. In general, the server does a build and probe phase using the first set of chunk files as the build and probe input. We load the first chunk file from the build input into the in-memory hash table. This explains why we want the largest chunk to fit exactly in memory; if the chunk files are too big we need to split it up in even smaller chunks. After the build chunk is loaded, we read the corresponding chunk file from the probe input and probe for matches in the hash table, just like when everything fits in memory. When the first pair of chunk files is processed, we move to the next pair of chunk files, continuing until all pair of chunk files has been processed.
{: .prompt-tip }

读取步骤如下：

- 将磁盘上build input中的第1块加载至内存上
- 读取probe input对应块中的数据
- 与内存中数据连接匹配
- 依次对后续磁盘块重复上述步骤

算法的开销主要在于：

- 两次读I/O（进行散列运算以及载入至内存两次）
- 一次写I/O（因内存不足而散列后写入对应编号的磁盘块中）

![Image.tiff](https://res.craft.do/user/full/5dbbba6d-7cd5-7f7a-23e0-93b375d4df25/doc/0421481B-0DA2-4C1E-A28D-8EBD158B4337/4B3C061B-C8EC-4F7B-B91B-980D2DAE1548_2/V5oS447xWu3zy1kKHr8pGyywxBCy3buQlB74syyJgZIz/Image.tiff)

