## Log-Structured File Organization

相比于向pages中保存tuples，DBMS只向其中保存log records（how the database was modified）。

为了read一个tuple，DBMS需要从后向前扫描，并recreates这个tuple。（可以建立indexes来实现在log中的快速跳转）

可以定期来compact the log

## Log-Structured Compaction

Compaction通过移除不必要的records，将大的log files合并为小的files。

- Level Compaction
- Universal Compaction

## Tuple Storage

DBMS的catalogs保存了关于tables的schema information，通过它来找出the tuple's layout。

## Data Representation

- INTEGER/BIGINT/SMALLINT/TINYINT ：C/C++ Representation
- FLOAT/REAL vs. NUMERIC/DECIMAL：IEEE-754 Standard / Fixed-point Decimals
- VARCHAR/VARBINARY/TEXT/BLOB：Header with length, followed by data bytes.
- TIME/DATE/TIMESTAMP：32/64-bit integer of (micro)seconds since Unix epoch

## Variable Precision Numbers

Inexact, variable-precision numeric type that uses the "native" C/C++ types. Examples: **FLOAT**, **REAL**/**DOUBLE**

Store directly as specified by **IEEE-754**.

通常比arbitrary precision numbers快，但是可能会有舍入误差。

## Fixed Precision Numbers

Numeric data types with (potentially) arbitrary precision and scale. Used when rounding errors are unacceptable.Example: **NUMERIC**, **DECIMAL**

Many different implementations.Example: Store in an exact, variable-length binary representation with additional meta-data.Can be less expensive if you give up arbitrary precision.

## Large Values

大多数DBMS不允许一个tuple的大小超过一个page的大小。

为了保存大于一个page的数据，DBMS使用separate **overflow** storage pages

![1.PNG](https://i.loli.net/2021/02/21/ZdOFhYT2tzQWKyA.png)

## External Value Storage

一些DBMS允许以 **BLOB** type的形式在一个external file中保存非常大的数据。

DBMS不能够操纵一个external file中的内容。（No durability protections，No transaction protections）（you can read it,but can't manipulate it）

![2.PNG](https://i.loli.net/2021/02/21/egWmOaJUHzNcDj2.png)

## System CataLogs

DBMS在内部的catalogs中保存数据库的meta-data。（Tables, columns, indexes, views | Users, permissions | Internal statistics）

几乎所有的DBMS都在另一个database中保存其他databases的catalogs（Wrap object abstraction around tuples.Specialized code for "bootstrapping" catalog tables.）

可以通过查询DBMS内部的**INFORMATION_SCHEMA** catalog来获取database的信息。（ANSI standard set of read-only views that provide info about all the tables, views, columns, and procedures in a database）

DBMSs also have non-standard shortcuts to retrieve this information.

## Database Workloads

- **On-Line Transaction Processing (OLTP)**（Fast operations that only read/update a small amount of 

  data each time. ）

- **On-Line Analytical Processing (OLAP)**（Complex queries that read a lot of data to compute 

  aggregates.）

- **Hybrid Transaction + Analytical Processing**（OLTP + OLAP together on the same database instance）

![3.PNG](https://i.loli.net/2021/02/21/ROGQF3M94sbXEcw.png)

## OLTP

**On-line Transaction Processing**

Simple queries that read/update a small amount of data that is related to a single entity in the database.

这通常是人们最先构建的那类程序。

## OLAP

**On-line Analytical Processing**

Complex queries that read large portions of the database spanning multiple entities.

You execute these workloads on the data you have collected from your OLTP application(s).

## Data Storage Models

The DBMS can store tuples in different ways that are better for either OLTP or OLAP workloads.

## N-ary Storage Model（NSM）

The DBMS stores all attributes for a single tuple contiguously in a page.

Ideal for OLTP workloads where queries tend to operate only on an individual entity and insert-heavy workloads

- 优点：快速的inserts，updates，deletes。对于需要整个tuple的那些queries有益。
- 缺点：当需要扫描table的大部分时，或扫描attributes的一个subset时，有害。

## Decomposition Storage Model（DSM）

The DBMS stores the values of a single attribute for all tuples contiguously in a page.（column store）

Ideal for OLAP workloads where read-only queries perform large scans over a subset of the table’s attributes

## Tuple Identification

- **Choice #1: Fixed-length Offsets**（Each value is the same length for an attribute.）（most common）
- **Choice #2: Embedded Tuple Ids**（Each value is stored with its tuple id in a column.）（less common）

## Conclusion

The storage manager is not entirely independent from the rest of the DBMS

It is important to choose the right storage model for the target workload:

- OLTP = Row Store

- OLAP = Column Store