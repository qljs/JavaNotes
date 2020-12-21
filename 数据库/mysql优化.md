[TOC]



## 一 explain

用 EXPLAIN 关键字可以模拟优化器执行SQL语句，执行查询返回执行计划的信息，其中主要的列：

![image-20201221144735444](D:\JavaNotes\JavaNotes\images\mysql\explain.png)

### 1. id

id列的编号是select的序号，有几个select就有几个id，并且id越大执行优先级越高，id相同则从上向下执行，id为null最后执行。



### 2.  select_type

select_type列表示对应行是简单还是复杂的查询。

- simple：简单查询。查询不包含子查询和union；
- primary：复杂查询中最外层的 select；
- subquery：包含在 select 中的子查询（不在 from 子句中）；
- derived：包含在 from 子句中的子查询。MySQL会将结果存放在一个临时表中，也称为 派生表（derived的英文含义）。



### 3. table

table列表示操作的是哪个表。



### 4. type

type列表示关联类型或访问类型，即MySQL如何查找表中的行。**性能从优到差：system > const > eq_ref > ref > range > index > all。**

一般来说，要保证查询到range级，最好能到ref级。

- NULL：mysql能够在优化阶段分解查询语句，在执行阶段用不着再访问表或索引。

![](..\images\mysql\null.png)



- system/const：用于 primary key 或 unique key 的进行查询时，只匹配到一条数据。system相当于时const的一个特例，

![](..\images\mysql\const.png)



- eq_ref：一般是primary key或unique key索引的所有部分被连接使用 ，最多只会返回一条符合 条件的记录。

![](..\images\mysql\qe_ref.png)



- ref：一般是普通索引或者唯一性索引的部分前缀，索引要 和某个值相比较，可能会找到多个符合条件的行。

![](..\images\mysql\ref.png)



- index：全表扫描，比all 快一点，例如查询索引。

![image-20201221153310893](D:\JavaNotes\JavaNotes\images\mysql\index.png)



### 5. possible_keys

possible_keys列表示可能使用那些索引来查找，过该列是null且没有索引时，可以考虑添加索引来提高性能。



### 6. key

key列表示MySQL执行时真正走的索引，可以使用 force index来指定要采用的索引，也可以使用 ignore index来忽略索引。



### 7. key_len

该列表示MySQL走的索引的字节数，通过这个可以计算出MySQL走的索引，计算方式如下（不同的编码长度有所不同，以utf8为例）：

- char(n)：n个字节长度；
- varchar(n)：长度为3*n + 2；
- int：4个字节；
- bigint：8个字节；
- date：3个字节；
- timestamp：4个字节；
- datetime：8个字节。



### 8. ref

该列显示了在key列记录的索引中，表查找值所用到的列或常量，常见的有const（常量），字段名。



### 9. rows

该列表示大约要读取的行数，与结果的行数可能有差别。



### 10 Extra

该列展示的额外信息，常见的如下：

- **Using index**：使用覆盖索引；
- **Using where**：查询的列未被索引覆盖；
- **Using index condition**：查询的列包括非索引列，where条件中使用一个前导列；
- **Using temporary**：需要创建临时表来处理，这种情况需要优化。
- **Using filesort**：用外部排序而不是索引排序，数据较小时从内存排序，否则需要在磁盘 完成排序。这种情况下一般也是要考虑使用索引来优化的。
- **Select tables optimized away：**使用了聚合函数来访问索引字段。



## 二 某些使索引无效的操作

基于MySQL版本为5.7

- ##### 最左前缀原则

最左前缀原则指查询条件从最左前开始且不跳过中间索引，因为聚合索引是根据建立的索引顺序来排序的，所以后面索引的有序性时建立在前面索引有序性上的，对于跳过中间索引则会导致后面的索引失效。

- ##### 不要再索引列上做任何操作，例如计算、函数、类型转换等，会导致索引失效；

- ##### 使用不等于会导致索引失效；

- ##### is null 和not null 也会导致索引失效；

- ##### like操作以通配符开头会导致索引失效，对于 like key%，相当于时常量，而 like %key则是范围查找；

- ##### 尽量避免使用in或on，mysql也可能会放弃走索引；

