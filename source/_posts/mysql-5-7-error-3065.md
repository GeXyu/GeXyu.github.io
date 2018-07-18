---
title: mysql5.7导致的ERROR 3065
comments: true
tags:
  - mysql
categories:
  - mysql
date: 2018-05-30 21:15:39
---

最近迁移服务器时把MySQL装到5.7，MYSQL5.7版本比之前版本的语法更严谨，如果GROUP BY的字段没有在select中出现，他会报3065错误。
### 解决这个问题有两种方法
- 在select中加入group by排序的字段
- 修改mysql 配置文件
```sql
[mysqld]
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
```
然后重启数据库。

### sqlModel
关于SQLModel在[官方文档](https://dev.mysql.com/doc/refman/5.6/en/sql-mode.html)有专门的介绍，其作用就是配置SQL的检查等级，这个检查分为SQL语句检查和数据验证检查。
- SQL语法检查
  - ONLY_FULL_GROUP_BY
    在ORDER BY聚合操作中， 如果在SELECT中的列、HAVING或者ORDER BY子句的列，没有在GROUP BY中出现，那么这个SQL是不合法的，反之同样。
  - ANSI_QUOTES
    启用 ANSI_QUOTES 后，不能用双引号来引用字符串，因为它被解释为识别符，作用与 ` 一样。
    设置它以后，`update t set f1="" ...，`会报 `Unknown column ‘’ in ‘field list `这样的语法错误。

- 数据检查
  - NO_ZERO_DATE
    认为日期 `‘0000-00-00’` 非法，与是否设置后面的严格模式有关。
    1.如果设置了严格模式，则 `NO_ZERO_DATE` 自然满足。但如果是 `INSERT IGNORE` 或 `UPDATE IGNORE`，`’0000-00-00’`依然允许且只显示warning
    2.如果在非严格模式下，设置了`NO_ZERO_DATE`，效果与上面一样，`’0000-00-00’`允许但显示warning；如果没有设置`NO_ZERO_DATE，no warning`，当做完全合法的值。

  - NO_ENGINE_SUBSTITUTION
    使用 `ALTER TABLE`或`CREATE TABLE` 指定 `ENGINE` 时， 需要的存储引擎被禁用或未编译，该如何处理。启用`NO_ENGINE_SUBSTITUTION`时，那么直接抛出错误；不设置此值时，`CREATE`用默认的存储引擎替代，`ATLER`不进行更改，并抛出一个 `warning` .
  - STRICT_TRANS_TABLES
    设置它，表示启用严格模式。
    注意 `STRICT_TRANS_TABLES` 不是几种策略的组合，单独指 `INSERT、UPDATE`出现少值或无效值该如何处理  

 
以上也不是全部的约束，只是挑几个常用的。具体还是要参考[官方文档](https://dev.mysql.com/doc/refman/5.6/en/sql-mode.html)
