---
title: PostgreSQL技术-CTE与递归的使用
date: 2020-08-30 19:23:00
categories:
  - technique
tags:
  - postgres
  - sql
  - recursive
  - cte
---

## PostgreSQL技术-CTE与递归的使用

我接触PostgreSQL已有一年有余了, 为了分享自己的经验，之后会撰写一些列与之相关的文章。

### 功能定义

早在[SQL:1999](https://crate.io/docs/sql-99/en/latest/index.html)中，就定义了通用表表达式和递归查询相关功能. 以WITH开始的部分我们将其称为公共表表达式(Common Table Expression, CTE), 它提供了一种在大型查询中的辅助语句，可以被看成是定义在一个查询中的临时表,它可以将复杂的查询分解为相对简单的多个部分. 虽然WITH中的辅助语句可以是INSERT, DELETE, UPDATE, SELECT, 但我们平时更多的是使用SELECT.

当CTE中添加RECURSIVE修饰符后, 即可实现递归功能.(其实这个处理严格来说不是递归，而是迭代，但RECURSIVE是SQL标准委员会选择的术语)，从结构上看, 可以认为其存在如下形式.

```sql
WITH RECURSIVE cte_name(
    CTE_query_definition -- 非递归项
    UNION [ALL]
    CTE_query definion   -- 递归项
) SELECT * FROM cte_name;
```

递归查询求值的流程

1. 计算非递归项。对UNION（但不对UNION ALL），抛弃重复行。把所有剩余的行包括在递归查询的结果中，并且也把它们放在一个临时的工作表中。

2. 只要工作表不为空，重复下列步骤：

2.1 计算递归项，用当前工作表的内容替换递归自引用。对UNION（不是UNION ALL），抛弃重复行以及那些与之前结果行重复的行。将剩下的所有行包括在递归查询的结果中，并且也把它们放在一个临时的中间表中。

2.2 用中间表的内容替换工作表的内容，然后清空中间表。

### 使用场景

#### 模拟1到100的和

一般情况下，我们很容易想到通过`generate`完成本功能
```sql
select sum(num)
from generate_series(1, 100) nums(num);
```

但是, 通过递归CTE也可以实现
```sql
with recursive num_generate(num) as (
    values (1)
    union all
    select num + 1
    from num_generate
    where num < 100
)
select sum(num)
from num_generate;
```

#### 加速查询唯一数

有如下的场景, 我们有一百万条日志数据, 它们分为10个类别

```sql
-- 我们选择postgresql 13

-- 创建表
create table test_data
(
    category_id int, -- 类别ID
    log_id      uuid -- 日志ID
);

-- 为类别ID创建索引
create index idx_category on test_data using btree (category_id);

-- 插入模拟数据,
insert into test_data
select (random() * 100)::int, gen_random_uuid()
from generate_series(1, 1000000);
```

那么如何快速地查询每一个类别, 常见方法有如下几种

##### distinct查询

查询方式:

```sql
select distinct category_id
from test_data;
```

查询计划与耗时:

```
HashAggregate  (cost=18870.00..18871.00 rows=100 width=4) (actual time=155.989..155.997 rows=100 loops=1)
  Group Key: category_id
  Batches: 1  Memory Usage: 24kB
  ->  Seq Scan on test_data  (cost=0.00..16370.00 rows=1000000 width=4) (actual time=0.006..39.345 rows=1000000 loops=1)
Planning Time: 0.041 ms
Execution Time: 156.018 ms
```

##### group by查询

查询方式:

```sql
select category_id
from test_data
group by 1;
```

查询计划与耗时:

```
Group  (cost=12582.68..12606.51 rows=100 width=4) (actual time=58.393..60.113 rows=100 loops=1)
  Group Key: category_id
  ->  Gather Merge  (cost=12582.68..12606.01 rows=200 width=4) (actual time=58.392..60.095 rows=300 loops=1)
        Workers Planned: 2
        Workers Launched: 2
        ->  Sort  (cost=11582.66..11582.91 rows=100 width=4) (actual time=55.155..55.163 rows=100 loops=3)
              Sort Key: category_id
              Sort Method: quicksort  Memory: 29kB
              Worker 0:  Sort Method: quicksort  Memory: 29kB
              Worker 1:  Sort Method: quicksort  Memory: 29kB
              ->  Partial HashAggregate  (cost=11578.33..11579.33 rows=100 width=4) (actual time=55.126..55.134 rows=100 loops=3)
                    Group Key: category_id
                    Batches: 1  Memory Usage: 24kB
                    Worker 0:  Batches: 1  Memory Usage: 24kB
                    Worker 1:  Batches: 1  Memory Usage: 24kB
                    ->  Parallel Seq Scan on test_data  (cost=0.00..10536.67 rows=416667 width=4) (actual time=0.006..15.642 rows=333333 loops=3)
Planning Time: 0.040 ms
Execution Time: 60.135 ms
```

这里之所以比distinct更快，主要原因是利用了并行查询, 我们可以暂时把并行度调整为1

```sql
set max_parallel_workers_per_gather = 1
```

查询计划与耗时:
```sql
HashAggregate  (cost=18870.00..18871.00 rows=100 width=4) (actual time=158.472..158.480 rows=100 loops=1)
  Group Key: category_id
  Batches: 1  Memory Usage: 24kB
  ->  Seq Scan on test_data  (cost=0.00..16370.00 rows=1000000 width=4) (actual time=0.007..39.888 rows=1000000 loops=1)
Planning Time: 0.040 ms
Execution Time: 158.499 ms
```

##### 递归查询

查询方式:

```sql
with recursive rcs(category_id) as (
    select min(category_id)
    from test_data
    union all
    select (select min(test_data.category_id) from test_data where test_data.category_id > r1.category_id)
    from rcs r1
    where r1.category_id is not null
)
select count(*)
from rcs
where category_id is not null;
```

查询计划与耗时:

```
Aggregate  (cost=52.59..52.60 rows=1 width=8) (actual time=0.361..0.363 rows=1 loops=1)
  CTE rcs
    ->  Recursive Union  (cost=0.45..50.32 rows=101 width=4) (actual time=0.015..0.340 rows=101 loops=1)
          ->  Result  (cost=0.45..0.46 rows=1 width=4) (actual time=0.015..0.015 rows=1 loops=1)
                InitPlan 3 (returns $1)
                  ->  Limit  (cost=0.42..0.45 rows=1 width=4) (actual time=0.013..0.013 rows=1 loops=1)
                        ->  Index Only Scan using idx_category on test_data test_data_1  (cost=0.42..20920.42 rows=1000000 width=4) (actual time=0.012..0.012 rows=1 loops=1)
                              Index Cond: (category_id IS NOT NULL)
                              Heap Fetches: 0
          ->  WorkTable Scan on rcs r1  (cost=0.00..4.78 rows=10 width=4) (actual time=0.003..0.003 rows=1 loops=101)
                Filter: (category_id IS NOT NULL)
                Rows Removed by Filter: 0
                SubPlan 2
                  ->  Result  (cost=0.45..0.46 rows=1 width=4) (actual time=0.003..0.003 rows=1 loops=100)
                        InitPlan 1 (returns $3)
                          ->  Limit  (cost=0.42..0.45 rows=1 width=4) (actual time=0.002..0.003 rows=1 loops=100)
                                ->  Index Only Scan using idx_category on test_data  (cost=0.42..7807.09 rows=333333 width=4) (actual time=0.002..0.002 rows=1 loops=100)
                                      Index Cond: ((category_id IS NOT NULL) AND (category_id > r1.category_id))
                                      Heap Fetches: 0
  ->  CTE Scan on rcs  (cost=0.00..2.02 rows=100 width=0) (actual time=0.017..0.355 rows=100 loops=1)
        Filter: (category_id IS NOT NULL)
        Rows Removed by Filter: 1
Planning Time: 0.088 ms
Execution Time: 0.419 ms
```

#### 查询树状结构

我们定于如下树状数据

```sql
-- 创建表
create table test_data
(
    node_id       int, -- 节点ID
    child_node_id int  -- 该节点的子节点
);

-- 插入数据
insert into test_data
values (0, 1),
       (0, 2),

       (1, 10),
       (1, 11),
       (1, 12),
       (2, 20),
       (2, 21),

       (10, 101),
       (10, 102),
       (11, 110),
       (21, 210);
```

查询语句

```sql
with recursive rsc(node_id, path) as (
    values (0, '0')
    union all
    select test_data.child_node_id, format('%s -> %s', r1.path, test_data.child_node_id)
    from rsc r1
             join test_data using (node_id)
)
select path
from rsc;
```

查询结果

```sql
0
0 -> 1
0 -> 2
0 -> 1 -> 10
0 -> 1 -> 11
0 -> 1 -> 12
0 -> 2 -> 20
0 -> 2 -> 21
0 -> 1 -> 10 -> 101
0 -> 1 -> 10 -> 102
0 -> 1 -> 11 -> 110
0 -> 2 -> 21 -> 210
```

