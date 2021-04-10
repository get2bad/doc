# HBase特点

1. 海量存储

2. 列式存储

3. 极易扩展

4. 高并发

5. **稀疏**

   > 稀疏主要是针对 Hbase 列的灵活性，在列族中，你可以指定任意多的列，在列数据为空的情况下，是不会占用存储空间的。



# Hbase架构图

![](http://image.tinx.top/img20210409114302.png)

## 主要组件

从上图中可以看出HBase是由Client、Zookeeper、Master、HRegionServer、HDFS等几个组件组成

1. Client

   Client包含了访问HBase接口，另外Client还维护了对应的cache来加速HBase访问，比如cache的.META元数据信息。

2. Zookeeper

   HBase通过Zookeeper来做master的高可用、RegionServer的监控、元数据的入口以及集群配置的维护等工作：

   ​	通过 Zoopkeeper 来保证集群中只有 1 个 master 在运行，如果 master 异常，会通过竞争机制产生新的 master 提供服务

   ​	通过 Zoopkeeper 来监控 RegionServer 的状态，当 RegionSevrer 有异常的时候，通过回调的形式通知 Master RegionServer 上下线的信息

   ​	通过 Zoopkeeper 存储元数据的统一入口地址

3. HMaster

   master主要职责：

   ​	1．监控 RegionServer

   ​	2．处理 RegionServer 故障转移

   ​	3．处理元数据的变更

   ​	4．处理 region 的分配或转移 

   ​	5．在空闲时间进行数据的负载均衡

   ​	6．通过 Zookeeper 发布自己的位置给客户端

4. **HregionServer**

   HregionServer 直接对接用户的读写请求，他的功能：

   ​	1．负责存储 HBase 的实际数据

   ​	2．处理分配给它的 Region

   ​	3．刷新缓存到 HDFS

   ​	4．维护 Hlog

   ​	5．执行压缩

   ​	6．负责处理 Region 分片

5. HDFS

   HDFS 为 Hbase 提供最终的底层数据存储服务，同时为 HBase 提供高可用（Hlog 存储在HDFS）的支持，具体功能概括如下：

   提供元数据和表数据的底层分布式存储服务 

   数据多副本，保证的高可靠和高可用性

## 其他组件

1. Write-Ahead logs(WAL)

**HBase 的修改记录，当对 HBase 读写数据的时候，数据不是直接写进磁盘，它会在内存中保留一段时间（时间以及数据量阈值可以设定）。但把数据保存在内存中可能有更高的概率引起数据丢失，为了解决这个问题，数据会先写在一个叫做 Write-Ahead logfile 的文件中，然后再写入内存中。所以在系统出现故障的时候，数据可以通过这个日志文件重建。**

2. Region

Hbase 表的分片，HBase 表会根据 RowKey值被切分成不同的 region 存储在 RegionServer中，在一个 RegionServer 中可以有多个不同的 region。

3. Store

HFile 存储在 Store 中，一个 Store 对应 HBase 表中的一个列族。

4. MemStore

内存存储，位于内存中，用来保存当前的数据操作，所以当数据保存在WAL 中之后，RegsionServer 会在内存中存储键值对。

5. HFile

这是在磁盘上保存原始数据的实际的物理文件，是实际的存储文件。StoreFile 是以 Hfile的形式存储在 HDFS 的。



# 数据结构

## RowKey

与 nosql 数据库们一样,**RowKey 是用来检索记录的主键**。访问 HBASE table 中的行，只

有三种方式：

​	1.通过单个 RowKey 访问

​	2.通过 RowKey 的 range（正则）

​	3.全表扫描

RowKey 行键 (RowKey)可以是任意字符串(最大长度是 64KB，实际应用中长度一般为10-100bytes)，在 HBASE 内部，RowKey 保存为字节数组。存储时，数据按照 RowKey 的字典序(byte order)排序存储。<font color=red>设计 RowKey 时，要充分排序存储这个特性，将经常一起读取的行存储放到一起。(位置相关性) 

## Column Family

列族：HBASE 表中的每个列，都归属于某个列族。列族是表的 schema 的一部 分(而列不是)，必须在使用表之前定义。列名都以列族作为前缀。例如 courses:history，courses:math都属于 courses 这个列族。

## Cell

由{rowkey, column Family:columu, version} 唯一确定的单元。cell 中的数据是没有类型的，全部是字节码形式存贮。

关键字：无类型、字节码

##  Time Stamp

HBASE 中通过 rowkey和 columns 确定的为一个存贮单元称为cell。每个 cell都保存 着同一份数据的多个版本。版本通过时间戳来索引。时间戳的类型是 64 位整型。时间戳可以由 HBASE(在数据写入时自动 )赋值，此时时间戳是精确到毫秒 的当前系统时间。时间戳也可以由客户显式赋值。如果应用程序要避免数据版 本冲突，就必须自己生成具有唯一性的时间戳。每个 cell 中，不同版本的数据按照时间倒序排序，即最新的数据排在最前面。

为了避免数据存在过多版本造成的的管理 (包括存贮和索引)负担，HBASE 提供 了两种数据版本回收方式。一是保存数据的最后 n 个版本，二是保存最近一段 时间内的版本（比如最近七天）。用户可以针对每个列族进行设置。

## 命名空间 Schema

![](http://image.tinx.top/img20210409144545.png)

**1) Table**：表，所有的表都是命名空间的成员，即表必属于某个命名空间，如果没有指定，

则在 default 默认的命名空间中。

**2) RegionServer group**：一个命名空间包含了默认的 RegionServer Group。

**3) Permission**：权限，命名空间能够让我们来定义访问控制列表 ACL（Access Control List）。

例如，创建表，读取表，删除，更新等等操作。

**4) Quota**：限额，可以强制一个命名空间可包含的 region 的数量。



# HBase原理

## 读

![](http://image.tinx.top/img20210409144707.png)

1. Client 先访问 zookeeper，从 meta 表读取 region 的位置，然后读取 meta 表中的数据。meta中又存储了用户表的 region 信息；
2. 根据 namespace、表名和 rowkey 在 meta 表中找到对应的 region 信息；
3. 找到这个 region 对应的 regionserver； 
4. 查找对应的 region； 
5. 先从 MemStore 找数据，如果没有，再到 BlockCache 里面读；
6. BlockCache 还没有，再到 StoreFile 上读(为了读取的效率)； 
7. <font color=red>如果是从 StoreFile 里面读取的数据，不是直接返回给客户端，而是先写入 BlockCache，再返回给客户端。</font>



## 写

![](http://image.tinx.top/img20210409145100.png)



1. Client 向Zookeeper 发送请求，获取meta表所在RegionServer
2. zookeeper返回meta表所在的RegionServer
3. 请求这个RegionServer
4. 返回RegionServer
5. 发送写入数据的请求
6. 将数据写到HLog，为了数据的持久化和恢复
7. HRegionServer将数据写到内存
8. 返回写入成功！

## Flush

1. 当 MemStore 数据达到阈值（默认是 128M，老版本是 64M），将数据刷到硬盘，将内存中的数据删除，同时删除 HLog 中的历史数据；
2. 并将数据存储到 HDFS 中；
3. 在 HLog 中做标记点。

## 数据合并过程

1. 当数据块达到 4 块，Hmaster 触发合并操作，Region 将数据块加载到本地，进行合并；

2. 当合并的数据超过 256M，进行拆分，将拆分后的 Region 分配给不同的 HregionServer管理；

3. 当HregionServer宕机后，将HregionServer上的hlog拆分，然后分配给不同的HregionServer加载，修改.META.； 

   注意：HLog 会同步到 HDFS



# HBase 和 Hive 的对比

## Hive

(1) 数据仓库

<font color=red>Hive 的本质其实就相当于将 HDFS 中已经存储的文件在 Mysql 中做了一个双射关系，以方便使用 HQL 去管理查询。</font>

(2) 用于数据分析、清洗

**Hive 适用于离线的数据分析和清洗，延迟较高。**

(3) 基于 HDFS、MapReduce

**Hive 存储的数据依旧在 DataNode 上，编写的 HQL 语句终将是转换为 MapReduce 代码执行。**



## HBase

(1) 数据库

**是一种面向列存储的非关系型数据库。**

(2) 用于存储结构化和非结构化的数据

**适用于单表非关系型数据的存储，不适合做关联查询，类似 JOIN 等操作。**

(3) 基于 HDFS

**数据持久化存储的体现形式是 Hfile，存放于 DataNode 中，被 ResionServer 以 region 的形式进行管理。**

(4) 延迟较低，接入在线业务使用

**面对大量的企业数据，HBase 可以直线单表大量数据的存储，同时提供了高效的数据访问速度。**



# HBase优化

## 配置高可用

1. 在 conf 目录下创建 backup-masters 文件

```bash
touch conf/backup-masters
```

2. 在 backup-masters 文件中配置高可用 HMaster 节点

```bash
echo hadoop103 > conf/backup-masters
```

3. 将整个 conf 目录 scp 到其他节点



## 预分区

每一个 region 维护着 startRow 与 endRowKey，如果加入的数据符合某个 region 维护的rowKey 范围，则该数据交给这个 region 维护。那么依照这个原则，我们可以将数据所要投放的分区提前大致的规划好，以提高 HBase 性能

1. 手动设定预分区

```hbase
hbase> create 'staff1','info','partition1',SPLITS => ['1000','2000','3000','4000']
```

2. 生成16进制序列预分区

```hbase
create 'staff2','info','partition2',{NUMREGIONS => 15, SPLITALGO => 'HexStringSplit'}
```

3. 按照文件中设置的规则预分区

创建 splits.txt 文件内容如下：

```text
aaaa
bbbb
cccc
dddd
```

执行：

```hbase
create 'staff3','partition3',SPLITS_FILE => 'splits.txt'
```



## RowKey设计

<font color=red>一条数据的唯一标识就是 rowkey，那么这条数据存储于哪个分区，取决于 rowkey 处于哪个一个预分区的区间内，设计 rowkey的主要目的 ，就是让数据均匀的分布于所有的 region中，**在一定程度上防止数据倾斜**。</font>

### 原则

#### 长度原则

rowKey是一个二进制，RowKey的长度被很多开发者建议说设计在10~100个字节，以byte[]形式保存，最大不能超过64kb。建议越短越好，不要超过16个字节。

太长的影响有几点点：

- 一是HBase的持久化文件HFile是按照KeyValue存储的，如果RowKey过长，比如说500个字节，1000万列数据，光是RowKey就要占用500*1000万=50亿个字节，将近1G数据，极大影响了HFile的存储效率。
- 二是缓存MemStore缓存部分数据到内存中，如果RowKey字段过长，内存的有效利用率会降低，系统无法缓存更多的数据，降低检索效率。
- 目前操作系统都是64位系统，内存8字节对齐，控制在16字节，8字节的整数倍利用了操作系统的最佳特性。

注意：不仅RowKey的长度是越短越好，而且列簇名、列名等尽量使用短名字，因为HBase属于列式数据库，这些名字都是会写入到HBase的持久化文件HFile中去，过长的RowKey、列簇、列名都会导致整体的存储量成倍增加。

#### 唯一原则

保证rowKey的唯一性。由于在HBase中数据存储是Key-Value形式，若HBase中同一表插入相同RowKey，则原先的数据会被覆盖掉（如果表的version设置为1的话）。

#### 散列原则

设计的RowKey应均匀分布在各个HBase节点上。如RowKey是按系统时间戳的方式递增，RowKey的第一部分如果是时间戳的话，将造成所有新数据都在一个RegionServer堆积的热点现象，也就是通常说的Region热点问题，热点发生在大量的client直接访问集中在个别RegionServer上（访问可能是读、写或者其他操作），导致单个RegionServer机器自身负载过高，引起性能下降甚至Region不可用，常见的是发生jvm full gc或者显示region too busy异常情况。



### 为写优化

#### 散列

> 如果你愿意在行健里放弃时间戳信息（每次你做什么事情都要扫描全表，或者每次要读数据时你都知道精确的键，这些情况下也是可行的），使用原始数据的散列值作为行健是一种可能的解决方案：

```hbase
hash("TheRealMan") -> random byte[]
```

每次当你需要访问以这个散列值为键的行时，需要精确知道“TheRealMT”。时间序列数据一般不这样处理。当你访问数据时，可能记住了一个时间范围，但不大可能知道精确的时间戳。但是有些情况下，能够计算散列值从而找到正确的行。为了得到一种跨所有region的、优秀的分布策略，你可以使用MD5、SHA-1或者其他提供随机分布的散列数。

#### salting

> 当你思考行健的构成时，salting是另一种技巧。让我们考虑之前的时间序列数据例子。假设你在读取时知道时间范围，但不想做全表扫描。对时间戳做散列运算然后把散列值作为行健的做法需要做全表扫描，这是很低效的，尤其是在你有办法限制扫描范围的时候。使用散列值作为行健在这里不是办法，但是你可以在时间戳前面加上一个随机数前缀。

例如：

![](https://img2018.cnblogs.com/blog/1217276/201903/1217276-20190326115003925-271541335.png)

但并非一切都是完美的。现在读操作需要把扫描命令分散到所有region上来查找相应的行。因为它们不再存储在一起，所以一个短扫描不能解决问题了。这是一种权衡，为了搭建成功的应用你需要做出选择。这是一个利用信息的位置来获得跨region分布的经典例子。

#### Reverse翻转

> 针对固定长度的RowKey反转后存储，这样可以使RowKey中经常改变的部分放在最前面，可以有效的随机RowKey。反转RowKey的例子通常以手机举例，可以将手机号反转后的字符串作为RowKey，这样就避免了以手机号那样比较固定开头导致热点问题。**这样做的缺点是牺牲了RowKey的有序性。**

### 为读优化

#### 时间戳反转

> 一个常见的数据处理问题是快速获取数据的最新版本，使用反转的时间戳作为RowKey的一部分对这个问题十分有用，可以用Long.Max_Value - timestamp追加到key的末尾。举例，在设计推帖流表时，你的焦点是为读优化行健，目的是把推帖流里最新的推帖存储在一起，以便于它们可以被快速读取，而不用做开销很大的硬盘搜索。在推贴流表里，你使用倒序时间戳（Long.MAX_VALUE - 时间戳）然后附加上用户ID来构成行健。现在你基于用户ID扫描紧邻的n行就可以找到用户需要的n条最新推帖。这里行健的结构对于读性能很重要。把用户ID放在开头有助于你设置扫描，可以轻松定义起始键。



### 小案例

## 设计订单状态表

#### 设计模式：反转+时间戳反转

RowKey：reverser(order_id) + (Long.MAX_VALUE - timestamp)

这样设计的好处一是通过reverse订单号避免Region热点，二是可以按时间倒排显示，可以获取到最新的订单。

同样适用于需要保存一个用户的操作记录，按照操作时间倒序排序。设计的rowKey为：reverser(userId) + (Long.MAX_VALUE - timestamp)。如果需要查询某段时间的操作记录，startRow是```[userId反转][Long.MAX_VALUE - 结束时间]```，stopRow是```[userId反转][Long.MAX_VALUE - 起始时间]```。

#### 登录、下单等等统称事件(event)的临时存储

HBase只存储了最近10分钟的热数据

设计模式：salt加盐

RowKey：两位随机数Salt + eventId + Date + kafka的Offset

这样设计的好处是：

设计加盐的目的是为了增加查询的并发性，假如Salt的范围是0~n，那我们在查询的时候，可以将数据分为n个split同时做scan操作。经过我们的多次测试验证，增加并发度能够将整体的查询速度提升5~20倍以上。随后的eventId和Date是用来做范围Scan来使用的。在我们的查询场景中，大部分都是指定了eventId的，因此我们在eventId放在了第二个位置上，同时呢，通过Salt + eventId的方式可以保证不会形成热点。把date放在RowKey的第三个位置上可以实现date做scan，批量Scan性能甚至可以做到毫秒级返回。

这样的RowKey设计能够很好的支持如下几个查询场景：

1. 全表scan。在这种情况下，我们仍然可以将全表数据切分成n份并发查询，从而实现查询的实时响应。
2. 只按照event_id查询。
3. 按照event_id和date查询。



## HBase索引设计

> 数据库查询可简单分解为两个步骤：1）键的查找；2) 数据的查找
>
> 因这两种数据组织方式的不同，在RDBMS领域有两种常见的数据组织表结构：
>
> **索引组织表：**键与数据存放在一起，查找到键所在的位置则意味着查找到数据本身。
>
> **堆表：**键的存储与数据的存储是分离的。查找到键的位置，只能获取到数据的物理地址，还需要基于该地址去获取数据。
>
> <font color=red>HBase数据表其实是一种**索引组织表结构**：查找到**RowKey**所在的位置则意味着找到数据本身。因此，**RowKey本身就是一种索引**。</font>

### RowKey查询的局限性/二级索引需求背景

如果提供的查询条件能够**尽可能丰富**的描述RowKey的**前缀信息**，则**查询时延**越能得到保障。如下面几种组合条件场景：

　　* Name + Phone + ID
  * Name + Phone
  * Name

如果查询条件不能提供Name信息，则RowKey的前缀条件是无法确定的，此时只能通过**全表扫描**的方式来查找结果。

一种业务模型的用户数据RowKey，只能采用单一结构设计。但事实上，查询场景可能是多纬度的。例如，在上面的场景基础上，还需要单独基于Phone列进行查询。这是HBase二级索引出现的背景。即，二级索引是为了让HBase能够提供更多纬度的查询能力。

注：HBase原生并不支持二级索引方案，但基于HBase的KeyValue数据模型与API，可以轻易的构建出二级索引数据。Phoenix提供了两种索引方案，而一些大厂家也都提供了自己的二级索引实现。

#### HBase 二级索引方案

##### 基于Coprocessor方案

从0.94版本，HBase官方文档已经提出了HBase上面实现二级索引的一种路径：

- 基于Coprocessor（0.92版本引入，达到支持类似传统RDBMS的触发器的行为）。
- 开发自定义数据处理逻辑，采用数据“双写”策略，在有数据写入同时同步到二级索引表。

###### 开源方案

业界比较知名的基于Coprocessor的开源方案：

- 华为的hindex：基于0.94版本，但版本比较旧，github上几年都没更新过。
- Apache Phoenix：功能围绕SQL On HBase，支持和兼容多个hbase版本，二级索引只是其中一块功能。二级索引的创建和管理直接有SQL语法支持，适用起来简便，该项目目前社区活跃度和版本更新迭代情况都比较好。

###### Phoenix二级索引特点

- Covered Indexes（覆盖索引）：把关注的数据字段也附在索引表上，只需要通过索引表就能返回所要查询的数据（列），所以索引的列必须包含所需查询的列（SELECT的列和WHERE的列）。
- Functional Indexes（函数索引）：索引不局限于列，支持任意的表达式来创建索引。
- Global Indexes（全局索引）：适用于读多写少场景。通过维护全局索引表，所有的更新和写操作都会引起索引的更新，写入性能受到影响。在读数据时，Phoenix SQL会基于索引字段，执行快速查询。
- Local Indexes（本地索引）：适用于写多读少场景。在数据写入时，索引数据和表数据都会存储在本地。在数据读取时，由于无法预先确定region的位置，所以在读取数据时需要检查每个region（以找到索引数据），会带来一定性能（网络）开销。

##### 非Coprocessor方案

选择不基于Coprocessor开发，自行在外部构建和维护索引关系也是另外一种方式。

常见的是采用底层基于ElasticSearch（下面简称ES）或 Solr，来构建强大的索引能力、搜索能力，例如支持模糊查询、全文检索、组合查询、排序等。

其实对于在外部自定义构建二级索引的方式，有自己的大数据团队的公司一般都会针对自己的业务场景进行优化，自行构建ES/Solr的搜索集群。例如数说故事企业内部的百亿级数据全量库，就是基于ES构建海量索引和检索能力的案例。主要有优化点包括：

- 对企业的索引集群面向的业务场景和模式定制，对通用数据模型进行抽象和平台话复用
- 需要针对多业务、多项目场景进行ES集群资源的合理划分和运维管理
- 查询需要针对多索引集群、跨集群查询进行优化
- 共用集群场景需要做好防护、监控、限流

下面显示了数说基于ES做二级索引的两种构建流程，包含：

- 增量索引：日常持续接入的数据源，进行增量的索引更新
- 全量索引：配套基于Spark/MR的批量索引创建/更新程序，用于初次或重建已有HBase库表的索引。

![](https://img2018.cnblogs.com/blog/1217276/201903/1217276-20190327135747759-1125830138.png)

数据查询流程：

![](https://img2018.cnblogs.com/blog/1217276/201903/1217276-20190327135846473-974777912.png)



#### HBase表设计关注点

HBase表设计通常可以是宽表（wide table）模式，即一行包括很多列。同样的信息也可以用高表（tall table）形式存储，通常高表的性能比宽表要高出50%以上，所以推荐大家使用高表来完成表设计。表设计时，我们也应该要考虑HBase数据库的一些特性：

1. 在HBase表中是通过RowKey的字典序来进行数据排序的。
2. 所有存储在HBase表中的数据都是二进制的字节。
3. 原子性只在行内保证，HBase不支持跨行事务。
4. 列簇（Column Family）在表创建之前就要定义好
5. 列簇中的列标识（Column Qualifier）可以在表创建完以后动态插入数据时添加。



### 内存优化

HBase 操作过程中需要大量的内存开销，毕竟 Table 是可以缓存在内存中的，一般会分配整个可用内存的 70%给 HBase 的 Java 堆。但是不建议分配非常大的堆内存，因为 GC 过程持续太久会导致 RegionServer 处于长期不可用状态，一般 16~48G 内存就可以了，如果因为框架占用内存过高导致系统内存不足，框架一样会被系统服务拖死。



### **基础优化**

#### 允许在 HDFS 的文件中追加内容

hdfs-site.xml、hbase-site.xml

> 属性：dfs.support.append
>
> 解释：开启 HDFS 追加同步，可以优秀的配合 HBase 的数据同步和持久化。默认值为 true。 

#### 优化 DataNode 允许的最大文件打开数

hdfs-site.xml

> 属性：dfs.datanode.max.transfer.threads
>
> 解释：HBase 一般都会同一时间操作大量的文件，根据集群的数量和规模以及数据动作，设置为 4096 或者更高。默认值：4096

#### 优化延迟高的数据操作的等待时间

hdfs-site.xml

> 属性：dfs.image.transfer.timeout
>
> 解释：如果对于某一次数据操作来讲，延迟非常高，socket 需要等待更长的时间，建议把该值设置为更大的值（默认 60000 毫秒），以确保 socket 不会被 timeout 掉。

#### 优化数据的写入效率

mapred-site.xml

> 属性：
>
> mapreduce.map.output.compress
>
> mapreduce.map.output.compress.codec
>
> 解释：开启这两个数据可以大大提高文件的写入效率，减少写入时间。第一个属性值修改为true，第二个属性值修改为：org.apache.hadoop.io.compress.GzipCodec 或者其他压缩方式。

#### 设置 RPC 监听数量

hbase-site.xml

> 属性：hbase.regionserver.handler.count
>
> 解释：默认值为 30，用于指定 RPC 监听的数量，可以根据客户端的请求数进行调整，读写请求较多时，增加此值。

#### 优化 HStore 文件大小

hbase-site.xml

> 属性：hbase.hregion.max.filesize
>
> 解释：默认值 10737418240（10GB），如果需要运行 HBase 的 MR 任务，可以减小此值，因为一个 region 对应一个 map 任务，如果单个 region 过大，会导致 map 任务执行时间过长。该值的意思就是，如果 HFile 的大小达到这个数值，则这个 region 会被切分为两个 Hfile。

#### 优化 hbase 客户端缓存

hbase-site.xml

> 属性：hbase.client.write.buffer
>
> 解释：用于指定 HBase 客户端缓存，增大该值可以减少 RPC 调用次数，但是会消耗更多内存，反之则反之。一般我们需要设定一定的缓存大小，以达到减少 RPC 次数的目的。

#### 指定 scan.next 扫描 HBase 所获取的行数

hbase-site.xml

> 属性：hbase.client.scanner.caching
>
> 解释：用于指定 scan.next 方法获取的默认行数，值越大，消耗内存越大。

#### flush、compact、split 机制

当 MemStore 达到阈值，将 Memstore 中的数据 Flush 进 Storefile；compact 机制则是把 flush出来的小文件合并成大的 Storefile 文件。split 则是当 Region 达到阈值，会把过大的 Region一分为二。

**涉及属性：**

即：128M 就是 Memstore 的默认阈值hbase.hregion.memstore.flush.size：134217728

即：这个参数的作用是当单个 HRegion 内所有的 Memstore 大小总和超过指定值时，flush 该HRegion 的所有 memstore。RegionServer 的 flush 是通过将请求添加一个队列，模拟生产消费模型来异步处理的。那这里就有一个问题，当队列来不及消费，产生大量积压请求时，可能会导致内存陡增，最坏的情况是触发 OOM。

hbase.regionserver.global.memstore.upperLimit：0.4

hbase.regionserver.global.memstore.lowerLimit：0.38

即：当 MemStore 使用内存总量达到 hbase.regionserver.global.memstore.upperLimit 指定值时，将会有多个 MemStores flush 到文件中，MemStore flush 顺序是按照大小降序执行的，直到刷新到 MemStore 使用内存略小于 lowerLimit













