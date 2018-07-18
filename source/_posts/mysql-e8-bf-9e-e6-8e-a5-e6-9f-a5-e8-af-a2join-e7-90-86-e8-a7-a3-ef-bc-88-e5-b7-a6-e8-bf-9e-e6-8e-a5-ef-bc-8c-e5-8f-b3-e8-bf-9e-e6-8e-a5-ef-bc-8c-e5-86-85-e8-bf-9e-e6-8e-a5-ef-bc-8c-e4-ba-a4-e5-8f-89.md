---
title: mysql 连接查询Join理解
comments: true
tags:
  - mysql 连接查询Join理解
categories:
  - mysql 连接查询Join理解
url: 345.html
id: 345
date: 2017-06-07 19:28:56
---

在开发中连接查询是被经常使用到的，连接查询包括内连接，外连接和交叉连接，连接查询通过连接运算符可以实现一条sql查询多个表，连接查询也是关系型数据库模型最主要的特点。在 像mysql，oracle等关系型数据库的设计中，我们常把一个实体的信息存放到一张表，通过外键，中间表或者是某种属性将多个实体的关系保存起来，。我们查询的时侯可以通过连接查询根据实体的关系查询出存放在多个表中的实体的信息。

sql连接关系：

![](http://www.zzcode.cn/wp-content/uploads/2017/06/mysql.png)

从多表查询join运算符的家族构成来看，多表查询可以分为 内查询，交叉连接，和外连接。其中外连接包括左连接，右连接，和全连接。但是mysql中没有全连接，可以通过union来实现全连接。全连接就是取两个集合的交集。

数据准备：

我准备了三张表，分别用于演示三种连接查询连接。附上[github](https://github.com/GeXyu/node/blob/master/sql/17_6_7/sql.sql)地址。

**外连接**
-------

外连接分为三种：左外连接，右外连接，全外连接，通常我们省略outer 这个关键字。写成：LEFT/RIGHT/FULL JOIN。

### **左连接（left join） **

**我所理解的左连接，就是以左表为基表，匹配右表的数据。如果基表的数据在另一张表没有记录。那么在查询的结果集中列显示为NULL，反之 右表没有匹配到左表的数据，则不会在结果集中显示。**

我们有一张学生表，用来存放学生的信息，还有一张投票表，存放学生投票的信息。

![](http://www.zzcode.cn/wp-content/uploads/2017/06/mysql-1.png)

![](http://www.zzcode.cn/wp-content/uploads/2017/06/2.png)

我们以左连接查询出学生信息及他们投票的信息。

SELECT v.*,s.*  FROM student s LEFT JOIN voter v  
ON s.stu\_id = v.voter\_id;

![](http://www.zzcode.cn/wp-content/uploads/2017/06/mysql-2.png)我们学生表有10条记录而投票信息表有13条记录，这里以学生表为基表通过查询出13条数据，这也就是**右表没有匹配到左表的数据，则不会在结果集中显示。**

### **右查询**

右连接和左连接唯一不同的就是左连接以join字句左表的表为基表而右查询以join子句右边的表为基表。其实左右查询实际上就是基表不同，结果以基表为准而已。

SELECT v.*,s.* 
FROM student s RIGHT JOIN voter v  ON s.stu\_id = v.voter\_id;

![](http://www.zzcode.cn/wp-content/uploads/2017/06/mysql-3.png)可以看出 以右表（投票信息表）为基表，查询出14条记录而且在没有匹配到学生表的记录时，显示的是NULL值，这也说明了**如果基表的数据在另一张表没有记录。那么在查询的结果集中列显示为NULL。**

### **全连接（full join）**

在mysql中没有full join连接运算符。全连接其实就是取两个表的并集，我们可以使用union来实现全连接。但是在union中两个查询的集合列数和集合类型必须是相同的，所以表结构无法演示。

### **交叉连接（cross join）**

交叉联接返回左表中的所有行，左表中的每一行与右表中的所有行组合。交叉联接也称作笛卡尔积。

select v.*,s.*  from student s cross join voter v

### **内连接（inner join）**

这里我们要使用到第三张表dept表，其实**内连接就是使用inner join运算发使表自己连接自己**。

![](http://www.zzcode.cn/wp-content/uploads/2017/06/mysql-4.png)

如果我们查询出子公司和父公司的名字而部门ID 和上级部门ID在同一张表。这时候我们可以使用自连接，当然也可以使用子查询。

SELECT d1.dept\_id, d1.dept\_name,(
	SELECT d2.dept\_name FROM dept d2 WHERE d1.dept\_parent = d2.dept_id
	) "上级部门"
FROM dept d1 ;

![](http://www.zzcode.cn/wp-content/uploads/2017/06/mysql-5.png)

当然，也可以使用自连接

SELECT  d1.dept\_id,d1.dept\_name,d2.dept_name "上级部门"
FROM dept d1 JOIN dept d2 ON d1.dept\_parent = d2.dept\_id;

![](http://www.zzcode.cn/wp-content/uploads/2017/06/mysql-7.png)

在这里，使用子查询和自连接的区别就是：因为国家电力公司是老大，没有上级部门也没有匹配到上级部门。所以自连接会过滤掉这条记录。而子查询不会过滤掉而是把上级部门处理为null。

**自连接使用最多的就是上表对权形结构的查询**