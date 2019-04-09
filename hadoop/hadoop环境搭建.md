# 环境搭建

####  本地模式搭建虚拟机

1. 克隆虚拟机

```java
1. vim /etc/udev/rules.d/70-persistent-net.rules
   进入如下页面，删除eth0该行；将eth1修改为eth0，同时复制物理ip地址
2. vim /etc/sysconfig/network-scripts/ifcfg-eth0
   HWADDR=00:0C:2x:6x:0x:xx   #MAC地址 
   IPADDR=192.168.1.101      #IP地址
   ONBOOT=yes #系统启动的时候网络接口是否有(yes/no)
   BOOTPROTO=static 
   GATEWAY=192.168.1.2 #网关 
   DNS1=192.168.1.2 #域名解析器
   然后
   service network restart
3. 修改主机名称
	vi /etc/sysconfig/network
      NETWORKING=yes
      NETWORKING_IPV6=no
      HOSTNAME= hadoop100
    vim /etc/hosts
      192.168.1.101 qying1
      192.168.1.102 qying2
      192.168.1.103 qying3
      192.168.1.104 qying4
	进入C:\Windows\System32\drivers\etc路径
	修改host文件
	
关闭防火墙 iptables:
  service  服务名 start（功能描述：开启服务）
  service  服务名 stop（功能描述：关闭服务）
  service  服务名 restart（功能描述：重新启动服务）
  service  服务名 status（功能描述：查看服务状态）
  chkconfig 设置后台服务的自启配置
  chkconfig iptables off
  chkconfig 服务名 on
  chkconfig 服务名 --list	
  
添加用户操作
  useradd tangseng
  passwd tangseng
  删除
  userdel  用户名
  userdel -r 用户名
vi /etc/sudoers
  root    ALL=(ALL)     ALL
  atguigu   ALL=(ALL)     ALL
  sudo mkdir module
  chown atguigu:atguigu module/
  加入用户到指定组
  usermod -g root zhubajie
重新启动服务器
安装jdk
   卸载
		rpm  -qa | grep java  查询jdk
        rpm -e 软件包 卸载jdk
   解压
    tar -zxvf jdk-8u144-linux-x64.tar.gz -C 	/opt/module/
    sudo vi /etc/profile 
    #JAVA_HOME
    export JAVA_HOME=/opt/module/jdk1.8.0_144
    export PATH=PATH:JAVA_HOME/bin
    source /etc/profile
				
```

hadoop配置

同步工具

1 cd bin/   2 touch  xsync  3 vi xsync   4 chmod  777  xsync

```c
#!/bin/bash
#1 获取输入参数个数，如果没有参数，直接退出
pcount=$#
if((pcount==0)); then
echo no args;
exit;
fi

#2 获取文件名称
p1=$1
fname=`basename $p1`
echo fname=$fname

#3 获取上级目录到绝对路径
pdir=`cd -P $(dirname $p1); pwd`
echo pdir=$pdir

#4 获取当前用户名称
user=`whoami`

#5 循环同步
for((host=1; host<4; host++)); do
        echo --------------------- qying$host ----------------
        rsync -rvl $pdir/$fname $user@qying$host:$pdir
done

```



```java
安装hadoop
   	tar -zxvf hadoop-2.7.2.tar.gz -C /opt/module/
   	vi /etc/profile
   	##HADOOP_HOME
    export HADOOP_HOME=/opt/module/hadoop-2.7.2
    export PATH=$PATH:$HADOOP_HOME/bin
    export PATH=$PATH:$HADOOP_HOME/sbin
配置集群：
   hadoop-env.sh
     export JAVA_HOME=/opt/module/jdk1.8.0_144
   core-site.xml
      <!-- 指定HDFS中NameNode的地址 -->
      <property>
          <name>fs.defaultFS</name>
          <value>hdfs://hadoop101:9000</value>
      </property>

      <!-- 指定hadoop运行时产生文件的存储目录 -->
      <property>
          <name>hadoop.tmp.dir</name>
          <value>/opt/module/hadoop-2.7.2/data/tmp</value>
      </property>
  hdfs-site.xml    
   	  <!-- 指定HDFS副本的数量 -->
      <property>
          <name>dfs.replication</name>
          <value>1</value>
      </property>
  yarn-env.sh
      export JAVA_HOME=/opt/module/jdk1.8.0_144
  yarn-site.xml
      <!-- reducer获取数据的方式 -->
      <property>
       <name>yarn.nodemanager.aux-services</name>
       <value>mapreduce_shuffle</value>
      </property>

      <!-- 指定YARN的ResourceManager的地址 -->
      <property>
      <name>yarn.resourcemanager.hostname</name>
      <value>hadoop101</value>
      </property>
      <!-- 日志聚集功能使能 -->
      <property>
      <name>yarn.log-aggregation-enable</name>
      <value>true</value>
      </property>
      <!-- 日志保留时间设置7天 -->
      <property>
      <name>yarn.log-aggregation.retain-seconds</name>
      <value>604800</value>
      </property>

  mapred-env.sh
      export JAVA_HOME=/opt/module/jdk1.8.0_144
  mv mapred-site.xml.template mapred-site.xml #复制
  vi mapred-site.xml
      <!-- 指定mr运行在yarn上 -->
      <property>
          <name>mapreduce.framework.name</name>
          <value>yarn</value>
      </property>
      <!-- 配置历史服务器 -->
	  <property>
		<name>mapreduce.jobhistory.address</name>
      <value>hadoop101:10020</value>
      </property>
      <property> <name>mapreduce.jobhistory.webapp.address</name>
          <value>hadoop101:19888</value>
      </property>		
```

启动hadoop

```java
bin/hdfs  namenode -format
hdfs启动
sbin/hadoop-daemon.sh start namenode
sbin/hadoop-daemon.sh start datanode
创建文件夹
bin/hdfs dfs -mkdir -p /user/atguigu/input
删除文件夹
hdfs dfs -rm -r /user/atguigu/output
下载文件到本地
hadoop fs -get /user/atguigu/ output/part-r-00000 ./wcoutput/
http://192.168.1.101:50070/dfshealth.html查看
上传文件
bin/hdfs dfs -put wcinput/wc.input /user/atguigu/input/
yarn启动
sbin/yarn-daemon.sh start resourcemanager
sbin/yarn-daemon.sh start nodemanager
http://192.168.1.101:8088/cluster查看
历史服务器启动
目录查看
ls sbin/ | grep mr mr-jobhistory-daemon.sh
sbin/mr-jobhistory-daemon.sh start historyserver
http://192.168.1.101:19888/jobhistory
以上关闭需要将start变为stop
```

默认配置文件位置：

```java
[core-default.xml]
		hadoop-common-2.7.2.jar/ core-default.xml
[hdfs-default.xml]
		hadoop-hdfs-2.7.2.jar/ hdfs-default.xml
[yarn-default.xml]
		hadoop-yarn-common-2.7.2.jar/ yarn-default.xml
[mapred-default.xml]
		hadoop-mapreduce-client-core-2.7.2.jar/ mapred-default.xml

```



集群配置

```
hdfs-site.xml
    <property>
		<name>dfs.replication</name>
		<value>3</value>
	</property>

	<property>
        <name>dfs.namenode.secondary.http-address</name>
        <value>hadoop104:50090</value>
    </property>
yarn-site.xml
    <!-- reducer获取数据的方式 -->
	<property>
		 <name>yarn.nodemanager.aux-services</name>
		 <value>mapreduce_shuffle</value>
	</property>

	<!-- 指定YARN的ResourceManager的地址 -->
	<property>
		<name>yarn.resourcemanager.hostname</name>
		<value>hadoop103</value>
	</property>
mapred-site.xml
    <!-- 指定mr运行在yarn上 -->
	<property>
		<name>mapreduce.framework.name</name>
		<value>yarn</value>
	</property>
xsync /opt/module/hadoop-2.7.2/ 分发配置
hadoop namenode -format
配置slaves
vi slaves
    hadoop102
    hadoop103
    hadoop104
启动方式
start-dfs.sh
start-yarn.sh 

start-all.sh
集群时间同步
rpm -qa|grep ntp
vi /etc/ntp.conf
restrict 192.168.1.0 mask 255.255.255.0 nomodify notrap
修改2（设置为不采用公共的服务器）
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst为
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
添加3（添加默认的一个内部时钟数据，使用它为局域网用户提供服务。）
server 127.127.1.0
fudge 127.127.1.0 stratum 10
）修改/etc/sysconfig/ntpd 文件
vim /etc/sysconfig/ntpd
增加内容如下（让硬件时间与系统时间一起同步）
SYNC_HWCLOCK=yes
service ntpd status
service ntpd start
chkconfig ntpd on
crontab -e
*/10 * * * * /usr/sbin/ntpdate hadoop102
```

SSH无秘登陆

```
cd ~
ssh-keygen -t rsa 
ssh-copy-id hadoop102
```

#### 添加新节点

```java
1.添加host
2.修改hdfs-site.xml
    <property>
      <name>dfs.hosts</name>
      <value>/opt/module/hadoop-2.7.2/etc/hadoop/dfs.hosts</value>
     </property>
3.刷新namenode 
    hdfs dfsadmin -refreshNodes
4.更新resourcemanager节点
    yarn rmadmin -refreshNodes
5.在NameNode的slaves文件中增加新主机名称
    hadoop105
6. 单独命令启动新的数据节点和节点管理器
 sbin/hadoop-daemon.sh start datanode
 sbin/yarn-daemon.sh start nodemanager
7.在web浏览器上检查是否ok
/start-balancer.sh实现集群数据的平衡
```

#### 退役旧数据节点

```java
1. touch dfs.hosts.exclud
2.vi dfs.hosts.exclude
    添加如下主机名称  hadoop105
3.在namenode的的hdfs-site.xml配置文件中增加dfs.hosts.exclude属性
    <property>
      <name>dfs.hosts.exclude</name>
      <value>/opt/module/hadoop-2.7.2/etc/hadoop/dfs.hosts.exclude</value>
    </property>
4.刷新namenode、刷新resourcemanager
    hdfs dfsadmin -refreshNodes
    yarn rmadmin -refreshNodes
5.检查web浏览器，退役节点的状态为decommission in progress（退役中），说明数据节点正在复制块到其他节点
6.等待退役节点状态为decommissioned（所有块已经复制完成），停止该节点及节点资源管理器。注意：如果副本数是3，服役的节点小于等于3，是不能退役成功的，需要修改副本数后才能退役
7.从include文件中删除退役节点，再运行刷新节点的命令
    从namenode的dfs.hosts文件中删除退役节点hadoop105
    刷新namenode，刷新resourcemanager
    hdfs dfsadmin -refreshNodes
    yarn rmadmin -refreshNodes
8.从namenode的slave文件中删除退役节点hadoop105
使用sbin/start-balancer.sh平衡数据
```

namenode多目录配置 hdfs-site.xml

```
<property>
    <name>dfs.namenode.name.dir</name>
<value>file:///${hadoop.tmp.dir}/dfs/name1,file:///${hadoop.tmp.dir}/dfs/name2</value>
</property>
```

datanode多目录配置

```java
<property>
        <name>dfs.datanode.data.dir</name>
<value>file:///${hadoop.tmp.dir}/dfs/data1,file:///${hadoop.tmp.dir}/dfs/data2</value>
</property>
```

数据拷贝

```java
scp数据的集群间拷贝
scp -r hello.txt root@hadoop103:/user/atguigu/hello.txt	// 推 push
scp -r root@hadoop103:/user/atguigu/hello.txt  hello.txt // 拉 pull
discp命令实现两个hadoop集群之间的递归数据复制
bin/hadoop distcp hdfs://haoop102:9000/user/atguigu/hello.txt hdfs://hadoop103:9000/user/atguigu/hello.txt
```

高可用

```java
原理：
通过双namenode消除单点故障
1.故障检测：namenode在zk中维护了一个持久会话，如果机器崩溃，zk会话终止,通知另一个namenode触发故障转移
2.现役Namenode选择：namenode崩溃,另一个节点从zk获得特殊的排外锁表明它应该成为现役的namenode
ZKFC：一个自动故障转移的组件，职责为:
1.健康监测：ZKFC使用一个健康检查命令定期地ping与之在相同主机的NameNode，只要该NameNode及时地回复健康状态，ZKFC认为该节点是健康的。如果该节点崩溃，冻结或进入不健康状态，健康监测器标识该节点为非健康的。
2.ZooKeeper会话管理：当本地NameNode是健康的，ZKFC保持一个在ZooKeeper中打开的会话。如果本地NameNode处于active状态，ZKFC也保持一个特殊的znode锁，该锁使用了ZooKeeper对短暂节点的支持，如果会话终止，锁节点将自动删除。
3.基于ZooKeeper的选择：如果本地NameNode是健康的，且ZKFC发现没有其它的节点当前持有znode锁，它将为自己获取该锁。如果成功，则它已经赢得了选择，并负责运行故障转移进程以使它的本地NameNode为active。故障转移进程与前面描述的手动故障转移相似，首先如果必要保护之前的现役NameNode，然后本地NameNode转换为active状态
```

配置ZK

```java
 1. 解压安装
     tar -zxvf zookeeper-3.4.10.tar.gz -C /opt/module/
     mkdir -p zkData
     mv zoo_sample.cfg zoo.cfg
 2. 配置zoo.cfg文件
     dataDir=/opt/module/zookeeper-3.4.10/zkData
     #######################cluster##########################
server.2=hadoop102:2888:3888
server.3=hadoop103:2888:3888
server.4=hadoop104:2888:3888
Server.A=B:C:D。
A是一个数字，表示这个是第几号服务器；
B是这个服务器的ip地址；
C是这个服务器与集群中的Leader服务器交换信息的端口；
D是万一集群中的Leader服务器挂了，需要一个端口来重新进行选举，选出一个新的Leader，而这个端口就是用来执行选举时服务器相互通信的端口。
3.myid文件 	
    在zkData创建myid   touch myid
    编辑myid文件  vi myid 添加与server对应的编号：如2
    集群模式下配置一个文件myid，这个文件在dataDir目录下，这个文件里面有一个数据就是A的值，Zookeeper启动时读取此文件，拿到里面的数据与zoo.cfg里面的配置信息比较从而判断到底是哪个server
4.拷贝zk  scp -r zookeeper-3.4.10/ root@hadoop103.atguigu.com:/opt/app/
5.分别启动zookeeper
    bin/zkServer.sh start
    bin/zkServer.sh status
```

配置hdfs

```java
1.在opt目录下创建一个ha文件夹
mkdir HA
2.将/opt/app/下的 hadoop-2.7.2拷贝到/opt/ha目录下
cp -r hadoop-2.7.2/ /opt/HA/
3.配置hadoop-env.sh
export JAVA_HOME=/opt/module/jdk1.8.0_144
4.配置core-site.xml
    <configuration>
<!-- 把两个NameNode）的地址组装成一个集群mycluster -->
		<property>
			<name>fs.defaultFS</name>
        	<value>hdfs://mycluster</value>
		</property>

		<!-- 指定hadoop运行时产生文件的存储目录 -->
		<property>
			<name>hadoop.tmp.dir</name>
			<value>/opt/module/HA/hadoop-2.7.2/data/tmp</value>
		</property>
    </configuration>
5.配置hdfs-site.xml
    <configuration>
	<!-- 完全分布式集群名称 -->
	<property>
		<name>dfs.nameservices</name>
		<value>mycluster</value>
	</property>

	<!-- 集群中NameNode节点都有哪些 -->
	<property>
		<name>dfs.ha.namenodes.mycluster</name>
		<value>nn1,nn2</value>
	</property>

	<!-- nn1的RPC通信地址 -->
	<property>
		<name>dfs.namenode.rpc-address.mycluster.nn1</name>
		<value>hadoop102:9000</value>
	</property>

	<!-- nn2的RPC通信地址 -->
	<property>
		<name>dfs.namenode.rpc-address.mycluster.nn2</name>
		<value>hadoop103:9000</value>
	</property>

	<!-- nn1的http通信地址 -->
	<property>
		<name>dfs.namenode.http-address.mycluster.nn1</name>
		<value>hadoop102:50070</value>
	</property>

	<!-- nn2的http通信地址 -->
	<property>
		<name>dfs.namenode.http-address.mycluster.nn2</name>
		<value>hadoop103:50070</value>
	</property>

	<!-- 指定JournalNode为那个集群提供服务 -->
	<property>
		<name>dfs.namenode.shared.edits.dir</name>
	<value>qjournal://hadoop102:8485;hadoop103:8485;hadoop104:8485/mycluster</value>
	</property>

	<!-- 配置隔离机制，即同一时刻只能有一台服务器对外响应 -->
	<property>
		<name>dfs.ha.fencing.methods</name>
		<value>sshfence</value>
	</property>

	<!-- 使用隔离机制时需要ssh无秘钥登录-->
	<property>
		<name>dfs.ha.fencing.ssh.private-key-files</name>
		<value>/home/atguigu/.ssh/id_rsa</value>
	</property>

	<!-- 声明journalnode服务器存储目录-->
	<property>
		<name>dfs.journalnode.edits.dir</name>
		<value>/opt/moduel/HA/hadoop-2.7.2/data/jn</value>
	</property>

	<!-- 关闭权限检查-->
	<property>
		<name>dfs.permissions.enable</name>
		<value>false</value>
	</property>

	<!-- 访问代理类：client，mycluster，active配置失败自动切换实现方式-->
	<property>		<name>dfs.client.failover.proxy.provider.mycluster</name>
<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
	</property>
</configuration>
6.拷贝配置好的hadoop环境到其他节点
7.在各个JournalNode节点上，输入以下命令启动journalnode服务
    sbin/hadoop-daemon.sh start journalnode
8.在[nn1]上，对其进行格式化，并启动
    bin/hdfs namenode -format
	sbin/hadoop-daemon.sh start namenode
9.在[nn2]上，同步nn1的元数据信息
    bin/hdfs namenode -bootstrapStandby
10.启动[nn2]
    sbin/hadoop-daemon.sh start namenode
11.在[nn1]上，启动所有datanode
    sbin/hadoop-daemons.sh start datanode
12.将[nn1]切换为Active
    bin/hdfs haadmin -transitionToActive nn1
13.查看是否Active
    bin/hdfs haadmin -getServiceState nn1
```

配置HDFS-HA自动故障转移

```java
1.hdfs-site.xml中增加
    <property>
        <name>dfs.ha.automatic-failover.enabled</name>
        <value>true</value>
    </property>
 2.在core-site.xml文件中增加
    <property>
      <name>ha.zookeeper.quorum</name>     <value>hadoop102:2181,hadoop103:2181,hadoop104:2181</value>
      </property>

```

```
启动
	（1）关闭所有HDFS服务：
		sbin/stop-dfs.sh
	（2）启动Zookeeper集群：
		bin/zkServer.sh start
	（3）初始化HA在Zookeeper中状态：
		bin/hdfs zkfc -formatZK
	（4）启动HDFS服务：
		sbin/start-dfs.sh
（5）在各个NameNode节点上启动DFSZK Failover Controller，先在哪台机器启动，哪个机器的NameNode就是Active NameNode
		sbin/hadoop-daemin.sh start zkfc
3）验证
	（1）将Active NameNode进程kill
		kill -9 namenode的进程id
	（2）将Active NameNode机器断开网络
		service network stop

```

yarn集群

```java
具体配置
（1）yarn-site.xml
<configuration>

    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>

    <!--启用resourcemanager ha-->
    <property>
        <name>yarn.resourcemanager.ha.enabled</name>
        <value>true</value>
    </property>
 
    <!--声明两台resourcemanager的地址-->
    <property>
        <name>yarn.resourcemanager.cluster-id</name>
        <value>cluster-yarn1</value>
    </property>

    <property>
        <name>yarn.resourcemanager.ha.rm-ids</name>
        <value>rm1,rm2</value>
    </property>

    <property>
        <name>yarn.resourcemanager.hostname.rm1</name>
        <value>hadoop102</value>
    </property>

    <property>
        <name>yarn.resourcemanager.hostname.rm2</name>
        <value>hadoop103</value>
    </property>
 
    <!--指定zookeeper集群的地址--> 
    <property>
        <name>yarn.resourcemanager.zk-address</name>
        <value>hadoop102:2181,hadoop103:2181,hadoop104:2181</value>
    </property>

    <!--启用自动恢复--> 
    <property>
        <name>yarn.resourcemanager.recovery.enabled</name>
        <value>true</value>
    </property>
 
    <!--指定resourcemanager的状态信息存储在zookeeper集群--> 
    <property>
        <name>yarn.resourcemanager.store.class</name>     <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
</property>

</configuration>
	（2）同步更新其他节点的配置信息
3）启动hdfs 
（1）在各个JournalNode节点上，输入以下命令启动journalnode服务：
	sbin/hadoop-daemon.sh start journalnode
（2）在[nn1]上，对其进行格式化，并启动：
	bin/hdfs namenode -format
	sbin/hadoop-daemon.sh start namenode
（3）在[nn2]上，同步nn1的元数据信息：
	bin/hdfs namenode -bootstrapStandby
（4）启动[nn2]：
	sbin/hadoop-daemon.sh start namenode
（5）启动所有datanode
	sbin/hadoop-daemons.sh start datanode
（6）将[nn1]切换为Active
	bin/hdfs haadmin -transitionToActive nn1
4）启动yarn 
（1）在hadoop102中执行：
sbin/start-yarn.sh
（2）在hadoop103中执行：
sbin/yarn-daemon.sh start resourcemanager
（3）查看服务状态
bin/yarn rmadmin -getServiceState rm1

```

