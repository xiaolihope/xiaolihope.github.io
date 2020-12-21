---
date: 2016-07-03T07:47:18+08:00
title: Database
---
# Database commands

## Grant

MySQL 赋予用户权限命令的简单格式可概括为：

```
grant 权限 on 数据库对象 to 用户
```

## SQLite

1. SQLite 直接访问其存储文件
2. SQLite 命令：

```bash
create
alter
drop
insert
update
delete
select
```
3. sqlite3 
4. .show 查看SQLite命令提示符的

```bash
sqlite> .show
        echo: off
         eqp: off
     explain: auto
     headers: off
        mode: list
   nullvalue: ""
      output: stdout
colseparator: "|"
rowseparator: "\n"
       stats: off
       width:
```

5. sqlite commands

```bash
1) create database:
sqlite3 <database-name>.db
2) show databases .databases
     show tables: .tables
     show schema .schema <table-name>
     show version: sqlite3 -version
3) quit
.quit
4) dump and restore
sqlite3 dixiaoli.db .dump dixiaoli.sql
sqlite3 dixiaoli.db < dixiaoli.sql
5) 当有多个数据库时，可以通过attach来选择一个：
sqlite> ATTACH DATABASE 'testDB.db' as 'TEST';
6）当一个数据库有多个别名时，可以使用detach来分离数据库
```

## Mysql / MariaDB

### connect to remote db
```
mysql -uuser -hhostname -PPORT -ppassword
```
for example:

```mysql -ugringotts -h<hostname> -P3306 -pd0e437face6be95da74991c7```

### mysql commands
```
# show databases
# use gringoots;
# list tables;
# describe <table_name>;
# Select * from <table_name> where <column_id>=“***”;
# update <table_name> set <column_name>=“*” where user_id="47c68b2fc70e466ca0f0481b4bf56d34"
# delete from <table_name> where ;
# mysql -uroot "database-name" -e "select * from users" | grep 
```
推荐一个在mac上用的mysql tool：sequel pro 好处是可以通过界面来查看操作数据库，避免执行mysql语句错误的情况。
```commandline
# /usr/sbin/mysqld --user=mysql --init-file=

```

### mysql 最大文件打开数

```

查看当前打开文件数：

# lsof -p 66396 | wc -l
973

# losf -u mysql | wc -l

MariaDB [(none)]> show global variables like "%open_files_limit%";
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| open_files_limit | 1024  |
+------------------+-------+

设置OS参数—> ：单个进程能够打开的最大文件句柄数量
Vim /etc/security/limits.conf
*     soft nofile 655350
*     hard nofile 655350

查看：ulimit -n
修改mysql 参数

Vim /etc/my.cnf.d/server.cnf

重启mysql生效。

If mysql is started with systemd, this setting is important:

Vim the /lib/systemd/system/mariadb.service

# For example, if you want to increase mysql's open-files-limit to 10000,
# you need to increase systemd's LimitNOFILE setting, so create a file named
# "/etc/systemd/system/mariadb.service.d/limits.conf" containing:
#       [Service]
#       LimitNOFILE=10000

Touch /etc/systemd/system/mariadb.service.d/migrated-from-my.cnf-settings.conf

Cat migrated-from-my.cnf-settings.conf
# converted using /usr/bin/mariadb-service-convert
#

[Service]

User=mysql
LimitNOFILE=102400

Then systemctl daemon-reload
systemctl restart  maraidb

DONE

```

```
## 数据库索引

### 介绍

```
索引格式有三种：普通索引、UNIQUE 索引、PRIMARY KEY 索引。

为什么要创建索引呢？这是因为，创建索引可以大大提高系统的性能。

第一，通过创建唯一性索引，可以保证数据库表中每一行数据的唯一性。
第二，可以大大加快 数据的检索速度，这也是创建索引的最主要的原因。
第三，可以加速表和表之间的连接，特别是在实现数据的参考完整性方面特别有意义。
第四，在使用分组和排序 子句进行数据检索时，同样可以显著减少查询中分组和排序的时间。
第五，通过使用索引，可以在查询的过程中，使用优化隐藏器，提高系统的性能。

也许会有人要问：增加索引有如此多的优点，为什么不对表中的每一个列创建一个索引呢？这种想法固然有其合理性，然而也有其片面性。
虽然，索引有许多优点， 但是，为表中的每一个列都增加索引，是非常不明智的。这是因为，增加索引也有许多不利的一个方面。

第一，创建索引和维护索引要耗费时间，这种时间随着数据 量的增加而增加。
第二，索引需要占物理空间，除了数据表占数据空间之外，每一个索引还要占一定的物理空间，如果要建立聚簇索引，那么需要的空间就会更大。
第三，当对表中的数据进行增加、删除和修改的时候，索引也要动态的维护，这样就降低了数据的维护速度。

索引是建立在数据库表中的某些列的上面。因此，在创建索引的时候，应该仔细考虑在哪些列上可以创建索引，在哪些列上不能创建索引。
一般来说，应该在这些列 上创建索引，例如：

在经常需要搜索的列上，可以加快搜索的速度；
在作为主键的列上，强制该列的唯一性和组织表中数据的排列结构；
在经常用在连接的列上，这 些列主要是一些外键，可以加快连接的速度；
在经常需要根据范围进行搜索的列上创建索引，因为索引已经排序，其指定的范围是连续的；
在经常需要排序的列上创 建索引，因为索引已经排序，这样查询可以利用索引的排序，加快排序查询时间；
在经常使用在WHERE子句中的列上面创建索引，加快条件的判断速度。


同样，对于有些列不应该创建索引。一般来说，不应该创建索引的的这些列具有下列特点：

第一，对于那些在查询中很少使用或者参考的列不应该创建索引。这是因 为，既然这些列很少使用到，因此有索引或者无索引，
并不能提高查询速度。相反，由于增加了索引，反而降低了系统的维护速度和增大了空间需求。
第二，对于那 些只有很少数据值的列也不应该增加索引。这是因为，由于这些列的取值很少，例如人事表的性别列，在查询的结果中，
结果集的数据行占了表中数据行的很大比 例，即需要在表中搜索的数据行的比例很大。增加索引，并不能明显加快检索速度。
第三，对于那些定义为text, image和bit数据类型的列不应该增加索引。这是因为，这些列的数据量要么相当大，要么取值很少。
第四，当修改性能远远大于检索性能时，不应该创建索 引。这是因为，修改性能和检索性能是互相矛盾的。当增加索引时，会提高检索性能，
但是会降低修改性能。当减少索引时，会提高修改性能，降低检索性能。因 此，当修改性能远远大于检索性能时，不应该创建索引。

```

### 创建索引的方法和索引的特征
```
创建索引的方法
创建索引有多种方法，这些方法包括直接创建索引的方法和间接创建索引的方法。直接创建索引，
例如使用CREATE INDEX语句或者使用创建索引向导，间接创建索引，例如在表中定义主键约束或者唯一性键约束时，
同时也创建了索引。虽然，这两种方法都可以创建索引，但是，它们创建索引的具体内容是有区别的。

使用CREATE INDEX语句或者使用创建索引向导来创建索引，这是最基本的索引创建方式，并且这种方法最具有柔性，
可以定制创建出符合自己需要的索引。在使用这种方式 创建索引时，可以使用许多选项，例如指定数据页的充满度、进行排序、
整理统计信息等，这样可以优化索引。使用这种方法，可以指定索引的类型、唯一性和复合 性，也就是说，既可以创建聚簇索引，
也可以创建非聚簇索引，既可以在一个列上创建索引，也可以在两个或者两个以上的列上创建索引。

通过定义主键约束或者唯一性键约束，也可以间接创建索引。主键约束是一种保持数据完整性的逻辑，它限制表中的记录有相同的主键记录。
在创建主键约束时，系 统自动创建了一个唯一性的聚簇索引。虽然，在逻辑上，主键约束是一种重要的结构，但是，在物理结构上，
与主键约束相对应的结构是唯一性的聚簇索引。换句话 说，在物理实现上，不存在主键约束，而只存在唯一性的聚簇索引。同样，
在创建唯一性键约束时，也同时创建了索引，这种索引则是唯一性的非聚簇索引。因此， 当使用约束创建索引时，
索引的类型和特征基本上都已经确定了，由用户定制的余地比较小。

当在表上定义主键或者唯一性键约束时，如果表中已经有了使用CREATE INDEX语句创建的标准索引时，
那么主键约束或者唯一性键约束创建的索引覆盖以前创建的标准索引。也就是说，
主键约束或者唯一性键约束创建的索引的优先级高于使用CREATE INDEX语句创建的索引。
```
### 索引的特征

```
索引有两个特征，即唯一性索引和复合索引。
唯一性索引保证在索引列中的全部数据是唯一的，不会包含冗余数据。如果表中已经有一个主键约束或者唯一性键约束，
那么当创建表或者修改表时，SQL Server自动创建一个唯一性索引。然而，如果必须保证唯一性，
那么应该创建主键约束或者唯一性键约束，而不是创建一个唯一性索引。当创建唯一性索引 时，
应该认真考虑这些规则：当在表中创建主键约束或者唯一性键约束时，SQL Server自动创建一个唯一性索引；如果表中已经包含有数据，
那么当创建索引时，SQL Server检查表中已有数据的冗余性；每当使用插入语句插入数据或者使用修改语句修改数据时，
SQL Server检查a数据的冗余性：如果有冗余值，那么SQL Server取消该语句的执行，并且返回一个错误消息；
确保表中的每一行数据都有一个唯一值，这样可以确保每一个实体都可以唯一确认；只能在可以保证实体 完整性的列上
创建唯一性索引，例如，不能在人事表中的姓名列上创建唯一性索引，因为人们可以有相同的姓名。

复合索引就是一个索引创建在两个列或者多个列上。在搜索时，当两个或者多个列作为一个关键值时，最好在这些列上创建复合索引。
当创建复合索引时，应该考虑 这些规则：最多可以把16个列合并成一个单独的复合索引，构成复合索引的列的总长度不能超过900字节，
也就是说复合列的长度不能太长；在复合索引中，所 有的列必须来自同一个表中，不能跨表建立复合列；在复合索引中，
列的排列顺序是非常重要的，因此要认真排列列的顺序，原则上，应该首先定义最唯一的列，例 如在（COL1，COL2）上
的索引与在（COL2，COL1）上的索引是不相同的，因为两个索引的列的顺序不同；为了使查询优化器使用复合索引，
查询语句中的WHERE子句必须参考复合索引中第一个列；当表中有多个关键列时，复合索引是非常有用的；使用复合索引可以提高查询性能，
减少在一个表中所创建的 索引数量。

```

### 使用

(1) 使用ALTER TABLE 语句创建索引

```
alter table table_name add index index_name (column_list) ;
alter table table_name add unique (column_list) ;
alter table table_name add primary key (column_list) ;
```

（2）使用CREATE INDEX 语句对表增加索引

```
create index index_name on table_name (column_list) ;
create unique index index_name on table_name (column_list) ;
```

(3) 删除索引

```
drop index index_name on table_name ;
alter table table_name drop index index_name ;
alter table table_name drop primary key ;
```