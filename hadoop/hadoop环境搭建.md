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

