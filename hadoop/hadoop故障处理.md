# Hadoop故障处理

* nameNode故障处理

```java
方法一：将SecondaryNameNode中数据拷贝到NameNode存储数据的目录；
scp -r atguigu@hadoop104:/opt/module/hadoop-2.7.2/data/tmp/dfs/namesecondary/* ./name/
sbin/hadoop-daemon.sh start namenode
方法二：使用-importCheckpoint选项启动NameNode守护进程，从而将SecondaryNameNode中数据拷贝到NameNode目录中。
```

