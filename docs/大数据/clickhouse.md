# ClickHouse是什么

> ***ClickHouse，允许在运行时创建表和数据库，加载数据和运行查询，而无需重新配置和重新启动服务器，支持线性扩展，简单方便，高可靠性，容错。它在大数据领域没有走 Hadoop 生态，而是采用 Local attached storage 作为存储，这样整个 IO 可能就没有 Hadoop 那一套的局限。它的系统在生产环境中可以应用到比较大的规模，因为它的线性扩展能力和可靠性保障能够原生支持 shard + replication 这种解决方案。它还提供了一些 SQL 直接接口，有比较丰富的原生 client。另外就是它比较快。***



# ClickHouse特点

## 列式存储

### 行级存储和列式存储的区别

1. 行级存储

   > ![](http://image.tinx.top/img20210410115157.png)
   >
   > 好处是想查某个人所有的属性时，可以通过一次磁盘查找加顺序读取就可以。但是当想查所有人的年龄时，需要不停的查找，或者全表扫描才行，遍历的很多数据都是不需要的。

2. 列式存储

   > ![](http://image.tinx.top/img20210410161903.png)
   >
   > 这时想查所有人的年龄只需把年龄那一列拿出来就可以了

### 列式存储优点

+ 对于列的聚合，计数，求和等统计操作原因优于行式存储。
+ 由于某一列的数据类型都是相同的，针对于数据存储更容易进行数据压缩，每一列选择更优的数据压缩算法，大大提高了数据的压缩比重。
+ 由于数据压缩比更好，一方面节省了磁盘空间，另一方面对于 cache 也有了更大的发挥空间。

## DBMS功能

几乎覆盖了标准 SQL 的大部分语法，包括 DDL 和 DML，以及配套的各种函数，用管理及权限管理，数据的备份与恢复



## 多引擎支持



## 高吞吐写入能力

ClickHouse 采用类 LSM Tree 的结构，数据写入后定期在后台 Compaction。通过类LSM tree 的结构，ClickHouse 在数据导入时全部是顺序 append 写，写入后数据段不可更改，在后台 compaction 时也是多个段 merge sort 后顺序写回磁盘。顺序写的特性，充分利用了磁盘的吞吐能力，即便在 HDD 上也有着优异的写入性能。



##  数据分区与线程级并行

ClickHouse 将数据划分为多个 partition，每个 partition 再进一步划分为多个 index granularity，然后通过多个 CPU 核心分别处理其中的一部分来实现并行数据处理。在这种设计下，单条 Query 就能利用整机所有 CPU。极致的并行处理能力，极大的降低了查询延时。

所以，ClickHouse 即使对于大量数据的查询也能够化整为零平行处理。但是有一个弊端就是对于单条查询使用多 cpu，就不利于同时并发多条查询。所以对于高 qps 的查询业务，ClickHouse 并不是强项。



# 数据类型

## 整型

有符号整型范围 （-2n-1~2n-1-1）：
Int8 - [-128 : 127]

Int16 - [-32768 : 32767]

Int32 - [-2147483648 : 2147483647]

Int64 - [-9223372036854775808 : 9223372036854775807]

无符号整型范围（0~2n-1）：

UInt8 - [0 : 255]

UInt16 - [0 : 65535]

UInt32 - [0 : 4294967295]

UInt64 - [0 : 18446744073709551615]

**使用场景： 个数、数量、也可以存储型 id。**

## 浮点型

Float32 - float

Float64 – double

建议尽可能以整数形式存储数据。例如，将固定精度的数字转换为整数值，如时间用毫秒为单位表示，因为浮点型进行计算时可能引起四舍五入的误差。

**使用场景：一般数据值比较小，不涉及大量的统计计算，精度要求不高的时候。比如保存商品的重量。**

## **Decimal 型**

有符号的浮点数，可在加、减和乘法运算过程中保持精度。对于除法，最低有效数字会

被丢弃（不舍入）。

有三种声明：

- Decimal32(s)，相当于 Decimal(9-s,s)，有效位数为 1~9

- Decimal64(s)，相当于 Decimal(18-s,s)，有效位数为 1~18

-  Decimal128(s)，相当于 Decimal(38-s,s)，有效位数为 1~38

s 标识小数位

**使用场景： 一般金额字段、汇率、利率等字段为了保证小数点精度，都使用 Decimal 进行存储。**

## **字符串**

String

字符串可以任意长度的。它可以包含任意的字节集，包含空字节。

FixedString(N)

固定长度 N 的字符串，N 必须是严格的正自然数。当服务端读取长度小于 N 的字符串时候，通过在字符串末尾添加空字节来达到 N 字节长度。 当服务端读取长度大于 N 的字符串时候，将返回错误消息。与 String 相比，极少会使用 FixedString，因为使用起来不是很方便。

**使用场景：名称、文字描述、字符型编码。 固定长度的可以保存一些定长的内容，比如一些编码，性别等但是考虑到一定的变化风险，带来收益不够明显，所以定长字符串使用意义有限。**

## **枚举类型**

包括 Enum8 和 Enum16 类型。Enum 保存 'string'= integer 的对应关系。

Enum8 用 'String'= Int8 对描述。

Enum16 用 'String'= Int16 对描述。

### 小例子

```sql
-- 这个 x 列只能存储类型定义中列出的值：'hello'或'world',存储其他会报错！
CREATE TABLE t_enum
(
x Enum8('hello' = 1, 'world' = 2)
)
ENGINE = TinyLog;
```

**使用场景：对一些状态、类型的字段算是一种空间优化，也算是一种数据约束。但是实际使用中往往因为一些数据内容的变化增加一定的维护成本，甚至是数据丢失问题。所以谨慎使用。**

## **时间类型**

目前 ClickHouse 有三种时间类型

- Date 接受**年-月-日**的字符串比如 ‘2019-12-16’ 

- Datetime 接受**年-月-日 时:分:秒**的字符串比如 ‘2019-12-16 20:50:10’ 

-  Datetime64 接 受 **年 - 月 - 日 时 : 分 : 秒 . 亚 秒** 的 字 符 串 比 如 ‘ 2019-12-16 20:50:10.66’

日期类型，用两个字节存储，表示从 1970-01-01 (无符号) 到当前的日期值。

## **数组**

**Array(T)：**由 T 类型元素组成的数组。

T 可以是任意类型，包含数组类型。 但不推荐使用多维数组，ClickHouse 对多维数组的支持有限。例如，不能在 MergeTree 表中存储多维数组。



# 表引擎

表引擎是 ClickHouse 的一大特色。可以说， 表引擎决定了如何存储标的数据。包括：

+ 数据的存储方式和位置，写到哪里以及从哪里读取数据

+ 支持哪些查询以及如何支持。

+ 并发数据访问。

+ 索引的使用（如果存在）。

+ 是否可以执行多线程请求。

+ 数据复制参数。

表引擎的使用方式就是必须显式在创建表时定义该表使用的引擎，以及引擎使用的相关参数。

<font color="red">**特别注意：引擎的名称大小写敏感**</font>

## 日志引擎

### **TinyLog**

最简单的表引擎，用于将数据存储在磁盘上。每列都存储在单独的压缩文件中，写入时，数据将附加到文件末尾。

该引擎没有并发控制 

  \- 只支持并发读

\- 如果同时从表中读取和写入数据，则读取操作将抛出异常；

\- 如果同时写入多个查询中的表，则数据将被破坏。

```sql
create table t_tinylog ( id String, name String) engine=TinyLog;
```

这种表引擎的典型用法是 write-once：首先只写入一次数据，然后根据需要多次读取。此引擎适用于相对较小的表（建议最多1,000,000行）。如果有许多小表，则使用此表引擎是适合的，因为它比需要打开的文件更少。当拥有大量小表时，可能会导致性能低下。 不支持索引。

### StripeLog(数据分块列在一起)

在你需要写入许多小数据量（小于一百万行）的表的场景下使用这个引擎

```sql
CREATE TABLE stripe_log_table(
    timestamp DateTime,
    message_type String,
message String)ENGINE = StripeLog ;
```

StripeLog 引擎将所有列存储在一个文件中。对每一次 Insert 请求，ClickHouse 将数据块追加在表文件的末尾，逐列写入。

ClickHouse 为每张表写入以下文件：

data.bin — 数据文件。

index.mrk — 带标记的文件。标记包含了已插入的每个数据块中每列的偏移量。

StripeLog 引擎不支持 ALTER UPDATE 和 ALTER DELETE 操作。

读数据

带标记的文件使得 ClickHouse 可以并行的读取数据。这意味着 SELECT 请求返回行的顺序是不可预测的。使用 ORDER BY 子句对行进行排序。

我们使用两次 INSERT 请求从而在 data.bin 文件中创建两个数据块。

ClickHouse 在查询数据时使用多线程。每个线程读取单独的数据块并在完成后独立的返回结果行。这样的结果是，大多数情况下，输出中块的顺序和输入时相应块的顺序是不同的。

### Log(数据分块记录偏移量)

日志与 TinyLog 的不同之处在于，«标记» 的小文件与列文件存在一起。**这些标记写在每个数据块上，并且包含偏移量，这些偏移量指示从哪里开始读取文件以便跳过指定的行数。**这使得可以在多个线程中读取表数据。对于并发数据访问，可以同时执行读取操作，而写入操作则阻塞读取和其它写入。**Log 引擎不支持索引。**同样，**如果写入表失败，则该表将被破坏，并且从该表读取将返回错误。Log 引擎适用于临时数据，write-once 表以及测试或演示目的。**

 

-- 建表

create table tb_log(id Int8 , name String , age Int8) engine=Log ;

### 总结

#### 共同属性

> 1. 数据存储在磁盘上。
>
> 2. 写入时将数据追加在文件末尾。
>
> 3. 不支持突变操作。
>
> 4. 不支持索引。
>
>    这意味着 `SELECT` 在范围查询时效率不高。
>
> 5. 非原子地写入数据。
>
> 如果某些事情破坏了写操作，例如服务器的异常关闭，你将会得到一张包含了损坏数据的表。

#### 差异

> Log 和 StripeLog 引擎支持：
>
> 并发访问数据的锁。
>
> `INSERT` 请求执行过程中表会被锁定，并且其他的读写数据的请求都会等待直到锁定被解除。如果没有写数据的请求，任意数量的读请求都可以并发执行。
>
> 并行读取数据。
>
> 在读取数据时，ClickHouse 使用多线程。 每个线程处理不同的数据块。
>
> **Log 引擎为表中的每一列使用不同的文件。StripeLog 将所有的数据存储在一个文件中。因此 StripeLog 引擎在操作系统中使用更少的描述符，但是 Log 引擎提供更高的读性能。**
>
>  
>
> TinyLog 引擎是该系列中最简单的引擎并且提供了最少的功能和最低的性能。TingLog 引擎不支持并行读取和并发数据访问，并将每一列存储在不同的文件中。它比其余两种支持并行读取的引擎的读取速度更慢，并且使用了和 Log 引擎同样多的描述符。你可以在简单的低负载的情景下使用它。



## MergeTree家族引擎

MergeTree系列的表引擎是ClickHouse数据存储功能的核心。**<font color=red>它们提供了用于弹性和高性能数据检索的大多数功能：列存储，自定义分区，稀疏的主索引，辅助数据跳过索引等。</font>**

基本[MergeTree](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/mergetree/)表引擎可以被认为是单节点ClickHouse实例的默认表引擎，因为它在各种用例中通用且实用。

对于生产用途，[ReplicatedMergeTree](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/replication/)是必经之路，因为它为常规MergeTree引擎的所有功能增加了高可用性。一个额外的好处是在数据提取时自动进行重复数据删除，因此如果插入过程中出现网络问题，该软件可以安全地重试。

MergeTree系列的所有其他引擎为某些特定用例添加了额外的功能。通常，它是作为后台的其他数据操作实现的。

MergeTree引擎的主要缺点是它们很重。因此，典型的模式是没有太多。如果您需要许多小表（例如用于临时数据），请考虑使用[Log engine family](https://clickhouse.tech/docs/en/engines/table-engines/log-family/)。

· 允许快速写入不断变化的对象状态。

· 删除后台中的旧对象状态。 这显着降低了存储体积。



主要特点：

+ 存储按主键排序的数据。

这使您可以创建一个小的稀疏索引，以帮助更快地查找数据。

+ 如果指定了[分区键，](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/custom-partitioning-key/)则可以使用[分区](https://clickhouse.tech/docs/en/engines/table-engines/mergetree-family/custom-partitioning-key/)。

ClickHouse支持的某些分区操作比对相同数据，相同结果的常规操作更有效。ClickHouse还会自动切断在查询中指定了分区键的分区数据。这也提高了查询性能。

+ 数据复制支持。

ReplicatedMergeTree表族提供数据复制。有关更多信息.

+ 数据采样支持。

如有必要，可以在表中设置数据采样方法。

### **MergeTree**

ClickHouse 中最强大的表引擎当属 MergeTree（合并树）引擎及该系列（*MergeTree）中的其他引擎，支持索引和分区，地位可以相当于 innodb 之于 Mysql。 而且基于MergeTree，还衍生除了很多小弟，也是非常有特色的引擎。

```sql
create table t_order_mt(
id UInt32,
sku_id String,
total_amount Decimal(16,2),
create_time Datetime
) engine =MergeTree
partition by toYYYYMMDD(create_time)
primary key (id)
order by (id,sku_id);
```

#### **partition by 分区 （可选项）**

#### 作用

分区的目的主要是降低扫描的范围，优化查询速度

#### **分区目录**

MergeTree 是以列文件+索引文件+表定义文件组成的，但是如果设定了分区那么这些文件就会保存到不同的分区目录中。

#### **并行**

分区后，面对涉及跨分区的查询统计，ClickHouse 会以分区为单位并行处理。

#### **数据写入与分区合并**

任何一个批次的数据写入都会产生一个临时分区，不会纳入任何一个已有的分区。写入后的某个时刻（大概 10-15 分钟后），ClickHouse 会自动执行合并操作（等不及也可以手动通过 optimize 执行），把临时分区的数据，合并到已有分区中。

```sql
optimize table xxxx final;
```

#### **primary key 主键(可选)**

ClickHouse 中的主键，和其他数据库不太一样，<font color=red>**它只提供了数据的一级索引，但是却不是唯一约束**</font>。这就意味着是可以存在相同 primary key 的数据的。

主键的设定主要依据是查询语句中的 where 条件。

根据条件通过对主键进行某种形式的二分查找，能够定位到对应的 index granularity,避免了全表扫描。

index granularity： 直接翻译的话就是索引粒度，指在稀疏索引中两个相邻索引对应数据的间隔。ClickHouse 中的 MergeTree 默认是 8192。官方不建议修改这个值，除非该列存在大量重复值，比如在一个分区中几万行才有一个不同数据。

![](http://image.tinx.top/img20210410133107.png)

稀疏索引的好处就是可以用很少的索引数据，定位更多的数据，代价就是只能定位到索引粒度的第一行，然后再进行进行一点

#### **order by（必选）**

**order by 设定了分区内的数据按照哪些字段顺序进行有序保存。**

order by 是 MergeTree 中唯一一个必填项，甚至比 primary key 还重要，因为当用户不设置主键的情况，很多处理会依照 order by 的字段进行处理

<font color=red>**要求：主键必须是 order by 字段的前缀字段。**</font>

比如 order by 字段是 (id,sku_id) 那么主键必须是 id 或者(id,sku_id)

#### **二级索引 测试**

##### **使用二级索引前需要增加设置**

```sql
set allow_experimental_data_skipping_indices=1;
```

##### 创建测试表

```sql
create table t_order_mt2(
id UInt32,
sku_id String,
total_amount Decimal(16,2),
create_time Datetime,
INDEX a total_amount TYPE minmax GRANULARITY 5
) engine =MergeTree
partition by toYYYYMMDD(create_time)
primary key (id)
order by (id, sku_id);
```

其中 **GRANULARITY N** 是设定二级索引对于一级索引粒度的粒度。

#### TTL

TTL 即 Time To Live，MergeTree 提供了可以管理数据或者列的生命周期的功能。

##### **列级别 TTL**

```sql
create table t_order_mt3(
id UInt32,
sku_id String,
total_amount Decimal(16,2) TTL create_time+interval 10 SECOND,
create_time Datetime 
) engine =MergeTree
partition by toYYYYMMDD(create_time)
primary key (id)
order by (id, sku_id);
```

##### **表级 TTL**

```sql
alter table t_order_mt3 MODIFY TTL create_time + INTERVAL 10 SECOND;
```

> 涉及判断的字段必须是 Date 或者 Datetime 类型，推荐使用分区的日期字段。
>
> 能够使用的时间周期：
>
> \- SECOND
>
> \- MINUTE
>
> \- HOUR
>
> \- DAY
>
> \- WEEK
>
> \- MONTH
>
> \- QUARTER
>
> \- YEAR 

### **ReplacingMergeTree**

ReplacingMergeTree 是 MergeTree 的一个变种，它存储特性完全继承 MergeTree,只是多了一个去重的功能。 尽管 MergeTree 可以设置主键，但是 primary key 其实没有唯一约束的功能。如果你想处理掉重复的数据，可以借助这个 ReplacingMergeTree。 

#### **去重时机**

数据的去重只会在合并的过程中出现。合并会在未知的时间在后台进行，所以你无法预先作出计划。有一些数据可能仍未被处理。

#### **去重范围**

如果表经过了分区，去重只会在分区内部进行去重，不能执行跨分区的去重。

所以 ReplacingMergeTree 能力有限，ReplacingMergeTree 适用于在后台清除重复的数据以节省空间，但是它不保证没有重复的数据出现。

<font color=red>**ReplacingMergeTree() 填入的参数为版本字段，重复数据保留版本字段值最大的。如果不填版本字段，默认按照插入顺序保留最后一条。**</font>

### **SummingMergeTree**

对于不查询明细，只关心以维度进行汇总聚合结果的场景。如果只使用普通的MergeTree 的话，无论是存储空间的开销，还是查询时临时聚合的开销都比较大。

ClickHouse 为了这种场景，提供了一种能够“预聚合”的引擎 SummingMergeTree

+ 以 SummingMergeTree（）中指定的列作为汇总数据列 
+ 可以填写多列必须数字列，如果不填，以所有非维度列且为数字列的字段为汇总数据列
+ 以 order by 的列为准，作为维度列
+ 其他的列按插入顺序保留第一行
+ 不在一个分区的数据不会被聚合

#### **开发建议**

设计聚合表的话，唯一键值、流水号可以去掉，所有字段全部是维度、度量或者时间戳。

## 其他引擎

### **Memory**

内存引擎，数据以未压缩的原始形式直接保存在内存当中，服务器重启数据就会消失。读写操作不会相互阻塞，不支持索引。简单查询下有非常非常高的性能表现（超过 10G/s）。

一般用到它的地方不多，除了用来测试，就是在需要非常高的性能，同时数据量又不太大（上限大概 1 亿行）的场景。

### 分布式引擎

分布式引擎，是CH本身不存储数据, 但可以在多个服务器上进行分布式查询。 读是自动并行的。读取时，远程服务器表的索引（如果有的话）会被使用。 

Distributed(cluster_name, database, table [, sharding_key])

参数解析：

+ cluster_name  - 服务器配置文件中的集群名,在/etc/metrika.xml中配置的

+ database – 数据库名

+ table – 表名

+ sharding_key – 数据分片键

### 缓存引擎

缓冲数据以写入内存，并定期将其刷新到另一个表。在读取操作期间，同时从缓冲区和另一个表读取数据。

Buffer(database, table, num_layers, min_time, max_time, min_rows, max_rows, min_bytes, max_bytes)

引擎参数：

database- 数据库名称。可以使用返回字符串的常量表达式来代替数据库名称。

table –将数据刷新到的表。

num_layers–并行层。在物理上，该表将表示为num_layers独立的缓冲区。建议值：16。

min_time，max_time，min_rows，max_rows，min_bytes，和max_bytes-条件从缓冲区刷新数据。

如果满足所有min*条件或至少一个max*条件，则从缓冲区刷新数据并将其写入目标表。

min_time，max_time–从第一次写入缓冲区起的时间（以秒为单位）。

min_rows，max_rows–缓冲区中的行数条件。

min_bytes，max_bytes–缓冲区中字节数的条件。

在写操作期间，数据被插入到num_layers许多随机缓冲区中。或者，如果要插入的数据部分足够大（大于max_rows或max_bytes），则将其直接写入目标表，而忽略缓冲区。

对于每个num_layers缓冲区，分别计算刷新数据的条件。例如，如果num_layers = 16和max_bytes = 100000000，则最大RAM消耗为1.6 GB。

例：

CREATE TABLE merge.hits_buffer AS merge.hits ENGINE = Buffer(merge, hits, 16, 10, 100, 10000, 1000000, 10000000, 100000000)

创建一个具有与'merge.hits'相同结构的'merge.hits_buffer'表，并使用Buffer引擎。写入此表时，数据会缓冲在RAM中，然后再写入“ merge.hits”表。创建了16个缓冲区。如果经过了100秒，或者已写入一百万行，或者已写入100 MB数据，则将刷新其中每个数据；或者同时经过10秒并写入10,000行和10 MB数据。例如，如果只写了一行，则无论如何，在100秒后它将被刷新。但是，如果已写入许多行，则将更快地刷新数据。

**当服务器停止时，使用DROP TABLE或DETACH TABLE，缓冲区数据也将刷新到目标表。**

您可以在数据库名称和表名称的单引号中设置空字符串。这表明没有目标表。在这种情况下，当达到数据刷新条件时，只需清除缓冲区。这对于将数据窗口保留在内存中可能很有用。

从缓冲区表读取数据时，将从缓冲区和目标表（如果有的话）中处理数据。
请注意

+ 缓冲区表不支持索引。换句话说，缓冲区中的数据已被完全扫描，这对于大型缓冲区而言可能很慢。（对于下级表中的数据，将使用其支持的索引。）

+ 如果“缓冲区”表中的列集与从属表中的列集不匹配，则插入两个表中都存在的列子集。

+ 如果类型与缓冲区表和从属表中的任一列都不匹配，则会在服务器日志中输入错误消息，并清除缓冲区。
  如果刷新缓冲区时从属表不存在，也会发生相同的情况。

+ 如果需要对下级表和Buffer表运行ALTER，建议先删除Buffer表，对下级表运行ALTER，然后再次创建Buffer表。

+ 如果服务器异常重启，缓冲区中的数据将会丢失。

+ FINAL和SAMPLE对于缓冲区表不能正常工作。这些条件将传递到目标表，但不用于处理缓冲区中的数据。如果需要这些功能，建议从目标表读取时仅使用缓冲区表进行写入。

+ 将数据添加到缓冲区时，缓冲区之一被锁定。如果同时从表执行读取操作，则会导致延迟。

+ 插入到缓冲区表中的数据可能以不同的顺序和不同的块最终出现在从属表中。因此，很难使用Buffer表正确地写入CollapsingMergeTree。为了避免出现问题，可以将“ num_layers”设置为1。

+ 如果目标表被复制，则写入缓冲区表时，复制表的某些预期特性会丢失。数据部分的行顺序和大小的随机变化会导致重复数据删除退出工作，这意味着不可能对复制表进行可靠的“仅一次”写入。

+ 由于这些缺点，我们仅建议在极少数情况下使用Buffer表。

+ 当在一个单位时间内从大量服务器接收到太多INSERT且无法在插入之前对数据进行缓冲的情况下，将使用Buffer表，这意味着INSERT不能足够快地运行。

+ 注意，即使一次插入缓冲区表也没有意义。这样只会产生每秒几千行的速度，而插入更大的数据块则每秒会产生一百万行以上的速度（请参阅“性能”一节）。

### 文件引擎

File表引擎以特殊的[文件格式](#formats)（TabSeparated，Native等）将数据保存在文件中。

用法示例： File(Format)

+ 数据从ClickHouse导出到文件。

+ 将数据从一种格式转换为另一种格式。

+ 通过编辑磁盘上的文件来更新ClickHouse中的数据。

#### 示例

1. 建表

```sql
create table tb_file_demo1(uid UInt16 , name String) engine=File(TabSeparated) ;
```

2. 在指定的目录下创建数据文件

```sql
/var/lib/ClickHouse/data/default/tb_file_demo1
[root@linux01 tb_file_demo1]# vi data.TabSeparated   注意这个文件的名字不能变化
1001    TaoGe
1002    XingGe
1003    HANGGE
```

3. 查询数据

```sql
select * from tb_file_demo1;
```

![](http://image.tinx.top/img20210410150237.png)

### Merge

但可用于同时从任意多个其他的表中读取数据。 读是自动并行的，不支持写入。读取时，那些被真正读取到数据的表的索引（如果有的话）会被使用。 

Merge 引擎的参数：一个数据库名和一个用于匹配表名的正则表达式。

```sql
create table tt1 (id UInt16, name String) ENGINE=TinyLog;

create table tt2 (id UInt16, name String) ENGINE=TinyLog;

create table tt3 (id UInt16, name String) ENGINE=TinyLog;

 

insert into tt1(id, name) values (1, 'Taoge');

insert into tt2(id, name) values (2, 'Xingge');

insert into tt3(id, name) values (3, 'Hangge');

 

create table t (id UInt16, name String) ENGINE=Merge(currentDatabase(), '^tt');
```



## 集成引擎

+ Kafka

+ MySQL

+ ODBC

+ JDBC

+ HDFS

  ```sql
  CREATE TABLE hdfs_engine_table
  (
      `name` String,
      `age` UInt32
  )
  ENGINE = HDFS('hdfs://doit01:8020/data/ch/demo1.csv', 'CSV')
  ```

  



# 导出数据

```bash
clickhouse-client --query "select * from t_order_mt where create_time='2021-06-01 12:00:00'" --format CSVWithNames> /opt/module/data/rs1.csv
```



# 副本

> 副本的目的主要是保障数据的高可用性，即使一台 ClickHouse 节点宕机，那么也可以从其他服务器获得相同的数据。

## 写入流程

![](http://image.tinx.top/img20210410135200.png)

### 配置步骤

+ 启动 zookeeper 集群
+ 在hadoop202的/etc/clickhouse-server/config.d目录下创建一个名为metrika.xml的配置文件,内容如下：

```xml
<?xml version="1.0"?>
<yandex>
	<zookeeper-servers>
		<node index="1">
			<host>hadoop202</host>
			<port>2181</port>
		</node>
		<node index="2">
			<host>hadoop203</host>
			<port>2181</port>
		</node>
		<node index="3">
			<host>hadoop204</host>
			<port>2181</port>
		</node>
	</zookeeper-servers>
</yandex>
```

+ 在server的的/etc/clickhouse-server/config.xml 中增加

  ```xml
  <include_from>/etc/clickhouse-server/config.d/metrika.xml</include_from>
  ```

+ 同步配置文件(给所有集群)

+ 重启服务

<font color=red>**副本只能同步数据，不能同步表结构，所以我们需要在每台机器上自己手动建表**</font>

建表语句

```sql
create table t_order_rep (
id UInt32,
sku_id String,
total_amount Decimal(16,2),
create_time Datetime
) engine =ReplicatedMergeTree('/clickhouse/tables/01/t_order_rep','分片名称')
partition by toYYYYMMDD(create_time)
primary key (id)
order by (id,sku_id);
```

注意：

<font color=red>**第二个参数是**副本名称，相同的分片副本名称不能相同。</font>



# **分片集群**

副本虽然能够提高数据的可用性，降低丢失风险，但是每台服务器实际上必须容纳全量数据，对数据的横向扩容没有解决。



要解决数据水平切分的问题，需要引入分片的概念。通过分片把一份完整的数据进行切分，不同的分片分布到不同的节点上，再通过 Distributed 表引擎把数据拼接起来一同使用。



Distributed 表引擎本身不存储数据，有点类似于 MyCat 之于 MySql，成为一种中间件，通过分布式逻辑表来写入、分发、路由来操作多台节点不同分片的分布式数据。



<font color=red>注意：ClickHouse 的集群是表级别的，实际企业中，大部分做了高可用，但是没有用分片，避免降低查询性能以及操作集群的复杂性。</font>

## **集群写入流程（3 分片 2 副本共 6 个节点）**

![](http://image.tinx.top/img20210410140424.png)



## **集群读 取流 程 （ 3 分片 2 副本共 6 个 节 点）**

![](http://image.tinx.top/img20210410140516.png)

## **3 分片 2 副本共 6 个节点集群配置**

配置的位置还是在之前的/etc/clickhouse-server/config.d/metrika.xml，内容如下

```
<yandex>
    <clickhouse_remote_servers>
        <gmall_cluster> <!-- 集群名称-->
            <shard> <!--集群的第一个分片-->
                <internal_replication>true</internal_replication>
                <!--该分片的第一个副本-->
                <replica>
                    <host>hadoop201</host>
                    <port>9000</port>
                </replica>
                <!--该分片的第二个副本-->
                <replica>
                    <host>hadoop202</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard> <!--集群的第二个分片-->
                <internal_replication>true</internal_replication>
                <replica> <!--该分片的第一个副本-->
                    <host>hadoop203</host>
                    <port>9000</port>
                </replica>
                <replica> <!--该分片的第二个副本-->
                    <host>hadoop204</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard> <!--集群的第三个分片-->
                <internal_replication>true</internal_replication>
                <replica> <!--该分片的第一个副本-->
                    <host>hadoop205</host>
                    <port>9000</port>
                </replica>
                <replica> <!--该分片的第二个副本-->
                    <host>hadoop206</host>
                    <port>9000</port>
                </replica>
            </shard>
        </gmall_cluster>
    </clickhouse_remote_servers>
</yandex>
```





















