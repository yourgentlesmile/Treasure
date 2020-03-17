# 数仓

## 系统数据流程设计

![image-20200316215652461](assets/image-20200316215652461.png)

## HDFS参数调优

1、dfs.namenode.handler.count = 20*log2(cluster size)

> NameNode有一个工作线程池，用来处理不同DataNode的并发心跳以及客户端并发的元数据操作。对于大集群或者有大量客户端的集群来说，通常需要增大参数dfs.namenode.handler.count,其默认值为10.设置该值的一般原则是将其设置为集群大小的自然对数乘以20，即20log(N),N为集群大小

2、编辑日志存储路径dfs.namenode.edits.dir设置与镜像文件存储路径dfs.namenode.name.dir尽量分开，达到最低写入延迟。

