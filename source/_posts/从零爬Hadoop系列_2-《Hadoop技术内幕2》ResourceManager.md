---
title: 从零爬Hadoop系列_2-《Hadoop技术内幕2》ResourceManager
date: 2016-07-24 18:50:57
categories: Hadoop
tags:
	- Hadoop
	- 大数据
	- 读书笔记
---
## RM剖析
### 内部架构
从前边YARN的工作流程图中可以看到，RM作为一个处理中心，分别需要和Client、NM、AM交互做处理，那RM是怎么做到这些交互的呢？其实很简单，从面向对象的角度理解，RM作为一个综合的处理中心，是一个大部门，所以在RM内部，又按照服务类别细分了很多“专一”的模块，每个模块作为服务或事件调度器对外提供访问接口，所有这些访问接口组合起来才形成了上边工作流程图中的各种交互。其中每个模块内部还能继续功能细分，直到把单一功能落实到具体的类或接口中，总的内部架构图如下：
![RM内部架构图](http://cevxd.img48.wal8.com/img48/542077_20160404152451/146936452667.png)
### 模块简介：
1. Client交互模块:
    * `ClientRMService`：为普通用户提供服务，如提交应用、终止应用、获取应用运行状态等
    * `AdminService`：为管理员提供了一套独立的服务接口，防止大量普通用户请求使得管理命令饿死，如动态更新节点列表、更新ACL列表等
    * WebApp：Web界面，友好地展示集群资源使用情况和应用运行状态等信息
2. NM管理模块：
    * `NMLivelinessMonitor`：监控NM是否活着
    * `NodesListManager`：维护正常节点和异常节点列表
    * `ResourceTrackerService`：处理来自NM的请求，主要包括注册和心跳两种请求
3. AM管理模块：
    * `AMLivelinessMonitor`：监控AM是否活着
    * `ApplicationMasterLauncher`：与NM通信，要求它为某个应用启动AM
    * `ApplicationMasterService`：处理来自AM的请求，主要包括注册和心跳两种请求
4. Application管理模块：
    * `ApplicationACLsManager`：管理应用程序访问权限，查看权限和修改权限
    * `RMAppManager`：管理应用程序的启动（提交和后续对象的创建）和关闭
    * `ContainerAllocationExpirer`：AM没有在规定时间内在对应的NM上启动新分配的Container，则RM强制回收，是否回收由该类决定和执行
5. 状态机管理模块：状态机维护着有状态对象的生命周期，RM中共维护4类：
    * `RMApp`：维护的是同一个Application启动的所有运行实例的生命周期，从启动到运行结束
    * `RMAppAttempt`：app运行实例的每次启动称为一次运行尝试，该类维护着一次运行尝试的生命周期，从启动到运行结束
    * `RMContainer`：维护一个Container的运行周期，从创建到运行结束
    * `RMNode`：维护一个NM的生命周期，从启动到运行结束
6. 安全管理模块：
    * 认证：同时采用Kerberos和Token两种技术，保证各个组件之间通信内容的合法性
    * 授权：访问控制列表ACL
7. 资源分配模块：
    * `ResourceScheduler`：
        * 批处理调度器：`FIFO`
        * 多用户调度器
            * `Fair Scheduler`
            * `Capacity Scheduler`

### 容错机制：
1. AM容错：RM负责监控AM的运行状态，若运行失败或超时，会重新分配资源并重启它
2. NM容错：NM定时向RM发送心跳，若超时，则NM上的Container状态置为失败，并交由对应的AM处理其中的任务
3. Container容错：即前边提到的ContainerAllocationExpirer负责的超时未启动处理
4. RM容错：基于共享存储的HA解决方案，YARN采用基于Zookeeper的方案
    * 主备切换分为手动和自动
    * HA(High Availability)需要考虑的问题：
        * 脑裂：
            * 共享存储隔离：确保只有一个Master往共享存储中写数据
            * 客户端隔离：确保只有一个Master可以响应客户端的请求
            * Slave隔离：确保只有一个Master可以向Slave下发命令
        * 切换对外透明：多次连接失败后换一个连接

### 常见行为分析
因为RM相当于YARN整个系统的核心，具备很多功能，每个功能都是基于事件在各个组成结构之间驱动的，所以并没有一个固定或统一的处理流程，只能针对某一个功能来说，比如：  
* 启动AM
* 申请和分配Container
* 杀死AM
* Container超时
* AM超时
* NM超时  

流程复杂，这里不再详述，建议回去翻书，总之一定要结合几个常见行为的分析，从中理解：服务化、基于事件驱动、状态机这三者在其中的应用，以及相比之前的直接函数调用，这三者的优势和带来的改进。  
RM中的重头：资源管理器，将在下篇博文详细介绍。

## 思考问题
想不通的不要急，后续了解多了自然就明白了(带着问题看书懂得更多)。
1. YARN的工作流程第5步：
    * 为什么AM为任务申请到资源以后，不是RM通知对应的NM启动对应的Container？
    * 怎么保证AM没有撒谎？比如AM并没有向RM申请资源，绕过RM直接通知NM启动任务？
2. YARN的工作流程第6步，为什么运行脚本而不是直接运行命令？