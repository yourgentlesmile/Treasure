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
### Bulkload时的副本数
在Bulkload的时候使用与HBase设置的副本数相同的副本数，因为如果Bulkload时指定的副本数为1，一旦服务器磁盘出现问题，数据将会丢失。在Compaction之前，load上去的HFile都是1个副本。  
HDFS手动增加文件副本数`hadoop dfs -setrep -w 3 -R /path-to-set`
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
# HBase2.x中的修复工具 HBCK2
github地址：[HBCK2](https://github.com/apache/hbase-operator-tools/)
## 基本命令解析
### bypass [option] &#60;pid&#62;
HBCK2的核心功能，bypass可以将一个或多个卡住的procedure进行释放。
原理很简单，在procedure的类里有一个bypass的flag, 每次执行时会检查这个flag是否为true，如果为true则直接返回null, 这样procedure就会被认为执行成功。

而我们的bypass就是把这个procedure对象中的这个flag设为true。 这样stuck的procedure就能够不再执行,后续的修复工作才能继续。

返回值为true则是成功，false是失败。
参数解析：
```
-o,--overide
```
在执行bypass之前先会尝试去拿IdLock, 如果procedure还在运行就会超时返回null，但是设置了这个参数即使拿不到IdLock也会去将procedure的bypass flag设为true。
```
-r, --recursive  
```
在bypass一个procedure时也会将这个procedure的所有子procedure进行递归的bypass。例如我们bypass一个对table schema修改的procedure, 就需要加上-r参数，才能把这个操作的所有子procedure都bypass掉。
```
-w, --lockWait
```
上文提到的等待IdLock的超时时间配置，默认为1ms
### assigns [option] &#60;encoded_regionname&#62;
将一个或多个region再次随机assign到别的机器上，返回值是创建的pid则为成功，-1则为失败。
参数解析：
```
-o,--override
```
这里的override跟bypass的override不同，因为assign本身就会创建一个新的procedure, 所以肯定是不涉及到拿IdLock的，但是这里涉及到资源锁的问题。因为之前卡住的资源锁即使在bypass后也不会释放(用于fence, 防止更多未知的错误操作)，所以需要加一个-o去手动释放这个资源锁。
### unassigns [option]  &#60;encoded_regionname&#62;
将一个或多个region unassign，返回值是创建的pid则为成功，-1则为失败。
参数解析：
```
-o,--override，与assigns的一致
```
### setTableState &#60;table&#62; &#60;state&#62;
可能的table状态, ENABLED, DISABLED, DISABLING, ENABLING

在table的状态和所有的region状态不一致时可以用这个命令进行修复
### serverCrashProcedures &#60;serverName&#62;
手动schedule一个或多个serverCrashProcedure, 如果有serverCrashProcedure没有执行成功，但是procedure log已经丢失了，那么可以利用这个命令进行修复。返回值为创建的pid则为成功，-1则为失败。

patch在HBASE-21393[3]，目前这个功能在release版本还没有。
### hbck2 原生帮助文档
```text
Options:
 -d,--debug                                       run with debug output
 -h,--help                                        output this help message
 -p,--hbase.zookeeper.property.clientPort <arg>   port of target hbase
                                                  ensemble
 -q,--hbase.zookeeper.quorum <arg>                ensemble of target hbase
 -s,--skip                                        skip hbase version check
 -v,--version                                     this hbck2 version
 -z,--zookeeper.znode.parent <arg>                parent znode of target
                                                  hbase

Commands:
 assigns [OPTIONS] <ENCODED_REGIONNAME>...
   Options:
    -o,--override  override ownership by another procedure
   A 'raw' assign that can be used even during Master initialization.
   Skirts Coprocessors. Pass one or more encoded RegionNames.
   1588230740 is the hard-coded name for the hbase:meta region and
   de00010733901a05f5a2a3a382e27dd4 is an example of what a user-space
   encoded Region name looks like. For example:
     $ HBCK2 assign 1588230740 de00010733901a05f5a2a3a382e27dd4
   Returns the pid(s) of the created AssignProcedure(s) or -1 if none.

 bypass [OPTIONS] <PID>...
   Options:
    -o,--override   override if procedure is running/stuck
    -r,--recursive  bypass parent and its children. SLOW! EXPENSIVE!
    -w,--lockWait   milliseconds to wait on lock before giving up;
default=1
   Pass one (or more) procedure 'pid's to skip to procedure finish.
   Parent of bypassed procedure will also be skipped to the finish.
   Entities will be left in an inconsistent state and will require
   manual fixup. May need Master restart to clear locks still held.
   Bypass fails if procedure has children. Add 'recursive' if all
   you have is a parent pid to finish parent and children. This
   is SLOW, and dangerous so use selectively. Does not always work.

 unassigns <ENCODED_REGIONNAME>...
   Options:
    -o,--override  override ownership by another procedure
   A 'raw' unassign that can be used even during Master initialization.
   Skirts Coprocessors. Pass one or more encoded RegionNames:
   1588230740 is the hard-coded name for the hbase:meta region and
   de00010733901a05f5a2a3a382e27dd4 is an example of what a user-space
   encoded Region name looks like. For example:
     $ HBCK2 unassign 1588230740 de00010733901a05f5a2a3a382e27dd4
   Returns the pid(s) of the created UnassignProcedure(s) or -1 if none.

 setTableState <TABLENAME> <STATE>
   Possible table states: ENABLED, DISABLED, DISABLING, ENABLING
   To read current table state, in the hbase shell run:
     hbase> get 'hbase:meta', '<TABLENAME>', 'table:state'
   A value of \x08\x00 == ENABLED, \x08\x01 == DISABLED, etc.
   An example making table name 'user' ENABLED:
     $ HBCK2 setTableState users ENABLED
   Returns whatever the previous table state was.

 setRegionState <ENCODED_REGIONNAME> <STATE>
   Possible region states: OFFLINE, OPENING, OPEN, CLOSING, CLOSED,
SPLITTING, SPLIT, FAILED_OPEN, FAILED_CLOSE, MERGING, MERGED,
SPLITTING_NEW, MERGING_NEW
   WARNING: This is a very risky option intended for use as last resort.
    Example scenarios are when unassigns/assigns can't move forward
     due to region being in an inconsistent state in META. For example,
     'unassigns' command can only proceed
      if passed in region is in one of following states:
                [SPLITTING|SPLIT|MERGING|OPEN|CLOSING]
   Before manually setting a region state with this command,
   please certify that this region is not being handled by
   by a running procedure, such as Assign or Split. You can get a view of
   running procedures from hbase shell, using 'list_procedures' command.
   An example setting region 'de00010733901a05f5a2a3a382e27dd4' to
CLOSING:
     $ HBCK2 setRegionState de00010733901a05f5a2a3a382e27dd4 CLOSING
   Returns "0" SUCCESS code if it informed region state is changed, "1"
FAIL code otherwise.
```
## Crash场景
### 场景1：有些表的region没有deploy到任何一台regionserver上
通过hbase hbck检查发现有如下错误  
```
ERROR: Region { meta => complexnetwork:schema_si,,1562575787949.622e99b7eb0ac2c4ba6a4d645d4470011, deployed => , replicaId => 0 } not deployed on any region server.
ERROR: Region { meta => complexnetwork:vl,,1562575788707.891b0acf2dc392736c2c355d92b26780., hd=> , replicaId => 0 } not deployed on any region server.
ERROR: Region { meta => complexnetwork:el,,1562575786625.829a9bc159f7627e748b4d4aade744c1., hd=> , replicaId => 0 } not deployed on any region server.
19/07/16 14:19:27 INFO util.HBaseFsck: Handling overlap merges in parallel. set hbasefsck.over
ERROR: There is a hole in the region chain between  and .  You need to create a new .regioninf
ERROR: Found inconsistency in table complexnetwork:schema_si
ERROR: There is a hole in the region chain between  and .  You need to create a new .regioninf
ERROR: Found inconsistency in table complexnetwork:el
ERROR: There is a hole in the region chain between  and .  You need to create a new .regioninf
ERROR: Found inconsistency in table complexnetwork:vl
```
发现有三个region没有被任何regionserver部署，导致了最后inconsistent发生
此时，可以使用HBCK2中的assigns对region进行手动assign
```
hbase hbck -j hbase-hbck2-1.0.0-SNAPSHOT.jar assigns 829a9bc159f7627e748b4d4aade744c1 622e99b7eb0ac2c4ba6a4d645d447011 622e99b7eb0ac2c4ba6a4d645d4470011
```
当返回了3个不为-1的pid号时，则表示region assign成功

### 场景2：region没有均匀负载在所有的regionserver上
在这个场景中，有一台regionserver一个region都没有，而其他几台regionserver都有几十个region。  
此时可以使用hbase shell中自带的balancer命令对所有region进行一次负载均衡，使得region均匀分配在所有的regionserver上。
```
进入hbase shell后
hbase(main):001:0> balancer
true
Took 4.3756 seconds                                                             
=> 1
```
如果返回了-1，则表示负载均衡失败，要查看是否有region处在RIT当中，或者有些region没有上线。可以使用hbck对所有region进行一致性检查  
balance_switch：是否打开hbase的自动负载均衡  
**注意**：balance_switch的返回值是之前的balance_switch状态
```
hbase(main):001:0> balance_switch true
Previous balancer state : true
Took 0.3955 seconds                                                             
=> "true"
```