---
layout: post
title: "coding的坑之Mysql"
date: 2017-09-04 22:34:21 +0800
tags: tips
---

# charset
今天出现的一个奇怪的问题，mysql按照一个类型是varchar的字段排序，这个字段有'10001002710030005245'这种纯数字的，也有'ANS201603020283'这种以字母开头的，那么通过orderby正序排列的时候，我期待的是所有纯数字的排在最前边，但是事实是：  
![](/_pic/Screenshot from 2017-09-07 10-38-20.png)  
![](/_pic/Screenshot from 2017-09-07 10-38-59.png)  
把最后那条诡异的数据的LOAN_NO手工输入，更新后，再次正序排列，这条数据就成为了
![](/_pic/Screenshot from 2017-09-07 10-52-53.png)  
百度后，发现了一个解决方案：`SELECT * FROM {$tableName} order by  CONVERT(${columnName} USING gbk);`  
如此排序问题解决。  
问题来了，我的建表语句中已经设定了charset=utf8，为什么数据库中还会存在其他编码的字符呢？请教了下DBA同学，有一个可能是输入时强制指定了字符编码，且指定的字符编码不为utf8.  
不管这个问题是怎样产生的，一个通用的解决方案是，所有的数据库使用统一的编码格式，即目前通用的UTF8，或者utf8md2就可以避免这些诡异的编码问题  

# alias  
今天线上sql查询数据时sql报错，看下报错的sql：

~~~
<select id="queryData">
    SELECT <include refid="columns"></include>
    FROM table1 t1
    <if test="condition1">
        INNER JOIN table2 t2 ON t1.column1 = t2.column1
    </if>
    <if test="condition2">
        INNER JOIN table3 t3 ON t1.column1 = t3.column1
    </if>  
    LEFT JOIN table4 t4 ON t1.column1 = t4.column1   
</select>
~~~

错误日志：

~~~
### Error querying database.  Cause: com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: Not unique table/alias: 'kr'
### The error may exist in lixincui-core-sqlmap/lixincui/Workspace.xml
### The error may involve defaultParameterMap
### The error occurred while setting parameters
### Cause: com.mysql.jdbc.exceptions.jdbc4.MySQLSyntaxErrorException: Not unique table/alias: 'kr'
~~~

在mysql中，对数据表的别名是不能够重复的，否则就会出现上边的问题

