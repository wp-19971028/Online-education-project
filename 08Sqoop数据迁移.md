# 概述

> sqoop是apache旗下一款**“Hadoop和关系数据库服务器之间传送数据”**的工具。
>
> **导入数据**：MySQL，Oracle导入数据到Hadoop的HDFS、HIVE、HBASE等数据存储系统；
>
> **导出数据：**从Hadoop的HDFS、HIVE中导出数据到关系数据库mysql等

![1621344234826](./assets\1621344234826.png)

# sqoop1与sqoop2架构对比

## sqoop1架构

> Sqoop1以Client客户端的形式存在和运行。没有任务时是没有进程存在的。
>
> ![img](./assets\wps1-1621344323794.jpg) 

# sqoop2架构

> sqoop2是以B/S服务器的形式去运行的，始终会有Server服务端进程在运行。
>
> ![img](./assets\wps2-1621344364957.jpg) 

# **工作机制**

将导入或导出命令翻译成mapreduce程序来实现。

# sqoop安装

## 安装步骤

> 在上一节中已经安装

## 验证启动

~~~sh
[root@hadoop01 ~]# sqoop -version
Warning: /opt/cloudera/parcels/CDH-6.2.1-1.cdh6.2.1.p0.1425774/bin/../lib/sqoop/../accumulo does not exist! Accumulo imports will fail.
Please set $ACCUMULO_HOME to the root of your Accumulo installation.
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/cloudera/parcels/CDH-6.2.1-1.cdh6.2.1.p0.1425774/jars/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/opt/cloudera/parcels/CDH-6.2.1-1.cdh6.2.1.p0.1425774/jars/log4j-slf4j-impl-2.8.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.slf4j.impl.Log4jLoggerFactory]
No such sqoop tool: -version. See 'sqoop help'.
您在 /var/spool/mail/root 中有新邮件
~~~

# 数据抽取的两种方式

> 对于Mysql数据的采集，通常使用Sqoop来进行。
>
> 通过Sqoop将关系型数据库数据到Hive有两种方式，一种是原生Sqoop API，一种是使用HCatalog API。两种方式略有不同。
>
> HCatalog方式与Sqoop方式的参数基本都是相同，只是个别不一样，都是可以实现Sqoop将数据抽取到Hive。

## 区别

> **数据格式支持**
>
> Sqoop方式支持的数据格式较少，HCatalog支持的较多，比如Sqoop方式不支持ORC格式的表，但是HCatalog支持。
>
> **数据覆盖**
>
> Sqoop方式允许数据覆盖，HCatalog不允许数据覆盖，每次都只是追加。
>
> **字段名**
>
> Sqoop方式比较随意，不要求源表和目标表字段相同(字段名称和个数都可以不相同)，它抽取的方式是将字段按顺序插入，比如目标表有3个字段，源表有一个字段，它会将数据插入到Hive表的第一个字段，其余字段为NULL。但是HCatalog不同，源表和目标表字段名需要相同，字段个数可以不相等，如果字段名不同，抽取数据的时候会报NullPointerException错误。HCatalog抽取数据时，会将字段对应到相同字段名的字段上，哪怕字段个数不相等。

## Sqoop方式

```sql
sqoop import \
--hive-import \
--connect 'jdbc:mysql://localhost:3306/test' \
--username 'root' \
--password '123456' \
--query " select order_no from driver_action where  \$CONDITIONS" \
--hive-database test \
--hive-table driver_action \
--hive-partition-key pt \
--hive-partition-value 20190901 \
--null-string '' \
--null-non-string '' \
--num-mappers 1 \
--target-dir /tmp/test \
--delete-target-dir
```

## HCatalog方式

```sql
sqoop import \
--connect jdbc:mysql://localhost:3306/test\
--username 'root' \
--password 'root' \
--query "SELECT order_no FROM driver_action  WHERE \$CONDITIONS" \
--hcatalog-database test \
--hcatalog-table driver_action \
--hcatalog-partition-keys pt \
--hcatalog-partition-values 20200104 \
--hcatalog-storage-stanza 'stored as orcfile tblproperties ("orc.compress"="SNAPPY")' \
--num-mappers 1
```

# **项目选型**

> 因为项目采用的是**ORC** File文件格式，sqoop原始方式并不支持，因此使用HCatalog方式来进行数据的导入导出。

# Sqoop数据导入

> “导入工具”导入单个表从RDBMS到HDFS。表中的每一行被视为HDFS的记录。所有记录都存储为文本文件的文本数据（或者Avro、sequence文件等二进制数据）

## 列举出所有的数据库

```sh
/usr/bin/sqoop help

# 命令行查看帮助
/usr/bin/sqoop list-databases --help

# 列出主机所有的数据库
/usr/bin/sqoop list-databases --connect jdbc:mysql://192.168.88.151:3306/ --username root --password 123456

# 查看某一个数据库下面的所有数据表
/usr/bin/sqoop list-tables --connect jdbc:mysql://192.168.88.151:3306/hue --username root --password 123456
```

## 完整数据导入

### 造数据

```sql
-- 创建数据库
CREATE DATABASE test;
-- 使用数据库
USE test;

-- 创建emp表
CREATE TABLE emp
(
    id     INT     NOT NULL PRIMARY KEY,
    NAME   VARCHAR(32) NULL,
    deg    VARCHAR(32) NULL,
    salary INT         NULL,
    dept   VARCHAR(32) NULL
);
-- 给emp表插入数据
INSERT INTO emp (id, NAME, deg, salary, dept) VALUES (1201, 'gopal', 'manager', 50000, 'TP');
INSERT INTO emp (id, NAME, deg, salary, dept) VALUES (1202, 'manisha', 'Proof reader', 50000, 'TP');
INSERT INTO emp (id, NAME, deg, salary, dept) VALUES (1203, 'khalil', 'php dev', 30000, 'AC');
INSERT INTO emp (id, NAME, deg, salary, dept) VALUES (1204, 'prasanth', 'php dev', 30000, 'AC');
INSERT INTO emp (id, NAME, deg, salary, dept) VALUES (1205, 'kranthi', 'admin', 20000, 'TP');

-- 创建emp_add
CREATE TABLE emp_add
(
    id     INT         NOT NULL
        PRIMARY KEY,
    hno    VARCHAR(32) NULL,
    street VARCHAR(32) NULL,
    city   VARCHAR(32) NULL
);
-- 插入数据
INSERT INTO emp_add (id, hno, street, city) VALUES (1201, '288A', 'vgiri', 'jublee');
INSERT INTO emp_add (id, hno, street, city) VALUES (1202, '108I', 'aoc', 'sec-bad');
INSERT INTO emp_add (id, hno, street, city) VALUES (1203, '144Z', 'pgutta', 'hyd');
INSERT INTO emp_add (id, hno, street, city) VALUES (1204, '78B', 'old city', 'sec-bad');
INSERT INTO emp_add (id, hno, street, city) VALUES (1205, '720X', 'hitec', 'sec-bad');
-- 创建 emp_conn表
CREATE TABLE emp_conn
(
    id    INT         NOT NULL
        PRIMARY KEY,
    phno  VARCHAR(32) NULL,
    email VARCHAR(32) NULL
);
-- 插入数据
INSERT INTO emp_conn (id, phno, email) VALUES (1201, '2356742', 'gopal@tp.com');
INSERT INTO emp_conn (id, phno, email) VALUES (1202, '1661663', 'manisha@tp.com');
INSERT INTO emp_conn (id, phno, email) VALUES (1203, '8887776', 'khalil@ac.com');
INSERT INTO emp_conn (id, phno, email) VALUES (1204, '9988774', 'prasanth@ac.com');
INSERT INTO emp_conn (id, phno, email) VALUES (1205, '1231231', 'kranthi@tp.com');

```

### 导入数据库表数据到HDFS

下面的命令用于从MySQL数据库服务器中的emp表导入HDFS。

**注意，mysql地址必须为服务器IP，不能是localhost或者机器名。**

```sql
sqoop import \
--connect jdbc:mysql://192.168.88.151:3306/test \
--password 123456 \
--username root \
--table emp \
--m 1 
```

为了验证在HDFS导入的数据，请使用以下命令查看导入的数据

```sql
hdfs  dfs  -ls  /user/root/emp

Found 2 items
-rw-r--r--   3 root supergroup          0 2021-05-18 22:34 /user/root/emp/_SUCCESS
-rw-r--r--   3 root supergroup        151 2021-05-18 22:34 /user/root/emp/part-m-00000

```

也可以通过hdfs 把数据下载下来看

![1621348777002](./assets\1621348777002.png)

![1621348742497](./assets\1621348742497.png)

### 导入数据到指定目录

在导入表数据到HDFS时，使用Sqoop导入工具，我们可以指定目标目录。

使用参数 **--target-dir**来指定导出目的地，

使用参数**--delete-target-dir**来判断导出目录是否已存在，如果存在就删掉

```sql
sqoop import \
--connect jdbc:mysql://192.168.88.151:3306/test \
--password 123456 \
--username root \
--delete-target-dir \
--table emp \
--target-dir /wp/sqoop/emp \
--m 1 
```

![1621349142348](./assets\1621349142348.png)

### 导入到hdfs指定目录并指定字段之间的分隔符

```sql
sqoop import \
--connect jdbc:mysql://192.168.88.151:3306/test \
--password 123456 \
--username root \
--delete-target-dir \
--table emp \
--target-dir /wp/sqoop/emp2 \
--m 1 \
--fields-terminated-by '\t'
```

![1621349525565](./assets\1621349525565.png)

![1621349576872](./assets\1621349576872.png)

### 导入关系表到HIVE

- 准备Hive数据库与表

```sql
CREATE DATABASE sqooptohive;

use sqooptohive;

CREATE TABLE sqooptohive.emp_hive(
    id int,
    name STRING,
    deg STRING,
    salary INT,
    dept STRING
) ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
STORED AS ORC;
```

![1621350070891](./assets\1621350070891.png)

- 导入数据

```sh
sqoop import \
--connect jdbc:mysql://192.168.88.151:3306/test \
--password 123456 \
--username root \
--table emp \
--fields-terminated-by '\t' \
--hcatalog-database sqooptohive \
--hcatalog-table emp_hive \
-m 1
```

```sql
SELECT * FROM  sqooptohive.emp_hive;
```

![1621350479223](./assets\1621350479223.png)

# 条件部分导入

## wher导入到HDFS

> 我们可以导入表时使用Sqoop导入工具，"where"子句的一个子集。它执行在各自的数据库服务器相应的SQL查询，并将结果存储在HDFS的目标目录。
>
> where子句的语法如下。
>
> --where <condition>
>
> 按照条件进行查找，通过—where参数来查找表emp_add当中city字段的值为sec-bad的所有数据导入到hdfs上面去

```sh
sqoop import \
--connect jdbc:mysql://192.168.88.151:3306/test \
--password 123456 \
--username root \
--table emp_add \
--target-dir /wp/sqoop/emp_add \
-m 1 \
--delete-target-dir \
--where "city = 'sec-bad'"
```

![1621350925692](./assets\1621350925692.png)

![1621350900897](./assets\1621350900897.png)

## sql语句查找导入hdfs

我们还可以通过 –query参数来指定我们的sql语句，通过sql语句来过滤我们的数据进行导入

```sql
sqoop import \
--connect jdbc:mysql://192.168.88.151:3306/test \
--password 123456 \
--username root \
--delete-target-dir \
-m 1 \
--query 'select phno from emp_conn where 1 = 1 and $CONDITIONS' \
--target-dir /wp/sqoop/emp_conn
```

查看hdfs的数据

```sh
hdfs dfs -text /wp/sqoop/emp_conn/part*
```

![1621351271080](./assets\1621351271080.png)

## 增量导入数据到Hive表

```sql
sqoop import \
--connect jdbc:mysql://192.168.88.151:3306/test \
--password 123456 \
--username root \
--query "select * from emp where id>1203 and  \$CONDITIONS" \
--fields-terminated-by '\t' \
--hcatalog-database sqooptohive \
--hcatalog-table emp_hive \
-m 1
```

```sql
SELECT * FROM  sqooptohive.emp_hive;
```

![1621351628759](./assets\1621351628759.png)

Sqoop 的数据导出

- 创建数据表

```sql
CREATE TABLE `emp_out` (
  `id` INT(11) DEFAULT NULL,
  `name` VARCHAR(100) DEFAULT NULL,
  `deg` VARCHAR(100) DEFAULT NULL,
  `salary` INT(11) DEFAULT NULL,
  `dept` VARCHAR(10) DEFAULT NULL
) ENGINE=INNODB DEFAULT CHARSET=utf8;
```

![1621351796382](./assets\1621351796382.png)

- 执行导出命令

```sh
sqoop export \
--connect jdbc:mysql://192.168.88.151:3306/test \
--password 123456 \
--username root \
--table emp_out \
--hcatalog-database sqooptohive \
--hcatalog-table emp_hive \
-m 1
```

- 验证数据

![1621352249094](./assets\1621352249094.png)

# sqoop的常用参数

| 参数                                                         | 说明                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| --connect                                                    | 连接关系型数据库的URL                                        |
| --username                                                   | 连接数据库的用户名                                           |
| --password                                                   | 连接数据库的密码                                             |
| --driver                                                     | JDBC的driver class                                           |
| --query或--e <statement>                                     | 将查询结果的数据导入，使用时必须伴随参--target-dir，--hcatalog-table，如果查询中有where条件，则条件后必须加上$CONDITIONS关键字。如果使用双引号包含sql，则$CONDITIONS前要加上\以完成转义：\$CONDITIONS |
| --hcatalog-database                                          | 指定HCatalog表的数据库名称。如果未指定，default则使用默认数据库名称。提供 --hcatalog-database不带选项--hcatalog-table是错误的。 |
| --hcatalog-table                                             | 此选项的参数值为HCatalog表名。该--hcatalog-table选项的存在表示导入或导出作业是使用HCatalog表完成的，并且是HCatalog作业的必需选项。 |
| --create-hcatalog-table                                      | 此选项指定在导入数据时是否应自动创建HCatalog表。表名将与转换为小写的数据库表名相同。 |
| --hcatalog-storage-stanza 'stored as orc tblproperties ("orc.compress"="SNAPPY")' \ | 建表时追加存储格式到建表语句中，tblproperties修改表的属性，这里设置orc的压缩格式为SNAPPY |
| -m                                                           | 指定并行处理的MapReduce任务数量。-m不为1时，需要用split-by指定分片字段进行并行导入，尽量指定int型。 |
| --split-by id                                                | 如果指定-split by, 必须使用$CONDITIONS关键字, 双引号的查询语句还要加\ |
| --hcatalog-partition-keys--hcatalog-partition-values         | keys和values必须同时存在，相当于指定静态分区。允许将多个键和值提供为静态分区键。多个选项值之间用，（逗号）分隔。比如：--hcatalog-partition-keys year,month,day--hcatalog-partition-values 1999,12,31 |
| --null-string '\\N'--null-non-string '\\N'                   | 指定mysql数据为空值时用什么符号存储，null-string针对string类型的NULL值处理，--null-non-string针对非string类型的NULL值处理 |
| --hive-drop-import-delims                                    | 设置无视字符串中的分割符（hcatalog默认开启）                 |
| --fields-terminated-by '\t'                                  | 设置字段分隔符                                               |