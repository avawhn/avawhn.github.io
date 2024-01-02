---
title: Oracle ROW_NUMBER用法
categories: 
  - 数据库
  - Oracle
tags: [Oracle]
date: 2023-11-21 17:00:13
---

- [官网地址](#官网地址)
- [语法](#语法)
- [作用](#作用)
- [示例](#示例)

### 官网地址

[ROW_NUMBER 函数官网](https://docs.oracle.com/cd/E11882_01/server.112/e41084/functions156.htm#SQLRF06100)

[分析函数官网](https://docs.oracle.com/cd/E11882_01/server.112/e41084/functions004.htm#i81407)

### 语法

{% asset_img row_number.gif row_number.gif %}

```sql
ROW_NUMBER( )
   OVER ([ query_partition_clause ] order_by_clause)
```

### 作用

ROW_NUMBER 按照 order_by_clause 中指定的有序行序列，从 1 开始，为应用该函数的每一条记录（分区中的每一条记录或查询返回的每一条记录）分配一个唯一的编号。

通过在检索指定范围内 ROW_NUMBER 值的查询中嵌套使用 ROW_NUMBER 的子查询，可以从内部查询的结果中找到精确的行子集。通过使用该函数，可以实现顶部-N、底部-N 和内部-N 报表。为了获得一致的结果，查询必须确保确定的排序顺序。

使用 ROW_NUMBER 或任何其他解析函数都不能嵌套解析函数 expr。不过，可以为 expr 使用其他内置函数表达式。有关 expr 有效形式的信息，请参阅 "[关于 SQL 表达式](https://docs.oracle.com/cd/E11882_01/server.112/e41084/expressions001.htm#i1002626)"。

### 示例

下面的示例在 hr.employees 表中找出每个部门中薪酬最高的三名员工。对于员工人数少于 3 人的部门，返回的行数将少于 3 行。

```sql
SELECT department_id, first_name, last_name, salary
FROM
(
  SELECT
    department_id, first_name, last_name, salary,
    ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary desc) rn
  FROM employees
)
WHERE rn <= 3
ORDER BY department_id, salary DESC, last_name;
```

下面的示例是对 sh.sales 表的连接查询。它查找 1999 年最畅销的五种产品在 2000 年的销售额，并比较 2000 年和 1999 年的差异。在每个分销渠道中计算十种最畅销产品。

```sql
SELECT sales_2000.channel_desc, sales_2000.prod_name,
       sales_2000.amt amt_2000,  top_5_prods_1999_year.amt amt_1999,
       sales_2000.amt  - top_5_prods_1999_year.amt amt_diff
FROM
/* The first subquery finds the 5 top-selling products per channel in year 1999. */
  (SELECT channel_desc, prod_name, amt
   FROM
   (
     SELECT channel_desc, prod_name, sum(amount_sold) amt,
       ROW_NUMBER () OVER (PARTITION BY channel_desc
                           ORDER BY SUM(amount_sold) DESC) rn
     FROM sales, times, channels, products
     WHERE sales.time_id = times.time_id
       AND times.calendar_year = 1999
       AND channels.channel_id = sales.channel_id
       AND products.prod_id = sales.prod_id
     GROUP BY channel_desc, prod_name
   )
   WHERE rn <= 5
  ) top_5_prods_1999_year,
/* The next subquery finds sales per product and per channel in 2000. */
  (SELECT channel_desc, prod_name, sum(amount_sold) amt
     FROM sales, times, channels, products
     WHERE sales.time_id = times.time_id
       AND times.calendar_year = 2000
       AND channels.channel_id = sales.channel_id
       AND products.prod_id = sales.prod_id
     GROUP BY channel_desc, prod_name
  ) sales_2000
WHERE sales_2000.channel_desc = top_5_prods_1999_year.channel_desc
  AND sales_2000.prod_name = top_5_prods_1999_year.prod_name
ORDER BY sales_2000.channel_desc, sales_2000.prod_name
;
CHANNEL_DESC    PROD_NAME                                          AMT_2000   AMT_1999   AMT_DIFF
--------------- --------------==-------------------------------- ---------- ---------- ----------
Direct Sales     17" LCD w/built-in HDTV Tuner                     628855.7 1163645.78 -534790.08
Direct Sales     Envoy 256MB - 40GB                               502938.54  843377.88 -340439.34
Direct Sales     Envoy Ambassador                                2259566.96 1770349.25  489217.71
Direct Sales     Home Theatre Package with DVD-Audio/Video Play  1235674.15 1260791.44  -25117.29
Direct Sales     Mini DV Camcorder with 3.5" Swivel LCD           775851.87 1326302.51 -550450.64
Internet         17" LCD w/built-in HDTV Tuner                     31707.48   160974.7 -129267.22
Internet         8.3 Minitower Speaker                            404090.32  155235.25  248855.07
Internet         Envoy 256MB - 40GB                                28293.87  154072.02 -125778.15
Internet         Home Theatre Package with DVD-Audio/Video Play   155405.54  153175.04     2230.5
Internet         Mini DV Camcorder with 3.5" Swivel LCD            39726.23  189921.97 -150195.74
Partners         17" LCD w/built-in HDTV Tuner                    269973.97  325504.75  -55530.78
Partners         Envoy Ambassador                                1213063.59  614857.93  598205.66
Partners         Home Theatre Package with DVD-Audio/Video Play   700266.58  520166.26  180100.32
Partners         Mini DV Camcorder with 3.5" Swivel LCD           404265.85  520544.11 -116278.26
Partners         Unix/Windows 1-user pack                         374002.51  340123.02   33879.49

15 rows selected.
```
