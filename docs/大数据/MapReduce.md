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
4. 因为d.txt大于默认的4MB，但是小于 2 * 4MB，所以直接一分为二，每一篇为 3.4MB



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
		args = new String[] { "e:/input/inputinputformat", "e:/output1" };

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

