---
title: PostgreSQL技术-窗口函数
date: 2020-08-31 22:36:00
categories:
  - technique
tags:
  - postgres
  - sql
  - window function
---

## PostgreSQL技术-窗口函数

今天继续分享有关PostgreSQL的技术。

### 功能定义

早在[SQL:2003](http://www.wiscorp.com/sql_2003_standard.zip)中，就定义了窗口函数。 一个*窗口函数调用*表示在一个查询选择的行的某个部分上应用一个聚集类的函数。和非窗口聚集函数调用不同，这不会被约束为将被选择的行分组为一个单一的输出行 — 在查询输出中每一个行仍保持独立。不过，窗口函数能够根据窗口函数调用的分组声明（`PARTITION BY`列表）访问属于当前行所在分组中的所有行。一个窗口函数调用的语法是下列之一：

```sql
function_name ([expression [, expression ... ]]) [ FILTER ( WHERE filter_clause ) ] OVER window_name
function_name ([expression [, expression ... ]]) [ FILTER ( WHERE filter_clause ) ] OVER ( window_definition )
function_name ( * ) [ FILTER ( WHERE filter_clause ) ] OVER window_name
function_name ( * ) [ FILTER ( WHERE filter_clause ) ] OVER ( window_definition )
```

其中*`window_definition`*的语法是

```sql
[ existing_window_name ]
[ PARTITION BY expression [, ...] ]
[ ORDER BY expression [ ASC | DESC | USING operator ] [ NULLS { FIRST | LAST } ] [, ...] ]
[ frame_clause ]
```

可选的*`frame_clause`*是下列之一

```sql
{ RANGE | ROWS | GROUPS } frame_start [ frame_exclusion ]
{ RANGE | ROWS | GROUPS } BETWEEN frame_start AND frame_end [ frame_exclusion ]
```

其中*`frame_start`*和*`frame_end`*可以是下面形式中的一种

```sql
UNBOUNDED PRECEDING
offset PRECEDING
CURRENT ROW
offset FOLLOWING
UNBOUNDED FOLLOWING
```

而*`frame_exclusion`*可以是下列之一

```sql
EXCLUDE CURRENT ROW
EXCLUDE GROUP
EXCLUDE TIES
EXCLUDE NO OTHERS
```

关于更加详细的描述, 可以参考[PostgreSQL手册](https://www.postgresql.org/docs/13/sql-expressions.html#SYNTAX-WINDOW-FUNCTIONS)

### 使用场景

以下的查询将使用此处定义的数据
```sql
-- 员工薪水表
create table test_salary
(
    dep_name varchar(64), -- 部门名称
    emp_no   integer,     -- 职员编号
    salary   numeric      -- 薪水
);

-- 插入数据
insert into test_salary
values ('develop', 11, 5200),
       ('develop', 7, 4200),
       ('develop', 9, 4500),
       ('develop', 8, 6000),
       ('develop', 10, 5200),
       ('personnel', 5, 3500),
       ('personnel', 2, 3900),
       ('sales', 3, 4800),
       ('sales', 1, 5000),
       ('sales', 4, 4800);
```

#### 查询每个职员所处部门的平均薪水

查询语句:

```sql
select *, (avg(salary) over (partition by dep_name))::int dep_avg_salary
from test_salary;
```

查询结果:

| dep\_name | emp\_no | salary | dep\_avg\_salary |
| :-------- | :------ | :----- | :--------------- |
| develop   | 11      | 5200   | 5020             |
| develop   | 7       | 4200   | 5020             |
| develop   | 9       | 4500   | 5020             |
| develop   | 8       | 6000   | 5020             |
| develop   | 10      | 5200   | 5020             |
| personnel | 5       | 3500   | 3700             |
| personnel | 2       | 3900   | 3700             |
| sales     | 3       | 4800   | 4867             |
| sales     | 1       | 5000   | 4867             |
| sales     | 4       | 4800   | 4867             |

#### 查询每个职员与本部门的薪水的差值

查询语句:

```sql
select *, max(salary) over (partition by dep_name) - salary  diff_dep_max_salary
from test_salary;
```
查询结果:

| dep\_name | emp\_no | salary | diff\_dep\_max\_salary |
| :--- | :--- | :--- | :--- |
| develop | 11 | 5200 | 800 |
| develop | 7 | 4200 | 1800 |
| develop | 9 | 4500 | 1500 |
| develop | 8 | 6000 | 0 |
| develop | 10 | 5200 | 800 |
| personnel | 5 | 3500 | 400 |
| personnel | 2 | 3900 | 0 |
| sales | 3 | 4800 | 200 |
| sales | 1 | 5000 | 0 |
| sales | 4 | 4800 | 200 |

#### 查询每一个员工与本部门中薪资比自己更高的一位的差值

查询语句:

```sql
select *, min(salary) over w - salary diff_higher_salary
from test_salary
    window w as (partition by dep_name order by salary desc groups unbounded preceding exclude group);
```

查询结果:

| dep\_name | emp\_no | salary | diff\_higher\_salary |
| :-------- | :------ | :----- | :------------------- |
| develop   | 8       | 6000   | NULL                 |
| develop   | 10      | 5200   | 800                  |
| develop   | 11      | 5200   | 800                  |
| develop   | 9       | 4500   | 700                  |
| develop   | 7       | 4200   | 300                  |
| personnel | 2       | 3900   | NULL                 |
| personnel | 5       | 3500   | 400                  |
| sales     | 1       | 5000   | NULL                 |
| sales     | 3       | 4800   | 200                  |
| sales     | 4       | 4800   | 200                  |

#### 查询每一个部门的平均工资及其排名

查询语句:

```sql
select rank() over (order by avg(salary) desc ), dep_name, avg(salary)::int dep_avg_salary
from test_salary
group by dep_name;
```

查询结果:

| rank | dep\_name | dep\_avg\_salary |
| :--- | :--- | :--- |
| 1 | develop | 5020 |
| 2 | sales | 4867 |
| 3 | personnel | 3700 |

#### 查询与自身相邻工号的两位员工的薪资均值

查询语句:

```sql
select *, avg(salary) over w adj_emp_avg_salary
from test_salary
    window w as (order by emp_no range between 1 preceding and 1 following exclude current row );
```

查询结果:

| dep\_name | emp\_no | salary | adj\_emp\_avg\_salary |
| :--- | :--- | :--- | :--- |
| sales | 1 | 5000 | 3900 |
| personnel | 2 | 3900 | 4900 |
| sales | 3 | 4800 | 4350 |
| sales | 4 | 4800 | 4150 |
| personnel | 5 | 3500 | 4800 |
| develop | 7 | 4200 | 6000 |
| develop | 8 | 6000 | 4350 |
| develop | 9 | 4500 | 5600 |
| develop | 10 | 5200 | 4850 |
| develop | 11 | 5200 | 5200 |
