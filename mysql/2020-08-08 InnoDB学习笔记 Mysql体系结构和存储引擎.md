# 2020-08-08 InnoDB学习笔记 Mysql体系结构和存储引擎

## 一、数据库和实例

> 数据库

物理操作系统文件或其他形式**文件类型的集合**，MySQL数据库中，数据库文件可以是`frm`、`MYD`、`MYI`、`ibd`结尾的文件。

> 实例

MySQL数据库由后台线程以及一个共享内存区组成。**数据库实例**才是真正用于操作数据库文件的。   

在MySQL数据库中，实例与数据库的关系通常是一一对应的，在**集群**情况下可能存在一个数据库被多个数据实例使用的情况。   

MySQL被设计成一个单进程多线程架构的数据库。**MySQL数据库实例在系统上的表现就是一个进程**。

> 启动数据库实例

``` base
# 启动并后台运行
[root@localhost bin]# ./mysqld_safe&
# 查看实例进程
[root@localhost bin]# ps -ef | grep mysqld
root      1696     1  0 Mar28 ?        00:00:00 /bin/sh /usr/bin/mysqld_safe --datadir=/data/mysqldata/mysql --socket=/data/mysqldata/mysql/mysql.sock --pid-file=/var/run/mysqld/mysqld.pid --basedir=/usr --user=mysql
mysql     1960  1696  1 Mar28 ?        1-23:09:31 /usr/sbin/mysqld --basedir=/usr --datadir=/data/mysqldata/mysql --plugin-dir=/usr/lib64/mysql/plugin --user=mysql --log-error=/var/log/mysqld.log --pid-file=/var/run/mysqld/mysqld.pid --socket=/data/mysqldata/mysql/mysql.sock
```

进程号1960的进程，就是MySQL实例。

> 启动实例时的配置文件

当启动实例时，MySQL数据库会去读取**配置文件**，根据配置文件的参数来启动数据库实例。在启动实例中，可以没有配置文件，此时，MySQL会按照编译时的**默认参数**设置启动实例。

> 命令查看MySQL数据库实例启动的配置文件

```bash
[root@localhost bin]# mysql --help | grep my.cnf
                      order of preference, my.cnf, $MYSQL_TCP_PORT,
/etc/my.cnf /etc/mysql/my.cnf /usr/etc/my.cnf ~/.my.cnf 

```

MySQL数据库是按照上面配置文件**从左到右的顺序**读取配置文件，有相同参数以**最后一个**配置文件中的参数为准。

> 数据库所在路径

配置文件中有一个参数`datadir`，指定了**数据库所在的路径**，linux操作系统下默认`datadir`为/usr/local/mysql/data   

命令查看该参数值

```mysql
mysql> show variables like 'datadir'\g
+---------------+------------------------+
| Variable_name | Value                  |
+---------------+------------------------+
| datadir       | /data/mysqldata/mysql/ |
+---------------+------------------------+
1 row in set (0.00 sec)
```

用户必须保证以上路径的用户和权限，使得只有mysql用户和组可以访问。通常MySQL数据库的权限为mysql:mysql。

## 二、MySQL体系结构

从概念上说：数据库是**文件的集合**，是依照某种数据模型组织起来并存放于二级存储器中的数据集合；数据库实例时**程序**，是位于用户与操作系统之间得一层**数据管理软件**，用户对数据库数据的任何操作（数据库定义，数据查询，数据维护，数据库运行控制），都是在数据库实例下进行的，**应用程序只有通过数据库实例才能和数据库打交道**。

> 官方MySQL体系结构

![4b99c67cb47a0843b78fe042a2f651af6b6c76ce.png](https://i.loli.net/2020/08/08/wyB7XuphD29Z3gs.png)

MySQL由以下几部分组成：

- 连接池组件
- 管理服务和工具组件
- SQL接口组件
- 查询分析器组件
- 优化器组件
- 缓存（Cache）组件
- 插件式存储引起
- 物理文件

MySQL数据库区别于其他数据库的最重要一个特点就是其**插件式的表存储引擎**。MySQL插件式的存储引擎架构提供了一系列表中的管理和服务支持，这些标准与存储引擎本身无关，可能是每个数据库系统本身必需的，如SQL分析器和优化器等，而存储引擎是底层物理结构的实现，每个存储引擎开发者可以自己开发。   

**存储引擎时基于表的，而不是数据库的。**

## 三、MySQL存储引擎

存储引擎是MySQL区别于其他数据库的一个最重要组件，好处是每个存储引擎都有各自的特点，能够根据**具体的应用建立不同意存储引擎表**。由于MySQL数据库的开源性，用户可以根据MySQL预定义的存储引擎接口编写自己的存储引擎。

### 1.InnoDB存储引擎 

> InnoDB介绍

InnoDB存储引擎支持事务，设计目标主要面向**在线事务处理**（OLTP Online Transaction Processing）的应用，特点是`行锁设计`，`支持外键`，并支持类似于Oracle的`非锁定读`，即默认读取操作不产生锁。从MySQL数据库**5.5.8**版本开始，**InnoDB存储引擎是默认的存储引擎**。

> InnoDB数据存储文件

InnoDB存储引擎将**数据**放在一个**逻辑表空间**中，然后有InnoDB存储引擎自身进行管理。从MySQL**4.1**版本开始，它可以将每个InnoDB存储引擎的**表单独存放到一个独立的`ibd`文件中**。此外，InnoDB存储引擎支持用裸设备（row disk）用来建立其表空间。

> InnoDB的高可用和高性能

InnoDB通过使用`多版本并发控制器`（MVCC）来获得**高并发性**，并实现了SQL标准的**4种隔离级别**，默认为`REPEADABLE`级别。同时使用一种被称为`next-key-locking`的策略来避免**幻读**(phantom)现象的产生。InnoDB存储引擎还提供了**插入缓冲**（insert buffer）、**二次写**（double write）、**自适应哈希索引**（adaptive hash index）、预读（read ahead）等高可用和高性能的功能。

> InnoDB存储引擎的数据存储方式

InnoDB存储引擎采用了**聚集**（clustered）的方式对表中的数据进行存储（**每一行中的数据会跟主键进行一起存放**），因此每张表的存储都是**按照主键的顺序进行存放**。如果**没有显式地在表定义时指定主键**，InnoDB存储引擎会为每一行生成一个6字节的`ROWID`，并以此作为主键（**实际上是一个long类型的字段，但是有效位只有6*8=48个字，随着数据新增时默认自增，超过低48位上限后，新插入的数据就会报主键重复/冲突错误，id自增都会如此，建议分库分表解决该问题**）。

### 2.MyISAM存储引擎

> MyISAM介绍

MyISAM存储引擎**不支持事务**、**表锁设计**，**支持全文索引**，主要面向一些OLAP数据库应用。在MySQL**5.5.8**之前MyISAM存储引擎是默认的存储引擎（windows除外）。除了不支持事务外，MyISAM存储引擎的另一个与众不同点是**缓冲池只缓存（cache）索引文件，而不缓冲数据文件**。

> MyISAM数据存储文件

MySIAM存储引擎表由`MYD`和`MYI`组成，`MID`用来存放数据文件，`MYI`用来存放索引文件。可以使用**myisampack**工具来进一步压缩数据文件，因为该工具使用赫夫曼编码静态算法来压缩数据，所以**压缩后的表是只读的**，用户也可以使用**myisampack**工具来解压数据文件。   

在MySQL**5.0**版本之前，MyISAM默认支持的表大小为**4GB**，如果需要支持大于4GB的MyISAM表时，则需要制定`MAX_ROWS`和`AVG_ROW_LENGTH`属性。从MySQL**5.0**版本开始，MyISAM默认支持**256TB**的单表数据。   

> MySQL对MyISAM存储引擎表的数据文件处理

对于MyISAM存储引擎表，**MySQL数据库只缓存其索引文件，数据文件的缓存交由操作系统本身来完成**，这与其他使用LRU算法缓存数据的大部分数据库大不相同。在MySQL**5.1.23**版本之前，**缓存索引的缓冲区最大只能设置为4GB**，在之后的版本，**64位系统可以支持大于4GB的索引缓冲区**。

### 3.NDB存储引擎

> NDB介绍

NDB存储引擎时一个**集群存储引擎**，架构是share nothing的集群架构，能提供更高的可用性。NDB的特点是**数据全部放在内存中**（从MySQL5.1版本开始，**可以将非索引数据放在磁盘上**），因此**主键查找的速度极快**，并且通过添加**NDB数据存储节点（Data Node）可以线性提高数据库性能**，是高可用、高性能的集群系统。   

> NDB局限

NDB存储引擎的**连接操作（JOIN）是在MySQL数据库层完成**的，而不是在存储引擎层完成的，意味着**复杂的连接操作需要巨大的网络开销**，因此**查询速度很慢**。

### 4.Memory存储引擎

> Memory介绍

Memory存储引擎将表中的**数据存放在内存中**，如果数据库重启或发生崩溃，表中的数据都将消失。适用于存储**临时数据的临时表**，以及**数据仓库中的维度表**。该存储引擎默认使用**哈希索引**，而不是B+树索引。

> Memory局限

- **只支持表锁，并发性能较差**
- **不支持TEXT和BLOB列类型。**
- **存储边长字段（varchar）时是按照定长字段（char）的方式进行的，会有内存浪费**
- MySQL数据库使用Memory存储引擎作为**临时表**来**存放查询的中间结果**，如果中间结果集大于Memory存储引擎表的容量设置，又或者中间结果集含有**TEXT和BOLB**列类型字段，则MySQL数据库会把其**转换到MyISAM存储引擎表而存活到磁盘中**（MyISAM不缓存数据文件），会产生临时表的性能对于**查询会有损失**。

### 5.Archive存储引擎

Archive存储引擎**只支持INSERT和SELECT操作**，从MySQL**5.1开始支持索引**。Archive存储引擎**使用zlib算法讲数据行（row）进行压缩后存储**，压缩比一般达到**1:10**，所以该存储引擎非常适合存储归档数据，如日志信息。Archive存储引擎使用**行锁来实现高并发的插入操作**，但其本身**不是事务安全的存储引擎**，设计目标主要是**提供高速的插入和压缩功能**。

### 6.Federated存储引擎

Federated存储引擎**表并不存放数据**，它只是指向一台远程MySQL数据库服务器上的表，只支持MySQL数据库表，不支持异构数据库表。

### 7.Maria存储引擎

Maria存储引擎时**新开发的引擎**，设计目标是用来**取代原有的MyISAM存储引擎，成为MySQL的默认存储引擎**。它支持**缓存数据和索引文件，应用了行锁设计，提供了MVCC功能，支持事务和非事务安全的选项，已经更好的BLOB字符类型的处理性能**。

## 四、存储引擎之间的比较

MySQL的官方手册中，展现了一些常用MySQL存储引擎之间的不同之处，包括存储容量的限制、事务支持、锁的粒度、MVCC支持、支持的索引、备份和复制等。

![](https://i.loli.net/2020/08/09/ghpoOWju2Be4EY1.png)

可以通过`SHOW ENGINES`语句查看当前使用的MySQL数据库所支持的存储引擎，也可以通过查找`information_schema`架构下的**ENGINES**表，如下所示：

```mysql
mysql> show engines;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
9 rows in set (0.01 sec)
```

## 五、连接MySQL

连接MySQL**操作是一个连接进程和MySQL数据库实例进行通信，本质上是进程通信**。

> TCP/IP

**TCP/IP套接字方式是MySQL数据库在任何平台下都提供的连接方式，也是网络中使用最多的一种方式**。通过TCP/IP建立一个基于网络的连接请求，用户可在一个服务器请求另一个服务器下的MySQL实例。如下：

```bash
C:\> mysql -h192.168.10.6 -u root -p
Enter password:
```

通过TCP/IP连接到MySQL实例时，**MySQL数据库会先检查一张权限视图，用来判断发起请求的客户端IP是否允许连接到MySQL实例**。该视图在`mysql`架构下，表明为`user`，如下所示：

```mysql
mysql> use mysql;
Database changed
mysql> select host,user from user;
+-----------+---------------+
| host      | user          |
+-----------+---------------+
| localhost | mysql.session |
| localhost | mysql.sys     |
| localhost | root          |
+-----------+---------------+
3 rows in set (0.00 sec)
```

> 命名管道和共享内存

两个进程需要通信的进程在同一台服务器上，那么可以使用命名管道。同时MySQL还提供了共享内存的连接方式。

> UNIX域套接字

在Linux和UNIX环境下，还可以使用UNIX域套接字。只能在MySQL客户端和数据库实例在同一台服务器上的情况下使用。用户可以在配置文件中指定套接字文件的路径，如`--socket=/usr/mysql/mysql.sock`;在实例启动后也可通过一下方式进行查找

```mysql
SHOW VARIABLES LIKE 'socket'\G
```

## 总结

- 本章介绍了”实例“和”数据库“的区别；
- 简单介绍MySQL的体系架构
- 讲解了各种存储引擎的特性
- 连接MySQL数据库实例的方式







