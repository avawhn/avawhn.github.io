---
title: MySQL中CHAR与VARCHAR(8.0)
date: 2023-12-15 13:50:33
categories: 
  - 数据库
  - MySQL
tags: [MYSQL]
---

### 一、官网内容
CHAR 长度 **0 - 255**，存储 CHAR 值时，会**用空格将其填充到指定长度**。检索 CHAR 值时，除非启用 [PAD_CHAR_TO_FULL_LENGTH](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html#sqlmode_pad_char_to_full_length) SQL 模式，否则会**删除尾部空格**。

VARCHAR 长度可变，范围 **0 - 65535**。VARCHAR 的有效最大长度取决于最大行大小（65535 字节，所有列共享）和使用的字符集。请参见第 [表列数和行大小的限制](https://dev.mysql.com/doc/refman/8.0/en/column-count-limit.html)。

VARCHAR 使用**1或2个字节+数据**的形式存储。如果数值不超过 255 字节，列使用 1byte；如果数值可能超过 255 字节，列使用 2byte。VARCHAR 值在存储时没有填充。根据标准 SQL，在存储和检索值时，**会保留尾部空格**。

如果**未启用严格 SQL 模式**，且为 CHAR 或 VARCHAR 列分配的值超过了列的最大长度，则会**截断该值以适应该列**，并生成警告。对于非空格字符的截断，可以通过使用严格 SQL 模式导致错误（而不是警告）并抑制值的插入。请参阅第 [服务器 SQL 模式](https://dev.mysql.com/doc/refman/8.0/en/sql-mode.html)。

对于 VARCHAR 列，无论使用何种 SQL 模式，在插入前都会截断超过列长度的尾部空格，并生成警告。对于 CHAR 列，无论使用何种 SQL 模式，都会无声地截断插入值中多余的尾部空格。


下表显示了在 CHAR(4) 和 VARCHAR(4) 列中存储各种字符串值的结果，从而说明了 CHAR 和 VARCHAR 之间的区别（假设列使用单字节字符集，如 latin1）。

下面使用 "_" 下划线代替空格

|值|CHAR(4)|大小|VACHAR(4)|大小|
|--|----|--|--|--|
|''|'____'|4 bytes|''|1byte|
|'ab'|'ab__'|4 bytes|'ab'|3 bytes|
|'abcd'|'abcd'|4 bytes|'abcd'|5bytes|
|'abcdefgh'|'abcd'|4bytes|'abcd'|5bytes|

表格最后一行显示的存储值仅适用于不使用严格 SQL 模式的情况；如果启用了严格模式，则不会存储超出列长度的值，并会导致错误。

InnoDB 会将长度大于或等于 **768** 字节的固定长度字段编码为可变长度字段，这些字段可以在页面外存储。例如，如果字符集的最大字节长度大于 3，CHAR(255) 列的长度就会超过 **768** 字节，utf8mb4 就是如此。

CHAR 与 VARCHAR 尾部空格：

```sql
mysql> CREATE TABLE vc (v VARCHAR(4), c CHAR(4));
Query OK, 0 rows affected (0.01 sec)

mysql> INSERT INTO vc VALUES ('ab  ', 'ab  ');
Query OK, 1 row affected (0.00 sec)

mysql> SELECT CONCAT('(', v, ')'), CONCAT('(', c, ')') FROM vc;
+---------------------+---------------------+
| CONCAT('(', v, ')') | CONCAT('(', c, ')') |
+---------------------+---------------------+
| (ab  )              | (ab)                |
+---------------------+---------------------+
1 row in set (0.06 sec)
```

可以从结果中发现：VARCHAR 保留了尾部的空格，而 CHAR 删除了尾部的空格

CHAR、VARCHAR 和 TEXT 列中的值会根据分配给列的字符集校对进行排序和比较。

MySQL 校对中，基于 UCA 9.0.0 及更高版本的 Unicode 的 pad 属性为 **PAD SPACE**，它们的 pad 属性是 **NO PAD**。 

要确定校对的 PAD 属性，请使用 INFORMATION_SCHEMA COLLATIONS 表，该表有一个 PAD_ATTRIBUTE 列。

对于非二进制字符串（CHAR、VARCHAR 和 TEXT 值），字符串校对 pad 属性决定在比较中如何处理字符串末尾的空格。NO PAD 字串校对将尾部空格视为比较中的重要字符，就像其他字符一样。PAD SPACE 在比较中将拖尾空格视为不重要字符；在比较字符串时不考虑拖尾空格。请参阅[比较中的尾部空格处理](https://dev.mysql.com/doc/refman/8.0/en/charset-binary-collations.html#charset-binary-collations-trailing-space-comparisons)。服务器 SQL 模式对拖尾空格的比较行为没有影响。

{% blockquote %}
有关 MySQL 字符集和校对的更多信息，请参阅 [字符集、校对、Unicode](https://dev.mysql.com/doc/refman/8.0/en/charset.html)。有关存储要求的更多信息，请参阅 [数据类型存储要求](https://dev.mysql.com/doc/refman/8.0/en/storage-requirements.html)。
{% endblockquote %}


### 二、总结

- CHAR：固定长度，设定长度为 0 - 255，存储时用空格填充，查询时删除空格
- VARCHAR 长度可变，设定长度为 0 - 65535，不会填充，查询时保留空格
- VARCHAR格式为 1或2字节 + 数据，值不超过255字节，使用1个字节，超过使用2个字节
- 插入超过长度的值进行截取并警告，在SQL严格模式下报错
- CHAR 与 VARCHAR 指定的大小是字符的数量，而非字节的大小
- CHAR 的检索速度要快于 VARCHAR，因为不需要存储额外的信息
- InnoDB 会将长度大于或等于 768 字节的固定长度字段编码为可变长度字段，这些字段可以在页面外存储