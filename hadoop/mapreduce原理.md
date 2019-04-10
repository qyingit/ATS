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

```

