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

```

```

