# 数仓

## 系统数据流程设计

![image-20200316215652461](assets/image-20200316215652461.png)

## HDFS参数调优

1、dfs.namenode.handler.count = 20*log2(cluster size)

> NameNode有一个工作线程池，用来处理不同DataNode的并发心跳以及客户端并发的元数据操作。对于大集群或者有大量客户端的集群来说，通常需要增大参数dfs.namenode.handler.count,其默认值为10.设置该值的一般原则是将其设置为集群大小的自然对数乘以20，即20log(N),N为集群大小

2、编辑日志存储路径dfs.namenode.edits.dir设置与镜像文件存储路径dfs.namenode.name.dir尽量分开，达到最低写入延迟。

## Flume HDFS小文件处理

官方默认的这三个参数配置写入HDFS后会产生小文件，hdfs.rollInterval、hdfs.rollSize、hdfs.rollCount  

基于以上hdfs.rollInterval=3600, hdfs.rollSize=134217728, hdfs.rollCount=0, hdfs.roundValue=10, hdfs.roundUnit=second几个参数综合作用，效果如下：  

1. tmp文件在达到128M时会滚动生成正式文件

2. tmp文件创建超过10秒时会滚动生成正式文件

举例：在2018-01-01 05:23的时候sink接收到数据，那会产生如下tmp文件：/path/to/save/201801010520.tmp

即使文件内容没有达到128M，也会在05:33时滚动生成正式文件  

# 数仓概念

## 数仓为什么要分层

![image-20200318171044316](assets/image-20200318171044316.png)  

**1、把复杂的问题简单化**  

将一个复杂的任务分解成多个步骤来完成，每一层只处理单一的步骤，比较简单、并且方便定位问题。  

**2、减少重复开发**  

规范数据分层，通过中间层数据，能够减少极大的重复计算，增加一次计算结果的复用性。  

**3、隔离原始数据**  

不论是数据的异常还是数据的敏感性，使真实数据与统计数据解耦开  

## 分层

![image-20200318195624096](assets/image-20200318195624096.png)  

![image-20200318200409281](assets/image-20200318200409281.png)

## 数据集市的概念

## 数据集市与数据仓库的区别

**数据集市(Data Market) 是一种微型的数据仓库**，它通常有更少的数据，更少的主题区域，以及更少的历史数据，因此是**部门级**的，一般只能为某个局部范围内的管理人员服务。  

**数据仓库是企业级的**，能为整个企业各部门的运行提供决策支持手段。  

## 数仓命名规范

ODS层命名为ods  

DWD层命名为dwd  

DWS层命名为dws  

ADS层命名为ads  

临时表数据库命名为xxx_tmp  

备份数据数据库命名为xxx_bak  

