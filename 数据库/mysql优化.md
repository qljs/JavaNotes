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

最左前缀原则指查询条件从最左前开始且不跳过中间索引，因为聚合索引是根据建立的索引顺序来排序的，所以单单比较第二个索引是无序的，对于跳过中间索引，会导致后面的索引失效。

- ##### 不要再索引列上做任何操作，例如计算、函数、类型转换等，会导致索引失效；

- ##### 使用不等于会导致索引失效；

- ##### is null 和not null 也会导致索引失效；

- ##### like操作以通配符开头会导致索引失效，对于 like key%，相当于时常量，而 like %key则是范围查找；

- ##### 尽量避免使用in或on，mysql也可能会放弃走索引；



## 三 order by的优化

1. **order by**：MySQL支持两种排序方式，分别是**filesort**和**index**，Using index是指MySQL扫描索引本身完成排序。index效率高，fielesort效率低。
2. order by满足两种情况会使用Using index。

- order by语句使用索引最左前列；
- 使用where子句与order by子句条件列组合满足索引最左前列。

3. 尽量在索引列上完成排序，遵循索引建立（索引创建的顺序）时的最左前缀法则。
4. 如果order by的条件不在索引列上，就会产生Using filesort。
5. 能用覆盖索引尽量用覆盖索引。
6. 、group by与order by很类似，其实质是先排序后分组，遵照索引创建顺序的最左前缀法则。对于groupby的优化如果不需要排序的可以加上order by null禁止排序。

> #### Using filesort文件排序原理

- 单路排序：是一次性取出满足条件行的所有字段，然后在sort buffer中进行排序；
- 双路排序（又叫回表排序模式）：是首先根据相应的条件取出相应的排序字段和可以直接定位行数据的行 ID，然后在 sort buffer 中进行排序，排序完后需要再次取回其它需要的字段。



## 四 trace工具

使用trace工具可以查看mysql底层的优化结果：

```json
# 开启trace，trace会收集各种信息，开启后影响mysql性能
set session optimizer_trace="enabled=on",end_markers_in_json=on; 

SELECT * from user order by id;
SELECT * FROM information_schema.OPTIMIZER_TRACE;

{
  "steps": [
    {
      "join_preparation": { # 第一阶段：SQL准备阶段
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `user`.`id` AS `id`,`user`.`username` AS `username`,`user`.`password` AS `password` from `user` order by `user`.`username`"
          }
        ] /* steps */
      } /* join_preparation */
    },
    {
      "join_optimization": { # SQL优化阶段
        "select#": 1,
        "steps": [
          {
            "substitute_generated_columns": {
            } /* substitute_generated_columns */
          },
          {
            "table_dependencies": [
              {
                "table": "`user`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ] /* depends_on_map_bits */
              }
            ] /* table_dependencies */
          },
          {
             "rows_estimation": [ #预估表的访问成本
              {
                "table": "`user`",
                "table_scan": { # 全表扫描情况
                  "rows": 1,    # 扫描行数
                  "cost": 0.25  # 查询成本
                } /* table_scan */
              }
            ] /* rows_estimation */
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ] /* plan_prefix */,
                "table": "`user`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "rows_to_scan": 1,
                      "access_type": "scan",
                      "resulting_rows": 1,
                      "cost": 0.35,
                      "chosen": true,
                      "use_tmp_table": true
                    }
                  ] /* considered_access_paths */
                } /* best_access_path */,
                "condition_filtering_pct": 100,
                "rows_for_plan": 1,
                "cost_for_plan": 0.35,
                "sort_cost": 1,
                "new_cost_for_plan": 1.35,
                "chosen": true
              }
            ] /* considered_execution_plans */
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": null,
              "attached_conditions_computation": [
              ] /* attached_conditions_computation */,
              "attached_conditions_summary": [
                {
                  "table": "`user`",
                  "attached": null
                }
              ] /* attached_conditions_summary */
            } /* attaching_conditions_to_tables */
          },
          {
            "optimizing_distinct_group_by_order_by": {
              "simplifying_order_by": {
                "original_clause": "`user`.`username`",
                "items": [
                  {
                    "item": "`user`.`username`"
                  }
                ] /* items */,
                "resulting_clause_is_simple": true,
                "resulting_clause": "`user`.`username`"
              } /* simplifying_order_by */
            } /* optimizing_distinct_group_by_order_by */
          },
          {
            "reconsidering_access_paths_for_index_ordering": {
              "clause": "ORDER BY",
              "steps": [
              ] /* steps */,
              "index_order_summary": {
                "table": "`user`",
                "index_provides_order": false,
                "order_direction": "undefined",
                "index": "unknown",
                "plan_changed": false
              } /* index_order_summary */
            } /* reconsidering_access_paths_for_index_ordering */
          },
          {
            "finalizing_table_conditions": [
            ] /* finalizing_table_conditions */
          },
          {
            "refine_plan": [
              {
                "table": "`user`"
              }
            ] /* refine_plan */
          },
          {
            "considering_tmp_tables": [
              {
                "adding_sort_to_table_in_plan_at_position": 0
              } /* filesort */
            ] /* considering_tmp_tables */
          }
        ] /* steps */
      } /* join_optimization */
    },
    {
      "join_execution": {
        "select#": 1,
        "steps": [
          {
            "sorting_table_in_plan_at_position": 0,
            "filesort_information": [
              {
                "direction": "asc",
                "table": "`user`",
                "field": "username"
              }
            ] /* filesort_information */,
            "filesort_priority_queue_optimization": {
              "usable": false,
              "cause": "not applicable (no LIMIT)"
            } /* filesort_priority_queue_optimization */,
            "filesort_execution": [
            ] /* filesort_execution */,
            "filesort_summary": {
              "memory_available": 262144,
              "key_size": 4093,
              "row_size": 4093,
              "max_rows_per_buffer": 63,
              "num_rows_estimate": 1213,
              "num_rows_found": 1,
              "num_initial_chunks_spilled_to_disk": 0,
              "peak_memory_used": 32776,
              "sort_algorithm": "none",
              "unpacked_addon_fields": "max_length_for_sort_data",
              "sort_mode": "<varlen_sort_key, rowid>"
            } /* filesort_summary */
          }
        ] /* steps */
      } /* join_execution */
    }
  ] /* steps */
}
```



