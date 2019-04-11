# Yarn熟悉

###概述

```java
    yarn是一个资源调度平台,负责为运算程序提供服务器资源,相当于一个分布式的操作系统,而mapreduce相当于运行于操作系统上的应用程序
```

###基本架构

```java
1.ResourceManger
    1.处理客户端的请求
    2.监控nodemanager
    3.启动或监控ApplicationMaster
    4.资源的分配与调度
2.Nodemanager
    1.管理单个节点上的资源
    2.处理来自resourcemanager的命令
    3.处理来自applictionmaster的命令
3.ApplicationMaster
    1.负责数据的切分
    2.为应用程序申请资源并分配给内部任务
    3.任务的监控与容错
4.Container
    1.container是yarn中的资源抽象,封装了某个节点上的多资源维度,如内存,cpu,磁盘,网络等
```

#### yarn的工作机制

```java
1.mr程序提交到客户端所在的节点
2.yarnrunner向resourcemanager申请一个application
3.rm将应用程序的资源路径返回给yarnrunner
4.程序将运行所需要资源提交到hdfs
5.程序资源提交完毕后,申请运行mrappMaster
6.rm将用户的请求初始化成一个task
7.其中一个NodeManager领取到task任务。
8.该NodeManager创建容器Container，并产生MRAppmaster。
9.Container从HDFS上拷贝资源到本地。
10.MRAppmaster向RM 申请运行maptask资源。
11.RM将运行maptask任务分配给另外两个NodeManager，另两个NodeManager分别领取任务并创建容器。
12.MR向两个接收到任务的NodeManager发送程序启动脚本，这两个NodeManager分别启动maptask，maptask对数据分区排序
13.MrAppMaster等待所有maptask运行完毕后，向RM申请容器，运行reduce task
14.reduce task向maptask获取相应分区的数据
15.程序运行完毕后，MR会向RM申请注销自己。
```

#### 提交过程

```java
第0步：client调用job.waitForCompletion方法，向整个集群提交MapReduce作业。
第1步：client向RM申请一个作业id。
第2步：RM给client返回该job资源的提交路径和作业id。
第3步：client提交jar包、切片信息和配置文件到指定的资源提交路径
第4步：client提交完资源后，向RM申请运行MrAppMaster。
（2）作业初始化
第5步：当RM收到client的请求后，将该job添加到容量调度器中。
第6步：某一个空闲的NM领取到该job。
第7步：该NM创建Container，并产生MRAppmaster。
第8步：下载client提交的资源到本地。
（3）任务分配
第9步：MrAppMaster向RM申请运行多个maptask任务资源。
第10步：RM将运行maptask任务分配给另外两个NodeManager，另两个NodeManager分别领取任务并创建容器。
（4）任务运行
第11步：MR向两个接收到任务的NodeManager发送程序启动脚本，这两个NodeManager分别启动maptask，maptask对数据分区排序。
第12步：MrAppMaster等待所有maptask运行完毕后，向RM申请容器，运行reduce task。
第13步：reduce task向maptask获取相应分区的数据。
第14步：程序运行完毕后，MR会向RM申请注销自己。
（5）进度和状态更新
YARN中的任务将其进度和状态(包括counter)返回给应用管理器, 客户端每秒(通过mapreduce.client.progressmonitor.pollinterval设置)向应用管理器请求进度更新, 展示给用户。
（6）作业完成
除了向应用管理器请求作业进度外, 客户端每5分钟都会通过调用waitForCompletion()来检查作业是否完成。时间间隔可以通过mapreduce.client.completion.pollinterval来设置。作业完成之后, 应用管理器和container会清理工作状态。作业的信息会被作业历史服务器存储以备之后用户核查。
2）作业提交过程之MapReduce
```

#### 资源调度器

```
目前，Hadoop作业调度器主要有三种：FIFO、Capacity Scheduler和Fair Scheduler。Hadoop2.7.2默认的资源调度器是Capacity Scheduler。

先进先出调度器（FIFO）
容量调度器（Capacity Scheduler）
公平调度器（Fair Scheduler）


<property>
    <description>The class to use as the resource scheduler.</description>
    <name>yarn.resourcemanager.scheduler.class</name>
<value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
</property>

```

#### 任务的推测执行

```
1）作业完成时间取决于最慢的任务完成时间
一个作业由若干个Map任务和Reduce任务构成。因硬件老化、软件Bug等，某些任务可能运行非常慢。
典型案例：系统中有99%的Map任务都完成了，只有少数几个Map老是进度很慢，完不成，怎么办？
2）推测执行机制：
发现拖后腿的任务，比如某个任务运行速度远慢于任务平均速度。为拖后腿任务启动一个备份任务，同时运行。谁先运行完，则采用谁的结果。
3）执行推测任务的前提条件
（1）每个task只能有一个备份任务；
（2）当前job已完成的task必须不小于0.05（5%）
（3）开启推测执行参数设置。Hadoop2.7.2 mapred-site.xml文件中默认是打开的。
4）不能启用推测执行机制情况
   （1）任务间存在严重的负载倾斜；
   （2）特殊任务，比如任务向数据库中写数据。

```

##  MapReduce 跑的慢的原因

```
Mapreduce 程序效率的瓶颈在于两点：
1）计算机性能
	CPU、内存、磁盘健康、网络
2）I/O 操作优化
（1）数据倾斜
（2）map和reduce数设置不合理
（3）map运行时间太长，导致reduce等待过久
（4）小文件过多
（5）大量的不可分块的超大文件
（6）spill次数过多
（7）merge次数过多等。
 MapReduce优化方法
MapReduce优化方法主要从六个方面考虑：数据输入、Map阶段、Reduce阶段、IO传输、数据倾斜问题和常用的调优参数。
数据输入
（1）合并小文件：在执行mr任务前将小文件进行合并，大量的小文件会产生大量的map任务，增大map任务装载次数，而任务的装载比较耗时，从而导致mr运行较慢。
（2）采用CombineTextInputFormat来作为输入，解决输入端大量小文件场景。
Map阶段
1）减少溢写（spill）次数：通过调整io.sort.mb及sort.spill.percent参数值，增大触发spill的内存上限，减少spill次数，从而减少磁盘IO。
2）减少合并（merge）次数：通过调整io.sort.factor参数，增大merge的文件数目，减少merge的次数，从而缩短mr处理时间。
3）在map之后，不影响业务逻辑前提下，先进行combine处理，减少 I/O。
Reduce阶段
1）合理设置map和reduce数：两个都不能设置太少，也不能设置太多。太少，会导致task等待，延长处理时间；太多，会导致 map、reduce任务间竞争资源，造成处理超时等错误。
2）设置map、reduce共存：调整slowstart.completedmaps参数，使map运行到一定程度后，reduce也开始运行，减少reduce的等待时间。
3）规避使用reduce：因为reduce在用于连接数据集的时候将会产生大量的网络消耗。
4）合理设置reduce端的buffer：默认情况下，数据达到一个阈值的时候，buffer中的数据就会写入磁盘，然后reduce会从磁盘中获得所有的数据。也就是说，buffer和reduce是没有直接关联的，中间多个一个写磁盘->读磁盘的过程，既然有这个弊端，那么就可以通过参数来配置，使得buffer中的一部分数据可以直接输送到reduce，从而减少IO开销：mapred.job.reduce.input.buffer.percent，默认为0.0。当值大于0的时候，会保留指定比例的内存读buffer中的数据直接拿给reduce使用。这样一来，设置buffer需要内存，读取数据需要内存，reduce计算也要内存，所以要根据作业的运行情况进行调整。
I/O传输
1）采用数据压缩的方式，减少网络IO的的时间。安装Snappy和LZO压缩编码器。
2）使用SequenceFile二进制文件。
```

数据倾斜问题

```
1）数据倾斜现象
数据频率倾斜——某一个区域的数据量要远远大于其他区域。
数据大小倾斜——部分记录的大小远远大于平均值。
2）如何收集倾斜数据
在reduce方法中加入记录map输出键的详细情况的功能。
3）减少数据倾斜的方法
方法1：抽样和范围分区
可以通过对原始数据进行抽样得到的结果集来预设分区边界值。
方法2：自定义分区
基于输出键的背景知识进行自定义分区。例如，如果map输出键的单词来源于一本书。且其中某几个专业词汇较多。那么就可以自定义分区将这这些专业词汇发送给固定的一部分reduce实例。而将其他的都发送给剩余的reduce实例。
方法3：Combine
使用Combine可以大量地减小数据倾斜。在可能的情况下，combine的目的就是聚合并精简数据。
方法4：采用Map Join，尽量避免Reduce Join。
 
```

#### HDFS小文件优化方法

```
HDFS小文件弊端
HDFS上每个文件都要在namenode上建立一个索引，这个索引的大小约为150byte，这样当小文件比较多的时候，就会产生很多的索引文件，一方面会大量占用namenode的内存空间，另一方面就是索引文件过大是的索引速度变慢。
6.3.2 解决方案
1）Hadoop Archive:
 是一个高效地将小文件放入HDFS块中的文件存档工具，它能够将多个小文件打包成一个HAR文件，这样就减少了namenode的内存使用。
2）Sequence file：
 sequence file由一系列的二进制key/value组成，如果key为文件名，value为文件内容，则可以将大批小文件合并成一个大文件。
3）CombineFileInputFormat：
  CombineFileInputFormat是一种新的inputformat，用于将多个文件合并成一个单独的split，另外，它会考虑数据的存储位置。
4）开启JVM重用
对于大量小文件Job，可以开启JVM重用会减少45%运行时间。
JVM重用理解：一个map运行一个jvm，重用的话，在一个map在jvm上运行完毕后，jvm继续运行其他map。
具体设置：mapreduce.job.jvm.numtasks值在10-20之间

```

#### 常见错误及解决方案

```
1）导包容易出错。尤其Text和CombineTextInputFormat。
2）Mapper中第一个输入的参数必须是LongWritable或者NullWritable，不可以是IntWritable.  报的错误是类型转换异常。
3）java.lang.Exception: java.io.IOException: Illegal partition for 13926435656 (4)，说明partition和reducetask个数没对上，调整reducetask个数。
4）如果分区数不是1，但是reducetask为1，是否执行分区过程。答案是：不执行分区过程。因为在maptask的源码中，执行分区的前提是先判断reduceNum个数是否大于1。不大于1肯定不执行。
5）在Windows环境编译的jar包导入到linux环境中运行，
hadoop jar wc.jar com.atguigu.mapreduce.wordcount.WordCountDriver /user/atguigu/ /user/atguigu/output
报如下错误：
Exception in thread "main" java.lang.UnsupportedClassVersionError: com/atguigu/mapreduce/wordcount/WordCountDriver : Unsupported major.minor version 52.0
原因是Windows环境用的jdk1.7，linux环境用的jdk1.8。
解决方案：统一jdk版本。
6）缓存pd.txt小文件案例中，报找不到pd.txt文件
原因：大部分为路径书写错误。还有就是要检查pd.txt.txt的问题。还有个别电脑写相对路径找不到pd.txt，可以修改为绝对路径。
7）报类型转换异常。
通常都是在驱动函数中设置map输出和最终输出时编写错误。
Map输出的key如果没有排序，也会报类型转换异常。
8）集群中运行wc.jar时出现了无法获得输入文件。
原因：wordcount案例的输入文件不能放用hdfs集群的根目录。
9）出现了如下相关异常
Exception in thread "main" java.lang.UnsatisfiedLinkError: org.apache.hadoop.io.nativeio.NativeIO$Windows.access0(Ljava/lang/String;I)Z
	at org.apache.hadoop.io.nativeio.NativeIO$Windows.access0(Native Method)
	at org.apache.hadoop.io.nativeio.NativeIO$Windows.access(NativeIO.java:609)
	at org.apache.hadoop.fs.FileUtil.canRead(FileUtil.java:977)
java.io.IOException: Could not locate executable null\bin\winutils.exe in the Hadoop binaries.
	at org.apache.hadoop.util.Shell.getQualifiedBinPath(Shell.java:356)
	at org.apache.hadoop.util.Shell.getWinUtilsPath(Shell.java:371)
	at org.apache.hadoop.util.Shell.<clinit>(Shell.java:364)
解决方案：拷贝hadoop.dll文件到windows目录C:\Windows\System32。个别同学电脑还需要修改hadoop源码。
10）自定义outputformat时，注意在recordWirter中的close方法必须关闭流资源。否则输出的文件内容中数据为空。

```

