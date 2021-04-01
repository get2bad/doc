## HDFS优缺点

### 优点

#### 高容错性

1. 数据自动保存多个副本。它通过增加副本的形式，提高容错性。
2. 某一个副本丢失以后，可以自动恢复

#### 适合处理大数据

1. 数据规模：能够处理数据规模达到GB TB PB
2. 文件规模： 能够处理百万规模以上的文件数量，数量相当大

#### 可以构建在廉价机器上，通过多副本机制，提高可靠性

### 缺点

#### 不适合低延时数据访问，比如毫秒级的存储数据，是做不到的

#### 无法高效的对大量小文件进行存储

1. 存储大量小文件的话，他会占用大量NameNode的内存来存储文件目录和块信息。这是不对的，因为NameNode的内存是有限的
2. 小文件存储的寻址时间会超过读取时间，**违反了HDFS的设计目标**

#### 不支持并发写入、文件随机修改

1. 一个文件只有一个写，不允许多个线程同时写
2. 仅仅支持文件数据的Append（追加），不支持文件的随机修改



## HDFS架构概述

![](http://image.tinx.top/img20210401134226.png)

### NameNode(NN)

存储文件的元数据，如：文件名、文件目录结构、文件属性(生成时间、副本数、文件权限)，以及每个文件的块列表和块所在的DataNode等

### DataNode(DN)

在本地文件系统存储文件块数据，以及块数据的校验和

### Secondary NameNode(2NN)

> 并非NameNode热备份，当NameNode挂掉的时候，它并不能马上替换NameNode并提供服务

1. 辅助NameNode 分担工作量，比如定期合并 Fsimage和Edits，并推送给NameNode
2. 在紧急情况下，可以辅助恢复NameNode

### Client

1. 文件切分 文件上传到HDFS的时候，Clinet将文件切分成一个个的Block，然后进行上传
2. 与NameNode交互，获取文件的位置信息
3. 与DataNode交互，读取或者写入数据
4. Client提供一些命令来管理HDFS，比如NameNode格式化
5. Client可以通过一些命令来访问HDFS，比如对HDFS进行增删改查操作



## HDFS文件块大小

> HDFS中的文件在物理上是分块存储，块的大小可以通过配置参数(dfs.blockSize)来规定，默认在Hadoop2.X上是128M，1.X是64M

<font color=green>如果寻址时间约为 10ms，即查找到目标block的时间为 10ms，寻址时间为传输时间的1%是，则为最佳状态，因此，传输时间 = 10ms / 0.01 = 1000ms = 1s ,假设当前磁盘的传输速率为120M/S，那么block大小为 1s * 120M/S = 120M</font>

### 小问题：为什么块的大小不能设置太小，也不能太大？

1. HDFS的块设置太小，会增加寻址时间，程序一直在找块开始的位置
2. 如果块设置太大，从磁盘传输数据的时间会明显大于定位这个块开始位置所需的时间。导致程序处理这块数据会特别慢

**总结：HDFS块的大小设置主要取决于磁盘传输速率**



## HDFS 写文件流程

### Java源码角度解析写流程

1. 调用客户端的对象DistributedFileSystem的create方法；



2. DistributedFileSystem会发起对namenode的一个RPC连接，请求创建一个文件，不包含关于block块的请求。

   a. namenode会执行各种各样的检查，确保要创建的文件不存在，并且客户端有创建文件的权限。

   ​	i. 如果检查通过，namenode会创建一个文件（在edits中）

   ​	ii. 否则创建失败，客户端抛异常IOException。

   

3. DistributedFileSystem返回一个FSDataOutputStream对象给客户端用于写数据。FSDataOutputStream封装了一个DFSOutputStream对象负责客户端跟datanode以及namenode的通信。



4. FSDataOutputStream对象将数据切分为小的数据包（64kb），并写入到一个内部队列（“数据队列”）。DataStreamer会读取其中内容，并请求namenode返回一个datanode列表来存储当前block副本。列表中的datanode会形成管线，DataStreamer将数据包发送给管线中的第一个datanode，第一个datanode将接收到的数据发送给第二个datanode，第二个发送给第三个。。。



5. DFSOoutputStream维护着一个数据包的队列，这的数据包是需要写入到datanode中的，该队列称为确认队列。当一个数据包在管线中所有datanode中写入完成，就从ack队列中移除该数据包。队列是在客户端维护的。



6. 如果在数据写入期间datanode发生故障，则执行以下操作

   a. 关闭管线，把确认队列中的所有包都添加回数据队列的最前端，以保证故障节点下游的datanode不会漏掉任何一个数据包。

   

   b、为存储在另一正常datanode的当前数据块指定一个新的标志，并将该标志传送给namenode，以便故障datanode在恢复后可以删除存储的部分数据块。

   

   c、如果在数据写入期间datanode发生故障，待确认消息队列迟迟得不到确认消息，这时会有一个超时时间，超过这个时间，从管线中删除故障数据节点并且把余下的数据块写入管线中另外两个正常的datanode（也就是这两个节点组成新的管线并且blockID的值要发生变化，另外注意正常的节点中包括之前上传的部分小的64K文件，所以需要对其就行一个统计，确认我现在数到第几个包了，避免重复提交）。namenode在检测到副本数量不足时，会在另一个节点上创建新的副本。

   

   d、后续的数据块继续正常接受处理。

   

7. 在一个块被写入期间可能会有多个datanode同时发生故障，但非常少见。只要设置了dfs.replication.min的副本数（默认为1），写操作就会成功，并且这个块可以在集群中异步复制，直到达到其目标副本数（dfs.replication默认值为3）。



8. 如果有多个block，则会反复从步骤4开始执行。



9. 当客户端完成了数据的传输，调用数据流的close方法。该方法将数据队列中的剩余数据包写到datanode的管线并等待管线的确认



10. 客户端收到管线中所有正常datanode的确认消息后，通知namenode文件写完了。



11. 客户端完成数据的写入后，对数据流调用close方法。该操作将剩余的所有数据包写入datanode管线，并在联系到namenode且发送文件写入完成信号之前，等待确认。



namenode已经知道文件由哪些块组成，所以它在返回成功前只需要等待数据块进行最小量的复制。

注意：如果在数据写入期间datanode发生故障，待确认消息队列迟迟得不到确认消息，这时会有一个超时时间，超过这个时间

文件写文件的时候只有一个客户端能写，保证数据上传成功



### Hadoop角度解析写入流程

![](http://image.tinx.top/img20210401142622.png)

1. 客户端向NameNode发出写文件请求。

2. namenode收到客户端的请求后，首先会检测元数据的目录树；检查权限并判断待上传的文件是否已存在，如果已存在，则拒绝client的上传。如果不存在，则响应客户端可以上传。

（注：WAL，write ahead log，先写Log，再写内存，因为EditLog记录的是最新的HDFS客户端执行所有的写操作。如果后续真实写操作失败了，由于在真实写操作之前，操作就被写入EditLog中了，故EditLog中仍会有记录，我们不用担心后续client读不到相应的数据块，因为在第5步中DataNode收到块后会有一返回确认信息，若没写成功，发送端没收到确认信息，会一直重试，直到成功）

3. 客户端收到可以上传的响应后，会把待上传的文件切块（hadoop2.x默认块大小为128M）;然后再次给namenode发送请求，上传第一个block块。

4. namenode收到客户端上传block块的请求后，首先会检测其保存的datanode信息，确定该文件块存储在那些节点上；最后，响应给客户端一组datanode节点信息。

5. 客户端根据收到datanode节点信息，首先就近与某台datanode建立网络连接；然后该datanode节点会与剩下的节点建立传输通道，通道连通后返回确认信息给客户端；表示通道已连通，可以传输数据。

6. 客户端收到确认信息后，通过网络向就近的datanode节点写第一个block块的数据；就近的datanode收到数据后，首先会缓存起来；然后将缓存里数据保存一份到本地，一份发送到传输通道；让剩下的datanode做备份。

client将NameNode返回的分配的可写的DataNode列表和Data数据一同发送给最近的第一个DataNode节点，此后client端和NameNode分配的多个DataNode构成pipeline管道，client端向输出流对象中写数据。client每向第一个DataNode写入一个packet，这个packet便会直接在pipeline里传给第二个、第三个…DataNode。

（注：并不是写好一个块或一整个文件后才向后分发）

每个DataNode写完一个块后，会返回确认信息。

（注：并不是每写完一个packet后就返回确认信息，个人觉得因为packet中的每个chunk都携带校验信息，没必要每写一个就汇报一下，这样效率太慢。正确的做法是写完一个block块后，对校验信息进行汇总分析，就能得出是否有块写错的情况发生）

7. 第一个block块写入完毕，若客户端还有剩余的block未上传；则客户端会从（3）开始，继续执行上述步骤；直到整个文件上传完毕。

写完数据，关闭输输出流。

发送完成信号给NameNode。

（注：发送完成信号的时机取决于集群是强一致性还是最终一致性，强一致性则需要所有DataNode写完后才向NameNode汇报。最终一致性则其中任意一个DataNode写完后就能单独向NameNode汇报，**<font color=red>HDFS一般情况下都是强调强一致性</font>**）



### 节点距离计算

上述写的流程有提到，```在HDFS写数据的过程中，NameNode会选择距离待上传数据最近距离的DataNode接收数据```，那么这最近距离要怎么计算呢？

<font color=red>节点距离：两个节点到达最近的共同祖先的距离总和</font>

![](http://image.tinx.top/img20210401153039.png)

### Hadoop 2.X副本节点选择

1. 第一个副本在Clinet所处的节点上，如果客户端在集群外，随机选一个
2. 第二个副本和第一个副本位于相同机架，随机节点
3. 第三个副本位于不同机架，随机节点



## HDFS 读数据流程

![](http://image.tinx.top/img20210401154434.png)

1. 客户端通过Distributed FileSystem向NameNode请求下载文件，NameNode通过查询元数据，找到文件块所在的DataNode地址。
2. 挑选一台DataNode（就近原则，然后随机）服务器，请求读取数据。
3. DataNode开始传输数据给客户端（从磁盘里面读取数据输入流，以Packet为单位来做校验）。
4. 客户端以Packet为单位接收，先在本地缓存，然后写入目标文件。



## NameNode和SecondaryNameNode工作机制

### 小思考： NameNode中的元数据是存储在哪里的？

首先，我们做个假设，如果存储在NameNode节点的磁盘中，**因为经常需要进行随机访问，还有响应客户请求，必然是效率过低。**因此，元数据需要存放在内存中。但如果只存在内存中，一旦断电，元数据丢失，整个集群就无法工作了。**因此产生在磁盘中备份元数据的FsImage。**

**这样又会带来新的问题，当在内存中的元数据更新时，如果同时更新FsImage，就会导致效率过低，但如果不更新，就会发生一致性问题，一旦NameNode节点断电，就会产生数据丢失。因此，引入Edits文件(只进行追加操作，效率很高)。每当元数据有更新或者添加元数据时，修改内存中的元数据并追加到Edits中。这样，一旦NameNode节点断电，可以通过FsImage和Edits的合并，合成元数据。**

但是，**如果长时间添加数据到Edits中，会导致该文件数据过大，效率降低，而且一旦断电，恢复元数据需要的时间过长。因此，需要定期进行FsImage和Edits的合并，如果这个操作由NameNode节点完成，又会效率过低。因此，引入一个新的节点SecondaryNamenode，专门用于FsImage和Edits的合并。**



### NameNode工作机制

![](http://image.tinx.top/img20210401155512.png)



1. 第一阶段：NameNode启动

   （1）第一次启动NameNode格式化后，创建Fsimage和Edits文件。如果不是第一次启动，直接加载编辑日志和镜像文件到内存。

   （2）客户端对元数据进行增删改的请求。

   （3）NameNode记录操作日志，更新滚动日志。

   （4）NameNode在内存中对数据进行增删改。

2. 第二阶段：Secondary NameNode工作

   （1）Secondary NameNode询问NameNode是否需要CheckPoint。直接带回NameNode是否检查结果。

   （2）Secondary NameNode请求执行CheckPoint。

   （3）NameNode滚动正在写的Edits日志。

   （4）将滚动前的编辑日志和镜像文件拷贝到Secondary NameNode。

   （5）Secondary NameNode加载编辑日志和镜像文件到内存，并合并。

   （6）生成新的镜像文件fsimage.chkpoint。

   （7）拷贝fsimage.chkpoint到NameNode。

   （8）NameNode将fsimage.chkpoint重新命名成fsimage。

***NN和2NN工作机制详解：***

Fsimage：NameNode内存中元数据序列化后形成的文件。

Edits：记录客户端更新元数据信息的每一步操作（可通过Edits运算出元数据）。

NameNode启动时，先滚动Edits并生成一个空的edits.inprogress，然后加载Edits和Fsimage到内存中，此时NameNode内存就持有最新的元数据信息。

Client开始对NameNode发送元数据的增删改的请求，这些请求的操作首先会被记录到edits.inprogress中（查询元数据的操作不会被记录在Edits中，因为查询操作不会更改元数据信息），如果此时NameNode挂掉，重启后会从Edits中读取元数据的信息。然后，NameNode会在内存中执行元数据的增删改的操作。 

由于Edits中记录的操作会越来越多，Edits文件会越来越大，导致NameNode在启动加载Edits时会很慢，所以需要对Edits和Fsimage进行合并（所谓合并，就是将Edits和Fsimage加载到内存中，照着Edits中的操作一步步执行，最终形成新的Fsimage）。

SecondaryNameNode的作用就是帮助NameNode进行Edits和Fsimage的合并工作。

SecondaryNameNode首先会询问NameNode是否需要CheckPoint（触发CheckPoint需要满足两个条件中的任意一个，定时时间到和Edits中数据写满了）。直接带回NameNode是否检查结果。SecondaryNameNode执行CheckPoint操作，首先会让NameNode滚动Edits并生成一个空的edits.inprogress，滚动Edits的目的是给Edits打个标记，以后所有新的操作都写入edits.inprogress，其他未合并的Edits和Fsimage会拷贝到SecondaryNameNode的本地，然后将拷贝的Edits和Fsimage加载到内存中进行合并，生成fsimage.chkpoint，然后将fsimage.chkpoint拷贝给NameNode，重命名为Fsimage后替换掉原来的Fsimage。NameNode在启动时就只需要加载之前未合并的Edits和Fsimage即可，因为合并过的Edits中的元数据信息已经被记录在Fsimage中。

**每次NameNode启动的时候，都会将Fsimage文件读入内存，加载Edits里面的更新操作，保证内存中的元数据信息是最新的、同步的，可以看成NameNode启动的时候就将Fsimage和Edits进行了合并**



### NameNode故障处理方法

```
NameNode故障后，可以通过下列两种方式进行恢复数据：
方法一（手动）：将SecondaryNameNode文件下的数据复制到NameNode中
方法二（程序）：使用-importCheckpoint选项启动NameNode的守护线程，
	从而将SecondaryNameNode文件目录下的数据拷贝到NamenNode中
```

具体操作方法
方法一
模拟NameNode故障，并采用方法一，恢复NameNode的数据。
（1）kill -9 NameNode进程
（2）删除NameNode存储的数据（$HADOOP_PATH/data/tmp/dfs/name）
	```$ rm -rf /opt/module/hadoop-2.7.2/data/tmp/dfs/name/*```
（3）拷贝SecondaryNameNode中的数据到原NameNode存储数据目录中
	```$ scp -r upuptop@hadoop104:/opt/module/hadoop-2.7.2/data/tmp/dfs/namesecondary/* ./name/```
（4）重启NameNode
	```$ sbin/hadoop-daemon.sh start namenode```
方法二
（1）修改hdfs-site.xml文件

```xml
<property>
  <name>dfs.namenode.checkpoint.period</name>
  <value>120</value>
</property>

<property>
  <name>dfs.namenode.name.dir</name>
  <value>/opt/module/hadoop-2.7.2/data/tmp/dfs/name</value>
</property>
```

（2）模拟NameNode挂掉

```kill -9 namenode进程```
（3）删除namenode存储的数据（/opt/module/hadoop-2.7.2/data/tmp/dfs/name）

```$ rm -rf /opt/module/hadoop-2.7.2/data/tmp/dfs/name/*```
（4）如果SecondaryNameNode不和Namenode在一个主机节点上，需要将SecondaryNameNode存储数据的目录拷贝到Namenode存储数据的平级目录，并删除in_use.lock文件。

```console
$ scp -r upuptop@hadoop104:/opt/module/hadoop-2.7.2/data/tmp/dfs/namesecondary ./

$ rm -rf in_use.lock

$ pwd
/opt/module/hadoop-2.7.2/data/tmp/dfs

$ ls
data  name  namesecondary
```

4）导入检查点数据（等待一会ctrl+c结束掉）

```console
$ bin/hdfs namenode -importCheckpoint
```

5）启动NameNode

```console
$ sbin/hadoop-daemon.sh start namenode
```



## DataNode工作机制

![](http://image.tinx.top/img20210401174212.png)

1. 一个数据块在DataNode上以文件形式存储在磁盘上，包括两个文件，一个是数据本身，一个是元数据包括数据块的长度，块数据的校验和，以及时间戳
2. DataNode启动后向NameNode注册，通过后，周期性(1小时)想NameNode上报所有的块信息
3. 心跳是每3秒一次，心跳返回结果带有NameNode给该DataNode命令 如复制块数据到另一台机器，或者删除某个数据块，如果超过10分钟没有收到某个DataNode心跳，则认为该节点不可用
4. 集群运行中可以安全加入和退出一些机器



## DataNode保存数据完整性的方法

![](http://image.tinx.top/img20210401174810.png)

1）当DataNode读取Block的时候，它会计算CheckSum。

2）如果计算后的CheckSum，与Block创建时值不一样，说明Block已经损坏。

3）Client读取其他DataNode上的Block。

4）DataNode在其文件创建后周期验证CheckSum



## DataNode掉线参数

![](http://image.tinx.top/img20210401175250.png)



相关参数可以在 ```hdfs-site.xml```中调整

~~~xml
<property>
    <name>dfs.namenode.heartbeat.recheck-interval</name>
    <value>300000</value>
</property>
<property>
    <name>dfs.heartbeat.interval</name>
    <value>3</value>
</property>
~~~



## HDFS-HA

![](http://image.tinx.top/img20210401180609.png)



## HDFS常见命令

### -ls: 显示目录信息

### -mkdir：在HDFS上创建目录

### -moveFromLocal：从本地剪切粘贴到HDFS

### -appendToFile：追加一个文件到已经存在的文件末尾

### -cat：显示文件内容

### -chgrp 、-chmod、-chown：Linux文件系统中的用法一样，修改文件所属权限

### -copyFromLocal：从本地文件系统中拷贝文件到HDFS路径去

### -copyToLocal：从HDFS拷贝到本地

### -cp ：从HDFS的一个路径拷贝到HDFS的另一个路径

### -mv：在HDFS目录中移动文件

### -get：等同于copyToLocal，就是从HDFS下载文件到本地

### -getmerge：合并下载多个文件，比如HDFS的目录 /user/atguigu/test下有多个文件:log.1, log.2,log.3,...

### -put：等同于copyFromLocal

### -tail：显示一个文件的末尾

### -rm：删除文件或文件夹

### -rmdir：删除空目录

### -du统计文件夹的大小信息

### -setrep：设置HDFS中文件的副本数量

