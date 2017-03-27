---
title: 'MySQL: BINLOG详解'
date: 2017-01-18 15:12:30
tags: MySQL
categories:
  - 数据库
  - MySQL
---

转自面神的[博客](http://zzzvvvxxxd.github.io/2016/02/13/mysql_binlog/#),感谢面神近两年来对我的帮助和教导.  

---

> The binary log contains “events” that describe database changes such as table creation operations or changes to table data. It also contains events for statements that potentially could have made changes (for example, a DELETE which matched no rows), unless row-based logging is used. The binary log also contains information about how long each statement took that updated data.  

binlog是mysql的二进制日志，记录了mysql数据的更新或者潜在的更新记录，同时还包含了每条语句的执行耗时和更新时间。  
注意： binlog主要有`statement-based`、`row-based logging`和`mixed logging` 三种格式，row-based的记录中不包括潜在更新记录。  
binlog的主要功能有如下两条:
- 用于复制
  > master发送了包含了所有events的binary log给slaves，slaves执行binary log中的event来保证和master之间的数据一致性
- 一些特定的数据恢复操作

<!-- more -->

# binlog相关的启动参数
```
--binlog-row-event-max-size=N
```
指定行格式复制日志event的最大大小，单位为byte，每一行数据会被切分到多个小于该限制的event包中，必须是256byte的倍数。
- 默认值：8192
- min：256
- max：18446744073709551615（64位平台）

```
--log-bin[=base_name]
```
开启binary log功能（用于复制和备份），指定日志名称的base name，mysql会使用指定的base name作为前缀，连续的数据作为后缀生成一系列的日志文件。默认会使用`host_name`作为base name。  
```
--log-bin-trust-function-creators[={0|1}]
```
指定mysql如何处理函数及存储过程的创建，取决于用户是否认为自己的存储过程及函数是否安全（确定的或者不修改数据），默认对于不安全的存储过程及函数不会写入binlog。这也就是，在开启binlog的时候，如果不设置该参数，使用自定义函数时会出现错误的原因。

# binlog格式
可以指定三种binary log的格式(启动时指定)：  
```
--binlog-format=STATEMENT
--binlog-format=ROW
--binlog-format=MIXED
```
- `statement-based logging`： 基于SQL语句，Mysql5.6默认，某些语句和函数如UUID, LOAD DATA INFILE等在复制过程可能导致数据不一致甚至出错。  
- `row-based logging`：基于行，记录影响table中每一行的事务，很安全，很安全。但是binlog会比其他两种模式大很多，在一些大表中清除大量数据时在binlog中会生成很多条语句，可能导致从库延迟变大。  
- `mixed logging`：使用statement-based logging作为默认，但是日志模式可能会在某些情况下自动切换到row-based logging。  

`statement-based logging`可能会带来复制上的安全问题：
> With statement-based replication, there may be issues with replicating nondeterministic statements. In deciding whether or not a given statement is safe for statement-based replication, MySQL determines whether it can guarantee that the statement can be replicated using statement-based logging. If MySQL cannot make this guarantee, it marks the statement as potentially unreliable and issues the warning, Statement may not be safe to log in statement format.  
You can avoid these issues by using MySQL's row-based replication instead.

除了开头提到的启动参数，binlog的格式还可以在运行时切换：  
```
# GLOBAL
mysql> SET GLOBAL binlog_format = ‘STATEMENT’;
mysql> SET GLOBAL binlog_format = ‘ROW’;
mysql> SET GLOBAL binlog_format = ‘MIXED’;
# SESSION
mysql> SET SESSION binlog_format = ‘STATEMENT’;
mysql> SET SESSION binlog_format = ‘ROW’;
mysql> SET SESSION binlog_format = ‘MIXED’;
```
一般基于以下几个理由会设置`SESSION`级别的binlog format切换：
- 在对数据库做出了很多小的改变时，可能会需要使用 row-based logging
- 在使用`WHERE`进行更新操作时，可能会影响很多行记录，使用`statement-based logging`来记录少量的事务语句日志，会比记录很多行的改动有效得多。
- 有些语句可能需要执行很长时间，但是实际只改动几行记录。使用`row-based logging`会对复制功能比较友好。

有些情况切换logging format可能会返回`error`:
1. 在使用InnoDB时，如果隔离级别是`READ COMMITTED`或`READ UNCOMMITTED`，只有row-based format可以使用。切换到statement-based其实也是可行的，但是很加就会导致错误，因为这种情况下InnoDB就无法插入数据.
2. 临时表存在时，不推荐切换logging format
  > Switching the replication format at runtime is not recommended when any temporary tables exist, because temporary tables are logged only when using statement-based replication, whereas with row-based replication they are not logged. With mixed replication, temporary tables are usually logged; exceptions happen with `user-defined functions` `(UDFs)` and with the `UUID()` function.

注意:
`ROW`格式下依然会有部分的语句按照`STATEMENT`格式记录，例如所有的DDL语句：`CREATE TABLE`, `ALTER TABLE`和 `DROP TABLE`

# Statement selection options
下面介绍的选项会影响写入到binary log中的记录语句，因此会控制master发送到slaves中的日志的内容。同样，slaves也有相应的参数，在日志选择哪些命令可以执行。  

## `--binlog-do-db`
```
--binlog-do-db=db_name
```
这个参数的影响取决是使用`statement-based`还是`row-based logging`格式的日志。  
STATEMENT-BASED LOGGING 只有操作默认数据库（USE语句指定）的语句才会被记录。如果要指定多个数据库，需要重复使用该参数。但是跨数据库的操作不会因此被记录，因为这里的检查原则，mysql为了尽可能的简单，只会去检查最近的`USE`语句指定的数据库是否在该参数指定的数据库中。例如：  
```
USE sales;
UPDATE prices.discounts SET percentage = percentage + 10;
```
该语句在参数为`--binlog-do-db=sales`的情况下会被记录进binlog的，虽然看起来不太能理解。  
再举个例子:  
```
USE prices;
UPDATE sales.january SET amount=amount+1000;
```
该语句在参数为--binlog-do-db=sales的情况下是不会被记录进binlog的。  
ROW-BASED LOGGING 日志会被严格限定在db_name所指定的数据库上，不再受USE的影响。  
在跨数据的修改操作上，需要举一个具体的例子来加深说明`statement-based`和`row-based logging`两者的区别，假设初始参数为`binlog-do-db=db1`：  
```
USE db1;
UPDATE db1.table1 SET col1 = 10, db2.table2 SET col2 = 20;
```
这条操作`statement-based`的情况下，对两个表的`UPDATE`操作都被记录，如果是`row-based logging`，只有针对table1的操作会被记录。  
现在假设更换默认数据库为db4:
```
USE db4;
UPDATE db1.table1 SET col1 = 10, db2.table2 SET col2 = 20;
```
`statement-based`不会记录这条操作，而`row-based logging`则会记录table1的`UPDATE`操作日志。
## `--binlog-ignore`
```
--binlog-ignore-db=db_name
```
这个参数看起来很好理解，但是要注意的是，`CREATE TABLE`和`ALTER TABLE`不会受其影响都一定会被写入log，一般这个参数影响的是`UPDATE`操作。
## `--binlog-format`
指定binlog使用的格式，可选：statement、row、mixed  
> 更多的配置参数可以参考[1]中的官方说明。

# 相关系统参数和配置
## `sync_binlog`
binlog刷新到磁盘的时机跟sync_binlog参数相关，如果设置为0，则表示MySQL不控制binlog的刷新，由文件系统去控制它缓存的刷新，而如果设置为不为0的值则表示每sync_binlog次事务，MySQL调用文件系统的刷新操作刷新binlog到磁盘中。设为1是最安全的，在系统故障时最多丢失一个事务的更新，但是会对性能有所影响，一般情况下会设置为100或者0，牺牲一定的一致性来获取更好的性能。  
## `expire_logs_days`
指定binlog保留时间
## 清理binlog
要手动清理binlog可以通过指定binlog名字或者指定保留的日期
```
purge master logs to BINLOGNAME;
purge master logs before DATE;
```
## 查看binlog情况
```
SHOW MASTER LOGS;
+------------------+-----------+
| mysql-bin.000018 |       515 |
| mysql-bin.000019 |       504 |
| mysql-bin.000020 |       107 |
+------------------+-----------+
```
第一列是binlog文件名，第二列是binlog文件大小  

# binlog和redo/undo log的区别
两者是完全不同的日志，主要有一下2个区别：  
- 层次不同。redo/undo log是innodb层维护的，而binlog是mysql server层维护的，跟采用何种引擎没有关系，记录的是所有引擎的更新操作的日志记录。
- 记录内容不同。redo/undo日志记录的是每个页的修改情况，属于物理日志+逻辑日志结合的方式，目的是保证数据的一致性。binlog记录的都是事务操作内容，格式是二进制的。

# 参考
[1] [mysql文档 18.1.6.4 Binary Logging Options and Variables](http://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html)  
[2] [mysql文档 6.4.4 The Binary Log](http://dev.mysql.com/doc/refman/5.7/en/binary-log.html)  


