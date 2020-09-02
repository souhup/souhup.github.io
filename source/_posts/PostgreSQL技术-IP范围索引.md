---
title: PostgreSQL技术-IP范围索引
date: 2020-09-02 23:40:00
categories:
  - technique
tags:
  - postgres
  - sql
  - ip range index
---

## PostgreSQL技术-IP范围索引

本文来源于自己的经验，但也主要参考一篇[文章](https://github.com/digoal/blog/blob/master/201701/20170126_02.md)

### 场景描述

在业务上，有时需要根据一个IP地址查询其城市，经纬度，运营商等信息, 那么该如何快速的查询该IP的信息.

比如，我们以IP2LOCATION家的一个免费数据库[LITE IP-COUNTRY Database](https://lite.ip2location.com/database/ip-country)为例, 该数据提供了IP段起始地址，IP段结束地址,国家代码, 国家名称.

如果你下载时遇到了一些困难，那么可以点击[这里](/download/IP2LOCATION-LITE-DB1.CSV.zip)

### 方案1. BTREE索引-int8

我们按照其官方下载页面提供的PostgreSQL建表建议:
```sql
CREATE TABLE ip2location_db1_test1(
	ip_from bigint NOT NULL,
	ip_to bigint NOT NULL,
	country_code character(2) NOT NULL,
	country_name character varying(64) NOT NULL,
	CONSTRAINT ip2location_db1_pkey PRIMARY KEY (ip_from, ip_to)
);
```

基准测试

构建文件: `./test-sql1.sql`

```sql
\set ip random(0, 2147483647)
select *
from ip2location_db1_test1
where ip_from <= :ip
  and ip_to >= :ip;
```

执行测试:

```bash
pgbench -M simple -c 4 -j 4 -f ./test-sql1.sql -n -T 30 -h localhost -U postgres test
```

测试结果

```
transaction type: ./test-sql1.sql
scaling factor: 1
query mode: simple
number of clients: 4
number of threads: 4
duration: 30 s
number of transactions actually processed: 39007
latency average = 3.077 ms
tps = 1299.848793 (including connections establishing)
tps = 1299.933199 (excluding connections establishing)
```

### 方案2. BTREE索引-int4

为了让字段占用空间更小, 我们在方案1的基础上选用int4 (int4即int)

建表:

```sql
create table ip2location_db1_test2
(
    ip_from      int         not null,
    ip_to        int         not null,
    country_code char(2)     not null,
    country_name varchar(64) not null
);
create index ip2location_db1_test2_pkey on ip2location_db1_test2 using btree (ip_from, ip_to);
```

基准测试

构建文件: `./test-sql2.sql`

```sql
\set ip random(0, 2147483647)
select *
from ip2location_db1_test2
where ip_from <= :ip - 2147483648
  and ip_to >= :ip - 2147483648;
```

执行测试:

```bash
pgbench -M simple -c 4 -j 4 -f ./test-sql2.sql -n -T 30 -h localhost -U postgres test
```

测试结果

```
transaction type: ./test-sql2.sql
scaling factor: 1
query mode: simple
number of clients: 4
number of threads: 4
duration: 30 s
number of transactions actually processed: 173751
latency average = 0.691 ms
tps = 5791.490212 (including connections establishing)
tps = 5791.971583 (excluding connections establishing)
```

### 方案3. Gist索引-iprange

我们选用Gist索引, 类型选用自定义的iprange

建表:

```sql
create type iprange as range
(
    subtype=inet
);
create table ip2location_db1_test3
(
    ip_range     iprange     not null,
    country_code char(2)     not null,
    country_name varchar(64) not null
);
create index ip2location_db1_test3_key on ip2location_db1_test3 using gist (ip_range);
```

基准测试

构建文件: `./test-sql3.sql`

```sql
\set ip random(0, 2147483647)
select *
from ip2location_db1_test3
where ip_range @> '0.0.0.0'::inet + :ip;
```

执行测试:

```bash
pgbench -M simple -c 4 -j 4 -f ./test-sql3.sql -n -T 30 -h localhost -U postgres test
```

测试结果

```
transaction type: ./test-sql3.sql
scaling factor: 1
query mode: simple
number of clients: 4
number of threads: 4
duration: 30 s
number of transactions actually processed: 198253
latency average = 0.605 ms
tps = 6608.167036 (including connections establishing)
tps = 6608.692985 (excluding connections establishing)
```

### 方案4. Gist索引-int8range

我们选用Gist索引, 类型选用int8range

建表:

```sql
create table ip2location_db1_test4
(
    ip_range     int8range             not null,
    country_code character(2)          not null,
    country_name character varying(64) not null
);
create index ip2location_db1_test4_range on ip2location_db1_test4 using gist (ip_range);
```

基准测试

构建文件: `./test-sql4.sql`

```sql
\set ip random(0, 2147483647)
select *
from ip2location_db1_test4
where ip_range @> :ip;
```

执行测试:

```bash
pgbench -M simple -c 4 -j 4 -f ./test-sql4.sql -n -T 30 -h localhost -U postgres test
```

测试结果

```
transaction type: ./test-sql4.sql
scaling factor: 1
query mode: simple
number of clients: 4
number of threads: 4
duration: 30 s
number of transactions actually processed: 1070154
latency average = 0.112 ms
tps = 35671.455573 (including connections establishing)
tps = 35674.666859 (excluding connections establishing)
```

### 方案5. Gist索引-int4range

我们对方案4进行改进, 将int8range修改为int4range

建表:

```sql
create table ip2location_db1_test5
(
    ip_range     int4range             not null,
    country_code character(2)          not null,
    country_name character varying(64) not null
);
create index ip2location_db1_test5_range on ip2location_db1_test5 using gist (ip_range);
```

基准测试

构建文件: `./test-sql5.sql`

```sql
\set ip random(0, 2147483647)
select *
from ip2location_db1_test5
where ip_range @> (:ip - 2147483648)::int4;
```

执行测试:

```bash
pgbench -M simple -c 4 -j 4 -f ./test-sql5.sql -n -T 30 -h localhost -U postgres test
```

测试结果

```
ransaction type: ./test-sql5.sql
scaling factor: 1
query mode: simple
number of clients: 4
number of threads: 4
duration: 30 s
number of transactions actually processed: 980291
latency average = 0.122 ms
tps = 32675.974742 (including connections establishing)
tps = 32678.161957 (excluding connections establishing)
```

### 方案6. Gist索引-box

我们在PostgreSQL的wiki里可以找到这样一个[页面](https://wiki.postgresql.org/wiki/GeoIP_index), 按照其建议

建表:

```sql
create table ip2location_db1_test6
(
    ip_range     box                   not null,
    country_code character(2)          not null,
    country_name character varying(64) not null
);
create index ip2location_db1_test6_range on ip2location_db1_test6 using gist (ip_range);
```

基准测试

构建文件: `./test-sql6.sql`

```sql
\set ip random(0, 2147483647)
select *
from ip2location_db1_test6
where ip_range @> box(point(:ip, :ip), point(:ip, :ip));
```

执行测试:

```bash
pgbench -M simple -c 4 -j 4 -f ./test-sql6.sql -n -T 30 -h localhost -U postgres test
```

测试结果

```sql
transaction type: ./test-sql6.sql
scaling factor: 1
query mode: simple
number of clients: 4
number of threads: 4
duration: 30 s
number of transactions actually processed: 1218063
latency average = 0.099 ms
tps = 40600.765922 (including connections establishing)
tps = 40604.364144 (excluding connections establishing)
```

### 数据与索引空间

最后, 我们展示各种方案下数据和索引所占空间

查询语句
```sql
SELECT table_name                                               "表名",
       pg_size_pretty(pg_relation_size(table_name::text))       "数据占用空间",
       pg_size_pretty(pg_indexes_size(table_name::text))        "索引占用空间",
       pg_size_pretty(pg_total_relation_size(table_name::text)) "数据和索引占用空间"
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_type = 'BASE TABLE'
  and table_name like 'ip2location_db1_%';
```

查询结果

| 表名                    | 数据占用空间 | 索引占用空间 | 数据和索引占用空间 |
| :---------------------- | :----------- | :----------- | :----------------- |
| ip2location\_db1\_test1 | 12 MB        | 10 MB        | 22 MB              |
| ip2location\_db1\_test2 | 10 MB        | 4080 kB      | 14 MB              |
| ip2location\_db1\_test3 | 12 MB        | 14 MB        | 26 MB              |
| ip2location\_db1\_test4 | 13 MB        | 14 MB        | 27 MB              |
| ip2location\_db1\_test5 | 11 MB        | 11 MB        | 22 MB              |
| ip2location\_db1\_test6 | 14 MB        | 18 MB        | 32 MB              |

### 总结

由于方案6的查询性能更高, 我们最终选用它。它的数据空间和索引空间更大，但随着IP信息表越来越宽，这点劣势就会变得不明显。




