# Hive

## 数据存储位置

>  Hive 是建立在 Hadoop 之上的，**所有 Hive 的数据都是存储在 HDFS 中的**。而数据库则可以将数据保存在块设备或者本地文件系统中。

<font color=red>数据仓库的内容是读多写少的。因此，Hive中不建议对数据的改写，所有的数据都是在加载的时候确定好的</font>

## 索引

Hive在加载数据的过程中不会对数据进行任何处理，甚至不会对数据进行扫描，因此也没有对数据中的某些Key建立索引。<font color=red>Hive要访问数据中满足条件的特定值时，需要暴力扫描整个数据，因此访问延迟较高。</font>由于 MapReduce 的引入， Hive 可以并行访问数据，因此即使没有索引，对于[大数据](http://lib.csdn.net/base/hadoop)量的访问，Hive 仍然可以体现出优势。数据库中，通常会针对一个或者几个列建立索引，因此对于少量的特定条件的数据的访问，数据库可以有很高的效率，较低的延迟。由于数据的访问延迟较高，**决定了 Hive 不适合在线数据查询。**



## Hive集合类型

### 基本数据类型

| Hive数据类型 | Java数据类型 | 长度                                                 | 例子                                 |
| ------------ | ------------ | ---------------------------------------------------- | ------------------------------------ |
| TINYINT      | byte         | 1byte有符号整数                                      | 20                                   |
| SMALINT      | short        | 2byte有符号整数                                      | 20                                   |
| INT          | int          | 4byte有符号整数                                      | 20                                   |
| BIGINT       | long         | 8byte有符号整数                                      | 20                                   |
| BOOLEAN      | boolean      | 布尔类型，true或者false                              | TRUE  FALSE                          |
| FLOAT        | float        | 单精度浮点数                                         | 3.14159                              |
| DOUBLE       | double       | 双精度浮点数                                         | 3.14159                              |
| STRING       | string       | 字符系列。可以指定字符集。可以使用单引号或者双引号。 | ‘now is the time’ “for all good men” |
| TIMESTAMP    |              | 时间类型                                             |                                      |
| BINARY       |              | 字节数组                                             |                                      |



### 集合数据类型

| 数据类型 | 描述                                                         | 语法示例 |
| -------- | ------------------------------------------------------------ | -------- |
| STRUCT   | 和c语言中的struct类似，都可以通过“点”符号访问元素内容。例如，如果某个列的数据类型是STRUCT{first STRING, last STRING},那么第1个元素可以通过字段.first来引用。 | struct() |
| MAP      | MAP是一组键-值对元组集合，使用数组表示法可以访问数据。例如，如果某个列的数据类型是MAP，其中键->值对是’first’->’John’和’last’->’Doe’，那么可以通过字段名[‘last’]获取最后一个元素 | map()    |
| ARRAY    | 数组是一组具有相同类型和名称的变量的集合。这些变量称为数组的元素，每个数组元素都有一个编号，编号从零开始。例如，数组值为[‘John’, ‘Doe’]，那么第2个元素可以通过数组名[1]进行引用。 | Array()  |







## DDL语句

### 创建数据库

```sql
create database if not exists test;
```

### 修改数据库的基本信息

```sql
alter database db_hive set dbproperties('createtime'='20170830');
```

### 创建表

```sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name 
[(col_name data_type [COMMENT col_comment], ...)] 
[COMMENT table_comment] 
[PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)] 
[CLUSTERED BY (col_name, col_name, ...) 
[SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS] 
[ROW FORMAT row_format] 
[STORED AS file_format] 
[LOCATION hdfs_path]
```

（1）CREATE TABLE 创建一个指定名字的表。如果相同名字的表已经存在，则抛出异常；用户可以用 IF NOT EXISTS 选项来忽略这个异常。

（2）<font color=red>EXTERNAL关键字可以让用户创建一个外部表，在建表的同时指定一个指向实际数据的路径（LOCATION），Hive创建内部表时，会将数据移动到数据仓库指向的路径；若创建外部表，仅记录数据所在的路径，不对数据的位置做任何改变。在删除表的时候，内部表的元数据和数据会被一起删除，而外部表只删除元数据，不删除数据。</font>

（3）COMMENT：为表和列添加注释。

（4）PARTITIONED BY创建分区表

（5）CLUSTERED BY创建分桶表

（6）SORTED BY不常用

（7）ROW FORMAT 

DELIMITED [FIELDS TERMINATED BY char] [COLLECTION ITEMS TERMINATED BY char]

​    [MAP KEYS TERMINATED BY char] [LINES TERMINATED BY char] 

  | SERDE serde_name [WITH SERDEPROPERTIES (property_name=property_value, property_name=property_value, ...)]

用户在建表的时候可以自定义SerDe或者使用自带的SerDe。如果没有指定ROW FORMAT 或者ROW FORMAT DELIMITED，将会使用自带的SerDe。在建表的时候，用户还需要为表指定列，用户在指定表的列的同时也会指定自定义的SerDe，Hive通过SerDe确定表的具体的列的数据。

SerDe是Serialize/Deserilize的简称，目的是用于序列化和反序列化。

（8）STORED AS指定存储文件类型

常用的存储文件类型：SEQUENCEFILE（二进制序列文件）、TEXTFILE（文本）、RCFILE（列式存储格式文件）

如果文件数据是纯文本，可以使用STORED AS TEXTFILE。如果数据需要压缩，使用 STORED AS SEQUENCEFILE。

（9）LOCATION ：指定表在HDFS上的存储位置。

（10）LIKE允许用户复制现有的表结构，但是不复制数据。

示例：

```sql
create EXTERNAL table if not exists test(
	id int comment '主键ID',
  name string comment '姓名',
  age int comment '年龄',
  birth TIMESTAMP comment '出生年月'
)
partition by time
row format delimited fields terminated by '\t'
location '/user/hive/warehource';
```

### 向外部表导入数据

```sql
load data local inpath '/opt/local/module/data/dep.txt' into table default.dep;
```

### 内部表转外部表

```sql
alter table student2 set tblproperties('EXTERNAL'='TRUE');
```

### 外部表转内部表

```sql
alter table student2 set tblproperties('EXTERNAL'='FALSE');
```

<font color=red>注意：('EXTERNAL'='TRUE')和('EXTERNAL'='FALSE')为固定写法，区分大小写！</font>

### 分区表

#### 创建分区表

```sql
hive (default)> create table dept_partition(
deptno int, dname string, loc string
)
partitioned by (month string)
row format delimited fields terminated by '\t';
```

#### 加载数据到分区表中

```sql
load data local inpath '/opt/module/datas/dept.txt' into table default.dept_partition partition(month='201709');
```

### 把数据直接上传到分区目录上，让分区表和数据产生关联的三种方式

#### 方式一：上传数据后修复

##### 上传数据

```sql
dfs -mkdir -p /user/hive/warehouse/dept_partition2/month=201709/day=12;

dfs -put /opt/module/datas/dept.txt  /user/hive/warehouse/dept_partition2/month=201709/day=12;
```

##### 执行修复命令

```sql
msck repair table dept_partition2;
```

#### 方式二：上传数据后添加分区

```shell
dfs -mkdir -p /user/hive/warehouse/dept_partition2/month=201709/day=11;

dfs -put /opt/module/datas/dept.txt  /user/hive/warehouse/dept_partition2/month=201709/day=11;
```

##### 执行添加分区

```sql
alter table dept_partition2 add partition(month='201709', day='11');
```

#### 方式三：创建文件夹后load数据到分区

##### 创建目录

```shell
dfs -mkdir -p /user/hive/warehouse/dept_partition2/month=201709/day=10;
```

##### 上传数据

```sql
load data local inpath '/opt/module/datas/dept.txt' into table dept_partition2 partition(month='201709',day='10');
```



## DML语句

### 加载数据

```sql
load data [local] inpath '/opt/module/datas/student.txt' overwrite | into table student [partition (partcol1=val1,…)];
```

（1）load data:表示加载数据

（2）local:表示从本地加载数据到hive表；否则从HDFS加载数据到hive表

（3）inpath:表示加载数据的路径

（4）overwrite:表示覆盖表中已有数据，否则表示追加

（5）into table:表示加载到哪张表

（6）student:表示具体的表

（7）partition:表示上传到指定分区

### 插入数据

```sql
insert into table  student partition(month='201709') values(1,'wangwu');
```

### 导出与导入

#### 导入

```sql
import table student2 partition(month='201709') from '/user/hive/warehouse/export/student';
```

#### 导出

```sql
insert overwrite local directory '/opt/module/datas/export/student'
```

### 函数

#### 查看所有函数

~~~sql
show functions;
~~~

#### 查看函数的用法

~~~sql
desc funtion upper;
~~~

#### 自定义函数

##### UDF

一进一出

##### UDAF

多进一出

##### UDTF

一进多出



##### 步骤

1. 继承org.apache.hadoop.hive.ql.UDF

2. 需要实现evaluate函数；evaluate函数支持重载；

3. 在hive的命令行窗口创建函数

   1. 添加jar

   ```sql
   add jar linux_jar_path
   ```

   2. 创建function

   ```sql
   create [temporary] function [dbname.]function_name AS class_name;
   ```

4. 在hive的命令行窗口删除函数

```sql
Drop [temporary] function [if exists] [dbname.]function_name;
```

5. 注意： UDF必须要有返回类型，可以返回null，但是返回类型不能为void；



## 开启压缩

### 在Map阶段开启压缩

#### 开启hive中间传输数据压缩功能

```sql
set hive.exec.compress.intermediate=true;
```

#### 开启mapreduce中map输出压缩功能

```sql
set mapreduce.map.output.compress=true;
```

#### 设置mapreduce中map输出数据的压缩方式

```sql
set mapreduce.map.output.compress.codec= org.apache.hadoop.io.compress.SnappyCodec;
```

#### 执行查询语句

```sql
select count(ename) name from emp;
```



### Reduce输出阶段压缩

#### 开启hive最终输出数据压缩功能

```sql
set hive.exec.compress.output=true;
```

#### 开启mapreduce最终输出数据压缩

```sql
set mapreduce.output.fileoutputformat.compress=true;
```

#### 设置mapreduce最终数据输出压缩方式

```sql
set mapreduce.output.fileoutputformat.compress.codec = org.apache.hadoop.io.compress.SnappyCodec;
```

#### 设置mapreduce最终数据输出压缩为块压缩

```sql
set mapreduce.output.fileoutputformat.compress.type=BLOCK;
```

#### 测试一下输出结果是否是压缩文件

```sql
insert overwrite local directory '/opt/module/datas/distribute-result' select * from emp distribute by deptno sort by empno desc;
```



## 文件存储格式

> Hive支持的存储数的格式主要有：TEXTFILE 、SEQUENCEFILE、ORC、PARQUET

### 列式存储和行式存储

#### 行存储的特点

查询满足条件的一整行数据的时候，列存储则需要去每个聚集的字段找到对应的每个列的值，行存储只需要找到其中一个值，其余的值都在相邻地方，所以此时行存储查询的速度更快。

#### 列存储的特点

因为每个字段的数据聚集存储，在查询只需要少数几个字段的时候，能大大减少读取的数据量；每个字段的数据类型一定是相同的，列式存储可以针对性的设计更好的设计压缩算法。

<font color=red>TEXTFILE和SEQUENCEFILE的存储格式都是基于行存储的；</font>

<font color=red>ORC和PARQUET是基于列式存储的。</font>

### TextFile格式

默认格式，数据不做压缩，磁盘开销大，数据解析开销大。可结合Gzip、Bzip2使用，但使用Gzip这种方式，hive不会对数据进行切分，从而无法对数据进行并行操作。

### Orc格式

![](http://image.tinx.top/img20210407153048.png)



1. Index Data：一个轻量级的index，默认是每隔1W行做一个索引。这里做的索引应该只是记录某行的各字段在Row Data中的offset。

2. Row Data：存的是具体的数据，先取部分行，然后对这些行按列进行存储。对每个列进行了编码，分成多个Stream来存储。

3. Stripe Footer：存的是各个Stream的类型，长度等信息。

每个文件有一个File Footer，这里面存的是每个Stripe的行数，每个Column的数据类型信息等；每个文件的尾部是一个PostScript，这里面记录了整个文件的压缩类型以及FileFooter的长度信息等。在读取文件时，会seek到文件尾部读PostScript，从里面解析到File Footer长度，再读FileFooter，从里面解析到各个Stripe信息，再读各个Stripe，即从后往前读。

### Parquet格式

<font color=red>Parquet文件是以二进制方式存储的，所以是不可以直接读取的，文件中包括该文件的数据和元数据，因此Parquet格式文件是自解析的。</font>

<font color=red>通常情况下，在存储Parquet数据的时候会按照Block大小设置行组的大小，由于一般情况下每一个Mapper任务处理数据的最小单位是一个Block，这样可以把每一个行组由一个Mapper任务处理，增大任务执行并行度。</font>

![](http://image.tinx.top/img20210407153714.png)

在Parquet中，有三种类型的页：数据页、字典页和索引页。数据页用于存储当前行组中该列的值，字典页存储该列值的编码字典，每一个列块中最多包含一个字典页，索引页用来存储当前行组下该列的索引，目前Parquet中还不支持索引页。

## Hive 外部表 和 内部表的区别

### 外部表

> 因为表是外部表，所以Hive并非认为其完全拥有这份数据。删除该表并不会删除掉这份数据，不过描述表的元数据信息会被删除掉。

### 内部表

> 默认创建的表都是内部表。Hive会（或多或少地）控制着数据的生命周期。Hive默认情况下会将这些表的数据存储在由配置项hive.metastore.warehouse.dir(例如，/user/hive/warehouse)所定义的目录的子目录下。	
>
> 当我们删除一个管理表时，Hive也会删除这个表中数据。管理表不适合和其他工具共享数据。



## 调优

### fetch 抓取

Hive中对某些查询可以不必使用MapReduce,而是直接读取文件即可

所以我们可以：

在hive-default.xml.template文件中hive.fetch.task.conversion默认是more，老版本hive默认是minimal，该属性修改为more以后，在全局查找、字段查找、limit查找等都不走mapreduce。



### 开启本地模式

> 大多数的Hadoop Job是需要Hadoop提供的完整的可扩展性来处理大数据集的。不过，有时Hive的输入数据量是非常小的。在这种情况下，为查询触发执行任务消耗的时间可能会比实际job的执行时间要多的多。对于大多数这种情况，Hive可以通过本地模式在单台机器上处理所有的任务。对于小数据集，执行时间可以明显被缩短。

```shell
set hive.exec.mode.local.auto=true;  //开启本地mr
//设置local mr的最大输入数据量，当输入数据量小于这个值时采用local  mr的方式，默认为134217728，即128M
set hive.exec.mode.local.auto.inputbytes.max=50000000;
//设置local mr的最大输入文件个数，当输入文件个数小于这个值时采用local mr的方式，默认为4
set hive.exec.mode.local.auto.input.files.max=10;
```



### 表优化

#### 大表 join 小表

<font color=red>尽量大表 join 小表，如果小表join大表可能会触发 oom 错误，或者查询速度特别慢</font>

#### 大表 Join 大表

##### 空key 过滤

##### 空key转换

#### 设置MapJoin

如果不指定MapJoin或者不符合MapJoin的条件，那么Hive解析器会将Join操作转换成Common Join，即：在Reduce阶段完成join。容易发生数据倾斜。可以用MapJoin把小表全部加载到内存在map端进行join，避免reducer处理。

1．开启MapJoin参数设置

（1）设置自动选择Mapjoin

```sql
set hive.auto.convert.join = true; 默认为true
```

（2）大表小表的阈值设置（默认25M一下认为是小表）：

```sql
set hive.mapjoin.smalltable.filesize=25000000;
```

#### GroupBy

默认情况下，Map阶段同一Key数据分发给一个reduce，当一个key数据过大时就倾斜了。

  并不是所有的聚合操作都需要在Reduce端完成，很多聚合操作都可以先在Map端进行部分聚合，最后在Reduce端得出最终结果。

1．开启Map端聚合参数设置

​	（1）是否在Map端进行聚合，默认为True

​		hive.map.aggr = true

（2）在Map端进行聚合操作的条目数目

​		hive.groupby.mapaggr.checkinterval = 100000

（3）有数据倾斜的时候进行负载均衡（默认是false）

​		hive.groupby.skewindata = true

  当选项设定为 true，生成的查询计划会有两个MR Job。第一个MR Job中，Map的输出结果会随机分布到Reduce中，每个Reduce做部分聚合操作，并输出结果，这样处理的结果是相同的Group By Key有可能被分发到不同的Reduce中，从而达到负载均衡的目的；第二个MR Job再根据预处理的数据结果按照Group By Key分布到Reduce中（这个过程可以保证相同的Group By Key被分布到同一个Reduce中），最后完成最终的聚合操作。



#### Count(Distinct) 去重统计

数据量小的时候无所谓，数据量大的情况下，由于COUNT DISTINCT操作需要用一个Reduce Task来完成，这一个Reduce需要处理的数据量太大，就会导致整个Job很难完成，一般COUNT DISTINCT使用先GROUP BY再COUNT的方式替换：

##### 采用GROUP by去重id

```sql
select count(id) from (select id from bigtable group by id) a;
```



### 数据倾斜

#### 合理设置Map数

#### 小文件进行合并

#### 复杂文件增加Map数

> 当input的文件都很大，任务逻辑复杂，map执行非常慢的时候，可以考虑增加Map数，来使得每个map处理的数据量减少，从而提高任务的执行效率。
>
> 增加map的方法为：根据computeSliteSize(Math.max(minSize,Math.min(maxSize,blocksize)))=blocksize=128M公式，调整maxSize最大值。让maxSize最大值低于blocksize就可以增加map的个数。

#### 合理设置Reduce数

##### 方法1

（1）每个Reduce处理的数据量默认是256MB

​		hive.exec.reducers.bytes.per.reducer=256000000

（2）每个任务最大的reduce数，默认为1009

​		hive.exec.reducers.max=1009

（3）计算reducer数的公式

​		N=min(参数2，总输入数据量/参数1)

##### 方法2

在hadoop的mapred-default.xml文件中修改

设置每个job的Reduce个数

```sql
set mapreduce.job.reduces = 15;
```



#### reduce个数并不是越多越好

1）过多的启动和初始化reduce也会消耗时间和资源；

2）另外，有多少个reduce，就会有多少个输出文件，如果生成了很多个小文件，那么如果这些小文件作为下一个任务的输入，则也会出现小文件过多的问题；

在设置reduce个数的时候也需要考虑这两个原则：处理大数据量利用合适的reduce数；使单个reduce任务处理数据量大小要合适；



#### 并行执行

Hive会将一个查询转化成一个或者多个阶段。这样的阶段可以是MapReduce阶段、抽样阶段、合并阶段、limit阶段。或者Hive执行过程中可能需要的其他阶段。默认情况下，Hive一次只会执行一个阶段。不过，某个特定的job可能包含众多的阶段，而这些阶段可能并非完全互相依赖的，也就是说有些阶段是可以并行执行的，这样可能使得整个job的执行时间缩短。不过，如果有更多的阶段可以并行执行，那么job可能就越快完成。

​	通过设置参数hive.exec.parallel值为true，就可以开启并发执行。不过，在共享集群中，需要注意下，如果job中并行阶段增多，那么集群利用率就会增加。

set hive.exec.parallel=true;        //打开任务并行执行

set hive.exec.parallel.thread.number=16;  //同一个sql允许最大并行度，默认为8。

当然，得是在系统资源比较空闲的时候才有优势，否则，没资源，并行也起不来。



#### 严格模式

Hive提供了一个严格模式，可以防止用户执行那些可能意想不到的不好的影响的查询。

​	通过设置属性hive.mapred.mode值为默认是非严格模式nonstrict 。开启严格模式需要修改hive.mapred.mode值为strict，开启严格模式可以禁止3种类型的查询

```xml
<property>
    <name>hive.mapred.mode</name>
    <value>strict</value>
</property>
```

1) 对于分区表，除非where语句中含有分区字段过滤条件来限制范围，否则不允许执行。换句话说，就是用户不允许扫描所有分区。进行这个限制的原因是，通常分区表都拥有非常大的数据集，而且数据增加迅速。没有进行分区限制的查询可能会消耗令人不可接受的巨大资源来处理这个表。

2) 对于使用了order by语句的查询，要求必须使用limit语句。因为order by为了执行排序过程会将所有的结果数据分发到同一个Reducer中进行处理，强制要求用户增加这个LIMIT语句可以防止Reducer额外执行很长一段时间。

3) 限制笛卡尔积的查询。对关系型数据库非常了解的用户可能期望在执行JOIN查询的时候不使用ON语句而是使用where语句，这样关系数据库的执行优化器就可以高效地将WHERE语句转化成那个ON语句。不幸的是，Hive并不会执行这种优化，因此，如果表足够大，那么这个查询就会出现不可控的情况。



### 开启JVM重用

JVM重用是Hadoop调优参数的内容，其对Hive的性能具有非常大的影响，特别是对于很难避免小文件的场景或task特别多的场景，这类场景大多数执行时间都很短。

JVM重用可以使得JVM实例在同一个job中重新使用N次

```xml
<property>
  <name>mapreduce.job.jvm.numtasks</name>
  <value>10</value>
</property>
```



