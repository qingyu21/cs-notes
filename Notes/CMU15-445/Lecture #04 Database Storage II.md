[TOC]

## Log-Structured File Organization

相比于向pages中保存tuples，DBMS只向其中保存log records（how the database was modified）。

为了read一个tuple，DBMS需要从后向前扫描，并recreates这个tuple。（可以建立indexes来实现在log中的快速跳转）

可以定期来compact the log

### Log-Structured Compaction

Compaction通过移除不必要的records，将大的log files合并为小的files。

- Level Compaction
- Universal Compaction

## Tuple Storage

DBMS的catalogs保存了关于tables的schema information，通过它来找出the tuple's layout。

## Data Representation

即DBMS是如何保存一个值的bytes的

### Integers

大多数DBMS使用IEEE-754标准规定的native C++ types来保存整数

这些数是固定长度的

例如：INTEGER，BIGINT，SMALLINT，TINYINT

### Variable Precision Numbers

Inexact, variable-precision numeric type that uses the "native" C/C++ types. 

同样是固定长度的

Store directly as specified by IEEE-754.

通常比arbitrary precision numbers快（因为CPU有可以直接利用它们的指令），但是计算时可能会有rounding errors（舍入误差。因为有些数字不能被准确表示）。

例如: FLOAT，REAL/DOUBLE

### Fixed-Point Precision Numbers

可能有任意的精度和大小的数值数据类型。 当可能有舍入误差时，使用它。

定点数，它们通常以variable-length binary的形式存储（就像一个string）。带有meta-data来告诉系统例如：数据的长度，小数位应该在哪里。

因为要保证任意精度，所以会有性能损失。

例如：NUMERIC, DECIMAL

### Variable-Length Data

可以表示具有任意长度的数据类型。

通常会同时保存一个header来记录string的长度，以此来更容易去跳转到下一个value。同时也保存一个checksum。

大多数DBMS不允许一个tuple的大小超过一个page的大小。为了保存这种数据，DBMS使用separate **overflow** storage pages

![1.PNG](https://i.loli.net/2021/02/21/ZdOFhYT2tzQWKyA.png)

例如：VARCHAR，VARBINARY，TEXT，BLOB

#### External Value Storage

一些DBMS允许以 **BLOB** type的形式在一个external file中保存非常大的数据。tuple会包含一个指向这个文件的指针。

DBMS不能够操纵一个external file中的内容。（No durability protections，No transaction protections）（you can read it,but can't manipulate it）

![2.PNG](https://i.loli.net/2021/02/21/egWmOaJUHzNcDj2.png)

### Datas and Times

不同的系统区别很大。通常它们被保存为与UNIX时间戳相差的时间（以micro/milli seconds的形式）

例如：TIME，DATE，TIMESTAMP

### System Catalogs

DBMS在内部的catalogs中保存数据库的meta-data。（Tables, columns, indexes, views | Users, permissions | Internal statistics）

几乎所有的DBMS都在另一个database中保存其他databases的catalogs（Wrap object abstraction around tuples.它们用特殊的代码来“bootstrap”这些catalog tables。）

可以通过查询DBMS内部的**INFORMATION_SCHEMA** catalog来获取database的信息。（ANSI standard set of read-only views that provide info about all the tables, views, columns, and procedures in a database）

DBMSs also have non-standard shortcuts to retrieve this information.

## Database Workloads

- **On-Line Transaction Processing (OLTP)**（Fast operations that only read/update a small amount of data each time. ）

- **On-Line Analytical Processing (OLAP)**（Complex queries that read a lot of data to compute aggregates.）

- **Hybrid Transaction + Analytical Processing（HTAP）**（OLTP + OLAP together on the same database instance）

![3.PNG](https://i.loli.net/2021/02/21/ROGQF3M94sbXEcw.png)

### OLTP

**On-line Transaction Processing**

一个OLTP workload被刻画为fast，short running的操作。Simple queries that read/update a small amount of data on single entity at a time。repetitive operations。

一个OLTP workload通常会处理比reads更多的writes。

这通常是人们最先构建的那类程序。

例子：Amazon store front，用户向购物车添加物品，购买，这些行为只影响他们自己的账户。

### OLAP

**On-line Analytical Processing**

一个OLAP workload通常被刻画为long running，complex queries，reads on large portions of the database spanning multiple entities.。

在这些workloads中，可以用OLTP应用中收集的数据来执行数据分析。

例子：Amazon基于过去一个月的地理位置信息计算出被购买最多的五个商品。

### HTAP

**Hybrid Transaction + Analytical Processing**

it's like a combination which tries to do OLTP and OLAP together on the same database.

## Data Storage Models

The DBMS can store tuples in different ways that are better for either OLTP or OLAP workloads.

### N-ary Storage Model（NSM）

The DBMS stores all attributes for a single tuple contiguously in a single page.（row store）

Ideal for OLTP workloads where queries tend to operate only on an individual entity and insert-heavy workloads

- 优点：快速的inserts，updates，deletes。对于需要整个tuple的那些queries有益。
- 缺点：当需要扫描table的大部分时，或扫描attributes的一个subset时，有害（取出的不必要的数据污染了buffer pool）。

### Decomposition Storage Model（DSM）

The DBMS stores the values of a single attribute for all tuples contiguously in a page.（column store）

Ideal for OLAP workloads where read-only queries perform large scans over a subset of the table’s attributes

- 优点：减少浪费的I/O，因为DBMS只读那些需要的数据。Better query processing and data compression (因为具有相同attribute值的数据是连续存储的).
- 缺点：Slow for point queries, inserts, updates, and deletes because of tuple splitting/stitching.

#### Tuple Identification

## @

- **Choice #1: Fixed-length Offsets**（Each value is the same length for an attribute.）（most common）
- **Choice #2: Embedded Tuple Ids**（Each value is stored with its tuple id in a column.）（less common）

## Conclusion

The storage manager is not entirely independent from the rest of the DBMS

It is important to choose the right storage model for the target workload:

- OLTP = Row Store

- OLAP = Column Store