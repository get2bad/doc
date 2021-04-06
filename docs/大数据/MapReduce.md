## MapReduce架构概述

MapReduce 将计算过程分为两个阶段

1. Map阶段并行处理输入数据
2. Reduce阶段对Map结果进行汇总

一个完整的MapReduce程序在运行时有三类实例进程

1. MrAppMaster: 负责整个程序的过程调度及状态协调
2. MapTask: 负责Map阶段整个数据处理流程
3. ReduceTask: 负责Reduce阶段的整个数据处理流程



## Job提交流程源码和切片源码详解

![](http://image.tinx.top/img20210402152601.png)

```java
waitForCompletion()

submit();

// 1建立连接
	connect();	
		// 1）创建提交Job的代理
		new Cluster(getConfiguration());
			// （1）判断是本地yarn还是远程
			initialize(jobTrackAddr, conf); 

// 2 提交job
submitter.submitJobInternal(Job.this, cluster)
	// 1）创建给集群提交数据的Stag路径
	Path jobStagingArea = JobSubmissionFiles.getStagingDir(cluster, conf);

	// 2）获取jobid ，并创建Job路径
	JobID jobId = submitClient.getNewJobID();

	// 3）拷贝jar包到集群
copyAndConfigureFiles(job, submitJobDir);	
	rUploader.uploadFiles(job, jobSubmitDir);

// 4）计算切片，生成切片规划文件
writeSplits(job, submitJobDir);
		maps = writeNewSplits(job, jobSubmitDir);
		input.getSplits(job);

// 5）向Stag路径写XML配置文件
writeConf(conf, submitJobFile);
	conf.writeXml(out);

// 6）提交Job,返回提交状态
status = submitClient.submitJob(jobId, submitJobDir.toString(), job.getCredentials());
```



## FileInputFormat切片源码解析

1. 程序先找到数据的存储目录

2. 开始遍历处理目录下的每一个文件

3. 遍历第一个文件 xxx.txt

   1. 获取文件大小 fs.sizeOf(xxxx.txt)

   2. 计算切片大小

      ```java
      // minSize 默认为 mapreduce.input.fileinputformat.split.minsize =1
      // maxSize 默认值为 mapreduce.input.fileinputformat.split.maxsize = Integer.MAX_VALUE
      // 所以默认状态下 切片大小为 blockSize (1.X 默认为64M 2.X默认为128M)
      computeSplitSize(Math.max(minSize,Math.min(maxSize,blockSize))) = blockSize = 128M
      ```

   3. 默认情况下，切片大小 = blockSize

   4. 开始切，形成第一个切片， xxx.txt -> 0:128MB 第二个切片 128:256MB 第三个一次类推(<font color=red>每次切片时，都要判断切完城下的部分是否大于块的1.1倍，不大于1.1倍的就划分一块切片</font>)

   5. 将切片信息写到一个切片规划文件中

   6. 整个切片的核心过程在getSplit()方法中完成

   7. InputSplit只记录了切片的元数据信息，比如起始位置、长度以及所在节点的列表等

4. 提交切片规划文件到yarn上，yarn上的MrAppMaster就可以根据切片规划文件计算开启的MapTask个数



## CombineTextInputFormat 切片

> 框架默认的TextInputFormat切片机制是对任务按文件规划切片，不管文件多小，都会是一个单独的切片，都会交给一个MapTask，这样如果有大量小文件，就会产生大量的MapTask，处理效率极其低下。

### 应用场景

CombineTextInputFormat用于小文件过多的场景，它可以将多个小文件从逻辑上规划到一个切片中，这样，多个小文件就可以交给一个MapTask处理。

### 虚拟存储切片最大值设置

```java
CombineTextInputFormat.setMaxInputSplitSize(job, 4194304);// 4m
```

### 切片机制小例子

![](http://image.tinx.top/img20210402155234.png)



如果我们需要将：

1. a.txt 1.7MB
2. b.txt 5.1MB
3. c.txt 3.4MB
4. d.txt 6.8MB

这些文件切片，那么切片时怎么过分的呢？

1. 因为 a.txt 小于默认的 4MB，单独切一片
2. 因为b.txt 大于默认的4MB，但是小于 2 * 4MB ，所以直接一分为二 每一片为 2.55MB
3. 因为c.txt小于默认的4MB，所以单独切一片
4. 因为d.txt大于默认的4MB，但是小于 2 * 4MB，所以直接一分为二，每一片为 3.4MB
5. 最终会合并为 (1.7 + 2.55)M + (2.55 + 3.4)M + (3.4+3.4)M的组合



## FileInputFormat实现类

### TextInputFormat

TextInputFormat是默认的FileInputFormat实现类，按行读取每条记录，键是存储该行在整个文件中的起始字节偏移量，LongWritable类型。值是这行的内容，不包括任何行终止符(换行符和回车符)

### KeyValueTextInputFormat

每一行均为一条几率，被分隔符分割为key,value。可以通过在驱动类中设置 conf.set(KeyValueRecordReader/KEY_VALUE_SEPERATOR,"\t")来设定分隔符。默认的分隔符是tab(\t)

### NLineInputFormat

如果使用NLineInputFormat，代表每个map进程处理的InputSplit不再按Block块去划分，而是按NLineInputFormat指定的行数N来划分。即输入文件的总行数/N=切片数。如果不能整除，就切片数 = 商 + 1

### 自定义InputFormat

#### 步骤

##### 自定义一个类继承FileinputFormat

```java
// 定义类继承FileInputFormat
public class WholeFileInputformat extends FileInputFormat<Text, BytesWritable>{
	
	@Override
	protected boolean isSplitable(JobContext context, Path filename) {
		return false;
	}

	@Override
	public RecordReader<Text, BytesWritable> createRecordReader(InputSplit split, TaskAttemptContext context)	throws IOException, InterruptedException {
		
		WholeRecordReader recordReader = new WholeRecordReader();
		recordReader.initialize(split, context);
		
		return recordReader;
	}
}
```

##### 改写RecordReader，实现一次读取一个完整文件封装KV

```java
public class WholeRecordReader extends RecordReader<Text, BytesWritable>{

	private Configuration configuration;
	private FileSplit split;
	
	private boolean isProgress= true;
	private BytesWritable value = new BytesWritable();
	private Text k = new Text();

	@Override
	public void initialize(InputSplit split, TaskAttemptContext context) throws IOException, InterruptedException {
		
		this.split = (FileSplit)split;
		configuration = context.getConfiguration();
	}

	@Override
	public boolean nextKeyValue() throws IOException, InterruptedException {
		
		if (isProgress) {

			// 1 定义缓存区
			byte[] contents = new byte[(int)split.getLength()];
			
			FileSystem fs = null;
			FSDataInputStream fis = null;
			
			try {
				// 2 获取文件系统
				Path path = split.getPath();
				fs = path.getFileSystem(configuration);
				
				// 3 读取数据
				fis = fs.open(path);
				
				// 4 读取文件内容
				IOUtils.readFully(fis, contents, 0, contents.length);
				
				// 5 输出文件内容
				value.set(contents, 0, contents.length);

// 6 获取文件路径及名称
String name = split.getPath().toString();

// 7 设置输出的key值
k.set(name);

			} catch (Exception e) {
				
			}finally {
				IOUtils.closeStream(fis);
			}
			
			isProgress = false;
			
			return true;
		}
		
		return false;
	}

	@Override
	public Text getCurrentKey() throws IOException, InterruptedException {
		return k;
	}

	@Override
	public BytesWritable getCurrentValue() throws IOException, InterruptedException {
		return value;
	}

	@Override
	public float getProgress() throws IOException, InterruptedException {
		return 0;
	}

	@Override
	public void close() throws IOException {
	}
}
```

##### 在输出时使用SequenceFileOutputFormat输出合并文件

```java
public class SequenceFileMapper extends Mapper<Text, BytesWritable, Text, BytesWritable>{
	
	@Override
	protected void map(Text key, BytesWritable value,			Context context)		throws IOException, InterruptedException {

		context.write(key, value);
	}
}
```

```java
public class SequenceFileReducer extends Reducer<Text, BytesWritable, Text, BytesWritable> {

	@Override
	protected void reduce(Text key, Iterable<BytesWritable> values, Context context)		throws IOException, InterruptedException {

		context.write(key, values.iterator().next());
	}
}
```

```java
public class SequenceFileDriver {

	public static void main(String[] args) throws IOException, ClassNotFoundException, InterruptedException {
		
       // 输入输出路径需要根据自己电脑上实际的输入输出路径设置
		args = new String[] { "/usr/input/inputinputformat", "/usr/output1" };

       // 1 获取job对象
		Configuration conf = new Configuration();
		Job job = Job.getInstance(conf);

       // 2 设置jar包存储位置、关联自定义的mapper和reducer
		job.setJarByClass(SequenceFileDriver.class);
		job.setMapperClass(SequenceFileMapper.class);
		job.setReducerClass(SequenceFileReducer.class);

       // 7设置输入的inputFormat
		job.setInputFormatClass(WholeFileInputformat.class);

       // 8设置输出的outputFormat
	 job.setOutputFormatClass(SequenceFileOutputFormat.class);
       
// 3 设置map输出端的kv类型
		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(BytesWritable.class);
		
       // 4 设置最终输出端的kv类型
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(BytesWritable.class);

       // 5 设置输入输出路径
		FileInputFormat.setInputPaths(job, new Path(args[0]));
		FileOutputFormat.setOutputPath(job, new Path(args[1]));

       // 6 提交job
		boolean result = job.waitForCompletion(true);
		System.exit(result ? 0 : 1);
	}
}
```



## MapReduce详细工作流程

### 图解

![](http://image.tinx.top/img20210402161658.png)

![](http://image.tinx.top/img20210402161725.png)

### 文字解析

1. 待处理文本 a.txt (200MB)
2. 客户端submit之前，获取待处理数据的信息，然后根据参数配置，形成一个任务分配的计划。
   1. a.txt 0-128MB (split 1)
   2. a.txt 129 -256MB(split 2)
3. 提交信息
   1. Job.split
   2. Wc.jar
   3. Job.xml
4. 计算出 MapTask数量
5. 然后使用FileInputTask的子类读取文件内容
6. 进行 Map运算 map(K,V)
7. 向环形缓冲区写入<K,V>数据，
8. 分区、排序
9. 溢出到文件（分区并且分区内有序）
10. Merge归并排序
11. 合并
12. 所有的MapTask任务完成已经，启动相应数量的ReduceTask，并告知ReduceTask处理数据范围(数据分区)
13. 下载到ReduceTask本地磁盘
14. 合并文件 归并排序
15. 一次读取一组
16. 分组
17. 输出 默认 TextOutputFormat



上面的流程是整个MapReduce最全工作流程，但是Shuffle过程只是从第7步开始到第16步结束，

#### 具体Shuffle过程详解

1）MapTask收集我们的map()方法输出的kv对，放到内存缓冲区中

2）从内存缓冲区不断溢出本地磁盘文件，可能会溢出多个文件

3）多个溢出文件会被合并成大的溢出文件

4）在溢出过程及合并的过程中，都要调用Partitioner进行分区和针对key进行排序

5）ReduceTask根据自己的分区号，去各个MapTask机器上取相应的结果分区数据

6）ReduceTask会取到同一个分区的来自不同MapTask的结果文件，ReduceTask会将这些文件再进行合并（归并排序）

7）合并成大文件后，Shuffle的过程也就结束了，后面进入ReduceTask的逻辑运算过程（从文件中取出一个一个的键值对Group，调用用户自定义的reduce()方法）

3．注意

Shuffle中的缓冲区大小会影响到MapReduce程序的执行效率，原则上说，缓冲区越大，磁盘io的次数越少，执行速度就越快。

缓冲区的大小可以通过参数调整，参数：io.sort.mb默认100M。

### 源码步骤

context.write(k, NullWritable.get());

output.write(key, value);

collector.collect(key, value,partitioner.getPartition(key, value, partitions));

​	HashPartitioner();

collect()

​	close()

​	collect.flush()

sortAndSpill()

​	sort()  QuickSort 快速排序

mergeParts();

![](http://image.tinx.top/img20210402165450.png)

collector.close();



## Shuffle机制

> Map方法之后，Reduce方法之前的数据处理过程称之为Shuffle。

![](http://image.tinx.top/img20210402165907.png)



### Partition分区

#### 默认Paritition分区算法

```java
(key.hashCode & Integer.MAX_VALUE) % numReduceTasks;
```

#### 自定义Parition分区

1. 自定义类继承Parititoner，重写getParitition() 方法

2. 在Job驱动中，设置自定义parititioner

   ```java
   job.setPartitionCLass(XXX.class)
   ```

3. 自定义Parition后，要根据自定义paritition的逻辑设置相应数量的ReduceTask

   ```java
   job.setNumReduceTasks(5);
   ```

#### 总结

1. 如果ReduceTask的数量 大于 getPartition的数量，则会多产生几个空的输出文件 part-r-00XX
2. 如果 1 小于 ReduceTask的数量 小于 getPartition的数量，则有一部分分区数据无处安放，会抛出Exception
3. 如果ReduceTask的数量 = 1 ，则不管有几个分区文件，都会交给这一个ReduceTask，最终会产生一个文件
4. 分区号必须从0开始，逐一累加



## WritableComparable排序

MapTask和ReduceTask均会对数据按照Key进行排序。该操作数据Hadoop的默认行为，**任何应用程序中的数据都会被排序，而不管逻辑是否需要**

<font color=red>默认排序是按照字典顺序排序，且实现排序的方法是**快速排序**</font>

***重点！！！！***

<font color=red>***进行三次排序，其中map过程为2次排序，分别为环形缓冲区达到阈值，进行一次快速排序，当数据处理完毕对磁盘上的文件进行一次归并排序。***</font>

<font color=red>**然后在reduce过程，当他从mapTask上远程拷贝文件放入内存，如果磁盘上的文件数目达到一定的阈值，就进行一次归并排序生成一个更大的文件，当所有的文件拷贝完成以后，ReduceTask统一对内存和磁盘上的所有的数据进行一次归并排序。**</font>

### 排序分类

#### 部分排序

MapReduce根据输入几率的键对数据集排序。保证输出的每个文件内部有序

#### 全排序

最终输出结果只有一个文件，且文件内部有序。实现方式只设置了一个ReduceTask。蛋该方法在处理大型文件时效率极低，因为一台机器处理所有文件没完全丧事了mapReduce所提供的并行架构

#### 辅助排序(GroupingComparator分组)

在Reduce端对key进行分组。应用于：在接收的key为bean对象时，想让一个或几个字段相同（全部字段比较不相同）的key进入到同一个reduce方法时，可以采用分组排序。

#### 二次排序

在自定义排序过程中，如果compareTo中的判断条件为两个即为二次排序。

#### 自定义排序 WritableComparable

实现 WritableComparable接口重写compareTo方法，可以实现排序

```java
@Override
public int compareTo(FlowBean o) {

	int result;
		
	// 按照总流量大小，倒序排列
	if (sumFlow > bean.getSumFlow()) {
		result = -1;
	}else if (sumFlow < bean.getSumFlow()) {
		result = 1;
	}else {
		result = 0;
	}

	return result;
}
```



## Combiner 合并

> 1. Combiner是MR程序中Mapper和Reducer之外的一种组件
> 2. Combiner组件的父类就是Reducer
> 3. Combiner和Reducer的区别在于运行的位置
>    1. Combiner 运行在 每一个MapTask所在的节点
>    2. Reducer是接收全局所有Mapper的结果
> 4. Combiner的意义就是每一个MapTask的输出进行局部汇总，以减小网络传输量
> 5. Combiner能够应用的前提是不能影响业务逻辑，而且Combiiner的输出kv应该跟Reducer的输入kv类型要对应起来

### 自定义Combiner

1. 自定义一个Combiner继承Reducer，重写Reduce方法

```java
public class WordcountCombiner extends Reducer<Text, IntWritable, Text,IntWritable>{

	@Override
	protected void reduce(Text key, Iterable<IntWritable> values,Context context) throws IOException, InterruptedException {

        // 1 汇总操作
		int count = 0;
		for(IntWritable v :values){
			count += v.get();
		}

        // 2 写出
		context.write(key, new IntWritable(count));
	}
}
```

2. 在job中设置

```java
job.setCombinerClass(WordcountCombiner.class);
```



## MapTask机制

![](http://image.tinx.top/img20210406143345.png)

1. Read阶段：MapTask通过用户编写的RecordReader,从输入InputSplit中解析出一个个key/value

2. Map阶段：该节点主要是将解析出的key/value交给用户编写map()函数处理，并产生一系列新的key/value

3. Collect阶段：在用户编写的map()函数中，当数据处理完成后，一般会调用OutputCollector.collect()输出结果。在该函数背部，他将会产生key/value分区(调用Paritioner)，并写入一个环形内存缓冲区

4. Spill阶段：即"溢写"，当环形缓冲区放满后，MapReduce将数据写到本地磁盘上，生成一个临时文件。需要注意的是，将数据写入本地磁盘以前，先要对数据进行一次本地排序，并在必要时对数据进行合并，、压缩等操作

   溢写阶段详情：

   1. 利用快速排序算法对缓存区内的数据进行排序，排序方式是，先按照分区编号Partition进行排序，然后按照key进行排序，这样，经过排序以后，数据以分区为单位聚集在一起，并且在同一分区内所有数据按照key进行排序
   2. 按照分区编号由小到大依次将每个分区中的数据写入任务工作目录下的临时文件output/spillN.out中，如果用户设置了Combiner，则写入文件以前，对每个分区中的数据进行一次聚集操作
   3. 将分区数据的原信息写到内存索引数据结构SpillRecord中，其中每个分区的原信息包括在临时文件中的偏移量，压缩前数据大小和压缩后的数据大小。如果当前内存索引大小超过1MB，则将内存索引写到文件output/spillN.out.index中。

5. Combine阶段：当所有数据处理完成以后，MapTask对所有临时文件进行一次合并，以确保最终只会生成一个数据文件

当所有数据处理完后，MapTask会将所有临时文件合并成一个大文件，并保存到文件output/file.out中，同时生成相应的索引文件output/file.out.index。

在进行文件合并过程中，MapTask以分区为单位进行合并。对于某个分区，它将采用多轮递归合并的方式。每轮合并io.sort.factor（默认10）个文件，并将产生的文件重新加入待合并列表中，对文件排序后，重复以上过程，直到最终得到一个大文件。

让每个MapTask最终只生成一个数据文件，可避免同时打开大量文件和同时读取大量小文件产生的随机读取带来的开销。



## ReduceTask工作机制

![image-20210406145936492](/Users/wangshuai/Library/Application Support/typora-user-images/image-20210406145936492.png)

1. Copy阶段： ReduceTask从各个MapTask上远程拷贝一片数据，并针对某一片数据，如果其大小超过一定阈值，则写到磁盘上，否则直接写到内存中
2. Merge阶段： 在远程拷贝数据的同时，ReduceTask启动了两个后台线程对内存和磁盘上的文件进行合并，以防止内存使用过多或磁盘上文件过多
3. Sort阶段： 按照MapReduce语义，用户编写reduce()函数输入数据是按key进行聚集的一组数据，为了将key相同的数据聚在一起，Hadoop采用了基于排序的策略，由于各个MapTask已经实现对自己的处理结果进行了局部排序，因此，ReduceTask只需对所有数据进行一次归并排序即可。
4. Reduce阶段： reduce()0函数将计算结果写到HDFS上

### 设置ReduceTask并行度（个数）

ReduceTask的并行度同样影响整个Job的执行并发度和执行效率，但与MapTask的并发数由切片数决定不同，ReduceTask数量的决定是可以直接手动设置：

~~~java
// 默认值是1，手动设置为4
job.setNumReduceTasks(4);
~~~

### ReduceTask注意事项

1. ReduceTask=0 表示没有reduce阶段，输出文件个数和Map个数一致

2. ReduceTask默认值就是1，所以输出文件个数为1个

3. 如果数据分布不均匀，就有可能在Reduce阶段发生数据倾斜

4. ReduceTask数量并不是任意设置，还要考虑业务逻辑需求，有些情况下，需要计算全局汇总结果，就只能有一个ReduceTask

5. 具体多少个ReduceTask，需要根据集群性能而定

6. 如果分区不是1，但是ReduceMTask为1，是否执行分区过程？

   > 不执行分区过程。因为MapTask的源码中，执行分区的前提是先判断ReduceNum个数是否大于1.不大于1肯定不执行。



## OutputFormat数据输出

> OutputFormat是MapReduce输出的基类，所有实现MapReduce输出都实现了OutputFormat接口

### TextOutputFormat

默认的输出格式是 TextOutputFormat，**他把每条记录写为文本行**。他的键和值可以使任意类型，因为TextOutputFormat调用toString()方法把他们转换为字符串



### SequenceFileOutputFormat

将SequenceFileOutputFormat输出**作为后续MapReduce任务的输入**，这便是一种好的输出格式，以为它的格式更加 紧凑，因容易被压缩



### 自定义 outputFormat

1. 自定义一个类继承FileOutputFormat
2. 改写RecordWriter，具体改写输出数据的方法write()



## 压缩

### 基本原则

1. 运算密集型job，少用压缩
2. IO密集型job，多用压缩

### MapReduce支持的压缩编码

| 压缩格式 | hadoop自带？ | 算法    | 文件扩展名 | 是否可切分 | 换成压缩格式后，原来的程序是否需要修改 |
| -------- | ------------ | ------- | ---------- | ---------- | -------------------------------------- |
| DEFLATE  | 是，直接使用 | DEFLATE | .deflate   | 否         | 和文本处理一样，不需要修改             |
| Gzip     | 是，直接使用 | DEFLATE | .gz        | 否         | 和文本处理一样，不需要修改             |
| bzip2    | 是，直接使用 | bzip2   | .bz2       | 是         | 和文本处理一样，不需要修改             |
| LZO      | 否，需要安装 | LZO     | .lzo       | 是         | 需要建索引，还需要指定输入格式         |
| Snappy   | 否，需要安装 | Snappy  | .snappy    | 否         | 和文本处理一样，不需要修改             |

#### 编码/解码器

| 压缩格式 | 对应的编码/解码器                          |
| -------- | ------------------------------------------ |
| DEFLATE  | org.apache.hadoop.io.compress.DefaultCodec |
| gzip     | org.apache.hadoop.io.compress.GzipCodec    |
| bzip2    | org.apache.hadoop.io.compress.BZip2Codec   |
| LZO      | com.hadoop.compression.lzo.LzopCodec       |
| Snappy   | org.apache.hadoop.io.compress.SnappyCodec  |

要在Hadoop中启用压缩，可以配置如下参数：

| 参数                                                         | 默认值                                                       | 阶段        | 建议                                          |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ----------- | --------------------------------------------- |
| io.compression.codecs  （在core-site.xml中配置）             | org.apache.hadoop.io.compress.DefaultCodec, org.apache.hadoop.io.compress.GzipCodec, org.apache.hadoop.io.compress.BZip2Codec | 输入压缩    | Hadoop使用文件扩展名判断是否支持某种编解码器  |
| mapreduce.map.output.compress（在mapred-site.xml中配置）     | false                                                        | mapper输出  | 这个参数设为true启用压缩                      |
| mapreduce.map.output.compress.codec（在mapred-site.xml中配置） | org.apache.hadoop.io.compress.DefaultCodec                   | mapper输出  | 企业多使用LZO或Snappy编解码器在此阶段压缩数据 |
| mapreduce.output.fileoutputformat.compress（在mapred-site.xml中配置） | false                                                        | reducer输出 | 这个参数设为true启用压缩                      |
| mapreduce.output.fileoutputformat.compress.codec（在mapred-site.xml中配置） | org.apache.hadoop.io.compress. DefaultCodec                  | reducer输出 | 使用标准工具或者编解码器，如gzip和bzip2       |
| mapreduce.output.fileoutputformat.compress.type（在mapred-site.xml中配置） | RECORD                                                       | reducer输出 | SequenceFile输出使用的压缩类型：NONE和BLOCK   |

这边就速度来说建议使用 LZO  Snappy  压缩/解压



#### GZIP

##### 优点：

1. 压缩率高，而且压缩/解压速度比较快。Hadoop本身支持，在应用中处理GZip格式的文件就和直接处理文本一样，大部分linux系统都自带GZip命令，方便

##### 缺点：

不支持split切片

##### 应用场景：

每个文件压缩后在130M以内的，比如说每天的日志文件都压缩成一个GZIP文件



#### BZip2

##### 优点：

1. 支持Split切片，具有很高的压缩率，比GZip压缩率都高，Hadoop本身自带，使用方便

##### 缺点：

压缩速度很慢

##### 应用场景：

对速度要求不高，带需要较高的压缩率的时候。减少空间占用，又要支持切片Split的时候可以考虑使用BZip2



#### LZO

##### 优点：

1. 压缩/解压速度比较快
2. 支持Split

##### 缺点：

压缩率比GZip要低一些，Hadoop本身不支持，需要安装，在应用中对LZO格式的文件需要做一些特殊处理(为了支持Split需要建索引，还需要指定InputFormat为LZO格式)

##### 应用场景：

一个很大的文本文件，压缩后还大于200MB以上的可以考虑，而且单个文件越大，LZO优点越明显



#### Snappy

##### 优点：

1. 高速压缩速度和合理的压缩率

##### 缺点：

不支持Split，压缩率比GZIP低，Hadoop本身不支持，需要安装

##### 应用场景：

当MapReduce作业的Map输出的数据比较大的时候，作为Map到Reduce的中间数据的压缩格式，或者作为一个MapReduce作业的输出和另外一个MapReduce作业的输入



### 压缩位置选择

![](http://image.tinx.top/img20210406161017.png)



## 相关优化

### 数据输入

1. 合并小文件：在执行MR任务前将小文件进行合并，大量的小文件会产生大量的Map任务，增大Map任务装载次数，而任务的状态比较耗时，从而导致MR运行较慢
2. 采用CombineTextInputFormat来作为输入，解决输入端大量小的文件场景

### Map阶段

1. 减少溢写(Spill）次数：通过调整io.sort.mb及sort.spill.percent参数值，增大触发Spill的内存上线，减少Spill次数，从而减少磁盘IO
2. 减少合并(Merge)次数：通过调整io.sort.factor参数，增大Merge的文件数目，减少Merge次数，从而缩短MR处理时间。
3. 在Map之后，不影响业务逻辑的前提下，先进行Combine处理，减少IO

### Reduce阶段

1. 合理设置Map和Reduce数：两个都不能设置太小，也不能设置太多。太少，会导致Task等待，延长处理时间；太多，会导致Map、Reduce任务键竞争资源，造成处理超时等错误

2. 设置Map、Reduce共存：调整showstart.completedmaps参数，使Map运行到一定程度后，Reduce也开始运行，减少Reduce的等待时间

3. 规避使用Reduce: 因为Reduce在用于连接数据集的时候会产生大量的网络消耗

4. 合理的设置Reduce端的buffer: 默认情况下，数据达到一个阈值的时候，Buffer中的数据就会写入磁盘，然后Reduce会从磁盘中获得所有的数据。也就是说Buffer和Reduce是没有直接关联的，中间多次写磁盘->读磁盘的过程，既然有这个弊端，那么就可以通过参数来配置，使得Buffer中的一部分数据可以直接输送到Reduce，从而减少IO开销：

   mapreduce.reduce.input.buffer.percent，默认为0.0当值大于0的时候，会保留指定比例的内存读Buffer中的数据直接拿给Reduce使用，这样一来，设置Buffer需要内存，读取数据需要内存，Reduce计算也要内存，所以要根据作业的运行情况进行调整。

5. IO传输

   1. 采用数据压缩的方式，减少网络IO的时间，安装Snappy和LZO压缩编码器
   2. 使用SequenceFile二进制文件

6. 数据倾斜

   1. 数据倾斜现象

      1. 数据频率倾斜 - 某一个区域的数据量要远远大于其他区域
      2. 数据大小倾斜 - 部分记录的大小远远大于平均值

   2. 减少数据倾斜的方法

      1. 抽样和范围分区

         可以通过对原始数据进行抽样得到的结果集来预设分区边界值

      2. 自定义分区

         基于输出键的背景只是进行自定义分区。

      3. Combine

         精简数据，提前"reduce"

      4. 采用Map join ，尽量避免 reduce join



## 常用的调优参数

| 配置参数                                      | 参数说明                                                     |
| --------------------------------------------- | ------------------------------------------------------------ |
| mapreduce.map.memory.mb                       | 一个MapTask可使用的资源上限（单位:MB），默认为1024。如果MapTask实际使用的资源量超过该值，则会被强制杀死。 |
| mapreduce.reduce.memory.mb                    | 一个ReduceTask可使用的资源上限（单位:MB），默认为1024。如果ReduceTask实际使用的资源量超过该值，则会被强制杀死。 |
| mapreduce.map.cpu.vcores                      | 每个MapTask可使用的最多cpu core数目，默认值: 1               |
| mapreduce.reduce.cpu.vcores                   | 每个ReduceTask可使用的最多cpu core数目，默认值: 1            |
| mapreduce.reduce.shuffle.parallelcopies       | 每个Reduce去Map中取数据的并行数。默认值是5                   |
| mapreduce.reduce.shuffle.merge.percent        | Buffer中的数据达到多少比例开始写入磁盘。默认值0.66           |
| mapreduce.reduce.shuffle.input.buffer.percent | Buffer大小占Reduce可用内存的比例。默认值0.7                  |
| mapreduce.reduce.input.buffer.percent         | 指定多少比例的内存用来存放Buffer中的数据，默认值是0.0        |



Shuffle性能优化的关键参数，应在YARN启动之前就配置好（mapred-default.xml）

| 配置参数                         | 参数说明                          |
| -------------------------------- | --------------------------------- |
| mapreduce.task.io.sort.mb        | Shuffle的环形缓冲区大小，默认100m |
| mapreduce.map.sort.spill.percent | 环形缓冲区溢出的阈值，默认80%     |



容错相关参数(MapReduce性能优化)

| 配置参数                     | 参数说明                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| mapreduce.map.maxattempts    | 每个Map Task最大重试次数，一旦重试参数超过该值，则认为Map Task运行失败，默认值：4。 |
| mapreduce.reduce.maxattempts | 每个Reduce Task最大重试次数，一旦重试参数超过该值，则认为Map Task运行失败，默认值：4。 |
| mapreduce.task.timeout       | Task超时时间，经常需要设置的一个参数，该参数表达的意思为：如果一个Task在一定时间内没有任何进入，即不会读取新的数据，也没有输出数据，则认为该Task处于Block状态，可能是卡住了，也许永远会卡住，为了防止因为用户程序永远Block住不退出，则强制设置了一个该超时时间（单位毫秒），默认是600000。如果你的程序对每条输入数据的处理时间过长（比如会访问数据库，通过网络拉取数据等），建议将该参数调大，该参数过小常出现的错误提示是“AttemptID:attempt_14267829456721_123456_m_000224_0 Timed out after 300 secsContainer killed by the ApplicationMaster.”。 |









