# Spark核心组件
## Driver
Spark驱动器节点，用于执行Spark任务中的main方法，负责实际代码的执行工作。Driver在Spark作业执行时主要负责：  
1. 将用户程序转化为作业  
2. 在Executor之间调度任务(task)  
3. 跟踪Executor的执行情况  
4. 通过UI展示查询运行情况  

## Executor
CoarseGrainedExecutorBackend 节点是一个JVM进程，Executor存在于它当中，负责在Spark作业中运行具体任务，任务彼此之间相互独立。Spark应用启动时，Executor节点被同时启动，并且始终伴随着整个Spark应用的生命周期而存在。如果有Executor节点发生了故障或崩溃，Spark应用也可以继续执行，会将出错节点上的任务调度到其他Executor节点上继续运行。 

>1.在CoarseGrainedExecutorBackend启动时，向Driver注册Executor其实质是注册ExecutorBackend实例，和Executor实例之间没有直接的关系！！！
2.CoarseGrainedExecutorBackend是Executor运行所在的进程名称，Executor才是真正在处理Task的对象，Executor内部是通过线程池的方式来完成Task的计算的。
3.CoarseGrainedExecutorBackend和Executor是一一对应的。
4.CoarseGrainedExecutorBackend是一个消息通信体（其实现了ThreadSafeRpcEndpoint）。可以发送信息给Driver，并可以接收Driver中发过来的指令，例如启动Task等。

Executor有两个核心功能：  
1. 负责运行组成Spark应用的任务，并将结果返回给驱动器进程  
2. 它们通过自身的块管理器(block manager)为用户程序中要求缓存的RDD

# Spark部署流程
![image-20200201112610371](assets\image-20200201112610371.png)  
		不论Spark以何种模式进行部署，任务提交后，都会先启动Driver进程，随后Driver进行向集群管理器注册应用程序，之后集群管理器根据此任务的配置文件分配Executor并启动，当Driver所需的资源全部满足后，Driver开始执行main函数，Spark查询为懒执行，当执行到action算子时开始反向推算，根据宽依赖进行stage的划分，随后每一个stage对应一个taskset，taskset中有多个task，根据本地化原则，task会被分发到指定的Executor去执行，在任务执行的过程中，Executor也会不断与Driver进行通信，报告任务运行情况。  

# Spark部署模式
Spark支持3中集群管理器(Cluster Manager)，分别为:  
1. Standalone：独立模式，Spark原生的简单集群管理器，自带完整的服务，可单独部署到一个集群中，无需依赖任何其他资源管理系统，使用Standalone可以很方便的搭建一个集群。

2. Apache Mesos：一个强大的分布式管理框架，它允许多种不同的框架部署在其之上，包括yarn。  

3. Hadoop Yarn：统一的资源管理机制，在上面可以运行多套计算框架，如mapreduce,storm等，根据driver在集群中的位置不同，分为yarn client和yarn cluster。  

## Standalone Client模式

![image-20200201114140664](assets\image-20200201114140664.png)  

在standalone Client模式下，**Driver在任务提交的本地机器上运行**，Driver启动后向Master注册应用程序，Master根据submit脚本的资源需求找到内部资源至少可以启动一个Executor的所有Worker，然后在这些Worker之间分配Executor，Worker上的Executor启动后会向Driver反向注册，所有的Executor注册完成后，Driver开始执行main函数，之后执行到Action算子时，开始划分stage，每个stage生成对应的taskset，之后将task分发到各个Executor上执行。  

## Standalone Cluster模式
![image-20200201114634027](assets\image-20200201114634027.png)  

在Standalone Cluster模式下，任务提交后，**Master会找到一个worker启动Driver进程**，Driver启动后向Master注册应用程序，Master根据submit脚本的资源需求找到内部资源至少可以启动一个Executor的所有worker，然后在这些worker之间分配Executor，Worker上的Executor启动后会向Driver反向注册，所有的Executor注册完成后，Driver开始执行main函数，之后执行到Action算子时，开始划分stage，每个stage生成对应的taskset，之后将task分发到各个executor上执行。  

## Yarn Client模式

![image-20200201143922159](assets\image-20200201143922159.png)

![image-20200201144038137](assets\image-20200201144038137.png)

​		在Yarn Client模式下，Driver在任务提交的本地机器上运行，Driver启动后会和ResourceManager通讯，申请启动ApplicationMaster，随后ResourceManager分配container，在合适的NodeManager上启动ApplicationMaster，此时的ApplicationMaster的功能相当于一个ExecutorLauncher，只负责向ResourceManager申请Executor内存。   

​		ResourceManager接到ApplicationMaster的资源申请后会分配container，然后ApplicationMaster再资源分配指定的NodeManager上启动Executor进程，Executor进程启动后会向Driver反向注册，Executor全部注册完毕后Driver开始执行main函数，之后执行到Action算子时，触发一个job，并根据宽依赖开始划分stage，每个stage生成对应的taskset，之后将task分发到各个Executor上执行。  

## Yarn Cluster模式

![image-20200201150737858](assets\image-20200201150737858.png)

![image-20200201150654499](assets\image-20200201150654499.png)

在Yarn Cluster模式下，任务提交后会和ResourceManager通讯申请启动ApplicationMaster，随后ResourceManager分配container，在合适的NodeManager上启动ApplicationMaster，此时的ApplicationMaster就是Driver。

# Spark任务调度概述

​		当Driver起来后，Driver则会根据用户程序逻辑准备任务，并根据Executor资源情况逐步分发任务在详细阐述任务调度前，首先说明下Spark里的几个概念。一个Spark应用程序包括Job,Stage以及Task三个概念：  

1. Job是以Action算子方法为界，遇到一个Action算子则触发一个Job；  
2. Stage是Job的子集，以RDD的宽依赖(即Shuffle)为界，遇到Shuffle做一次划分；  
3. Task是Stage的子集，以并行度(分区数)来衡量，分区数是多少，则有多少个task；  

Spark的任务调度总体来说分两路进行，一路是Stage级的调度，一路是Task级的调度；  

![image-20200202171301659](assets\image-20200202171301659.png)





# 尚未归类

spark的sortShuffle中，什么时候不会使用bypassMergeSort:    

![image-20200203153531216](assets\image-20200203153531216.png)

1. 首选条件是当出现了map-side aggregation的时候，不会使用bypass  
2. 当上面的条件不满足时，则使用是否小于spark.shuffle.sort.bypassMergeThreshold(默认值200)，如果大于，则不会使用bypassMergeSort  

## JAVA
Java序列化的时候，会尝试去寻找并调用需要序列化的类中的writeReplace方法，如果有则调用，如果没有则不调用