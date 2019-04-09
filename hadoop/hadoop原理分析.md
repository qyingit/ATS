# hadoop原理分析

### 架构分析

四个部分组成，分别为HDFS Client、NameNode、DataNode和Secondary NameNode

* Client
  1. 文件上传时，对文件切分为block，然后存储
  2. 与namenode交互，获取文件位置信息
  3. 提供命令管理hdfs
  4. 访问hdfs的命令


* Namenode
  1. 管理hdfs的名称空间
  2. 管理数据块映射信息
  3. 配置副本策略
  4. 处理客户端读写请求
* Datanode
  1. 存储数据块
  2. 执行数据读写操作
* Secondary Namenode
  1. 辅助namenode，分单工作量
  2. 定期合并fsimage和edits，推送给namenode
  3. 紧急情况恢复namenode

寻址时间计算与设置

```java
默认块大小配置dfs.blocksize
一般寻址时间占传入时间1%,寻址时间10ms
若传输100m/s 则块大小 size = 10ms*100/1000*100=100m
```

## hdfs文件写入流程

```java
1.客户端通过distribute filesystem向namenode请求上传文件,namenode检查目标文件是否存在
2.Namenode返回是否可以上传
3.客户端请求第一个block上传到哪几个datanode服务器
4.namenode返回3个datanode节点，为dn1,dn2,dn3
5.客户端通过FSDataOutputStream请求dn1上传数据,dn1收到请求调用dn2,dn2调用dn3将通信信道建立完成
6.dn1,dn2,dn3应答客户端
7.客户端上传block到dn1（先从磁盘读取数据放到一个本地内存缓存）dn1收到一个packet就会传给dn2，dn2传给dn3；dn1每传一个packet会放入一个应答队列等待应答。
8.当一个block传输完成之后，客户端再次请求NameNode上传第二个block的服务器。（重复执行3-7步）。
```

## HDFS读取流程

```java
1.客户端通过Distributed FileSystem向NameNode请求下载文件，NameNode通过查询元数据，找到文件块所在的DataNode地址
2.挑选一台DataNode（就近原则，然后随机）服务器，请求读取数据
3.DataNode开始传输数据给客户端（从磁盘里面读取数据输入流，以packet为单位来做校验）。
4.客户端以packet为单位接收，先在本地缓存，然后写入目标文件

```

副本节点选择，机架感知

```java
低版本Hadoop副本节点选择
    第一个副本在Client所处的节点上。如果客户端在集群外，随机选一个。
    第二个副本和第一个副本位于不相同机架的随机节点上。
    第三个副本和第二个副本位于相同机架，节点随机。
Hadoop2.7.2副本节点选择
	第一个副本在Client所处的节点上。如果客户端在集群外，随机选一个。
	第二个副本和第一个副本位于相同机架，随机节点。
	第三个副本位于不同机架，随机节点。

```

## NameNode和SecondaryNameNode

```java
namenode启动:
    1.首次启动格式化后创建fsimage和edits文件。如果不是第一次启动,直接加载编辑日志和镜像文件到内存
    2.客户端对元数据进行增删改请求
    3.namenode记录操作日志，更新滚动日志
    4.namenode在内存中对数据进行增删改查
Secondary NameNode工作
    1.secondary Namenode询问namenode是否checkpoint。直接带回namenode是否检查结果
    2.secondary namenode请求checkpoint
    3.namenode滚动正在写的edits日志
    4.将滚动前的编辑日志和镜像文件拷贝到Secondary NameNode。
    5.Secondary NameNode加载编辑日志和镜像文件到内存，并合并
    6.生成新的镜像文件fsimage.chkpoint
    7.拷贝fsimage.chkpoint到NameNode
    8.NameNode将fsimage.chkpoint重新命名成fsimage
```

namenode数据在/opt/module/hadoop-2.7.2/data/tmp/dfs/name/current目录产生文件如下

```java
edits_0000000000000000000
fsimage_0000000000000000000.md5
seen_txid
VERSION
Fsimage文件：HDFS文件系统元数据的一个永久性的检查点，其中包含HDFS文件系统的所有目录和文件idnode的序列化信息
Edits文件：存放HDFS文件系统的所有更新操作的路径，文件系统客户端执行的所有写操作首先会被记录到edits文件中
seen_txid文件：保存的是一个数字，就是最后一个edits_的数字
每次NameNode启动的时候都会将fsimage文件读入内存，并从00001开始到seen_txid中记录的数字依次执行每个edits里面的更新操作，保证内存中的元数据信息是最新的、同步的，可以看成NameNode启动的时候就将fsimage和edits文件进行了合并
```

查看fsimage文件

hdfs oiv -p XML -i
fsimage_0000000000000000025 -o /opt/module/hadoop-2.7.2/fsimage.xml

checkpoint时间设置:hdfs-default.xml

```java
<property>
  <name>dfs.namenode.checkpoint.period</name>
  <value>3600</value>
</property>
<!--一分钟检查一次操作次数，当操作次数达到1百万时，SecondaryNameNode执行一次-->
  <property>
  <name>dfs.namenode.checkpoint.txns</name>
  <value>1000000</value>
<description>操作动作次数</description>
</property>

<property>
  <name>dfs.namenode.checkpoint.check.period</name>
  <value>60</value>
<description> 1分钟检查一次操作次数</description>
</property>
```

namenode的安全模式

```java
namenode在启动的时候，首先将fsimage载入内存，并执行edits文件中的各项操作，当满足最小副本条件namenode在30s退出安全模式
（1）bin/hdfs dfsadmin -safemode get		（功能描述：查看安全模式状态）
（2）bin/hdfs dfsadmin -safemode enter  	（功能描述：进入安全模式状态）
（3）bin/hdfs dfsadmin -safemode leave	（功能描述：离开安全模式状态）
（4）bin/hdfs dfsadmin -safemode wait	（功能描述：等待安全模式状态）

```

## Datanode工作

工作机制

```java
1）一个数据块在DataNode上以文件形式存储在磁盘上，包括两个文件，一个是数据本身，一个是元数据包括数据块的长度，块数据的校验和，以及时间戳。
2）DataNode启动后向NameNode注册，通过后，周期性（1小时）的向NameNode上报所有的块信息。
3）心跳是每3秒一次，心跳返回结果带有NameNode给该DataNode的命令如复制块数据到另一台机器，或删除某个数据块。如果超过10分钟没有收到某个DataNode的心跳，则认为该节点不可用。
4）集群运行中可以安全加入和退出一些机器。
```

datanode根据校验和去验证block是否损坏，若损坏就读其它的datanode

datanode的超时时长设置 与配置

```java
timeout = 2*dfs.namenode.heartbeat.recheck-interval+10 * dfs.heartbeat.interval

配置：
<property>
    <name>dfs.namenode.heartbeat.recheck-interval</name>
    <value>300000</value>
</property>
<property>
    <name> dfs.heartbeat.interval </name>
    <value>3</value>
</property>
```

hdfs存储小文件弊端:

```java
1.会耗尽namenode内存
解决存储小文件办法之一
Hadoop存档文件或HAR文件，是一个更高效的文件存档工具，它将文件存入HDFS块，在减少NameNode内存使用的同时，允许对文件进行透明的访问
```

demo

```
 start-yarn.sh
 bin/hadoop archive -archiveName myhar.har -p /user/atguigu   /user/my
 hadoop fs -lsr /user/my/myhar.har
 hadoop fs -lsr har:///myhar.har
 hadoop fs -cp har:/// user/my/myhar.har /* /user/atguigu
```

快照管理

```
快照相当于对目录做一个备份。并不会立即复制所有文件，而是指向同一个文件。当写入发生时，才会产生新文件。
    （1）hdfs dfsadmin -allowSnapshot 路径   （功能描述：开启指定目录的快照功能）
	（2）hdfs dfsadmin -disallowSnapshot 路径 （功能描述：禁用指定目录的快照功能，默认是禁用）
	（3）hdfs dfs -createSnapshot 路径        （功能描述：对目录创建快照）
	（4）hdfs dfs -createSnapshot 路径 名称   （功能描述：指定名称创建快照）
	（5）hdfs dfs -renameSnapshot 路径 旧名称 新名称 （功能描述：重命名快照）
	（6）hdfs lsSnapshottableDir         	（功能描述：列出当前用户所有可快照目录）
	（7）hdfs snapshotDiff 路径1 路径2 	（功能描述：比较两个快照目录的不同之处）
	（8）hdfs dfs -deleteSnapshot <path> <snapshotName>  （功能描述：删除快照）
	
	
hdfs dfsadmin -allowSnapshot /user/atguigu/data
 hdfs dfs -createSnapshot /user/atguigu/data
 hdfs dfs -lsr /user/atguigu/data/.snapshot/ 
 hdfs dfs -createSnapshot /user/atguigu/data miao170508
 hdfs dfs -renameSnapshot /user/atguigu/data/ miao170508 atguigu170508
 hdfs ls SnapshottableDir
hdfs snapshotDiff /user/atguigu/data/  .  .snapshot/atguigu170508	
hdfs dfs -cp /user/atguigu/input/.snapshot/s20170708-134303.027 /user
```

回收站

```
默认值fs.trash.interval=0，0表示禁用回收站，可以设置删除文件的存活时间。
默认值fs.trash.checkpoint.interval=0，检查回收站的间隔时间。如果该值为0，则该值设置和fs.trash.interval的参数值相等。
要求fs.trash.checkpoint.interval<=fs.trash.interval
core-site.xml启用回收站
<property>
    <name>fs.trash.interval</name>
    <value>1</value>
</property>
进入回收站的用户名称
<property>
  <name>hadoop.http.staticuser.user</name>
  <value>atguigu</value>
</property>

```

 HDFS Federation架构设计

```java
1. namonode架构得局限性：
    单个namenode存储得对象受jvm Heap的限制,50G的heap能够存储20亿（200million）个对象，这20亿个对象支持4000个datanode，12PB的存储（假设文件平均大小为40MB）。随着数据的飞速增长，存储的需求也随之增长。单个datanode从4T增长到36T，集群的尺寸增长到8000个datanode。存储的需求从12PB增长到大于100PB。
2.隔离问题
    由于HDFS仅有一个namenode，无法隔离各个程序，因此HDFS上的一个实验程序就很有可能影响整个HDFS上运行的程序
3.性能的瓶颈
    由于是单个namenode的HDFS架构，因此整个HDFS文件系统的吞吐量受限于单个namenode的吞吐量。
    
```

