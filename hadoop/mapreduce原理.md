# mapreduce原理
#### mapreduce核心思想
1. 分布式计算分为两个阶段
2. maptask阶段，完全并行运行，互不相干
3. reducetask阶段,依赖于上一个阶段maptask并发实例的输出
4. mapreduce只能包含一个map和一个reduce阶段，如果用户逻辑复杂，则需要多个mapreduce程序,串行运行
#### mapreduce进程
*mapreduce程序运行时有三类实例*
1. MrAppMaster:负责整个程序的过程调度及状态协调
2. MapTask：负责map阶段的整个数据处理流程
3. ReduceTask：负责reduce阶段整个数据处理流程
#### 编程规范
*用户编写的程序分成三个部分：Mapper、Reducer和Driver*
1. mapper阶段
   (1）用户自定义的Mapper要继承自己的父类
   （2）Mapper的输入数据是KV对的形式（KV的类型可自定义）
   （3）Mapper中的业务逻辑写在map()方法中
   （4）Mapper的输出数据是KV对的形式（KV的类型可自定义）
   （5）map()方法（maptask进程）对每一个<K,V>调用一次
2. reducer阶段
  （1）用户自定义的Reducer要继承自己的父类
  （2）Reducer的输入数据类型对应Mapper的输出数据类型，也是KV
  （3）Reducer的业务逻辑写在reduce()方法中
  （4）Reducetask进程对每一组相同k的<k,v>组调用一次reduce()方法
3. Driver阶段
  整个程序需要一个Drvier来进行提交，提交的是一个描述了各种必要信息的job对象
###工作流程

```java
1. maptask收集map()方法输出的kv对,放到内存缓冲区
2. 从内存中不断溢出本地磁盘文件，可能溢出多个文件
3. 多个溢出文件合并为大文件
4. 在溢出过程中，及合并的过程中，都要调用partitioner进行分区和针对key进行排序
5. reducetask根据自己的分区号，去各个maptask机器上取相应的结果分区数据
6. reducetask会取到同一个分区的来自不同maptask的结果文件，reducetask会将这些文件再进行合并（归并排序）
7. 合并成大文件后，shuffle的过程也就结束了，后面进入reducetask的逻辑运算过程（从文件中取出一个一个的键值对group，调用用户自定义的reduce()方法）
注意:
Shuffle中的缓冲区大小会影响到mapreduce程序的执行效率，原则上说，缓冲区越大，磁盘io的次数越少，执行速度就越快。
缓冲区的大小可以通过参数调整，参数：io.sort.mb  默认100M。
```

#### FileInputFormat源码解析(input.getSplits(job))

```java
1. 找到你数据存储的目录。
2. 开始遍历处理（规划切片）目录下的每一个文件
3. 遍历第一个文件ss.txt
		1.获取文件大小fs.sizeOf(ss.txt)
		2.计算切片大小
computeSliteSize(Math.max(minSize,Math.min(maxSize,blocksize)))=blocksize=128M
        3.默认情况下，切片大小=blocksize
        4.开始切，形成第1个切片：ss.txt—0:128M 第2个切片ss.txt—128:256M 第3个切片ss.txt—256M:300M（每次切片时，都要判断切完剩下的部分是否大于块的1.1倍，不大于1.1倍就划分一块切片）
        5.将切片信息写到一个切片规划文件中
        6.整个切片的核心过程在getSplit()方法中完成
        7.数据切片只是在逻辑上对输入数据进行分片，并不会再磁盘上将其切分成分片进行存储。InputSplit只记录了分片的元数据信息，比如起始位置、长度以及所在的节点列表等
        8.注意：block是HDFS物理上存储的数据，切片是对数据逻辑上的划分
   4.提交切片规划文件到yarn上，yarn上的MrAppMaster就可以根据切片规划文件计算开启maptask个数
```

切片机制

```java
1.简单地按照文件的内容长度进行切片
2.切片大小，默认等于block大小
3.切片时不考虑数据集整体，而是逐个针对每一个文件单独切片
缺点：小文件过多时，会导致切片过多，消耗系统资源
优化策略：
1.在预处理时将小文件合并成大文件,再上传到hdfs
2.使用CombineTextInputFormat，可以将多个小文件从逻辑上规划到一个切片中
    demo: 优先满足最小切片大小，不超过最大切片大小
CombineTextInputFormat.setMaxInputSplitSize(job, 4194304);// 4m
CombineTextInputFormat.setMinInputSplitSize(job, 2097152);// 2m
```

#####InputFormat接口实现类

```java
InputFormat常见的接口实现类包括：
1. TextInputFormat
每条记录是一行输入。键是LongWritable类型，存储该行在整个文件中的字节偏移量。值是这行的内容，不包括任何行终止符
2. KeyValueTextInputFormat
每一行均为一条记录,被分隔符分割为key,value。可以通过在驱动类中设置conf.set(KeyValueLineRecordReader
.KEY_VALUE_SEPERATOR, " ");来设定分隔符。默认分隔符是tab（\t）。
3. NLineInputFormat
如果使用NlineInputFormat，代表每个map进程处理的InputSplit不再按block块去划分，而是按NlineInputFormat指定的行数N来划分。即输入文件的总行数/N=切片数，如果不整除，切片数=商+1。
NLineInputFormat.setNumLinesPerSplit(job, 3);
4. CombineTextInputFormat
指定文件大小的切片
5.自定义InputFormat等
    1.自定义一个类继承FileInputFormat
    2.改写RecordReader，实现一次读取一个完整文件封装为KV
    3.在输出时使用SequenceFileOutPutFormat输出合并文件
```

MapTask工作机制

```java
mapTask决定机制:
一个job的map阶段MapTask并行度（个数），由客户端提交job时的切片个数决定。
1. Read阶段：Map Task通过用户编写的RecordReader，从输入InputSplit中解析出一个个key/value。
2. Map阶段：该节点主要是将解析出的key/value交给用户编写map()函数处理，并产生一系列新的key/value。
3. Collect收集阶段：在用户编写map()函数中，当数据处理完成后，一般会调用OutputCollector.collect()输出结果。在该函数内部，它会将生成的key/value分区（调用Partitioner），并写入一个环形内存缓冲区中。
4. Spill阶段：即“溢写”，当环形缓冲区满后，MapReduce会将数据写到本地磁盘上，生成一个临时文件。需要注意的是，将数据写入本地磁盘之前，先要对数据进行一次本地排序，并在必要时对数据进行合并、压缩等操作。
   数据溢写:
       1. 利用快排对缓存区数据排序，排序方式，先按照分区号排序，然后按照key排序，经过排序后，数据以分区为单位聚集再一起，同一分区内所有数据按照key有序。
       2. 按照分区编号由小到大依次将每个分区中的数据写入任务工作目录下的临时文件output/spillN.out（N表示当前溢写次数）中。如果用户设置了Combiner，则写入文件之前，对每个分区中的数据进行一次聚集操作。
       3. 将分区数据的元信息写到内存索引数据结构SpillRecord中，其中每个分区的元信息包括在临时文件中的偏移量、压缩前数据大小和压缩后数据大小。如果当前内存索引大小超过1MB，则将内存索引写到文件output/spillN.out.index中。
       4.Combine阶段：当所有数据处理完成后，MapTask对所有临时文件进行一次合并，以确保最终只会生成一个数据文件。
       所有数据以分区为单位合并生成一个大文件,每轮合并io.sort.factor（默认100）个文件,重复生成一个大文件,让每个MapTask最终只生成一个数据文件，可避免同时打开大量文件和同时读取大量小文件产生的随机读取带来的开销
```

### Shuffer机制

```
shuffer机制：
Mapreduce确保每个reducer的输入都是按键排序的。系统执行排序的过程（即将map输出作为输入传给reducer）称为shuffle
```

### Partition分区

```
分区后要设置对应的reduceTask数量
注意：
如果reduceTask的数量> getPartition的结果数，则会多产生几个空的输出文件part-r-000xx；
如果1<reduceTask的数量<getPartition的结果数，则有一部分分区数据无处安放，会Exception；
如果reduceTask的数量=1，则不管mapTask端输出多少个分区文件，最终结果都交给这一个reduceTask，最终也就只会产生一个结果文件 part-r-00000；
```

### WritableComparable排序

```java
排序的分类：
1.部分排序：
MapReduce根据输入记录的键对数据集排序。保证输出的每个文件内部排序。
2.全排序：
如何用Hadoop产生一个全局排序的文件？最简单的方法是使用一个分区。但该方法在处理大型文件时效率极低，因为一台机器必须处理所有输出文件，从而完全丧失了MapReduce所提供的并行架构。
替代方案：首先创建一系列排好序的文件；其次，串联这些文件；最后，生成一个全局排序的文件。主要思路是使用一个分区来描述输出的全局排序。例如：可以为上述文件创建3个分区，在第一分区中，记录的单词首字母a-g，第二分区记录单词首字母h-n, 第三分区记录单词首字母o-z。
3.辅助排序：（GroupingComparator分组）
	Mapreduce框架在记录到达reducer之前按键对记录排序，但键所对应的值并没有被排序。甚至在不同的执行轮次中，这些值的排序也不固定，因为它们来自不同的map任务且这些map任务在不同轮次中完成时间各不相同。一般来说，大多数MapReduce程序会避免让reduce函数依赖于值的排序。但是，有时也需要通过特定的方法对键进行排序和分组等以实现对值的排序
4.二次排序：
在自定义排序过程中，如果compareTo中的判断条件为两个即为二次排序。
5.自定义排序WritableComparable
bean对象实现WritableComparable接口重写compareTo方法，就可以实现排序
@Override
public int compareTo(FlowBean o) {
	// 倒序排列，从大到小
	return this.sumFlow > o.getSumFlow() ? -1 : 1;
}

```

### Combiner合并

```
1）combiner是MR程序中Mapper和Reducer之外的一种组件。
2）combiner组件的父类就是Reducer。
3）combiner和reducer的区别在于运行的位置：
Combiner是在每一个maptask所在的节点运行;
Reducer是接收全局所有Mapper的输出结果；
4）combiner的意义就是对每一个maptask的输出进行局部汇总，以减小网络传输量。
5）combiner能够应用的前提是不能影响最终的业务逻辑，而且，combiner的输出kv应该跟reducer的输入kv类型要对应起来。
```

### GroupingComparator分组（辅助排序）

```java
public class OrderGroupingComparator extends WritableComparator {

	protected OrderGroupingComparator() {
		super(OrderBean.class, true);
	}

	@SuppressWarnings("rawtypes")
	@Override
	public int compare(WritableComparable a, WritableComparable b) {
		OrderBean aBean = (OrderBean) a;
		OrderBean bBean = (OrderBean) b;
		int result;
		if (aBean.getOrder_id() > bBean.getOrder_id()) {
			result = 1;
		} else if (aBean.getOrder_id() < bBean.getOrder_id()) {
			result = -1;
		} else {
			result = 0;
		}
		return result;
	}
}


job.setGroupingComparatorClass(OrderGroupingComparator.class);
```



###ReduceTask工作机制

```java
运行流程：
    1.copy阶段：reducetask从maptask拷贝一片数据，如果某一片数据大小超过阈值，就写道磁盘中，否则到内存中
    2.merge阶段：在远程拷贝数据的同时，reducetask启动了两个后台线程对内存和磁盘文件合并，防止文件过多
    3.sort阶段：根据mapredeuce语义,用户编写reduce()函数输入数据按key进行聚集。为了将key相同的数据聚在一起，hadoop采用基于排序的策略，由于各个maptask已经实现对自己处理结果进行了局部排序，只需要在进行一次归并排序
    4.Reduce阶段：reduce()将计算结果写道hdfs上

注意事项：
1.设置reduce task并行度
    reducetask的并行度影响job的执行并发度与执行效率，但maptask的并发数与切片数决定不同,reducetask的数量可以手动设置
    job.setNumReduceTasks(4);
2.注意：
    reducetask=0，表示没有reduce阶段，输出文件个数和map个数一致
    reducetask默认值就是1，所以输出文件个数为一个
    如果数据分布不均匀，就有可能在reduce阶段产生数据倾斜
    reducetask数量并不是任意设置，还要考虑业务逻辑需求，有些情况下，需要计算全局汇总结果，就只能有1个reducetask
    具体多少个reducetask，需要根据集群性能而定
    如果分区数不是1，但是reducetask为1，是否执行分区过程。答案是：不执行分区过程。因为在maptask的源码中，执行分区的前提是先判断reduceNum个数是否大于1。不大于1肯定不执行
```

###OutputFormat接口实现类

```java
几种常见的OutputFormat实现类
1.TextOutputFormat文本输出
    默认的输出格式是TextOutputFormat，它把每条记录写为文本行。它的键和值可以是任意类型，因为TextOutputFormat调用toString()方法把它们转换为字符串。
2.SequenceFileOutputFormat
    SequenceFileOutputFormat将它的输出写为一个顺序文件。如果输出需要作为后续 MapReduce任务的输入，这便是一种好的输出格式，因为它的格式紧凑，很容易被压缩
3.自定义OutputFormat
    根据用户需求，自定义实现输出
    demo:
        1.自定义一个类继承FileOutputFormat。
        2.改写recordwriter，具体改写输出数据的方法write()。
```

### JOIN应用

```java
reducejoin：
    1.map端：为不同表的key与value打标签，区分不同的来源记录，用连接字段作为key，其余部分和新加的标志作为value输出
    2.reduce端: 在reduce端以连接字段作为key，只需要在分组中将那些来源不同的文件记录分开,最后合并就ok了
    缺点:造成map和reduce端也就是shuffle阶段出现大量的数据传输，效率很低。
mapjoin:
    具体方法:
        1.在mapper的setup阶段,将文件读取到缓存集合中
        2.在驱动函数中加载缓存
        job.addCacheFile(new URI("file:/e:/
 mapjoincache/pd.txt"));// 缓存普通文件到task运行节点
        DistributedCacheDriver缓存文件
        mapper读取文件的数据
        setup()方法中
           1.获取缓存的文件
           2.循环读取缓存文件一行
           3.切割
           4.缓存数据到集合
           5.关流
        map()方法中
           1.获取一行
           2.截取
           3.获取订单id
           4.获取商品名称
           5.拼接
           6.写出
    解决方案:在map端缓存多张表,提前处理业务逻辑，增加map端业务，减少reduce端数据的压力，尽力减少数据倾斜
    场景:一张表十分小，一张表十分大
```

### 计数器应用

```java
Hadoop为每个作业维护若干内置计数器，以描述多项指标。例如，某些计数器记录已处理的字节数和记录数，使用户可监控已处理的输入数据量和已产生的输出数据量
API
(1）采用枚举的方式统计计数
enum MyCounter{MALFORORMED,NORMAL}
//对枚举定义的自定义计数器加1
context.getCounter(MyCounter.MALFORORMED).increment(1);
(2）采用计数器组、计数器名称的方式统计context.getCounter("counterGroup","countera").increment(1);组名和计数器名称随便起，但最好有意义。
(3）计数结果在程序运行后的控制台上查看。

```

###数据清洗（ETL）

```java
在运行核心业务Mapreduce程序之前，往往要先对数据进行清洗，清理掉不符合用户要求的数据。清理的过程往往只需要运行mapper程序，不需要运行reduce程序
```



## MapReduce开发总结

```java
在编写mapreduce程序时，需要考虑的几个方面
1:输入数据接口：InputFormat
   默认使用的实现类是：TextInputFormat 
   TextInputFormat的功能逻辑是：一次读一行文本，然后将该行的起始偏移量作为key，行内容作为value返回。
KeyValueTextInputFormat每一行均为一条记录，被分隔符分割为key，value。默认分隔符是tab（\t）。
NlineInputFormat按照指定的行数N来划分切片。
CombineTextInputFormat可以把多个小文件合并成一个切片处理，提高处理效率。
用户还可以自定义InputFormat。
2:逻辑处理接口：Mapper  
   用户根据业务需求实现其中三个方法：map()   setup()   cleanup () 
3.Partitioner分区
	有默认实现 HashPartitioner，逻辑是根据key的哈希值和numReduces来返回一个分区号；key.hashCode()&Integer.MAXVALUE % numReduces
	如果业务上有特别的需求，可以自定义分区。
4.Comparable排序
	当我们用自定义的对象作为key来输出时，就必须要实现WritableComparable接口，重写其中的compareTo()方法。
	部分排序：对最终输出的每一个文件进行内部排序。
	全排序：对所有数据进行排序，通常只有一个Reduce。
	二次排序：排序的条件有两个。
5.Combiner合并
Combiner合并可以提高程序执行效率，减少io传输。但是使用时必须不能影响原有的业务处理结果
6.reduce端分组：Groupingcomparator
	reduceTask拿到输入数据（一个partition的所有数据）后，首先需要对数据进行分组，其分组的默认原则是key相同，然后对每一组kv数据调用一次reduce()方法，并且将这一组kv中的第一个kv的key作为参数传给reduce的key，将这一组数据的value的迭代器传给reduce()的values参数。
	利用上述这个机制，我们可以实现一个高效的分组取最大值的逻辑。
	自定义一个bean对象用来封装我们的数据，然后改写其compareTo方法产生倒序排序的效果。然后自定义一个Groupingcomparator，将bean对象的分组逻辑改成按照我们的业务分组id来分组（比如订单号）。这样，我们要取的最大值就是reduce()方法中传进来key。
7.逻辑处理接口：Reducer
	用户根据业务需求实现其中三个方法：reduce()   setup()   cleanup () 
8.输出数据接口：OutputFormat
	默认实现类是TextOutputFormat，功能逻辑是：将每一个KV对向目标文本文件中输出为一行。
 SequenceFileOutputFormat将它的输出写为一个顺序文件。如果输出需要作为后续 MapReduce任务的输入，这便是一种好的输出格式，因为它的格式紧凑，很容易被压缩。
用户还可以自定义OutputFormat。

```

## Hadoop数据压缩

```java
如果磁盘I/O和网络带宽影响了MapReduce作业性能，在任意MapReduce阶段启用压缩都可以改善端到端处理时间并减少I/O和网络流量
压缩Mapreduce的一种优化策略：通过压缩编码对Mapper或者Reducer的输出进行压缩，以减少磁盘IO，提高MR程序运行速度（但相应增加了cpu运算负担）
注意：压缩特性运用得当能提高性能，但运用不当也可能降低性能。
基本原则：
（1）运算密集型的job，少用压缩
（2）IO密集型的job，多用压缩

压缩方式:
1. Gzip压缩
优点：压缩率比较高，而且压缩/解压速度也比较快；hadoop本身支持，在应用中处理gzip格式的文件就和直接处理文本一样；大部分linux系统都自带gzip命令，使用方便。
缺点：不支持split。
应用场景：当每个文件压缩之后在130M以内的（1个块大小内），都可以考虑用gzip压缩格式。例如说一天或者一个小时的日志压缩成一个gzip文件，运行mapreduce程序的时候通过多个gzip文件达到并发。hive程序，streaming程序，和java写的mapreduce程序完全和文本处理一样，压缩之后原来的程序不需要做任何修改。
2.Bzip2压缩
优点：支持split；具有很高的压缩率，比gzip压缩率都高；hadoop本身支持，但不支持native；在linux系统下自带bzip2命令，使用方便。
缺点：压缩/解压速度慢；不支持native。
应用场景：适合对速度要求不高，但需要较高的压缩率的时候，可以作为mapreduce作业的输出格式；或者输出之后的数据比较大，处理之后的数据需要压缩存档减少磁盘空间并且以后数据用得比较少的情况；或者对单个很大的文本文件想压缩减少存储空间，同时又需要支持split，而且兼容之前的应用程序（即应用程序不需要修改）的情况。
3.Lzo压缩
优点：压缩/解压速度也比较快，合理的压缩率；支持split，是hadoop中最流行的压缩格式；可以在linux系统下安装lzop命令，使用方便。
缺点：压缩率比gzip要低一些；hadoop本身不支持，需要安装；在应用中对lzo格式的文件需要做一些特殊处理（为了支持split需要建索引，还需要指定inputformat为lzo格式）。
应用场景：一个很大的文本文件，压缩之后还大于200M以上的可以考虑，而且单个文件越大，lzo优点越越明显。
4. Snappy压缩
优点：高速压缩速度和合理的压缩率。
缺点：不支持split；压缩率比gzip要低；hadoop本身不支持，需要安装；
应用场景：当Mapreduce作业的Map输出的数据比较大的时候，作为Map到Reduce的中间数据的压缩格式；或者作为一个Mapreduce作业的输出和另外一个Mapreduce作业的输入。


压缩位置选择:
1.输入端采用压缩
在有大量数据并计划重复处理的情况下，应该考虑对输入进行压缩。然而，你无须显示指定使用的编解码方式。Hadoop自动检查文件扩展名，如果扩展名能够匹配，就会用恰当的编解码方式对文件进行压缩和解压。否则，Hadoop就不会使用任何编解码器。
2.mapper输出采用压缩
当map任务输出的中间数据量很大时，应考虑在此阶段采用压缩技术。这能显著改善内部数据Shuffle过程，而Shuffle过程在Hadoop处理过程中是资源消耗最多的环节。如果发现数据量大造成网络传输缓慢，应该考虑使用压缩技术。可用于压缩mapper输出的快速编解码器包括LZO或者Snappy。
注：LZO是供Hadoop压缩数据用的通用压缩编解码器。其设计目标是达到与硬盘读取速度相当的压缩速度，因此速度是优先考虑的因素，而不是压缩率。与gzip编解码器相比，它的压缩速度是gzip的5倍，而解压速度是gzip的2倍。同一个文件用LZO压缩后比用gzip压缩后大50%，但比压缩前小25%~50%。这对改善性能非常有利，map阶段完成时间快4倍。
3.reducer输出采用压缩
在此阶段启用压缩技术能够减少要存储的数据量，因此降低所需的磁盘空间。当mapreduce作业形成作业链条时，因为第二个作业的输入也已压缩，所以启用压缩同样有效



压缩参数配置:
io.compression.codecs   
（在core-site.xml中配置）	org.apache.hadoop.io.compress.DefaultCodec, org.apache.hadoop.io.compress.GzipCodec, org.apache.hadoop.io.compress.BZip2Codec
	输入压缩	Hadoop使用文件扩展名判断是否支持某种编解码器
mapreduce.map.output.compress（在mapred-site.xml中配置）	false	mapper输出	这个参数设为true启用压缩
mapreduce.map.output.compress.codec（在mapred-site.xml中配置）	org.apache.hadoop.io.compress.DefaultCodec	mapper输出	使用LZO或snappy编解码器在此阶段压缩数据
mapreduce.output.fileoutputformat.compress（在mapred-site.xml中配置）	false	reducer输出	这个参数设为true启用压缩
mapreduce.output.fileoutputformat.compress.codec（在mapred-site.xml中配置）	org.apache.hadoop.io.compress. DefaultCodec	reducer输出	使用标准工具或者编解码器，如gzip和bzip2
mapreduce.output.fileoutputformat.compress.type（在mapred-site.xml中配置）	RECORD	reducer输出	SequenceFile输出使用的压缩类型：NONE和BLOCK


数据流的压缩和解压缩
 CompressionCodec有两个方法可以用于轻松地压缩或解压缩数据。要想对正在被写入一个输出流的数据进行压缩，我们可以使用createOutputStream(OutputStreamout)方法创建一个CompressionOutputStream，将其以压缩格式写入底层的流。相反，要想对从输入流读取而来的数据进行解压缩，则调用createInputStream(InputStreamin)函数，从而获得一个CompressionInputStream，从而从底层的流读取未压缩的数据。
 
  Map输出端采用压缩
  即使你的MapReduce的输入输出文件都是未压缩的文件，你仍然可以对map任务的中间结果输出做压缩，因为它要写在硬盘并且通过网络传输到reduce节点，对其压缩可以提高很多性能，这些工作只要设置两个属性即可，我们来看下代码怎么设置：
```

