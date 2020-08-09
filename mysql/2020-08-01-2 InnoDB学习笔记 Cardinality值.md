# 2020-08-01 InnoDB学习笔记 *Cardinality*值

## 一、什么是*Cardinality*

对于什么时候添加B+树索引，一般经验是，在访问表中很少一部分时使用B+树索引才有意义。性别、地区、类型等字段，可取值范围很小，称为低选择性。相反，如果某个字段的取值范围很广，几乎没有重复，属于高选择性，则此时使用B+树索引是最适合的。   

*Cardinality*是一个`预估值`，表示***索引中不重复记录数量的预估值***。在实际应用中，*Cardinality*/n_rows_in_table应尽可能接近1。

## 二、InnoDB存储引擎的*Cardinality*统计

Mysql数据库中有各种不同的存储引擎，而每种存储引擎对于B+树索引的实现又各不相同，索引对*Cardinality*的统计是放在存储引擎层进行。   

> 索引更新时统计*Cardinality*的缺陷

- 生成环境中，在索引的**更新操作非常频繁**的情况下，如果每次索引发生操作时就对其进行*Cardinality*的统计，那么将会给数据库带来很大的负担。
- 如果一个***表的数据非常大***，那么统计一次*Cardinality*信息所需要的时间也要相应地更长。

基于以上情况，索引更新及时统计*Cardinality*不现实。

> 通过采样（Sample）的方法来完成对*Cardinality*的统计

在InnoDB存储引擎中，*Cardinality*统计信息的更新发生在`UPDATE`和`INSERT`两个操作中，对更新**Cardinality**信息的策略为：

1. 表中`1/16的数据`发生过变化

2. > `stat_modified_counter`>2 000 000 000

##### 第一种策略

自从上次统计*Cardinality*信息后，表中1/16的数据已发生过变化，更新*Cardinality*信息

##### 第二组策略

如果对表中某一行数据频繁地进行更新操作，其实表中数据实际没有增加，发生变化的只是这一行数据，第一种策略无法使用该情况。InnoDB存储引擎内部有一个计数器`stat_modified_counter`，用来表示发生变化的次数，当`stat_modified_counter`大于2 000 000 000时，同样需要更新*Cardinality*信息。

> InnoDB存储引擎对Cardinality统计和更新具体操作

默认InnodB存储引起对`8个叶子节点`（Leaf Page）进行采样，过程如下

1. 取得B+树索引中叶子节点的`总数量`，记为A。
2. 随机取得B+树索引节点中`8个叶子节点`。统计每个页不同记录的个数P1，P2，P3...P8。
3. 根据采样信息给出Cardinality的预估值：Cardinality=（P1+P2+...+P8）* A/8。

Cardinality值是通过对8个叶子节点预估而得的，不是一个实际精确的值，同时，每次统计得到Cardinality值可能是不同的。当表足够小，表的叶子节点数小于或者等于8时，这时的随机采样结果得到的Cardinality值是相同的。

> 查看索引Cardinality

```mysql
SHOW INDEX FROM `table_name`
```

> InnoDB存储引擎对Cardinality统计时的参数

| 参数                                 | 说明                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| innodb_stats_sample_pages            | 设置统计Cardinality时每次采样页的数量，默认为8               |
| innodb_stats_method                  | 判断如何对待索引中出现的NULL值记录。默认值为**nulls_equal**,表示将NULL值记录视为相等的记录；**nulls_unequal**表示将NULL值视为不同的记录；**nulls_ignored**表示忽略NULL值记录 |
| innodb_stats_persistent              | 是否将命令ANALYZE_TABLE计算得倒的Cardinality值存放磁盘上。好处是可以减少重新计算每个索引的Cardinality值（Mysql重启时），默认**OFF** |
| innodb_stats_on_metadata             | 当通过命令SHOW TABLE STATUS、SHOW INDEX及访问INFORMATION_SCHEMA架构下的表TABLS和STATISTICS时，是否需要重新计算索引Cardinality值。默认**OFF** |
| innodb_stats_persistent_sample_pages | 若参数**innodb_stats_persistent**设置为**ON**，该参数表示ANALYZE TABLE更新Cardinality值时每次采样页的数量。默认值：20 |
| innodb_stats_transient_sample_pages  | 用来取代之前版本的参数**innodb_stats_sample_pages**，表示每次采样页的数量。默认值：8 |











