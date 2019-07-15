# HBase 学习纪要
当需要通过`FileSystem`去操作HDFS，且hdfs地址不在hdfs-site.xml中时，需要在Configuration类中配置`fs.defaultFS`指向目标HDFS  
例子：
```scala
val config = new Configuration()
config.set("fs.defaultFS","hdfs://cdh1:8020")
```
## Bulkload 相关
默认单个region加载的HFile不能超过32个，所以当切分完的HFile个数超过32的时候，会出现如下错误
```java
Exception in thread "main" java.lang.reflect.InvocationTargetException
at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
at java.lang.reflect.Method.invoke(Method.java:606)
at org.apache.hadoop.hbase.mapreduce.Driver.main(Driver.java:55)
Caused by: java.io.IOException: Trying to load more than 32 hfiles to one family of one region
at org.apache.hadoop.hbase.mapreduce.LoadIncrementalHFiles.doBulkLoad(LoadIncrementalHFiles.java:377)
at org.apache.hadoop.hbase.mapreduce.LoadIncrementalHFiles.run(LoadIncrementalHFiles.java:960)
at org.apache.hadoop.util.ToolRunner.run(ToolRunner.java:70)
at org.apache.hadoop.hbase.mapreduce.LoadIncrementalHFiles.main(LoadIncrementalHFiles.java:967)
```
 这个值可以通过在HBase的配置文件(hbase-site.xml)中添加如下配置来解决，也可以在代码中的HBaseConfiguration里进行配置
```xml
<property>
　　　<name>hbase.mapreduce.bulkload.max.hfiles.perRegion.perFamily</name>
　　　<value>3200</value>
</property>
```
```scala
val HBaseConf = HBaseConfiguration.create(config)
HBaseConf.set("hbase.mapreduce.bulkload.max.hfiles.perRegion.perFamily", "3200")
```
### Bulkload导入时的注意事项
写入的HFile文件内一定要**有序**,并且是升序，有序是指rowkey + columnFamily + qualifier三者有序，HFile存的数据本质是一个keyvalue
Spark生成HFile之前的数据排序代码
```scala
.sortBy(x => (new String(CellUtil.cloneRow(x._2)), new String(CellUtil.cloneFamily(x._2)),new String(CellUtil.cloneQualifier(x._2))))
```
#### 进阶，使用repartitionAndSortWithinPartitions
只使用sortby的确可以完成对KV的排序，但生成的HFile一般都会跨region，导致在load的过程中需要对HFile进行切割，使得一个HFile中的数据只属于一个region。这一步也会花费一定的时间。所以，切分的工作也可以在spark生成HFile这个阶段完成。就要使用到repartitionAndSortWithinPartitions算子。
此算子可以在数据shuffle阶段对数据进行排序，使得数据可以通过用户自定义的方式，让同种的数据落入同一个分区当中，并且分区内有序。   
例子 :  
需要定义一个分区器，告诉spark该如何对数据进行分区  
```scala
 /**
  * region分区器
  */
class RegionPartitioner(regionStartKey : Array[Array[Byte]]) extends Partitioner {
    //总共有多少个分区
    override def numPartitions: Int = regionStartKey.length
    //传入一条数据，返回一个分区序号，从0开始
    override def getPartition(key: Any): Int = {
        val value = key.asInstanceOf[KeyValue]
        for (i <- 1 until regionStartKey.length) {
            if (Bytes.compareTo(CellUtil.cloneRow(value), regionStartKey(i)) < 0) {
                return i - 1
            }
        }
        regionStartKey.length - 1
    }
}
```
自定义排序函数:   
repartitionAndSortWithinPartitions算子有一个排序的隐函数参数，可以通过创建隐函数来指定数据如何排序,此处是根据rowkey columnFamily qualifier排序升序排列 
```scala
//根据rowkey columnFamily qualifier排序
implicit val kv_ordering = new Ordering[KeyValue] {
    override def compare(x: KeyValue, y: KeyValue): Int = {
        val key = Bytes.compareTo(CellUtil.cloneRow(x),CellUtil.cloneRow(y))
        val family = Bytes.compareTo(CellUtil.cloneFamily(x),CellUtil.cloneFamily(y))
        val qualifier = Bytes.compareTo(CellUtil.cloneQualifier(x),CellUtil.cloneQualifier(y))
        if(key == 0) {
            if(family == 0){
                qualifier
            }else {
                family
            }
        } else {
          key
        }
    }
}
```
最后使用该算子对数据进行排序即可
```scala
.repartitionAndSortWithinPartitions(regionPartitioner).map(x => (x._2,x._1))
```

### Bulkload的时候要多关注HDFS容量
Bulkload的时候数据大小会膨胀，所以load到HDFS上的时候要关注HDFS容量是否足够
另外，如果线上HDFS与HBase的**副本数**是2，在load HFile到HDFS上的时候也可以将副本数降为2，避免HFile占用过多的HDFS
### Bulkload在load HFile时候会对文件进行SPLIT
当HFile中的数据有跨region的情况存在的时候，会对HFile进行分裂，如果导入表是一个snappy压缩的表，这时候Bulkload会调用客户端本地的snappy对HFile进行分裂，如果客户端本地的snappy native库有问题，**或者java运行的时候没有指定java.library.path，或者那个路径里面没有snappy的so文件**，都会报`java.lang.UnsatisfiedLinkError org.apache.hadoop.util.NativeCodeLoader.buildSupportsSnappy()Z`
所以，通过在运行java程序的时候加上-Djava.library.path来指定so文件位置之外
# HBase 迁移数据
## 使用snapshot进行数据迁移
1、先在源HBase集群上将需要迁移的表进行一次snapshot `snapshot "src_table","snapshot_name"`
2、使用HBase自带的工具进行snapshot的迁移，`hbase org.apache.hadoop.hbase.snapshot.ExportSnapshot -snapshot test_table_snapshot -copy-from hdfs://src-cluster:8020/hbase -copy-to hdfs://dist-cluster:8080/hbase -mappers n`,此命令会将生成的snapshot通过mapreduce任务，把数据传输到目标集群的hbase的HDFS的相关的目录(数据会写入/hbase/archive/data/)
3、在目标集群的hbase shell内，先disable需要装载的表，再使用`restore_snapshot "snapshot_name","dist_table"`,再将表enable
4、如果没有建表，也可以使用clone_snapshot命令将snapshot恢复成一张表`clone_snapshot "snapshot_name","dist_table"`
> **注意** 使用clone_snapshot恢复的表，其实在/hbase/data/default/dist_table/只是生成了引用数据文件(在/hbase/data/archive内)的一个符号链接，并没有真正的将数据文件移进表目录下，需要经过一次major_compact才会将数据文件compact之后写入表的文件夹内