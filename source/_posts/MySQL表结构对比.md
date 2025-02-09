---
title: MySQL表结构对比
tags:
  - MySQL
date: 2025-02-09 21:46:38
---

- [1. 背景](#1-背景)
- [2. Navicate的表结构比较功能](#2-navicate的表结构比较功能)
- [3. 表信息存储位置](#3-表信息存储位置)


## 1. 背景

公司的 MES/WMS 系统被多家厂商使用，均是部署在厂商的服务器上（厂商本地），这导致了数据库版本太多。且由于人力成本问题，厂商使用的是一套系统，开发环境也是统一数据库，并没有分开维护。这导致修改数据库的开发人员往往是多个人，当更新系统的时候，数据库的变更容易遗漏，导致报错，影响用户体验。这中情况可以通过数据库版本管理工具（例如：Flyway、Liquibase等）进行版本控制，也可以通过比较数据库的表结构然后在更新。

这里公司的情况比较简单，就是开发并测试完成后，将数据库表结构更新到指定厂商，并不涉及太多需求，所以采用的是表结构比较并同步。

关于数据库版本管理工具选型可以看下这篇文章：https://zhuanlan.zhihu.com/p/559857853

## 2. Navicate的表结构比较功能

Navicate工具就可以进行数据库表结构对比，工具 -> 结构同步

{% asset_img "Navicate结构同步功能位置.png" "Navicate结构同步功能位置" %}

这里创建 test1、test2 两个数据库，然后分别创建相同的 student 表，然后对比。

```sql
CREATE TABLE `student`  (
  `id` bigint NOT NULL COMMENT 'PK',
  `name` varchar(64) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `age` int NULL DEFAULT NULL,
  PRIMARY KEY (`id`) USING BTREE
)
```

比较的选项如图所示：

{% asset_img "Navicate表结构比较选项.png" "Navicate表结构比较选项" %}

比较结果如图所示，这时候两个表完全相同，所以没有需要修改的选项：

{% asset_img "第一次比较结果.png" "第一次比较结果" %}

修改 test1 数据库的 student 表，id 改为自动递增，name 改为非必填项，修改 age 字段类型为 tinyint 无符号类型并添加注释为“年龄”，添加 stu_no varchar(11) 字段，再进行比较，比较结果如图所示：

{% asset_img "修改后的比较结果.png" "修改后的比较结果" %}

同步表结构的脚本内容如下，可以看到对我们的修改内容都进行了同步：

{% asset_img "同步表结构的脚本.png" "同步表结构的脚本" %}

## 3. 表信息存储位置

在 `information_schema` 系统数据库中存储了数据库、表、列、索引等数据。

- `TABLES`：存储了表的基本信息，如表名、表类型、存储引擎、行格式等。
- `COLUMNS`：存储了列（字段）的基本信息，如字段名、字段类型、字段长度等。
- `STATISTICS`：存储了表的索引信息，如所属表名、索引名、索引类型等。
- `KEY_COLUMN_USAGE`：存储着约束信息（主键、外键等）

**TABLES 表结构信息**

| Field           | Type                                                               | Null | Key | Default | Extra |
|-----------------|--------------------------------------------------------------------|------|-----|---------|-------|
| TABLE_CATALOG   | varchar(64)                                                        | NO   |     | NULL    |       |
| TABLE_SCHEMA    | varchar(64)                                                        | NO   |     | NULL    |       |
| TABLE_NAME      | varchar(64)                                                        | NO   |     | NULL    |       |
| TABLE_TYPE      | enum('BASE TABLE','VIEW','SYSTEM VIEW')                            | NO   |     | NULL    |       |
| ENGINE          | varchar(64)                                                        | YES  |     | NULL    |       |
| VERSION         | int                                                                | YES  |     | NULL    |       |
| ROW_FORMAT      | enum('Fixed','Dynamic','Compressed','Redundant','Compact','Paged') | YES  |     | NULL    |       |
| TABLE_ROWS      | bigint unsigned                                                    | YES  |     | NULL    |       |
| AVG_ROW_LENGTH  | bigint unsigned                                                    | YES  |     | NULL    |       |
| DATA_LENGTH     | bigint unsigned                                                    | YES  |     | NULL    |       |
| MAX_DATA_LENGTH | bigint unsigned                                                    | YES  |     | NULL    |       |
| INDEX_LENGTH    | bigint unsigned                                                    | YES  |     | NULL    |       |
| DATA_FREE       | bigint unsigned                                                    | YES  |     | NULL    |       |
| AUTO_INCREMENT  | bigint unsigned                                                    | YES  |     | NULL    |       |
| CREATE_TIME     | timestamp                                                          | NO   |     | NULL    |       |
| UPDATE_TIME     | datetime                                                           | YES  |     | NULL    |       |
| CHECK_TIME      | datetime                                                           | YES  |     | NULL    |       |
| TABLE_COLLATION | varchar(64)                                                        | YES  |     | NULL    |       |
| CHECKSUM        | bigint                                                             | YES  |     | NULL    |       |
| CREATE_OPTIONS  | varchar(256)                                                       | YES  |     | NULL    |       |
| TABLE_COMMENT   | text                                                               | YES  |     | NULL    |       |

**COLUMNS 表结构信息**

| Field                    | Type                       | Null | Key | Default | Extra |
|--------------------------|----------------------------|------|-----|---------|-------|
| TABLE_CATALOG            | varchar(64)                | NO   |     | NULL    |       |
| TABLE_SCHEMA             | varchar(64)                | NO   |     | NULL    |       |
| TABLE_NAME               | varchar(64)                | NO   |     | NULL    |       |
| COLUMN_NAME              | varchar(64)                | YES  |     | NULL    |       |
| ORDINAL_POSITION         | int unsigned               | NO   |     | NULL    |       |
| COLUMN_DEFAULT           | text                       | YES  |     | NULL    |       |
| IS_NULLABLE              | varchar(3)                 | NO   |     |         |       |
| DATA_TYPE                | longtext                   | YES  |     | NULL    |       |
| CHARACTER_MAXIMUM_LENGTH | bigint                     | YES  |     | NULL    |       |
| CHARACTER_OCTET_LENGTH   | bigint                     | YES  |     | NULL    |       |
| NUMERIC_PRECISION        | bigint unsigned            | YES  |     | NULL    |       |
| NUMERIC_SCALE            | bigint unsigned            | YES  |     | NULL    |       |
| DATETIME_PRECISION       | int unsigned               | YES  |     | NULL    |       |
| CHARACTER_SET_NAME       | varchar(64)                | YES  |     | NULL    |       |
| COLLATION_NAME           | varchar(64)                | YES  |     | NULL    |       |
| COLUMN_TYPE              | mediumtext                 | NO   |     | NULL    |       |
| COLUMN_KEY               | enum('','PRI','UNI','MUL') | NO   |     | NULL    |       |
| EXTRA                    | varchar(256)               | YES  |     | NULL    |       |
| PRIVILEGES               | varchar(154)               | YES  |     | NULL    |       |
| COLUMN_COMMENT           | text                       | NO   |     | NULL    |       |
| GENERATION_EXPRESSION    | longtext                   | NO   |     | NULL    |       |
| SRS_ID                   | int unsigned               | YES  |     | NULL    |       |

**STATISTICS 表结构信息**

| Field         | Type          | Null | Key | Default | Extra |
|---------------|---------------|------|-----|---------|-------|
| TABLE_CATALOG | varchar(64)   | NO   |     | NULL    |       |
| TABLE_SCHEMA  | varchar(64)   | NO   |     | NULL    |       |
| TABLE_NAME    | varchar(64)   | NO   |     | NULL    |       |
| NON_UNIQUE    | int           | NO   |     | 0       |       |
| INDEX_SCHEMA  | varchar(64)   | NO   |     | NULL    |       |
| INDEX_NAME    | varchar(64)   | YES  |     | NULL    |       |
| SEQ_IN_INDEX  | int unsigned  | NO   |     | NULL    |       |
| COLUMN_NAME   | varchar(64)   | YES  |     | NULL    |       |
| COLLATION     | varchar(1)    | YES  |     | NULL    |       |
| CARDINALITY   | bigint        | YES  |     | NULL    |       |
| SUB_PART      | bigint        | YES  |     | NULL    |       |
| PACKED        | binary(0)     | YES  |     | NULL    |       |
| NULLABLE      | varchar(3)    | NO   |     |         |       |
| INDEX_TYPE    | varchar(11)   | NO   |     |         |       |
| COMMENT       | varchar(8)    | NO   |     |         |       |
| INDEX_COMMENT | varchar(2048) | NO   |     | NULL    |       |
| IS_VISIBLE    | varchar(3)    | NO   |     |         |       |
| EXPRESSION    | longtext      | YES  |     | NULL    |       |

**KEY_COLUMN_USAGE 表结构信息**

| Field                         | Type         | Null | Key | Default | Extra |
|-------------------------------|--------------|------|-----|---------|-------|
| CONSTRAINT_CATALOG            | varchar(64)  | NO   |     | NULL    |       |
| CONSTRAINT_SCHEMA             | varchar(64)  | NO   |     | NULL    |       |
| CONSTRAINT_NAME               | varchar(64)  | YES  |     | NULL    |       |
| TABLE_CATALOG                 | varchar(64)  | NO   |     | NULL    |       |
| TABLE_SCHEMA                  | varchar(64)  | NO   |     | NULL    |       |
| TABLE_NAME                    | varchar(64)  | NO   |     | NULL    |       |
| COLUMN_NAME                   | varchar(64)  | YES  |     | NULL    |       |
| ORDINAL_POSITION              | int unsigned | NO   |     | 0       |       |
| POSITION_IN_UNIQUE_CONSTRAINT | int unsigned | YES  |     | NULL    |       |
| REFERENCED_TABLE_SCHEMA       | varchar(64)  | YES  |     | NULL    |       |
| REFERENCED_TABLE_NAME         | varchar(64)  | YES  |     | NULL    |       |
| REFERENCED_COLUMN_NAME        | varchar(64)  | YES  |     | NULL    |       |
