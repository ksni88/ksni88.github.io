---
title: 从零爬Hadoop系列_5-《Hadoop技术内幕2》计算框架
date:2016-07-24 19:54:48
categories: Hadoop
tags:
	- Hadoop
	- 大数据
	- 计算框架
	- 读书笔记
---
# 计算框架
现在有很多计算框架，各有特点，各有应用场景，该篇主要介绍MR，简短介绍Tez、Storm、Spark。
## MapReduce On YARN
预备知识：由于YARN是一个资源管理系统，如果想要将一个新的应用程序（采用某个计算框架的作业）运行在YARN之上，通常需要编写两个组件来“对接移植”：  
1. Client客户端：用户通过客户端向RM提交AM，并查询作业的运行状态
2. AM：AM负责向RM申请资源，并和NM通信启动各个Contianer（内部任务），以及监控各个任务运行状态，失败重新申请等  

所以对于YARN来说，不变的是RM（共享集群的资源调度）、NM（节点代理），可变的是计算框架（应用程序），这里的可变即通过编写自己的AM完成，再明了一点，即各个计算框架是通过编写各自的AM集成到YARN上以实现多个框架共享集群的，而本章的主要内容就是讲MR的AM。  

看过前边的应该记得，YARN是由MRv1进化而来的，所以YARN天生支持MR，即提供了可以直接使用的AM：MRAppMaster，为了更好的理解，可以先来看一下两者的异同：  

![MRV12异同](http://obd791hyv.bkt.clouddn.com/hexo/hadoop/MRV12%E5%BC%82%E5%90%8C.PNG)  

即只有原来负责资源调度和任务管理的运行时环境发生了替换，其他还是原来的代码。

名词解释：
  1. 作业：前文中“应用程序”的另一个叫法
  2. 任务：前文中应用程序的内部子任务
  3. MT：Map Task
  4. RT：Reduce Task

## MR客户端
1. MR客户端是MR用户与YARN（和MRAppMaster）进行通信的唯一途径，通过客户端完成提交作业、获取作业运行状态和控制作业（杀死作业、杀死任务等）
2. 两个RPC通信协议：
    * ApplicationClientProtocol：任何客户端需使用该协议完成提交作业、杀死作业、改变作业优先级等作业（等同于前文一直提的应用程序）级别的操作
    * MRClientProtocol：当作业的AM启动成功以后，它会启动MRClientService，该服务实现了此协议，客户端通过该协议和AM通信，完成控制作业和查询作业运行状态等操作（偏向任务级别的操作），减轻RM的负载

## MRAppMaster
1. 基本构成：  
    * `ContainerAllocator`：与RM通信，为MR任务申请资源（因为AM的提交时由客户端完成的，而且对应Container的启动是RM直接告知NM的，所以不经它手，它只负责内部任务）
    * `ClientService`：是一个接口，由MRClientService实现，负责上边提到的MRClientProtocol通信协议的功能
    * `Job`：表示一个MR作业，即一个MR应用程序，功能和MRv1的JobInProgress类似，监控一个作业的运行状态，它维护了一个作业状态机，实现异步操作
    * `Task`：表示一个MR作业中的某个任务，与MRv1中的TaskInProgress功能类似，监控一个任务的运行状态，它维护了一个任务状态机，实现异步操作
    * `TaskAttempt`：表示一个任务的运行实例，它的执行逻辑和MRv1中的MapTask和ReduceTask运行实例完全一致，只是有一些优化
    * `TaskCleaner`：负责清理失败任务或被杀死任务使用的目录和产生的临时结果，它为了一个线程池和共享队列，异步删除
    * `Speculator`：负责推测执行功能，即当一个作业的某个任务运行速度明显慢于其他任务时，它会为其启动一个备份任务，谁先执行完谁作为最终结果，并杀死另一个
    * `ContainerLauncher`：负责和NM通信，以启动Container，这里需要注意，后文详细介绍
    * `TaskAttemptListener`：负责管理各个任务的心跳信息，同MRv1中的TaskTracker类似
    * `JobHistoryEventHandler`：负责对作业的各个事件记录日志，这些日志会写到HDFS上，主要用于MRAppMaster故障时作业恢复和历史查看
2. 三种作业运行模式：  
    * 本地模式：通常用于作业调试，和MRv1一样
    * Uber模式：所有内部任务以先Map Task后Reduce Task的顺序串行重用同一个Container，主要用于小作业，降低延时
    * Non-Uber模式：用于大作业：
        * Task分为四种状态：  
            * `pending`：刚启动但尚未向RM请求资源
            * `scheduled`：向RM请求资源但尚未分配
            * `assigned`：分配到资源且正在运行
            * `completed`：已运行完成
        * Map Task生命周期：scheduled -> assigned -> completed
        * Reduce Task生命周期：pending -> scheduled -> assigned -> completed
        * 因为reduce依赖于map的结果，而且在YARN中不在有Map Slot和Reduce Slot之分，为了防止过早启动Reduce Task造成资源利用率低下和Map Task饿死，MRAppMaster由三个参数控制其启动时机：  
            * `mapreduce.job.reduce.slowstart.completedmaps`：当Map Task完成的比例达到该值后才会为Reduce Task申请资源，默认为0.5
            * `yarn.app.mapreduce.am.job.reduce.rampup.limit`：在MT完成前，最多启动的RT比例，默认为0.5
            * `yarn.app.mapreduce.am.job.reduce.preemption.limit`：当MT需要资源暂时无法获取资源时，为了保证至少一个MT可以得到资源，最多可以抢占的RT比例，默认为0.5
3. MRAppMaster工作流程：  
    * 和之前YARN的工作流程完全相同，只是把AM替换为MRAppMaster
4. MR作业表示方式：  
    * `Job`：一个MR作业（应用程序），负责整个job即各个task的监控和管理
    * `Task`：作业内部的任务，此处有MT和RT，负责下面各个taskAttemp的监控和管理
    * `TaskAttempt`：task真正的运行实例，负责真正的任务计算工作，在YARN中，任务实例是运行在Container中的
![作业表示方式](http://obd791hyv.bkt.clouddn.com/hexo/hadoop/%E4%BD%9C%E4%B8%9A%E8%A1%A8%E7%A4%BA%E6%96%B9%E5%BC%8F.png)
5. MR作业生命周期：  
    * 三种状态机：`Job状态机`、`Task状态机`、`TaskAttempt状态机`，属于层级包含关系
    * 状态机内部即基于事件驱动的状态转移，详情请翻书
![作业声明周期](http://obd791hyv.bkt.clouddn.com/hexo/hadoop/MR%E4%BD%9C%E4%B8%9A%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F.png)
6. 资源申请和再分配（ContainerAllocator）：  
    * 当用户提交作业之后，MRAppMaster会为之初始化，并创建一系列MT和RT，并为它们申请资源
    * 资源请求是通过getResource()函数向RM发送心跳信息完成的：  
        * 大部分情况下，心跳信息中不包含新的资源请求信息
        * 作用：  
            * 周期性心跳，告诉RM自己活着
            * 周期性询问RM，获取分配的资源、节点更新列表（新增的、不可用的）等
    * 工作流程：  
        * 步骤1：将MT的资源需求发送给RM
        * 步骤2：如果达到了RT的启动条件，则开始为RT申请资源
        * 步骤3：如果为某个Task申请到了资源，则取消其他重复资源的申请，由于在HDFS中，任何一份数据通常有三备份，而对于一个任务而言，考虑到reack和any级别的本地性，它可能对应7个资源请求：  
```shell
            1. <20, "node1", "memory:1G", 1, true>
            2. <20, "node2", "memory:1G", 1, true>
            3. <20, "node3", "memory:1G", 1, true>
            4. <20, "rack1", "memory:1G", 1, true>
            5. <20, "rack1", "memory:1G", 1, true>
            6. <20, "rack1", "memory:1G", 1, true>
            7. <20, "*", "memory:1G", 1, true>
            8. <优先级，期望资源所在的host，资源量，Container数目，是否松弛本地性>
```
    * 在作业运行过程中，会出现资源重新申请和资源取消的行为：  
        * 如果任务运行失败，则会重新为其申请资源
        * 如果一个任务运行速度过慢，则会为其额外申请资源以启动备份任务（推测执行功能）
        * 如果一个节点失败的任务数目过多，则会撤销对该节点的所有资源的申请请求
    * 资源再分配：  
        * 再分配即之前资源调度模型：双层调度模型的第二层，AM申请到Container以后，再自主的分配给各个任务，具体的分配策略由AM自己决定
        * MRAppMaster再分配过程：  
            * 判断收到的Container包含的资源是否满足要求，如果不满足，则通过下次心跳通知RM释放该Container
            * 判断收到的Container所在节点是否被加入黑名单，如果是，则寻找一个与该Container匹配的任务，并重新为该任务申请资源，同时通过下次心跳通知RM释放该Container
            * 根据Container的优先级，将它分配给对应类型的任务：  
                *  优先级为PRIORITY_FAST_FAIL_MAP：则分配给失败的MT，直接从对应的队列中取出第一个任务即可
                *  优先级为PRIORITY_REDUCE：则分配给RT，直接从对应的队列中取出第一个任务即可
                *  其他的，分配给正常的MT，依次尝试从node-local（输入数据和Container同一个节点）、rack-local和no-local几个任务列表中查找MT
        * 黑名单：  
            * 当作业在一个节点上失败的任务实例数目超过一定的上限（默认3，默认开启），则该节点将被加入黑名单
            * 为了防止大量节点被加入黑名单，可以通过参数设定最多加入的比例，默认33%
        * 个人理解：
            * Container是否满足要求是分配过来以后AM判断的，RM只按照对应的调度器策略进行分配，而没有检查是否满足需求量？
            * RM的黑名单和AM的黑名单是不同的两个黑名单，不要搞混；
            * AM提交的只是资源需求，而且这个请求没有和内部的某个具体任务绑定在一起，只是通过级别绑定了任务类型，拿到资源以后还要再次挑选任务进行分配，分配到具体的任务身上以后，再把任务运行所需的资源放进Container，再发送给对应的NM启动Container；所以Container在不同的组件中表示的含义不太相同，在RM中只表示资源需求，在AM中表示任务描述，在NM中表示任务
7. Container启动和释放  
    * 在YARN中，运行Task所需的全部信息被封装到Container中，包括所需资源、依赖的外部文件、JAR包、运行时环境变量、运行命令等
    * ContainerLauncher通过RPC协议ContainerManager和NM通信，以控制Container的启动和释放，进而达到控制任务（启动、杀死等）的效果
    * 释放：Container运行结束以后，不是自己主动清理物理资源进行上报的，而是等着AM发送kill命令，然后才清理物理资源上报RM的
8. 推测执行机制  
    * 作用：防止个别任务拖后腿，通过内部竞争竞速完成
    * 算法核心思想：  
```shell
        estimatedRunTime  = (currentTimestamp - taskAttemptStartTime) / progress
        estimatedEndTime1 = estimatedRunTime + taskAttemptStartTime
        estimatedEndTime2 = currentTimestamp + averageRunTime 
        estimatedEndTime1  - estimatedEndTime2 差值越大，理论收益越大；
```
	* 为了避免浪费资源，启动备份任务之前还要检查一些条件：  
        * 每个任务同时启动的备份任务实例的上限，三个参数值中的最大值：
            * `MINIMUM_ALLOWED_SPECULATIVE_TASKS（常量10）`
            * `PROPORTION_TOTAL_TASKS_SPECULATABLE(常量0.01)*totalTaskNumber`
            * `PROPORTION_RUNNING_TASKS_SPECULATABLE(常量0.1)*numberRunningTasks`
        * 已经完成的任务数目比例不小于`MINIMUM_COMPLETE_PROPORTION_TO_SPECULATE(默认0.05)`，只有这样才能有足够的历史信息估算`averageRunTime` ；
9. 作业恢复
    * 三种恢复粒度： 
        * `作业级别`
        * `任务级别`（MRAppMater采用该级别）
        * `记录级别`
    * 实现方式：MRAppMater采用Avro记录了运行时的各种事件，通过读取解析，重做这些日志，以重构作业或任务的内存信息，进而使作业从断点继续执行
    * 过程：
        * MRAppMater初始化时，解析前一次记录的事件日志，并将运行完成的任务存放到`completedTasksFromPreviousRun`（类型为`Map<TaskId,TaskInfo>`）中
        * MRAppMater构建新的`JobImpl`对象，并将`completedTasksFromPreviousRun`传递给该对象的构造函数
        * 当JobImpl经过初始化后，开始调度内部的TaskImpl，如果任务处于`completedTasksFromPreviousRun`中，则向`TaskImpl`发送一个T_RECOVER事件，以恢复该任务之前的运行状态和信息，否则按照正常逻辑，发送一个`T_SCHEDULE`事件
        * `TaskImpl`收到`T_RECOVER`事件后，通过上次运行信息恢复状态，即根据日志信息将状态直接转移到最终状态：`FAILED`、`KILLED`或者`SUCCEED`
    * 当前MRAppMater仅能恢复上次已运行完成的任务，以避免重复运行，但对于正在运行的任务，则由RM强制杀死回收资源，造成大量浪费，更好的实现是把正在运行的任务直接划归到重启的MRAppMater中，但有一定复杂性，社区尚未考虑

## MR编程接口简述
### InputFormat
1. 两个功能：
    * 数据切分：按照某个策略将输入数据切分成若干个split，以便确定MT个数以及对应的split，对应的函数：
```java
        List<InputSplit> getSplits(in context：JobContext)
```
    * 为Mapper提供输入数据：给定某个split，将其解析成一个个key/value对，对应的函数：  
```java
        RecordReader<KEYIN,VALUEIN>  createRecordReader(in split：InputSplit, in context：TaskAttemptContext)
```
2. 如下图所示，所有基于文件的基类都是FileInputFormat，由公共基类统一对各种输入文件进行切分，即第一个功能，然后由各个派生类自己提供机制进一步解析InputSplit，即第二个功能
	![InputFormat继承树](http://obd791hyv.bkt.clouddn.com/hexo/hadoop/inputformat.png)
3. 举例说明：对于TextInputFormat，每一行内容即为value，该行在整个文件中的偏移量即为key，对应为Mapper中map()方法参数中的`key/value`，所以实现的map()方法，即可将value视为文件中的一行进行进一步数据处理
### Mapper
```java
void map(in key, in value, in context：Mapper.Context)
```
1. MR框架会通过InputFormat获取一个个key/value，然后交给map()方法处理；
2. 对于TextInputFormat，map()方法处理的还是文件中的原始数据，主要工作是从输入数据提取目标数据，然后以key/value的形式输出到磁盘，等待reduce处理

### Partitioner
作用是对map结果进行分片，以便将同一分片的数据交给同一个Reducer处理，直接影响Reduce阶段的负载均衡，对应的方法：
```java
int getPartition(in key：KEY，in value：VALUE，in numPartitions：int)
```
### Reducer：
```java
void reduce(in key：KEYIN, in values：Iterable<VALUEIN>, in context：Reducer.Context)
```
可以看到，reduce的输入，value是一个集合，即Map阶段产生的`K,V`对中，所有key相同的value集合，进行归约处理。
### OutputFormat
用于描述Reduce阶段输出数据的格式。

## Task运行过程
当我们需要编写一个简单的MR作业时，只需实现`map()`和`reduce()`两个函数即可，一旦将作业提交到集群上后，Hadoop内部会将这两个函数封装到Map Task和Reduce Task中，同时将它们调度到多个节点上并行执行，而任务执行过程中可能涉及的数据跨节点传输、记录按key分组等操作均由Task内部实现好了，用户无需关心。
### Map Task内部实现
1. 整体流程：  
    * `Read阶段`：通过用户编写的RecordReader，从输入InputSplit中解析出以一个个key/value对
    * `Map阶段`：将解析出的key/value对交给用户编写的map()函数处理，并产生一系列新的key/value对
    * `Collect阶段`：在map()函数中，当数据处理完成后，一般会调用OutputCollector.collect()输出结果，在该函数内部，它会将生成的key/value对分片（通过调用Partitioner），并写入一个环形内存缓冲区中
    * `Spill阶段`：即“溢出”，当环形缓冲区满后，MR会将数据写到本地磁盘上，生成一个临时文件，需要注意的是，将数据写入本地磁盘之前，先要对数据进行一次本地排序，并在必要时对数据进行合并、压缩等操作
    * `Conbine阶段`：当所有数据处理完成后，MT对所有临时文件进行一次合并，确保最终只会生成一个数据文件
2. Collect过程分析（详情请翻书）：  
	* 首先通过Partitioner获取记录的分片号，组成三元组<key，value，partitionNum>，然后交给collect()函数写入环形缓冲区；
	* 环形缓冲区内部使用两级索引结构
	![MapOutputBuffer两级索引结构](http://obd791hyv.bkt.clouddn.com/hexo/hadoop/MapTask%E4%B8%A4%E7%BA%A7%E7%B4%A2%E5%BC%95%E7%BB%93%E6%9E%84.png)
	* 之所以适用环形缓冲区，是为了当缓冲区满了以后，Collect阶段和Spill阶段可以并行进行读写，类似于生产者消费者模型；
3. Spill过程分析：  
    * 步骤1：利用快排对缓冲区内溢出（默认参数80%）的数据进行排序，排序方式是：先按照分区编号partition排序，然后按照key排序，经过排序后，数据以分区为单位聚集在一起，同一分区内所有数据按照Key有序
    * 步骤2：按照分区编号从小到大依次将每个分区中的数据写入任务工作目录下的临时文件output/spillN.out中（N表示当前溢写次数），如果用户设置了Combiner，则写入文件前，对每个分区中的数据进行一次聚集操作
    * 步骤3：将分区数据的元信息写到内存索引数据结构中，其中每个分区的元信息包括在临时文件中的偏移量、压缩前数据大小和压缩后数据大小，输入内存中的索引大于1MB，则写到文件output/spillN.out.index中
4. Combine过程分析：  
    * 在所有数据处理完后，MT会将所有临时文件合并成一个大文件，并保存到文件output/file.out中，同时生成相应的索引文件output/file.out.index
    * 在进行文件合并过程中，MT以分区为单位进行合并，对于某个分区，它将采用多轮递归合并的方式：每轮合并io.sort.factor（默认100）个文件，然后将合并后的文件重新加入待合并列表，重复直到最终得到一个大文件

### Reduce Task内部实现
1. 整体流程：  
    * `Shuffle阶段`：也称为Copy阶段，RT从各个MT上远程拷贝一片数据，并针对某一片数据，如果其大小超过一定阈值，则写到磁盘，否则直接放到内存
    * `Merge阶段`：在远程拷贝数据的同时，RT启动了两个后台线程对内存和磁盘上的文件进行合并，以防止内存使用过多或磁盘上文件过多
    * `Sort阶段`：按照MR语义，为了在reduce之前，将key相同的数据聚在一起，RT对所有数据进行一次归并排序（因为MR排序，所以局部有序）
    * `Reduce阶段`：将每组数据依次交给用户编写的reduce()函数处理
    * `Write阶段`：reduce()函数将计算结果写到HDFS上
2. Shuffle和Merge阶段分析：
    * 两个阶段是并行进行的
    * 可以分为三个子阶段：
        * 准备运行完成的MT列表：GetMapEventsThread周期性通过RPC获取已完成MT列表，并保存到映射表mapLocations中。为了防止出现网络热点，RT通过对所有host进行“混洗”以打乱拷贝顺序，并保存到scheduledCopies列表中
        * 远程拷贝数据：RT同时启动多个MapOutputCopier线程，这些线程从scheduledCopies列表中获取MT输出位置，通过HTTP GET远程拷贝数据
        * 合并内存文件和磁盘文件：为了防止内存或磁盘文件数据过多，RT启动了两个不同的线程分别对内存上和磁盘上的文件进行合并
    * Sort和Reduce阶段分析：
        * 为了提高效率，RT将这两个阶段并行进行
        * 在Sort阶段，RT为内存和磁盘中的文件建立了小顶堆，保存了指向堆根节点的迭代器，在Reduce阶段，RT不断地移动迭代器，以将key相同的数据顺次交给reduce()处理，启动迭代器的过程实际上就是不断调整小顶堆的过程，从而实现并行进行
3. 用到的两个算法：
    * 快速排序，四个优化：
        * 中轴的选择：避免退化为O（N2），取序列收尾和中间元素中的中位数
        * 子序列划分方法：两个索引，从两端交替扫描
        * 对相同元素的优化：一次排序后划分为三部分：小于中轴、等于中轴、大于中轴，其中等于中轴的不再参与后续递归处理
        * 减少递归次数：当子序列元素数目小于13时，直接使用插入排序法
    * 文件合并使用多轮合并方式：
        * 建小顶堆 
        * 将堆顶指定数量（默认10）的文件进行合并
        * 把合并后的文件加入堆尾，重新建堆
        * 不断重复，直到合并成为一个大文件

### MRAppMaster对Task的优化
1. Map端：用Netty代替Jetty，提高RT数据读取请求的处理性能
2. Reduce端：批拷贝，原来是每个分片建立一个HTTP连接，即使在同一个TaskTracker上，现在是对于同一个TaskTracker上的多个MT建立同一个HTTP，一次读取多个分片
3. Reduce端：  
    * Shuffle插件化：用户可以根据需要自定义实现，比如用RDMA代替HTTP
    * 排序插件化：用户可以根据需求定制排序方法

## 其他计算框架
### DAG计算框架Tez
1. 背景：在一些应用场景中，为了套用MR模型解决问题，不得不将问题分解成若干个有依赖关系的子问题，每个子问题对应一个MR作业，最终所有这些作业形成一个有向图（Directed Acyclic Graph，DAG）。在该DAG中，由于每个节点是一个MR作业，因此它们均会从HDFS上读一次数据和写一次数据（默认三份），即使中间节点的产生数据仅是临时数据，这种表达作业依赖关系的方式非常低效，而且浪费磁盘和网络IO资源
2. 低效的根本原因：作业之间的数据不是直接流动的，而是借助HDFS作为共享数据存储系统通过多次读写进行转移的。更高效的方式应该是有依赖关系的前一个作业直接将产生的数据传输给依赖它的作业，省去中间的磁盘读写
3. Tez核心思想：将Map和Reduce两个操作进一步拆分，即Map拆分成Input、Processor、Sort、Merge和Output，Reduce拆分成Input、Shuffle、Sort、Merge、Processor和Output等，这些分解后的元操作可以灵活组合，产生新的操作，经过一些控制程序的组装后，可形成一个大的DAG作业
4. Tez特点：  
    * 运行在YARN之上，充分利用YARN的管理资源和容错等功能（由Hadoop 2.0核心人员打造）
    * 提供了丰富的数据流API
    * 扩展性良好的“Input-Processor-Output”运行时模型
    * 动态生成物理数据流关系
5. Tez适用场景：  
    * 数据挖掘和自然语言处理：
        * 用户既可以像适用MR一样，直接使用Tez编写DAG类型的应用程序
        * 也可以把Tez作为MR之下的数据处理引擎，即用户依然使用MR编写程序，但使用Tez将之组装成一个DAG作业，Tez提供了一个MR到DAG的转换工具
    * 优化Pig、Hive等引擎

### 实时计算框架Storm（Twitter开源的）
1. 背景：MR因为以下几个特点，决定了它只适合离线批处理的应用场景：  
    * 启动时间长：一个MR作业由启动任务、Map任务、Reduce任务和清理任务四种组成，即使最简单的MR作业也会运行几秒到十几秒
    * 调度开销大：当任务数目大时，全部调度完成耗时长，且资源不足时需排队等待
    * 中间数据写磁盘：为了保证容错性，MT产生的中间结果总要写到磁盘，小作业延时大
2. 新的需求：实时计算和近实时计算相关的应用场景，比如广告推荐、用户行为日志实时分析、处理消息和更新数据库（消息流处理）
3. 实时计算：是指被处理的数据像流水一样不断流入系统，而系统需要针对每条数据进行实时处理和计算，并永不停止（直到用户显式杀死进程），考虑到数据源中数据流动的特点，也称为“流式计算”
4. Storm和MR最大的不同：  
    * Storm的数据源是动态的，即受到一条处理一条
    * MR的数据源是静态的，即数据被处理前整个数据集已经确定，且计算过程中不能被修改

### 内存计算框架Spark（伯克利分校AMPLab实验室开发）
1. 背景：因为MR每次均需要从HDFS上读取数据进行处理，且不同计算之间只能通过HDFS重用数据，使其在以下一个应用场景中效率缓慢：  
    * 迭代计算：即相邻两轮计算要共享一部分数据，如果使用MR，只能通过HDFS读写共享，比如数据挖掘、机器学习、自然语言处理等领域的算法：PageRank、K-means聚类、逻辑回归等
    * 交互式计算：在数据仓库应用中，在用户大量的查询语句中，有很多是作用在同一个数据集上的相同运算，即查询语句中的部分计算是相同的，但当使用Hive或Pig这样的系统时，它们每次都会重复计算，而不能重用计算结果，效率极低
2. 为了解决以上问题，Spark引入了弹性分布式数据集（RDD），它是一个有容错机制、可以被并行操作的数据集合，能够被缓存到内存中，共其他计算使用，而不必像MR那样每次都从HDFS上重新读取数据

### Storm和Spark比较
主要从运行的应用程序类型看：    
1. Storm通常是长作业，实际上是服务，永不终止，直到管理员显式杀死或服务故障
2. Spark通常是短作业，运行时间通常是分钟或者小时级别的




