# ClickHouse 简介

#### 介绍

###### 什么是ClickHouse

ClickHouse(https://clickhouse.yandex/)是一个面向 OLAP 的列式数据库，源自俄罗斯最大的搜索公司 Yandex。官方 benchmark 秒天秒地，比 Hive 快 100 倍，比 vertica、greenplum 等商业化数据库快 6-10 倍。

###### 关键特性

- 支持 SQL&丰富的数据和聚合函数;
- 大部分分析场景下，只用到了数据集中少量的列。例如，如果查询需要 100 列中的 5 列，在面向列的数据库中，通过只读取所需的数据，I/O 可能会减少 20 倍；

官网对行式存储和列式存储的可视化对比如下：



![img](https://clickhouse.com/docs/assets/images/row-oriented-d515facb5bffb48cbd09dc7d064c8816.gif#)



![img](https://clickhouse.com/docs/assets/images/column-oriented-b992c529fa4085b63b57452fbbeb27ba.gif#)



上图为行式存储，下图为列式存储，通过只加载所需的数据可以有效加速查询。

- 受益于列存及向量化引擎，ClickHouse 可以提供极快的查询速度。在官方测试中，10 亿级别数据分组聚合耗时约 1 秒，主要瓶颈在与磁盘读取速度。
- ClickHouse 提供了多种表引擎，其中 MergeTree 相关引擎支持实时写入，实时查询。
- 没有事务、低并发、对长文本支持差，官方建议列中数据是数字和短字符串
- ClickHouse 没有事务，因此不适用与 OLTP 类需求；同时不适用于长文本的存储(可以考虑 ES)。

#### 表引擎

###### MergeTree家族

MergeTree（合并树）系列表引擎是 ClickHouse 提供的最具特色的存储引擎。MergeTree 引擎支持数据按主键、数据分区、数据副本以及数据采样等特性。官方提供了包括 MergeTree、ReplacingMergeTree、SummingMergeTree、AggregatingMergeTree、CollapsingMergeTree、VersionedCollapsingMergeTree、GraphiteMergeTree 等 7 种不同类型的 MergeTree 引擎的实现

- **ReplacingMergeTree**： 在后台数据合并期间，对具有相同排序键的数据进行去重操作。
- **SummingMergeTree**： 当合并数据时，会把具有相同主键的记录合并为一条记录。根据聚合字段设置，该字段的值为聚合后的汇总值，非聚合字段使用第一条记录的值，聚合字段类型必须为数值类型。
- **AggregatingMergeTree**： 在同一数据分区下，可以将具有相同主键的数据进行聚合。
- **CollapsingMergeTree**： 在同一数据分区下，对具有相同主键的数据进行折叠合并。
- **VersionedCollapsingMergeTree**：
- **CollapsingMergeTree**，增添了数据版本信息字段配置选项。在数据依据 ORDER BY 设置对数据进行排序的基础上，如果数据的版本信息列不在排序字段中，那么版本信息会被隐式的作为 ORDER BY 的最后一列从而影响数据排序。
- **GraphiteMergeTree**： 用来存储时序数据库 Graphites 的数据。

###### MergeTree

1. 表创建

创建 MergeTree 的 DDL 如下所示：



```
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster] (      name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],       name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],       ... ) ENGINE = MergeTree()  ORDER BY expr   [PARTITION BY expr]   [PRIMARY KEY expr]   [SAMPLE BY expr]   [TTL expr [DELETE|TO DISK 'xxx'|TO VOLUME 'xxx'], ...]   [SETTINGS name=value, ...
```

复制代码



这里说明一下 MergeTree 引擎的主要参数：



- **必填选项**
- **ENGINE** ：引擎名字，MergeTree 引擎无参数。
- **ORDER BY** ：排序键，可以由一列或多列组成，决定了数据以何种方式进行排序，例如 ORDER BY（CounterID, EventDate）。如果没有显示指定 PRIMARY KEY，那么将使用 ORDER BY 作为 PRIMARY KEY。通常只指定 ORDER BY 即可。
- **选填选项**
- **PARTITION BY** ：分区键，指明表中的数据以何种规则进行分区。分区是在一个表中通过指定的规则划分而成的逻辑数据集。分区可以按任意标准进行，如按月、按日或按事件类型。为了减少需要操作的数据，每个分区都是分开存储的。
- **PRIMARY KEY** ：主键，设置后会按照主键生成一级索引（primary.idx），数据会依据索引的设置进行排序，从而加速查询性能。默认情况下，PRIMARY KEY 与 ORDER BY 设置相同，所以通常情况下直接使用 ORDER BY 设置来替代主键设置。
- **SAMPLE BY** ：数据采样设置，如果显示配置了该选项，那么主键配置中也应该包括此配置。例如 ORDER BY CounterID / EventDate / intHash32（UserID）、SAMPLE BY intHash32（UserID）。
- **TTL** ：数据存活时间，可以为某一字段列或者一整张表设置 TTL，设置中必须包含 Date 或 DateTime 字段类型。如果设置在列上，那么会删除字段中过期的数据。如果设置的是表级的 TTL，那么会删除表中过期的数据。如果设置了两种类型，那么按先到期的为准。例如，TTL createtime + INTERVAL 1 DAY，即一天后过期。使用场景包括定期删除数据，或者定期将数据进行归档。
- **index_granularity** ：索引间隔粒度。MergeTree 索引为稀疏索引，每 index_granularity 个数据产生一条索引。index_granularity 默认设置为 8092。
- **enable_mixed_granularity_parts** ：是否启动 index_granularity_bytes 来控制索引粒度大小。
- **index_granularity_bytes** ：索引粒度，以字节为单位，默认 10Mb。
- **merge_max_block_size** ：数据块合并最大记录个数，默认 8192。
- **merge_with_ttl_timeout** ：合并频率最小时间间隔，默认 1 天。

### 2.2 数据存储结构

首先创建一个 test 表，DDL 如下:



```
CREATE TABLE test.test (     id        UInt64,     type      UInt8,     create_time DateTime ) ENGINE = MergeTree()   PARTITION BY toYYYYMMDD(create_time)   ORDER BY (id)   SETTINGS index_granularity = 4;
```

复制代码



test 表包括 id、type、create 等三个字段，其中以 create_time 日期字段作为分区键，并将日期格式转化为 YYYYMMDD。按照 id 字段进行排序。由于没有显式设置主键，所以引擎默认使用 ORDER BY 设置的 id 列作为索引字段，并生成索引文件。index_granularity 设置为 4，意味着每 4 条数据产生一条索引数据。



插入一条测试数据：



```
insert into test.test(id, type, create_time) VALUES (1, 1, toDateTime('2021-03-01 00:00:00'));
```

复制代码



使用如下命令查看 test 表分区相关信息：



```
 SELECT         database,         table,         partition,         partition_id,         name,         active,         path  FROM system.parts  WHERE table = 'test' 
```

复制代码



返回结果如下图所示：



![img](https://static001.infoq.cn/resource/image/43/f1/43f14146a075b8a09417b34ec1abyyf1.png)



从上图中可以看到 test 表中返回了一条 partitionid 为 20210301 的数据分区的记录，从 name 字段中我们可以得知，此分区的目录名为 20210301_8_8_0。20210301_8_8_0 这个目录名字到底有什么含义呢？下面来介绍一下分区规则以及分区目录的命名规则。

#### 2.2.1 数据分区 ID 生成规则

数据分区规则由分区 ID 决定，分区 ID 由 PARTITION BY 分区键决定。根据分区键字段类型，ID 生成规则可分为：



- **未定义分区键**
- 没有定义 PARTITION BY，默认生成一个目录名为 all 的数据分区，所有数据均存放在 all 目录下。
- **整型分区键**
- 分区键为整型，那么直接用该整型值的字符串形式做为分区 ID。
- **日期类分区键**
- 分区键为日期类型，或者可以转化成日期类型。
- **其他类型分区键**
- String、Float 类型等，通过 128 位的 Hash 算法取其 Hash 值作为分区 ID。



上面我们插入一条日期为 2021-03-01 00:00:00 的数据，对该字段格式化后生成的数据分区 id 就是 20210301。

#### 2.2.2 数据分区目录命名规则

目录命名规则如下：



```
PartitionId_MinBlockNum_MaxBlockNum_Level
```

复制代码



- **PartitionID**
- 分区 id，例如 20210301。
- **MinBlockNum**
- 最小分区块编号，自增类型，从 1 开始向上递增。每产生一个新的目录分区就向上递增一个数字。
- **MaxBlockNum**
- 最大分区块编号，新创建的分区 MinBlockNum 等于 MaxBlockNum 的编号。
- **Level**
- 合并的层级，被合并的次数。合并次数越多，层级值越大。



![img](https://static001.infoq.cn/resource/image/da/b5/da1e10c251c73085ddffe598db5e07b5.png)



从上图可知，此分区的分区 id 为 20210301，当前分区的 MinBlockNum 和 MinBlockNum 均为 8，而 level 为 0，表示此分区没有合并过。

### 2.3 数据分区文件组织结构

在了解了分区目录名字的生成规则后，下面来看看数据分区目录下的文件组织结构。以 2021030188_0 分区为例：



![img](https://static001.infoq.cn/resource/image/6c/f3/6cc0b607ce0a9c35bc2aecc0e2e495f3.png)



从图中可以看到，目录中的文件主要包括 bin 文件、mrk 文件、primary.idx 文件以及其他相关文件。



- **bin 文件**
- 数据文件，存储的是某一列的数据。数据表中的每一列都对应一个与其字段名相同的 bin 文件，例如 id.bin 存储的是表 test 中 id 列的数据。
- **mrk 文件**
- 标记文件，每一列都对应一个与其字段名相同的标记文件，标记文件在 idx 索引文件和 bin 数据文件之间起到了桥梁作用。以 mrk2 结尾的文件，表示该表启用了自适应索引间隔。
- **primary.idx 文件**
- 主键索引文件，用于加快查询效率。
- **count.txt**
- 数据分区中数据总记录数。上述 20210301_8_8_0 的数据分区中，该文件中的记录总数为 1。
- **columns.txt**
- 表中所有列数的信息，包括字段名和字段类型。
- **partion.dat**
- 用于保存分区表达式的值。上述 20210301_8_8_0 的数据分区中该文件中的值为 20210301。
- **minmax_create_time.idx**
- 分区键的最大最小值。
- **checksums.txt**
- 校验文件，用于校验各个文件的正确性。存放各个文件的 size 以及 hash 值。

#### 2.3.1 数据文件

MergeTree 中，每列都对应一个 bin 文件单独存放该列数据。例如，id.bin 存放的是 id 列的数据。所有数据都经过数据压缩、排序，最后以数据块的形式写入 bin 文件中。bin 中数据以压缩数据块为单位写入文件中。每个数据块由头信息和压缩数据组成。头部信息包括校验和、数据压缩算法、数据压缩前大小和压缩后大小组成。压缩数据由 granule 组成，granule 大小与 index_granularity 相关。

#### 2.3.2 索引文件

MergeTree 索引为稀疏索引，它并不索引单条数据，而是索引一定范围的数据。也就是从已排序的全量数据中，间隔性的选取一些数据记录主键字段的值来生成 primary.idx 索引文件，从而加快表查询效率。间隔设置参数为 index_granularity。



![img](https://static001.infoq.cn/resource/image/bb/4c/bbc302f044278bbfcc3c16d63f15a34c.png)



我们向表 test 中插入 9 条数据，



```
insert into test.test(id, type, create_time) VALUES (1, 1, toDateTime('2021-03-01 00:00:00')); insert into test.test(id, type, create_time) VALUES (1, 2, toDateTime('2021-03-01 00:00:00')); insert into test.test(id, type, create_time) VALUES (1, 3, toDateTime('2021-03-01 00:00:00')); insert into test.test(id, type, create_time) VALUES (2, 1, toDateTime('2021-03-01 00:00:00')); insert into test.test(id, type, create_time) VALUES (2, 1, toDateTime('2021-03-01 00:00:00')); insert into test.test(id, type, create_time) VALUES (3, 1, toDateTime('2021-03-01 00:00:00')); insert into test.test(id, type, create_time) VALUES (3, 1, toDateTime('2021-03-01 00:00:00')); insert into test.test(id, type, create_time) VALUES (4, 1, toDateTime('2021-03-01 00:00:00')); insert into test.test(id, type, create_time) VALUES (5, 1, toDateTime('2021-03-01 00:00:00'));
```

复制代码



因为 index_granularity 设置为 4，所以每 4 条数据就会生成一条索引记录，即使用插入的第 1、5、9 条数据 id 字段的值生成索引文件记录。



![img](https://static001.infoq.cn/resource/image/d6/9c/d6ef6c6d12d669a97b0940e852caf29c.png)

#### 2.3.3 标记文件

mrk 标记文件在 primary.idx 索引文件和 bin 数据文件之间起到了桥梁作用。primary.idx 文件中的每条索引在 mrk 文件中都有对应的一条记录。一条记录的组成包括：



- **offset-compressed bin file**
- 表示指向的压缩数据块在 bin 文件中的偏移量。
- **offset-decompressed data block**
- 表示指向的数据在解压数据块中的偏移量。
- **row counts**
- 代表数据记录行数，小于等于 index_granularity 所设置的值。



![img](https://static001.infoq.cn/resource/image/57/d6/57c2c50d02c4bf7637007fb3b337d6d6.png)



索引，标记和数据文件下图所示：



![img](https://static001.infoq.cn/resource/image/b2/85/b2dabc9da1d48abca91627a46c08dd85.png)

#### 实践

