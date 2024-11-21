---
title:  "Clickhouse学习总结"
categories: 
  - 大数据
tags:
  - 大数据
  - 基础组件
  - clickhouse
toc: true
gallery1:
  - url: /assets/posts/2024-11-20-clickhouse-learn-distribute.png
    image_path: /assets/posts/2024-11-20-clickhouse-learn-distribute.png
  - url: /assets/posts/2024-11-20-clickhouse-learn-distribute-read.png
    image_path: /assets/posts/2024-11-20-clickhouse-learn-distribute-read.png
gallery2:
  - url: /assets/posts/2024-11-20-clickhouse-learn-cluster-1.png
    image_path: /assets/posts/2024-11-20-clickhouse-learn-cluster-1.png
  - url: /assets/posts/2024-11-20-clickhouse-learn-cluster-2.png
    image_path: /assets/posts/2024-11-20-clickhouse-learn-cluster-2.png
  - url: /assets/posts/2024-11-20-clickhouse-learn-cluster-3.png
    image_path: /assets/posts/2024-11-20-clickhouse-learn-cluster-3.png
header:
  overlay_image: /assets/images/unsplash-gallery-image-3.jpg
  teaser: assets/images/unsplash-gallery-image-3-th.jpg
tagline: ""
---

## Clickhouse学习总结

#### 一、 Clickhouse学习资料

- clickhouse官方中文文档: [clickhouse](https://clickhouse.ac.cn/docs/en/intro)
- clickhouse书籍: 
  - **ClickHouse原理解析与应用实践(朱凯2020)**(重点章节: 2、4、6、7、8、9、10)

#### 二、Clickhouse核心特性
1. 完备的DBMS功能（Database Management System，数据库管理系统）: 包含DDL、DML、权限控制、数据备份与恢复、分布式管理功能
2. 列式存储与数据压缩: 数据按列存储, 每列数据按压缩算法压缩数据
3. 向量化执行引擎: 利用CPU的SIMD指令, 即用单条指令操作多条数据,通过数据并行以提高性能的一种实现方式
4. 关系模型与SQL查询: 完全使用SQL作为查询语言（支持GROUP BY、ORDER BY、JOIN、IN等大部分标准SQL）
5. 多样化的表引擎:共拥有合并树、内存、文件、接口和其他6大类20多种表引擎
6. 多线程与分布式:大量使用了多线程技术以实现提速; 在数据存取方面，既支持分区（纵向扩展，利用多线程原理），也支持分片（横向扩展，利用分布式原理）
7. 多主架构:采用Multi-Master多主架构，集群中的每个节点角色对等，客户端访问任意一个节点都能得到相同的效果。

#### 三、Clickhouse缺点
1. 不支持事物(完全不支持事物, 不支持回滚操作)
2. 不擅长根据主键按行粒度进行查询(虽然支持, 但相比其他数据库没有性能优势, 如果是select *取全部列的查询, 相比其他数据库可能有性能劣势)
3. 不擅长按行删除数据(虽然支持,但支持不好, 数据删除后, 要等分区合并之后数据才会真的删除, 在分区合并前大部分引擎可能会查出来已经删除的数据)

#### 四、 表和视图

```sql
#创建表
CREATE TABLE [IF NOT EXISTS] [db_name.]table_name (
    name1 [type] [DEFAULT|MATERIALIZED|ALIAS expr],
    name2 [type] [DEFAULT|MATERIALIZED|ALIAS expr],
    省略…
) ENGINE = engine

#创建视图
CREATE [MATERIALIZED] VIEW [IF NOT EXISTS] [db.]table_name [TO[db.]name] [ENGINE = engine] [POPULATE] AS SELECT ..
```
clickhouse创建表与一般sql没有太大区别, 主要区别是要指定表引擎, 主要介绍经常使用的到的表引擎:

##### 4.1 *MergeTree系列表引擎
```sql
CREATE TABLE [IF NOT EXISTS] [db_name.]table_name (
    name1 [type] [DEFAULT|MATERIALIZED|ALIAS expr],
    name2 [type] [DEFAULT|MATERIALIZED|ALIAS expr],
    省略...
) ENGINE = MergeTree()
[PARTITION BY expr]
[ORDER BY expr]
[PRIMARY KEY expr]
[SAMPLE BY expr]
[SETTINGS name=value, 省略...]
```

![](/assets/posts/2024-11-20-clickhouse-learn-mergetree.png)

**PARTITION BY expr 分区键**: 按指定字段或表达式对数据进行分区, 不同分区内的数据不相同, 分区可以加快检索, 不会检索不需要的分区;如果不使用分区键，则分区ID默认取名为all，所有的数据都会被写入这个all分区

**ORDER BY expr 排序键**: 按指定字段或表达式对数据进行排序, clickhouse与其他数据库不同, 一般只用定义排序键, 不用定义主键,不定义主键的话, 主键等于排序键(clickhouse我理解是弱主键的数据库)

**PRIMARY KEY expr**: 按指定字段或表达式识别唯一数据, 及其他数据库中主键概念, 但是clickhouse是弱主键数据库, 实际上允许主键重复, 一般不用定义主键, 如果定义了主键要求是排序键的前缀


##### 4.2 kafka表引擎
```sql
CREATE TABLE [IF NOT EXISTS] [db_name.]table_name (
    name1 [type] [DEFAULT|MATERIALIZED|ALIAS expr],
    name2 [type] [DEFAULT|MATERIALIZED|ALIAS expr],
    省略…
) ENGINE = Kafka()
SETTINGS
    kafka_broker_list = 'host:port,... ',
    kafka_topic_list = 'topic1,topic2,...',
    kafka_group_name = 'group_name',
    kafka_format = 'data_format'[,]
    [kafka_row_delimiter = 'delimiter_symbol']
    [kafka_schema = '']
    [kafka_num_consumers = N]
    [kafka_skip_broken_messages = N]
    [kafka_commit_every_batch = N]
```

kafka表引擎按建表是的kafka配置, 使用librdkafka库, 开启相应数量的消费者线程消费kafka数据, 注意kafka表引擎是外部存储表引擎, 不负责数据写入保存, 每次对kafka表做select操作, 返回的都是新消费的数据, 所以一般配合视图一起使用


##### 4.3 join表引擎 
```sql
ENGINE = Join(join_strictness, join_type, key1[, key2, ...])
```

1. Join表引擎可以说是为JOIN查询而生的，它等同于将JOIN查询进行了一层简单封装。
2. 数据首先会被写至内存，然后被同步到磁盘文件。它既能够作为JOIN查询的连接表，也能够被直接查询使用.
3. 和MergeTree系列表引擎相比, 相当于没有分区、排序等功能的简单表


##### 4.4 Distributed表引擎
```sql
ENGINE = Distributed(cluster, database, table [,sharding_key])
```

{% include gallery id="gallery1" %}

上面说的表引擎实际上都是本地表, clickhouse是多主架构, 如果你有多个clickhouse节点, 在其中一个clickhouse节点执行了建表语句的话, 其他节点的clickhouse实际是没有这张表的, 除非在建表的时候加上 CREATE TABLE test_name ON CLUSTER cluster_name ,或者在每个节点单独建表(我们是这么做的)

如果有多个clickhouse节点, 使用了CREATE TABLE test_name ON CLUSTER cluster_name 语句建表(或在每个节点单独建表),  那怎么查询和写入呢, 这时候就需要Distributed表引擎来做代理, Distributed表引擎自身不存储任何数据, Distributed表引擎在建表的时候也需要使用ON CLUSTER cluster_name(或每个节点单独建表)

##### 4.5 VIEW视图
```sql
CREATE [MATERIALIZED] VIEW [IF NOT EXISTS] [db.]table_name [TO[db.]name] [ENGINE = engine] [POPULATE] AS SELECT ...
```

视图分为VIEW 普通视图和MATERIALIZED VIEW 物化视图

普通视图不存储数据, 它只是一层单纯的SELECT查询映射，起着简化查询、明晰语义的作用，对查询性能不会有任何增强

物化视图会存储数据, 数据保存形式由它的表引擎决定

两种视图我们都有用到, 其中物化视图还有一个作用, 是搭配kafka表引擎同步数据

![](/assets/posts/2024-11-20-clickhouse-learn-kafka.png)

1. 表A是kafka表引擎，它充当的角色是一条数据管道，负责拉取Kafka中的数据
2. 表B是MergeTree表引擎，它充当的角色是面向终端用户的查询表
3. 物化视图C它负责将表A的数据实时同步到表B

```sql
CREATE MATERIALIZED VIEW IF NOT EXISTS bdp.consumer_tracks TO bdp.tracks_local AS
SELECT *, _timestamp as updated_time
FROM bdp.kafka_tracks;
```

这个物化视图创建语句, 在bdp数据库创建物化视图consumer_tracks, 这个物化视图从kafka_tracks表中实时同步数据到tracks_local表中, tracks_local是MergeTree表引擎, 负责检索

#### 五、分区

MergeTree系列表引擎支持定义分区键, 作为clickhouse使用者, 需要重点理解clickhouse分区机制和实现原理, **目前clickhouse的现场问题都是分区导致的**

##### 5.1 分区id

![](/assets/posts/2024-11-20-clickhouse-learn-partition.png)

MergeTree表的每行数据,都会按分区键定义的表达式, 计算出这一行的分区ID, 不同的分区ID存储在不同的分区目录里, 不同分区ID的数据存储相互隔离

##### 5.2分区目录

分区目录命令规则:   **PartitionID_MinBlockNum_MaxBlockNum_Level**

![](/assets/posts/2024-11-20-clickhouse-learn-partition-id.png)

1. MergeTree表分区的数据的物理存储不是存在一个目录中, 而是存在多个目录中, 每个目录的命令规则如上所示
2. MinBlockNum和MaxBlockNum：最小数据编号与最大数据编号, 数据编号BlockNum是一个整型的自增长编号,计数在单张MergeTree数据表内全局累加, 数据编号BlockNum从1开始计数, 每当新创建一个分区目录时，计数n就会累积加1
3. 每次执行INSERT语句, MergeTree都会生成一批新的分区目录, 对于一个新的分区目录而言，MinBlockNum与MaxBlockNum取值一样,例如201905_1_1_0、201905_2_2_0
4. Level：合并的层级，可以理解为某个分区被合并过的次数, 初始值为0, 当多个分区目录合并成一个分区目录时, 新分区目录的level会取多个目录最大的level再+1

##### 5.3 分区合并

由上可知,  每次INSERT只要某个分区有数据, 这个分区就会产生新目录, 目录多了就需要合并, 分区目录的合并由clickhouse后台进程控制, 目前没有参数控制合并频率, 执行合并后, 原来的目录会在某个时刻后台删除;**分区合并是现场目前clickhouse的首要问题**

合并规则:

1. 属于同一个分区的多个目录，在合并之后会生成一个全新的目录, 不同分区的目录不会合并
2. MinBlockNum取要合并的目录中最小的MinBlockNum值
3. MaxBlockNum取要合并的目录中最大的MaxBlockNum值
4. Level取同要合并的目录中最大的Level值+1

![](/assets/posts/2024-11-20-clickhouse-learn-partition-merge.png)

对于分区合并, 需要着重注意的是:

1. 上面只说了分区目录合并时目录名字的变化规则, 分区合并的过程中, 分区目录里的数据文件也需要合并, 这时候会有大量的io操作, 是目前clickhouse的性能瓶颈
2. 分区越多, 要合并的目录越多, 因为不同分区的目录不会合并, 所以分区不能过细, 否者由大量分区目录需要合并, io过大, 卡死clickhouse

#### 六、集群、副本、分片、

##### 6.1 集群

![](/assets/posts/2024-11-20-clickhouse-learn-cluster.png)

clickhouse集群的概念和其他组件不一样:

1. clickhouse支持分布式部署, 支持多台节点
2. clickhouse支持不同的节点之间组成集群, 这个集群是逻辑层面的,  由上所示, 集群1可以由N1、N2组成,集群2由N3、N4、N5组成,集群3由N4、N5、N6组成, 集群4由 N1-N6组成, 不同集群之间使用的节点可以重复

##### 6.2 副本和分片

表开启副本和分片有两种方式, 一种是使用Replicated*MergeTree表引擎, 一种是使用Distributed分布式表引擎, 项目使用的是Distributed分布式表引擎

```sql
ENGINE = Distributed(cluster, database, table [,sharding_key])

--按照随机数划分
Distributed(cluster, database, table ,rand())
--按照用户id的散列值划分
Distributed(cluster, database, table , intHash64(userid))
```

**分区**: 数据按分区键区分, 存储在不同的分区目录, 如果在集群中定义了带分区键的表, 每台机器都有对应的分区目录, 不同分区目录存储的数据不一样

**分片**: 数据按Distributed分布式表引擎分片键区分, 没有定义分区键则只有一个分片, 即只能用到本地表; 数据按分片键存储在不同的节点上, 如果有分片键和分区键, 则先按分片键确定数据存储的节点, 再按分区键决定数据存储的分区目录

**副本**: 副本是分片的数据冗余备份, 本质上和分片没有区别, 副本需要依赖zk做数据同步能力, 所以开启副本的话, clickhouse需要配置zk

注意: 一个clickhouse对于同一张表, 只能存在一个分片或一个副本

##### 6.3 项目集群、分片、副本介绍

{% include gallery id="gallery2" layout="half" %}

clickhouse集群配置在clickhouse的配置文件中, 项目中使用了3种集群:

**clicks_cluster_s1rn** 集群包含1个分片(shard), 该分片有3个副本(replica) (副本就是分片, 分片就是副本), 使用clickhouse group0-0、group0-1、group0-2三台机器

**clicks_cluster_human** 集群包含1个分片(shard), 该分片有3个副本(replica) , 使用clickhouse group0-0、group0-1、group0-2三台机器

**clicks_cluster** 集群包含3个分片(shard), 每个分片有1个副本(replica), 使用clickhouse group0-0、group0-1、group0-2三台机器

##### 6.4 数据写入
```sql
#tracks_local 轨迹本地表, 注意create语句没有加ON CLUSTER
CREATE TABLE IF NOT EXISTS bdp.tracks_local
(
    `cluster_id`          UInt64 comment '类id',
    `object_id`           String comment '对象id',
    `sequence`            UInt16 comment '序号',
    `object_type`         String comment '对象类型',
    `captured_time`       DateTime comment '抓拍时间',
    `camera_map_id`       UInt32 comment '时空点位编码信息',
    `updated_time`        DateTime comment '更新时间'
)
    ENGINE = MergeTree()
        PRIMARY KEY (cluster_id, captured_time, camera_map_id)
        PARTITION BY (toYYYYMMDD(captured_time), object_type)
        ORDER BY (cluster_id, captured_time, camera_map_id)
        TTL captured_time + INTERVAL 5 YEAR
        SETTINGS index_granularity = 8192;


#kafka_tracks kafka轨迹本地表, 注意create语句没有加ON CLUSTER
CREATE TABLE IF NOT EXISTS bdp.kafka_tracks
(
    cluster_id          UInt64,
    object_id           String,
    sequence            UInt16,
    object_type         String,
    captured_time       DateTime,
    camera_map_id       UInt32,
)
    ENGINE = Kafka
        SETTINGS kafka_broker_list =
                '$KAFKA_BROKER_LIST',
            kafka_topic_list = 'clickhouse_row_data_tracks_${GROUP}',
            kafka_group_name = 'consumer_bdp_tracks_${GROUP}_${INDEX}',
            kafka_format = 'JSONEachRow',
            kafka_num_consumers = 1,
            kafka_max_block_size = 1048576;

#物化视图
CREATE MATERIALIZED VIEW IF NOT EXISTS bdp.consumer_tracks TO bdp.tracks_local AS
SELECT *, _timestamp as updated_time
FROM bdp.kafka_tracks;

#tracks_all 轨迹分布式表
CREATE TABLE IF NOT EXISTS bdp.tracks_all AS bdp.tracks_local
    engine = Distributed('clicks_cluster_human', 'bdp', 'tracks_local');
```

![](/assets/posts/2024-11-20-clickhouse-learn-data.png)
