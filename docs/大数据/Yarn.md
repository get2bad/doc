## Yarn基本架构

YARN主要由ResourceManager、NodeManager、ApplicationMaster和Container等组件构成

![](http://image.tinx.top/img20210406162113.png)

### ResourceManager

ResourceManager由两个关键组件Scheduler和ApplicationsManager组成。

#### Scheduler

Scheduler在容量和队列限制范围内负责为运行的容器分配资源。Scheduler是一个纯调度器（pure scheduler），只负责调度，它不会监视或跟踪应用程序的状态，也不负责重启失败任务，这些全都交给ApplicationMaster完成。Scheduler根据各个应用程序的资源需求进行资源分配。

#### ApplicationsManager

ApplicationManager负责接收作业的提交，然后启动一个ApplicationMaster容器负责该应用。它也会在ApplicationMaster容器失败时，重新启动ApplicationMaster容器。



### NodeManager

Hadoop 2集群中的每个DataNode都会运行一个NodeManager来执行Yarn的功能。每个节点上的NodeManager执行以下功能：

- 定时向ResourceManager汇报本节点资源使用情况和各个Container的运行状况
- 监督应用程序容器的生命周期
- （监控资源）监控、管理和提供容器消耗的有关资源（CPU/内存）的信息
- （监控容器）监控容器的资源使用情况，杀死失去控制的程序
- （启动/停止容器）接受并处理来自ApplicationMaster的Container启动/停止等各种请求。



### ApplicationMaster(上图的App Mstr)

**提交到Yarn上的每一个应用都有一个专用的ApplicationMaster**(注意，ApplicationMaster需要和ApplicationManager区分）。

ApplicationMaster运行在应用程序启动时的第一个容器内。

ApplicationMaster会与ResourceManager协商获取容器来执行应用中的mappers和reducers，之后会将ResourceManager分配的容器资源呈现给运行在每个DataNode上的NodeManager。ApplicationMaster请求的资源是具体的。包括：

- 处理作业需要的文件块
- 为应用程序创建的以容器为单位的资源
- 容器大小（例如，1GB内存和一个虚拟核心）
- 资源在何处分配，这个依据从NameNode获取的块存储位置信息（如机器1的节点10上分配4个容器，机器2的节点20上分配8个容器）
- 资源请求的优先级



### Container

Container是对于资源的抽象, 它封装了某个节点上的多维度资源，如内存、CPU等。



## Yarn工作流程

![](https://pic2.zhimg.com/80/v2-33ffdec8222c48ca30b70b30347b553d_720w.jpg)

1. 客户端向ResourceManager提交应用程序。
2. ResourceManager的ApplicationManager组件指示NodeManager（运行在每一个工作节点的其中一个）为应用程序启动一个新的ApplicationMaster容器。
3. ApplicationMaster首先向ResourceManager注册，这样用户可以直接通过NodeManager查看应用程序的运行状态。
4. ApplicationMaster计算应用完成所需要的资源，然后向ResourceManager申请需要的资源（容器）。ApplicationMaster在应用程序的整个生命周期内与ResourceManager保持联系，确保其所需要资源的列表被ResourceManager严格遵守，并且发送一些必要的Kill请求杀死任务。
5. 申请到资源后，ApplicationMaster指示NodeManager在对应的节点上创建容器。
6. NodeManager创建容器，设置好运行环境，然后启动容器。
7. 各个容器定时向ApplicationMaster发送任务的运行状态和进度，这样ApplicationMaster随时掌握各个任务的运行状态，从而可以在任务失败时重新启动任务。
8. 应用程序完成后，ApplicationMaster会通知ResourceManager该作业已经完成并注销关闭自己。

> 这里要注意几点。第一，NodeManager会将节点状态和健康状况发送到ResourceManager，ResourceManager拥有全局资源视图才能分配资源。第二，ResourceManager的Scheduler组件决定容器在哪个节点上运行。



## 作业提交全过程

![](http://image.tinx.top/img20210406162810.png)



1. 作业提交

   1. Client调用 Job.waitForCompletion方法，想整个集群提交MapReduce作业
   2. Client向RM申请一个作业id
   3. RM给Client返回该job资源的提交路径和作业id
   4. Client提交jar包，切片信息和配置文件到指定的资源提交路径
   5. Client提交完资源后，向RM申请运行MrAppMaster

2. 作业初始化

   6. 当RM收到Client请求后，将该Job添加到容量调度器中
   7. 某一个空闲的NM领取到该Job
   8. 该NM创建Container，并产生MRAppmaster
   9. 下载Client提交的资源到本地

3. 任务分配

   10. MrAppMaster向RM申请运行多个MapTask任务资源
   11. RM将运行MapTask任务分配给另外两个NodeManager,另外两个NodeManager分别领取任务并创建容器

4. 任务运行

   12. MR向两个接收到的任务的NodeManager发送程序启动脚本，这两个NodeManager分别启动MapTask，MapTask对数据分区排序
   13. MrAppMaster等待所有MapTask运行完毕后，向RM申请容器，运行ReduceTask
   14. ReduceTask向MapTask获取相应分区的数据
   15. 程序运行完毕后，MR会向RM申请注销自己

5. 进度和状态更新

   YARN中的任务将其进度和状态(包括counter)返回给应用管理器, 客户端每秒(通过mapreduce.client.progressmonitor.pollinterval设置)向应用管理器请求进度更新, 展示给用户。

6. 作业完成

   除了向应用管理器请求作业进度外, 客户端每5秒都会通过调用waitForCompletion()来检查作业是否完成。时间间隔可以通过mapreduce.client.completion.pollinterval来设置。作业完成之后, 应用管理器和Container会清理工作状态。作业的信息会被作业历史服务器存储以备之后用户核查。



## 资源调度器

目前，Hadoop作业调度器主要有三种：**FIFO、Capacity Scheduler和Fair Scheduler。Hadoop2.7.2默认的资源调度器是Capacity Scheduler**。

具体设置详见：yarn-default.xml文件

~~~xml
<property>

  <description>The class to use as the resource scheduler.</description>

  <name>yarn.resourcemanager.scheduler.class</name>

	<value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>

</property>
~~~

### FIFO

顾名思义，就是先进先出，队列模式，按照时间排序先到先服务

![](http://image.tinx.top/img20210406164439.png)

### Capacity Scheduler

这个调度器使用的是资源占用模式，即多个作业可以同时运行(队列模式的升级版)，但是占用的性能百分比不同

![](http://image.tinx.top/img20210406164531.png)

### 公平调度器（Fair Scheduler）

![](http://image.tinx.top/img20210406164612.png)

### 推测执行算法原理

![](http://image.tinx.top/img20210406164831.png)

## 优化参数

应该在YARN启动之前就配置在服务器的配置文件中才能生效（yarn-default.xml）

| 配置参数                                 | 参数说明                                        |
| ---------------------------------------- | ----------------------------------------------- |
| yarn.scheduler.minimum-allocation-mb     | 给应用程序Container分配的最小内存，默认值：1024 |
| yarn.scheduler.maximum-allocation-mb     | 给应用程序Container分配的最大内存，默认值：8192 |
| yarn.scheduler.minimum-allocation-vcores | 每个Container申请的最小CPU核数，默认值：1       |
| yarn.scheduler.maximum-allocation-vcores | 每个Container申请的最大CPU核数，默认值：32      |
| yarn.nodemanager.resource.memory-mb      | 给Containers分配的最大物理内存，默认值：8192    |