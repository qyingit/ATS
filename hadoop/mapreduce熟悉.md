# Mapreduce熟悉

* 定义：数据分析框架

### 优缺点

* 优点
  1. MapReduce 易于编程
  2. 良好的扩展性,通过增加机器
  3. 高容错性,一台机器挂了,可以通过转移计算任务,防止程序运行失败
  4. 适合PB级以上海量数据的离线处理,不能实时处理
* 缺点
  1. MapReduce不擅长做实时计算、流式计算、DAG（有向图）计算
  2. 不能实时计算
  3. 不能基于流计算
  4. 不能做DAG有向图计算,只能写到磁盘

