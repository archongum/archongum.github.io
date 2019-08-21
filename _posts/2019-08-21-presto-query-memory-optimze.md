---
layout: post
title: "Presto Query Memory Optimize"
subtitle: "Presto查询内存优化，可缓解内存不足的症状"
author: "Archon"
header-img: "img/header.jpg"
catalog: true
tags:
  - java
  - presto
  - database
  - data warehouse
  - big data
---

# 使用条件

- Hive v1 bucketing table: v1版本的分桶表（v2没测试，presto对hive 3.x的支持目前还在进行中）
> 其他支持分桶的数据源connector，需要实现presto特定的方法  
> @david: Assuming it’s hashing as in Hive, and two tables bucketed the same way are compatible, then that could in theory be implemented in the Kudu connector.  
> The connector needs to expose the bucketing and splits to the engine in a specific way.

---

# 原理

Presto的[Grouped Execution][1]特性。

根据**相同字段**（orderid）分桶（bucketing）且**分桶数量相同**的两个表（orders，orders_item），
在通过orderid进行join的时候，由于两个表相同的orderid都分到相同id的桶里，所以是可以独立进行join以及聚合计算的（参考MapReduer的partition过程）。

通过控制并行处理桶的数量来限制内存的占用。

计算理论占用的内存：优化后的内存占用=原内存占用/表的桶数量*并行处理桶的数量

---

# 测试环境

- Ubuntu 14.04
- PrestoSQL-317
- Hive connector (Hive 3.1)
- TPCH connector

---

# [测试步骤][2]

使用Hive作为默认的数据源连接（免写hive前缀）

## 1 建表

```sql
-- 复制数据到hive
create table orders as select * from tpch.sf1.orders;

-- drop table test_grouped_join1;
CREATE TABLE test_grouped_join1
WITH (bucket_count = 13, bucketed_by = ARRAY['key1']) as
SELECT orderkey key1, comment value1 FROM orders;

-- drop table test_grouped_join2;
CREATE TABLE test_grouped_join2
WITH (bucket_count = 13, bucketed_by = ARRAY['key2']) as
SELECT orderkey key2, comment value2 FROM orders;

-- drop table test_grouped_join3;
CREATE TABLE test_grouped_join3
WITH (bucket_count = 13, bucketed_by = ARRAY['key3']) as
SELECT orderkey key3, comment value3 FROM orders;
```


## 2 测试不使用Grouped Execution特性

```sql
-- 默认
set session colocated_join=false;
set session grouped_execution=false;

-- 查看执行计划
-- explain analyze
explain (TYPE DISTRIBUTED)
SELECT key1, value1, key2, value2, key3, value3
FROM test_grouped_join1
JOIN test_grouped_join2
ON key1 = key2
JOIN test_grouped_join3
ON key2 = key3
```

执行计划结果（太长，可忽略）

```
Fragment 0 [SINGLE]
    Output layout: [key1, value1, key1, value2, key1, value3]
    Output partitioning: SINGLE []
    Stage Execution Strategy: UNGROUPED_EXECUTION
    Output[key1, value1, key2, value2, key3, value3]
    │   Layout: [key1:bigint, value1:varchar(79), key1:bigint, value2:varchar(79), key1:bigint, value3:varchar(79)]
    │   Estimates: {rows: 1500000 (268.28MB), cpu: 1.85G, memory: 204.60MB, network: 447.13MB}
    │   key2 := key1
    │   key3 := key1
    └─ RemoteSource[1]
           Layout: [key1:bigint, value1:varchar(79), value2:varchar(79), value3:varchar(79)]

Fragment 1 [hive:buckets=13, hiveTypes=[bigint]]
    Output layout: [key1, value1, value2, value3]
    Output partitioning: SINGLE []
    Stage Execution Strategy: UNGROUPED_EXECUTION
    InnerJoin[("key1" = "key3")][$hashvalue, $hashvalue_34]
    │   Layout: [key1:bigint, value1:varchar(79), value2:varchar(79), value3:varchar(79)]
    │   Estimates: {rows: 1500000 (242.53MB), cpu: 1.85G, memory: 204.60MB, network: 204.60MB}
    │   Distribution: PARTITIONED
    ├─ InnerJoin[("key1" = "key2")][$hashvalue, $hashvalue_31]
    │  │   Layout: [key1:bigint, value1:varchar(79), $hashvalue:bigint, value2:varchar(79)]
    │  │   Estimates: {rows: 1500000 (178.85MB), cpu: 971.52M, memory: 102.30MB, network: 102.30MB}
    │  │   Distribution: PARTITIONED
    │  ├─ ScanProject[table = hive:test:test_grouped_join1 bucket=13, grouped = false]
    │  │      Layout: [key1:bigint, value1:varchar(79), $hashvalue:bigint]
    │  │      Estimates: {rows: 1500000 (102.30MB), cpu: 89.43M, memory: 0B, network: 0B}/{rows: 1500000 (102.30MB), cpu: 191.73M, memory: 0B, network: 0B}
    │  │      $hashvalue := "combine_hash"(bigint '0', COALESCE("$operator$hash_code"("key1"), 0))
    │  │      key1 := key1:bigint:0:REGULAR
    │  │      value1 := value1:varchar(79):1:REGULAR
    │  └─ LocalExchange[HASH][$hashvalue_31] ("key2")
    │     │   Layout: [key2:bigint, value2:varchar(79), $hashvalue_31:bigint]
    │     │   Estimates: {rows: 1500000 (102.30MB), cpu: 396.33M, memory: 0B, network: 102.30MB}
    │     └─ RemoteSource[2]
    │            Layout: [key2:bigint, value2:varchar(79), $hashvalue_32:bigint]
    └─ LocalExchange[HASH][$hashvalue_34] ("key3")
       │   Layout: [key3:bigint, value3:varchar(79), $hashvalue_34:bigint]
       │   Estimates: {rows: 1500000 (102.30MB), cpu: 396.33M, memory: 0B, network: 102.30MB}
       └─ RemoteSource[3]
              Layout: [key3:bigint, value3:varchar(79), $hashvalue_35:bigint]

Fragment 2 [hive:buckets=13, hiveTypes=[bigint]]
    Output layout: [key2, value2, $hashvalue_33]
    Output partitioning: hive:buckets=13, hiveTypes=[bigint] [key2]
    Stage Execution Strategy: UNGROUPED_EXECUTION
    ScanProject[table = hive:test:test_grouped_join2 bucket=13, grouped = false]
        Layout: [key2:bigint, value2:varchar(79), $hashvalue_33:bigint]
        Estimates: {rows: 1500000 (102.30MB), cpu: 89.43M, memory: 0B, network: 0B}/{rows: 1500000 (102.30MB), cpu: 191.73M, memory: 0B, network: 0B}
        $hashvalue_33 := "combine_hash"(bigint '0', COALESCE("$operator$hash_code"("key2"), 0))
        key2 := key2:bigint:0:REGULAR
        value2 := value2:varchar(79):1:REGULAR

Fragment 3 [hive:buckets=13, hiveTypes=[bigint]]
    Output layout: [key3, value3, $hashvalue_36]
    Output partitioning: hive:buckets=13, hiveTypes=[bigint] [key3]
    Stage Execution Strategy: UNGROUPED_EXECUTION
    ScanProject[table = hive:test:test_grouped_join3 bucket=13, grouped = false]
        Layout: [key3:bigint, value3:varchar(79), $hashvalue_36:bigint]
        Estimates: {rows: 1500000 (102.30MB), cpu: 89.43M, memory: 0B, network: 0B}/{rows: 1500000 (102.30MB), cpu: 191.73M, memory: 0B, network: 0B}
        $hashvalue_36 := "combine_hash"(bigint '0', COALESCE("$operator$hash_code"("key3"), 0))
        key3 := key3:bigint:0:REGULAR
        value3 := value3:varchar(79):1:REGULAR
```

## 3 测试使用Grouped Execution特性

```sql
set session colocated_join=true;
set session grouped_execution=true;
-- 并行处理桶的数量：0为一次性处理全部
set session concurrent_lifespans_per_task=1;
-- 此属性设为默认，其作用不在这里说明
set session dynamic_schedule_for_grouped_execution=false;

-- 查看执行计划
-- explain (TYPE DISTRIBUTED)
explain analyze
SELECT key1, value1, key2, value2, key3, value3
FROM test_grouped_join1
JOIN test_grouped_join2
ON key1 = key2
JOIN test_grouped_join3
ON key2 = key3
```

执行计划结果（太长，可忽略）

```
Fragment 0 [SINGLE]
    Output layout: [key1, value1, key1, value2, key1, value3]
    Output partitioning: SINGLE []
    Stage Execution Strategy: UNGROUPED_EXECUTION
    Output[key1, value1, key2, value2, key3, value3]
    │   Layout: [key1:bigint, value1:varchar(79), key1:bigint, value2:varchar(79), key1:bigint, value3:varchar(79)]
    │   Estimates: {rows: 1500000 (268.28MB), cpu: 1.65G, memory: 204.60MB, network: 242.53MB}
    │   key2 := key1
    │   key3 := key1
    └─ RemoteSource[1]
           Layout: [key1:bigint, value1:varchar(79), value2:varchar(79), value3:varchar(79)]

Fragment 1 [hive:buckets=13, hiveTypes=[bigint]]
    Output layout: [key1, value1, value2, value3]
    Output partitioning: SINGLE []
    Stage Execution Strategy: FIXED_LIFESPAN_SCHEDULE_GROUPED_EXECUTION
    InnerJoin[("key1" = "key3")][$hashvalue, $hashvalue_33]
    │   Layout: [key1:bigint, value1:varchar(79), value2:varchar(79), value3:varchar(79)]
    │   Estimates: {rows: 1500000 (242.53MB), cpu: 1.65G, memory: 204.60MB, network: 0B}
    │   Distribution: PARTITIONED
    ├─ InnerJoin[("key1" = "key2")][$hashvalue, $hashvalue_31]
    │  │   Layout: [key1:bigint, value1:varchar(79), $hashvalue:bigint, value2:varchar(79)]
    │  │   Estimates: {rows: 1500000 (178.85MB), cpu: 869.21M, memory: 102.30MB, network: 0B}
    │  │   Distribution: PARTITIONED
    │  ├─ ScanProject[table = hive:test:test_grouped_join1 bucket=13, grouped = true]
    │  │      Layout: [key1:bigint, value1:varchar(79), $hashvalue:bigint]
    │  │      Estimates: {rows: 1500000 (102.30MB), cpu: 89.43M, memory: 0B, network: 0B}/{rows: 1500000 (102.30MB), cpu: 191.73M, memory: 0B, network: 0B}
    │  │      $hashvalue := "combine_hash"(bigint '0', COALESCE("$operator$hash_code"("key1"), 0))
    │  │      key1 := key1:bigint:0:REGULAR
    │  │      value1 := value1:varchar(79):1:REGULAR
    │  └─ LocalExchange[HASH][$hashvalue_31] ("key2")
    │     │   Layout: [key2:bigint, value2:varchar(79), $hashvalue_31:bigint]
    │     │   Estimates: {rows: 1500000 (102.30MB), cpu: 294.03M, memory: 0B, network: 0B}
    │     └─ ScanProject[table = hive:test:test_grouped_join2 bucket=13, grouped = true]
    │            Layout: [key2:bigint, value2:varchar(79), $hashvalue_32:bigint]
    │            Estimates: {rows: 1500000 (102.30MB), cpu: 89.43M, memory: 0B, network: 0B}/{rows: 1500000 (102.30MB), cpu: 191.73M, memory: 0B, network: 0B}
    │            $hashvalue_32 := "combine_hash"(bigint '0', COALESCE("$operator$hash_code"("key2"), 0))
    │            key2 := key2:bigint:0:REGULAR
    │            value2 := value2:varchar(79):1:REGULAR
    └─ LocalExchange[HASH][$hashvalue_33] ("key3")
       │   Layout: [key3:bigint, value3:varchar(79), $hashvalue_33:bigint]
       │   Estimates: {rows: 1500000 (102.30MB), cpu: 294.03M, memory: 0B, network: 0B}
       └─ ScanProject[table = hive:test:test_grouped_join3 bucket=13, grouped = true]
              Layout: [key3:bigint, value3:varchar(79), $hashvalue_34:bigint]
              Estimates: {rows: 1500000 (102.30MB), cpu: 89.43M, memory: 0B, network: 0B}/{rows: 1500000 (102.30MB), cpu: 191.73M, memory: 0B, network: 0B}
              $hashvalue_34 := "combine_hash"(bigint '0', COALESCE("$operator$hash_code"("key3"), 0))
              key3 := key3:bigint:0:REGULAR
              value3 := value3:varchar(79):1:REGULAR
```

---

# 分析

表的桶数量为13（设为t）一个表读到内存之后是102MB，所以一个桶占用内存=102MB/13=7.8MB（设为m）。

测试Presto为单机，-Xmx=1GB，单个query最大占用（query.max-memory-per-node）为102MB（设为a，默认0.1*Max JVM大小）。

最大并行处理桶的数量（设为n）

上述的SQL join了3个表（数据相同），所以

<img src="https://latex.codecogs.com/gif.latex?n&space;=&space;\frac{a}{m&space;*&space;3}&space;=&space;\frac{102MB}{7.8MB&space;*&space;3}&space;\approx&space;4.4" title="n = \frac{a}{m * 3} = \frac{102MB}{7.8MB * 3} \approx 4.4" />

`concurrent_lifespans_per_task`设置小于4.4才能不OOM

测试情况核实：
当设置`concurrent_lifespans_per_task=5`的时候

```
SQL Error [131079]: Query failed (#20190821_054413_00220_r4jkt): Query exceeded per-node user memory limit of 102.40MB [Allocated: 102.38MB, Delta: 59.11kB, Top Consumers: {HashBuilderOperator=102.38MB}]
```

**注意：这是理论值，仅供参考价值。**（受“分桶不可能做到平均”等因素影响）

---

# 参考文档

- [Presto Unlimited: MPP SQL Engine at Scale][1]
- [TestHiveIntegrationSmokeTest][2]

[1]: https://prestodb.github.io/blog/2019/08/05/presto-unlimited-mpp-database-at-scale
[2]: https://github.com/prestosql/presto/blob/master/presto-hive/src/test/java/io/prestosql/plugin/hive/TestHiveIntegrationSmokeTest.java#L2832
