---
title: PostgreSQL技术-索引
date: 2020-09-05 22:00:00
categories:
  - technique
tags:
  - postgres
  - sql
  - index
---

## PostgreSQL技术-索引

为了更加高效地查询数据, 我们应该考虑使用索引.

在PostgreSQL中, 我们可以通过如下地方式创建索引

```sql
CREATE [ UNIQUE ] INDEX [ CONCURRENTLY ] [ [ IF NOT EXISTS ] name ] ON [ ONLY ] table_name [ USING method ]
    ( { column_name | ( expression ) } [ COLLATE collation ] [ opclass ] [ ASC | DESC ] [ NULLS { FIRST | LAST } ] [, ...] )
    [ INCLUDE ( column_name [, ...] ) ]
    [ WITH ( storage_parameter = value [, ... ] ) ]
    [ TABLESPACE tablespace_name ]
    [ WHERE predicate ]
```

### 1. btree

#### 介绍

绝大部分场景下btree索引均适用. 该索引可用来进行范围查找, 支持以下操作符

* `>`,`>=`,`=`,`<`, `<=`
* like
* between
* in

同时，该类索引支持`多字段索引`, `只用索引的扫描`,`覆盖索引`和`唯一约束`。 

但是该索引无法使用长度较长的key. 否则就会出现: `54000] ERROR: index row requires 21792 bytes, maximum size is 8191`

#### 用法

**创建表与数据**

```sql
create table test_btree
(
    key   varchar(32),
    value int
);
insert into test_btree
select md5(random()::text), (random() * 1000000)::int
from generate_series(1, 1000000);
```

**创建索引**

```sql
create index test_btree_key on test_btree (key) include (value);
```

**查询示例**

```sql
select sum(value)
from test_btree
where key in ('323816e2d4bb47d85d669b90089a2382', 'f3180734688c85d1cf5b173b261e9521');
```
**查询计划**

```
Aggregate  (cost=8.89..8.90 rows=1 width=8) (actual time=0.024..0.024 rows=1 loops=1)
  ->  Index Only Scan using test_btree_key on test_btree  (cost=0.42..8.88 rows=2 width=4) (actual time=0.016..0.021 rows=2 loops=1)
"        Index Cond: (key = ANY ('{323816e2d4bb47d85d669b90089a2382,f3180734688c85d1cf5b173b261e9521}'::text[]))"
        Heap Fetches: 0
Planning Time: 0.052 ms
Execution Time: 0.038 ms
```

### 2. hash

#### 介绍

hash索引适用于字段value非常长的场景. 只支持等值连接。 

#### 用法

**创建表**

```sql
create table test_hash
(
    key   text,
    value int
);
insert into test_hash
select md5(random()::text), (random() * 1000000)::int
from generate_series(1, 10000);
```

**创建索引**

```sql
create index test_hash_key on test_hash using hash (key);
```

**查询示例**

```sql
select key, value
from test_hash
where key = '70913971e3676ba202faba804a12f9bd';
```

**查询计划**

```
Index Scan using test_hash_key on test_hash  (cost=0.00..8.02 rows=1 width=37) (actual time=0.010..0.011 rows=1 loops=1)
  Index Cond: (key = '70913971e3676ba202faba804a12f9bd'::text)
Planning Time: 0.037 ms
Execution Time: 0.021 ms
```

### 3. GiST

#### 介绍

Gist表示通用搜索树, 它是一种平衡的树结构的访问方法. 它可以用来解决空间，范围等几何问题. 例如，对于各种数据类型，执行各类操作符

| 名称           | 索引数据类型   | 可索引操作符                                                 | 排序操作符 |
| -------------- | -------------- | ------------------------------------------------------------ | ---------- |
| `box_ops`      | `box`          | `&&` `&>` `&<` `&<|` `>>` `<<` `<<|` `<@` `@>` `@` `|&>` `|>>` `~` `~=` |            |
| `circle_ops`   | `circle`       | `&&` `&>` `&<` `&<|` `>>` `<<` `<<|` `<@` `@>` `@` `|&>` `|>>` `~` `~=` | `<->`      |
| `inet_ops`     | `inet`, `cidr` | `&&` `>>` `>>=` `>` `>=` `<>` `<<` `<<=` `<` `<=` `=`        |            |
| `point_ops`    | `point`        | `>>` `>^` `<<` `<@` `<@` `<@` `<^` `~=`                      | `<->`      |
| `poly_ops`     | `polygon`      | `&&` `&>` `&<` `&<|` `>>` `<<` `<<|` `<@` `@>` `@` `|&>` `|>>` `~` `~=` | `<->`      |
| `range_ops`    | 任何范围类型   | `&&` `&>` `&<` `>>` `<<` `<@` `-|-` `=` `@>` `@>`            |            |
| `tsquery_ops`  | `tsquery`      | `<@` `@>`                                                    |            |
| `tsvector_ops` | `tsvector`     | `@@`                                                         |            |

#### 用法

**创建表**

```
create table test_gist
(
    range int4range,
    value int,
    exclude using gist (range with &&)
);
insert into test_gist
with a as (
    select (random() * 1000)::int v
    from generate_series(1, 100)
)
select int4range(v, v + 11), (random() * 100)::int
from a
on conflict do nothing;
```

**查询示例**

```sql
select range, value
from test_gist
where range @> 400;
```

**查询计划**

```
Bitmap Heap Scan on test_gist  (cost=4.19..13.66 rows=6 width=36) (actual time=0.049..0.049 rows=1 loops=1)
  Recheck Cond: (range @> 400)
  Heap Blocks: exact=1
  ->  Bitmap Index Scan on test_gist_range_excl  (cost=0.00..4.19 rows=6 width=0) (actual time=0.009..0.009 rows=1 loops=1)
        Index Cond: (range @> 400)
Planning Time: 0.142 ms
Execution Time: 0.068 ms
```

### 4. SP-GiST

#### 介绍

SP-GiST是空间分割的(Space-Partitioned)GiST的省略语. 他的使用场景与GiST类似.  它支持二维点的SP-Gist操作符如下

* `<<`, 是否严格在左
* `>>`, 是否严格在右
* `~=`, 与...相同
* `<@`, 包含或在...上
* `<^`, 低于(允许接触)
* `>^`,高于(允许接触)

#### 用法

**创建表**

```sql
create table test_spgist
(
    key   point,
    value int
);
insert into test_spgist
select point(random() * 1000, random() * 1000), (random() * 10000)::int
from generate_series(1, 1000000);
```

**创建索引**

```sql
create index test_spgist_key on test_spgist using spgist (key);
```

**查询示例**

```sql
select key, value
from test_spgist
where key >> point(100, 100);
```

**查询计划**

```
Bitmap Heap Scan on test_spgist  (cost=2951.28..10571.28 rows=100000 width=20) (actual time=64.614..119.100 rows=899566 loops=1)
"  Recheck Cond: (key >> '(100,100)'::point)"
  Heap Blocks: exact=6370
  ->  Bitmap Index Scan on test_spgist_key  (cost=0.00..2926.28 rows=100000 width=0) (actual time=64.098..64.099 rows=899566 loops=1)
"        Index Cond: (key >> '(100,100)'::point)"
Planning Time: 0.048 ms
Execution Time: 135.882 ms
```

### 5. GIN

#### 介绍

GIN为通用倒排索引, 用于处理索引项为组合值的情况. 比如所搜包含特定词的文档, 或拥有特定值的数组. 对于一维数组, 它支持以下的操作符

* `<@`, 被包含
* `@>`, 包含
* `=`, 等于
* `&&`, 重叠

#### 用法

**创建表**

```sql
create table test_gin
(
    uuid uuid,
    tags integer[]
);
insert into test_gin
select gen_random_uuid(), array [(random() * 100)::int,(random() * 100)::int,(random() * 100)::int]
from generate_series(1, 1000);
```

**创建索引**

```sql
create index test_gin_tags on test_gin using gin (tags);
```

**查询示例**

```sql
select uuid, tags
from test_gin
where tags @> array [10, 2];
```

**查询计划**

```
Bitmap Heap Scan on test_gin  (cost=12.00..16.02 rows=1 width=49) (actual time=0.018..0.019 rows=1 loops=1)
"  Recheck Cond: (tags @> '{10,2}'::integer[])"
  Heap Blocks: exact=1
  ->  Bitmap Index Scan on test_gin_tags  (cost=0.00..12.00 rows=1 width=0) (actual time=0.012..0.012 rows=1 loops=1)
"        Index Cond: (tags @> '{10,2}'::integer[])"
Planning Time: 0.078 ms
Execution Time: 0.065 ms
```

### 6. BRIN

#### 介绍

BRIN索引(块范围索引, Block Range Indexes)维护不同范围内数据块的统计信息. 相比于btree索引, BRIN索引占用的空间更小.该索引用于线性相关较强的精确和范围查询.

#### 用法

**创建表**

```sql
create table test_brin
(
    key   int,
    value int
);
insert into test_brin
select g, (random() * 1000000)::int
from generate_series(1, 1000000) g;
```

**创建索引**

```sql
create index test_brin_key on test_brin (key) include (value);
```

**查询示例**

```sql
select key, value
from test_brin
where key between 11100 and 11150;
```

**查询计划**

```sql
Index Only Scan using test_brin_key on test_brin  (cost=0.42..5.42 rows=50 width=8) (actual time=0.014..0.018 rows=51 loops=1)
  Index Cond: ((key >= 11100) AND (key <= 11150))
  Heap Fetches: 0
Planning Time: 0.046 ms
Execution Time: 0.029 ms
```

### 7. bloom

#### 介绍

bloom索引虽然只能用于等值查询，但可通过多列字段索引高效的查询每一种可能的情况.

#### 用法

**创建表**

```sql
create table test_bloom as
select md5(random()::text) as i1,
       md5(random()::text) as i2,
       md5(random()::text) as i3
from generate_series(1, 1000000);
```

**创建索引**

```sql
create index test_bloom_all on test_bloom using bloom (i1, i2, i3);
```

**查询示例**

```sql
select i1,i2,i3
from test_bloom
where i2 = '2efeb41703124b0c5a3ac7cfbc86029f';
```

**查询计划**

```sql
Bitmap Heap Scan on test_bloom  (cost=15348.00..15352.01 rows=1 width=99) (actual time=5.933..8.660 rows=1 loops=1)
  Recheck Cond: (i2 = '2efeb41703124b0c5a3ac7cfbc86029f'::text)
  Rows Removed by Index Recheck: 4380
  Heap Blocks: exact=3890
  ->  Bitmap Index Scan on test_bloom_all  (cost=0.00..15348.00 rows=1 width=0) (actual time=5.615..5.616 rows=4381 loops=1)
        Index Cond: (i2 = '2efeb41703124b0c5a3ac7cfbc86029f'::text)
Planning Time: 0.039 ms
Execution Time: 8.678 ms
```







