---
title: 从零爬Hadoop系列_4-《Hadoop技术内幕2》NodeManager
date: 2016-07-24 19:33:27
categories: Hadoop
tags:
	- Hadoop
	- 大数据
	- 读书笔记
---
# NM剖析
NM是YARN中单个节点上的代理，负责管理Hadoop集群中单个计算节点。和RM剖析一样，我们可以先从最开始的YARN的工作流程中观察，NM都需要和哪些组件进行交互，分别需要完成什么功能，则在NM内部，便会细分出来各个功能类别的模块，在每个模块内部，还可以继续按照功能细分，落实到各个组成的类或接口。
## NM基本职能
1. NM作为RPC的client端，周期性的主动向RM发起请求，这些请求包括向RM注册、汇报节点健康状况和节点上所有Container运行状态；并以请求应答的形式，领取RM下达的命令
2. NM作为RPC的server端，接收并处理AM下达的命令，这些命令包括启动Container、杀死Container、获取Container执行状态等，并返回执行结果
3. 个人理解：其实以上两点都是NM作为整体对外负责的功能，而且并不是它最主要的功能，不要忘了，所有计算任务都是以Container的形式运行在NM上，所以NM最主要的功能是内部Container的管理，比如：  
    * 既然要运行任务，就需要下载任务运行所需的资源，每个NM上有很多Container，需要考虑并行下载和文件管理问题
    * 资源下载完以后，还要监控各个Container的运行状态，进行日志记录
    * Container运行完成以后，还要进行之前下载资源的清理

## NM内部架构
通过对YARN工作流程和NM职能的分析，可以猜测NM至少包含一下模块：
1. AM管理模块：每个NM上可以运行多个AM，需要有对应的管理模块负责信息归总和管理
2. Container管理模块：每个NM上可以运行多个Container，且可以属于不同的AM，需要有对应的管理模块负责信息归总和管理  

同时和RM一样，NM内部各个组件也是基于事件驱动的，NM内部架构组成如下：  
1. NodeStatusUpdater：是NM和RM通信的唯一通道，对应职能1
2. ContainerManager：是NM中最核心组件之一，有一下组件：  
    * RPC Servier：是AM和NM通信的唯一通道，对应职能2
    * ResourceLocalizationService：负责Container运行所需资源的本地化，即从HDFS上下载，同时尽量分摊到各个磁盘上，避免访问热点，同时添加访问权限
    * ContainersLauncher：维护了一个线程池并行完成Container相关的操作，比如启动或杀死Container，其中启动的命令来自AM，杀死的命令来自AM或RM（资源抢占时）
    * AuxServices：NM允许用户以附属服务的方式扩展自己的功能
    * ContainerMonitor：负责监控Container的资源使用量，防止超过RM分配的量，在YARN中，只有内存是通过它监控的
    * LogHandler：可插拔式组件，用户通过它控制Container日志保存方式：写到本地磁盘或上传到一个文件系统中
    * ContainerEventDispatcher：Container事件调度器，负责将ContainerEvent类型的事件调度给对应的Container的状态机ContainerImpl
    * ApplicationEventDispatcher：Application事件调度器，负责将ApplicationEvent类型的事件调度给对应的Application的状态机ApplicationImpl
3. NodeHealthCheckerService：周期性运行自定义脚本和向磁盘写文件，以检查节点健康状况，然后通过NodeStatusUpdater汇报给RM，若发现不健康RM会将其加入黑名单，不再分配，直到其转为健康状态
4. DeletionService：负责删除Container运行完成后遗留的无用的资源文件或临时输出文件，为了避免同步删除文件带来的开销，NM将删除文件功能服务化，进行异步删除
5. Security：安全模块，主要两部分：  
    * ApplicationACLsManager：确保访问NM的用户的合法性，比如Web UI访问，该组件为每个应用程序维护了一个ACL列表
    * ContainerTokenSecretManager：检查收到的各种访问请求的合法性，确保这些请求操作已被RM授权
6. WebServer：通过Web界面展示该节点上所有应用程序运行状态、健康状况、日志等信息
![NM内部架构图](http://cevxd.img48.wal8.com/img48/542077_20160404152451/146936491468.png)

## 健康节点状况监测
1. 自定义Shell脚本：  
    * NodeHealthScriptRunner服务周期性的执行监测脚本并检查其输出，如果标准输出打印出以字符串“ERROR”开头的语句，则认为节点不健康，通过心跳告诉RM
    * 自定义脚本的好处：  
        * 可作为节点负载的反馈：因为当前YARN只对CPU和内存资源进行分配和隔离，所以用户可以通过自定义脚本检查网络、磁盘、文件系统等运行状况缓解这个问题
        * 认为暂时维护NM：可以通过脚本输出ERROR停止NM接受新任务比便进行维护
2. 检测磁盘损坏数目：  
    * 另外一种健康检测机制，默认开启，即与脚本检测同时进行，双重检查
    * 检测方法：如果一个目录具有rwx权限，则认为它是正常的，否则加入坏磁盘列表，当比例过低时，便认为NM不健康，通过心跳告诉RM

## 分布式缓存机制
1. 主要作用：将用户应用程序执行时需要的外部文件资源自动透明地下载并缓存到各个节点上，省去用户手动部署。同时NM并不是一下子把所有资源下载下来，而是根据启动的Container按需下载，而且采用一定的缓存置换算法定期清理失效文件（默认10分钟）
2. 工作流程：  
    * 步骤1：客户端将应用程序所需的文件资源提交到HDFS上
    * 步骤2：客户端将应用程序提交到RM上
    * 步骤3：RM与某个NM通信，让NM启动对应的AM，NM收到命令后，首先从HDFS下载文件（缓存），然后再启动AM
    * 步骤4：AM与RM通信，以请求和获取Container
    * 步骤5：AM收到新分配的Container后，与对应的NM通信，启动任务
    * 步骤6：如果该应用程序第一次在该节点上启动任务，NM首先从HDFS上下载文件缓存到本地，然后启动任务
    * 步骤7：NM后续收到启动任务请求后，如果有缓存，则直接启动，否则先等缓存成功再启动
3. 资源可见性（三种）：  
    * PUBLIC：节点上所有用户共享，即只要存在，其他用户不用再下载
    * PRIVATE：节点上同一用户的所有应用程序共享
    * APPLICATION：节点上同一应用程序的所有Container共享
4. 资源分类（三类）：  
    * ARCHIVE：归档文件，当前支持.jar、.zip、.tar.gz、.tgz、和.tar5种，NM可以自动解压缩
    * FILE：普通文件，NM只下载，没有额外处理
    * PATTERN：以上两种类型的混合体，用户需要通过正则表达式指定哪些属于ARCHIVE需要自动解压
5. YARN分布式缓存的实现
    * 三种可见性资源分别放置在不同的目录下，通过目录权限实现不同的可见性
    * PUBLIC资源下载：由公共服务ResourceLocalizationService中的一个公用线程PublicLocalizer下载，它内部维护了个线程池并行下载
    * PRIVATE和APPLICATION：由公共服务ResourceLocalizationService中的一个专门线程下载：一个Container对应一个LocalizerRunner线程

## 目录结构管理
1. 由于同一个NM上不同的Container往往并行向本地磁盘写数据，为了避免占用大量IO资源造成相互干扰，尽量提高写数据的可靠性和并发写性能，YARN允许NM配置多个挂在不同磁盘的目录作为中间结果存放目录，对于任意一个应用程序，YARN会在每个磁盘中创建相同的目录结构，然后采用轮询策略使用这些目录
![NM目录结构](http://cevxd.img48.wal8.com/img48/542077_20160404152451/146936491521.png)
2. 目录分类：  
    * 数据目录：存放Container运行所需和中间数据
    * 日志目录：运行时输出的日志
3. 日志清理机制：  
    * 定期删除（默认）：默认3小时，超时删除
    * 日志聚集转存：把日志上传到HDFS上，上传后删除，可以指定三种可以转存的类型（顾名思义）：  
        * ALL_CONTAINER
        * APPLICATION_MASTER_ONLY
        * AM_AND_FAILED_CONTAINER_ONLY

## 状态机管理
1. 三类状态机：  
    * Application状态机：维护的信息是RM端维护的子集，方便对一个节点上同一个应用程序的所有Container进行统一管理
    * Container状态机：维护一个Container的生命周期，三个阶段：  
        * 资源本地化
        * Container启动
        * 资源清理（资源本地化逆过程，这里的资源指临时文件，而非硬件资源）：  
            * 这一步的清理，只是Container的临时数据，如Tokens文件和Shell运行脚本等
            * 因为各个Container之间可能有依赖关系，所以NM规定只有在一个应用程序的所有Container全部结束以后，由RM发送广播，NM才能清理程序所占的所有资源和中间数据
        * 可以翻书了解详细的过程，进一步理解基于事件驱动的应用
    * LocalizedResource状态机：维护一种资源的生命周期，从初始化、到下载中、到下载完成/下载失败
2. 个人理解：三类状态机的关系：从大流程上来看，基本可以认为是以下关系：  
    * NM启动一个AM，创建对应的Application状态机
    * Application状态机在收到AM发送的启动Container请求后，会创建对应的Container状态机（一个或多个）
    * Container运行之前，需要先下载缺少的资源，即会创建对应的LocalizedResource状态机（一个或多个）进行资源下载，并等待所有资源下载完，然后进入运行状态

## 资源隔离
1. 内存资源隔离：基于线程监控：  
    * 由服务ContainersMonitorImpl实现，它保存了每个Container进程pid，内部的MonitoringThread每隔一段时间（默认3000ms）扫描所有正在运行的Container进程树
    * 步骤：  
        * 读取/proc/<pid>/stat文件构造进程树
        * 判断单个任务内存使用量是否超过最大值内存量
    * 由于JVM创建一个子进程时，采用了“fork()+exec()”模型，意味着进程创建之后、执行之前会复制一份父进程内存空间，进而使得进程数在某一小段时间内存使用量翻倍，为了防止误杀，Hadoop赋予每个进程“年龄”属性，并规定刚启动的进程年龄为1，MonitoringThread线程每更新一次，各个进程年龄加1，则内存超量使用而被杀死的判断标准为：  
        * 如果一个Container对应的进程树中所有进程（年龄大于0）总内存超过用户设定最大值的两倍
        * 或者所有年龄大于1的进程总内存量超过用户设定的最大值
        * 只要满足其一，则向对应的Container发送ContainerEventType.CKILL_CONTAINER事件将其杀死
    * 个人理解：内存隔离并不是真正的物理上的隔离，其他内存还是可见的，只是简单的一个线程监控使用量而已，所以需要用户在设定内存最大值时有良好的预估和分配
2. CPU资源隔离：使用Cgroups（Control  groups）
    * Cgroups是Linux内核提供的一种可以限制、记录、隔离进程组所使用的物理资源的机制，YARN通过使用其子系统CPU实现CPU资源隔离，该子系统使用调度程序控制任务对CPU的访问
    * Cgroups相比于虚拟机，是一种轻量级资源隔离方案
    * Cgroups只能保证应用程序的CPU使用下限，但不能限制上限。举例说明，某个节点上有10个虚拟CPU：  
        * 若该节点上只有一个应用程序A，且设定的CPU需求为4，则A最多能够使用全部CPU
        * 若该节点上有两个应用程序A和B，且CPU需求都为4，则A和B都最多能够使用全部CPU，最少使用一半
        * 若该节点上有三个应用程序A、B、C，CPU需求依次为4、4、2，则它们最多均可以使用全部CPU，最少可使用CPU资源比例依次为：4/(4+4+2)*10=40%、4/(4+4+2)*10=40%、2/(4+4+2)*10=20%
    * NM默认没有启用任何CPU资源隔离机制
    * 个人理解：CPU隔离也不是真正的物理上的隔离，而且按照上边的说明，基本可以说没有隔离，只能保证下限而已，同样需要用户对资源的使用量有良好的分配和设定
3. 所以总的来讲，所谓的资源隔离是比较虚的，不能顾名思义

## 小结
1. 对NM的理解：作为资源管理系统YARN的一个重要服务，NM管理的是Container，而不是任务，一个Container中可能运行着各种任务，但是对NM而言是透明的，它只负责Container相关的操作，比如管理Container的生命周期等。这也是YARN能够作为一个通用的资源管理平台的重要原因，它不管具体的任务类型或内容，只要按它规定的规范进行封装就可以提交运行
2. 对Container的理解：Container从头到尾都只是一个逻辑抽象概念,RM把资源抽象为Container发送给AM，AM再发给NM，NM则会在Container描述的运行环境中启动任务，其实就是启动一个进程，然后在该进程下跑任务和各个状态机

## 思考问题
1. 第一天的疑问1：怎么保证AM没有撒谎？比如AM并没有向RM申请资源，绕过RM直接通知NM启动任务？
    * 就是由Secret模块下的ContainerTokenSecretManager通过Token检查保证的
2. 第一天疑问2：YARN的工作流程第6步，为什么运行脚本而不是直接运行命令？
    * 因为直接执行命令可能让一些特殊符号发生转义
3. 第二天的疑问3：Container只是对资源逻辑上的划分和规定，在实际使用中，如何保证严格的物理划分或隔离？
    * 没有严格的物理划分或隔离，对于内存是监控上限，对于CPU是保证下限
4. 本章只介绍了节点的健康检测，那NM向RM心跳时汇报的所有Container的运行状态都包括哪些呢？又是怎么汇总的？
5. 根据YARN工作流程的第七步的描述，听起来应该是各个Container心跳式的主动向AM汇报运行状态，但按照基本职能2的描述，只有一个AM主动查询运行状态的接口，那主动汇报的接口呢？本章好像没有提。



