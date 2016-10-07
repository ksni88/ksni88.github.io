---
title: 从零爬Hadoop系列_9-各组件相关命令和WebUI使用
date: 2016-09-26 15:38:36
categories: Hadoop
tags:
	- Hadoop配置
---
本文主要是HDFS、YARN、MR三个组件相关命令和对应的Web UI使用说明，以及相关的配置说明。
## HDFS  
### hdfs-site.xml配置说明  
1. `fs.name.dir`：这是NameNode结点存储hadoop文件系统信息(fsimage)的本地路径，可以配置多个路径，但这些目录汇总的文件是一样的（防止某个磁盘挂掉，做备份）。**Deprecated，use {dfs.namenode.name.dir} instead，当前配置依然生效，向前兼容**  
2. `dfs.data.dir`：hdfs上的数据在本地磁盘的存储路径，如果有多磁盘最好每个磁盘都配置一个路径，这样hdfs会轮询在这些路径中写入数据。所以datanode在`dfs.data.dir`每一项位置汇总存的数据是不一样的，这个和namenode不同。**Deprecated，use {dfs.datanode.name.dir} instead，当前配置依然生效，向前兼容**  
3. `dfs.replication`：设置数据块的复制次数，默认是3。如果大于节点数，则每个节点中都会存一份备份，而不会超过节点数  
4. `dfs.namenode.http-address`：NM Web UI地址，默认值为`0.0.0.0:50070`  

### 启动
在etc/hadoop/slave和ssh配置互信的前提下，通过执行`sbin/start-dfs.sh`启动。之后在master节点通过jps可看到三个进程：DataNode，NameNode，SecondrayNameNode，在slave节点可看到DataNode  

### 停止
同上，通过执行`sbin/stop-dfs.sh`停止  

### 常用命令  
1. **格式：`hadoop fs <-command>`**（可通过hadoop fs -help [command]查看说明，另外这里主要是hdfs文件操作命令，有关hdfs系统管理命令，如启动datanode等，参考`hdfs`命令）  
2. 常用的文件操作命令同Linux，如cat、mkdir、ls、rm、mv、cp、find、df、du、chmod、chown、chgrp等  
3. `-put <本地路径> <hdfs路径>`：文件上传（复制），可多个，用空格分隔，即最后一个参数是hdfs目的路径，前边的都是要上传的本地路径。其中目录路径一定要存在（通过mkdir创建），不然会报错（这点和Linux命令不同，在Linux中，cp命令会自动创建不存在的目的路径并完成复制）  
4. `-copyFromLocal <本地路径> <hdfs路径>`：和put完全相同  
5. `-moveFromLocal  <本地路径> <hdfs路径>`：剪切本地文件到hdfs，可多个，规则同上  
6. `-get <hdfs路径> <本地路径>`：文件下载（复制），可多个，规则同上  
7. `-copyToLocal <hdfs路径> <本地路径>`：和get完全相同  
8. `-moveToLocal  <hdfs路径> <本地路径>`：此命令还未实现，“Option '-moveToLocal' is not implemented yet”  

### Web UI
1. 访问地址由配置项`dfs.namenode.http-address`决定，默认`0.0.0.0:50070`  
2. 当前如果在浏览器中输入http://m1:50070，并不能正常访问，而是显示了一段html，仔细看得话，其中包含一个跳转语句，跳转到dfshealth.html，但不知道为什么没有跳转，所以需要手动输入地址http://m1:50070/dfshealth.html进行UI访问  
3. 主要功能  
    * Overview：所有datanode整体概况，如总容量、已用容量、剩余容量等  
    * Datanodes：各个datanode的概况，如总容量、已用容量、剩余容量等（相加等于上一条）  
    * Datanode Volume Failures：节点故障  
    * Snapshot：文件快照  
    * Startup Progress：启动过程中各个阶段的完成情况和耗时  
    * Utilities  
        * UI查看日志文件  
        * UI查看hdfs文件系统  

## YARN  
### yarn-site.xml配置说明  
1. `yarn.nodemanager.aux-services`：NM上运行的附属服务。需配置成mapreduce_shuffle才能运行MR程序
2. `yarn.nodemanager.aux-services.mapreduce.shuffle.class`：顾名思义，默认配置为`org.apache.hadoop.mapred.ShuffleHandler`，如果没有自定义实现，可缺省
3. `yarn.resourcemanager.hostname`：RM所在节点的IP，默认值`0.0.0.0`，可以不用另外配置
4. `yarn.resourcemanager.webapp.address`：RM Web UI地址，默认值`${yarn.resourcemanager.hostname}:8088`，可以不用另外配置  

### 启动  
在etc/hadoop/slave和ssh配置互信的前提下，通过执行`sbin/start-yarn.sh`启动。之后在master节点通过jps可看到两个进程：ResourceManager，NodeManager，在slave节点可看到NodeManager  

### 停止
同上，通过执行`sbin/stop-yarn.sh`停止  

### 常用命令  
1. `yarn resourcemanager`：启动RM  
2. `yarn nodemanager`：在每个slave节点启动一个NM  
3. `yarn rmadmin [-options]`：RM管理员命令，如刷新队列、刷新节点等  
4. `yarn jar <jarFile> [mainClassName] <inputDir>   <outputDir>`：运行指定的jar文件jarFile，如果打包时指定了mainClass，则[mainClassName]可不指定，否则需要指定，输入数据目录为inputDir，输出数据到目录outputDir  
5. `yarn application <-command>`：application 相关的命令  
    * `-list [-options]`：列出所有applications，通过options可以筛选  
        * `-appStates   <States>`：通过States筛选applications，其中<States>可以是逗号分隔的多个项，有效的States如下：`ALL、NEW、NEW_SAVING、SUBMITTED、ACCEPTED、RUNNING、FINISHED、FAILED、KILLED`  
        * `-appTypes <Types>`：通过Types筛选applications  
    * `-kill <ApplicationID>`：杀死指定的application  
    * `-status <ApplicationID>`：打印指定application的状态  
6. `yarn applicationattempt <-command>`：applicationattempt 相关的命令  
    * `-list <ApplicationID>`：列出指定application的所有运行实例applicationattempt  
    * `-status <ApplicationAttemptID>`：打印指定applicationattempt 的状态  
7. `yarn container <-command>`：container相关的命令  
    * `-list <ApplicationAttemptID>`：列出指定applicationattempt的所有Container  
    * `-status <ContainerID>`：打印指定Container的状态  
8. `yarn node <-command>`：node相关的命令  
    * `-list [-options]`：默认列出所有正常运行的nodes  
        * `-all`：列出所有nodes，包括不健康的等非正常状态  
        * `-states <States>`：根据states筛选列出对应的nodes  
        * `-status <NodeID>`：打印指定node的状态  
9. `yarn queue -status <QeueuName>`：打印指定队列的状态，如负载等，但没有对应的获取队列名的命令，只能通过`mapred queue -showacls`获取queueName  
10. `yarn logs -application <applicationID> [-options]`：打印指定application的日志  
11. `yarn cluster -lnl`：list node labels  
12. `yarn daemonlog <-command>`  
    * `-getlevel <host:port> <name>`：获取指定hadoop守护进程的日志级别  
    * `-setlevel <host:port> <name> <level>`：设置指定hadoop守护进程的日志级别  
13. `yarn top`：类似Linux的`top`命令，动态查看集群状态  

### Web UI  
1. 访问地址由配置项`yarn.resourcemanager.webapp.address`决定，默认`${yarn.resourcemanager.hostname}:8088`  
2. 主要功能：  
    * Nodes：集群整体的apps概况和资源概况，调度器的配置，以及各个nodes的资源概况  
    * Node Labels：查看各个Label的情况，如NM数量、资源总量（Label主要用于一种调度策略`Label based scheduling`，该策略是apache hadoop2.6.0和hdp2.2引入的，只有`Capacity Scheduler`支持该特性，其主要思想是：用户可以为每个NM标注几个标签，比如highmeme，highdisk等，以表明该NM的特性，同时用户可以为调度器中每个队列标注几个标签，这样，提交到某个队列中的作业，只会被分配到标注有对应标签的NM上的资源。该特性是为了让YARN更好的运行在异构集群中，更好地管理和调度混合类型的应用程序）  
    * Applications：查看各个application的状态，包括开始时间、结束时间、使用的资源量、进度等，其中还可以根据状态分类查看  
    * Scheduler：查看调度器的情况，如Container情况、分配情况、抢占情况等  
    * Tools  
        * UI查看本地日志  
        * UI查看RM服务状态  
		
## MR  
### mapred-site.xml配置说明  
1. `mapreduce.framework.name`：指使用哪种框架来运行任务，三个选项：`classic`，`yarn`，`local`，默认为`local`  
    * `classic`：任务提交给JobTracker，它的地址通过`mapreduce.jobtracker.address`配置  
    * `yarn`：任务提交给RM中的applications manager，它的地址通过`yarn.resourcemanager.address`配置(在`yarn-site.xml`中)  
    * `local`：任务提交给本地JobTracker，即在本地使用MR，把`mapreduce.framework.name`和`mapreduce.jobtracker.address`都配置为local即可  
2. 为了方便用户查看历史作业信息，`MRAppMaster`提供了一个`JobHistory-Server`，该服务由四个子服务组成，其中除了负责扫描删除的服务，其他三个服务都是对外的，相关配置如下：  
    * `mapreduce.jobhistory.webapp.address`：Web UI访问地址，默认`0.0.0.0:19888`  
    * `mapreduce.jobhistory.admin.address`：对外暴露的执行管理员命令的服务接口，通过执行mapred hsadmin输入的命令，都是通过该接口提交执行的，默认`0.0.0.0:10033`  
    * `mapreduce.jobhistory.address`：JobHistory服务负责从HDFS上读取MR历史作业日志，然后解析成格式化信息，供UI查看，该项即该服务对UI服务进程暴露的IPC接口，默认`0.0.0.0:10020`  
	
### 常用命令：
1. `mapred queue <-command>`：  
    * `-list`：列出所有队列信息和负载状态  
    * `-info <job-queue-name> [-showJobs]`：打印指定队列信息和负载状态[获取指定队列中所有任务的详细信息]；该命令只能指定`mapred queue -list`列出的队列名  
    * `-showacls`：打印当前用户可以访问的所有队列的acl列表；该命令列出的队列可能比-list列出的多（如root用户），通过`yarn queue -status   <name>`可以查看该命令列出的所有队列的状态  
2. `mapred job <-command>`：MR任务相关的命令，如list、kill、submit、status等14条命令  
3. `mapred pipes <-command>`：运行pipes job相关的命令  

### Job History  
1. 说明：整个集群中，只用在任意一个节点(默认配置无绑定)启动一个Job History服务就可以查看整个集群的作业历史，而不用在每个节点上都启动，因为它是从HDFS上读取各个节点数据的，不过一般和RM在同一个节点上  
2. 启动，两种方式：  
    * `mapred historyserver`：一直运行，只能通过ctrl+c停止，不建议使用  
    * `sbin/mr-jobhistory-daemon.sh start historyserver`  
3. 停止：`sbin/mr-jobhistory-daemon.sh stop historyserver`  
4. 常用命令：**`格式：mapred hsadmin <-command>`**：history server管理员命令，主要更新三种信息：管理员列表、超级用户组列表、用户和用户组映射关系  
5. Web UI  
    * 访问地址由配置项`mapreduce.jobhistory.webapp.address`决定，默认`0.0.0.0:19888`  
    * 主要功能：  
        * Jobs：所有历史作业的信息，如时间、完成进度等  
        * Tools：  
            * UI查看本地日志  
            * UI查看Job History服务状态  
			
## 问题解决清单  
1. 问题描述：通过`start-all.sh`启动hadoop后，通过jps没有发现NodeManager进程，通过web访问m1:8042也不能正常显示  
    **问题解决**：通过查看nodemanager启动日志发现有异常：`cannot support recovery with an ephemeral server port. Check the setting of yarn.nodemanager.address`。由于我并没有在yarn-site.xml中配置该项，所以通过查看官网提供的默认配置发现，`yarn.nodemanager.address`的默认配置是`${yarn.nodemanager.hostname}:0`。（网上很多资料记载的默认配置端口是8041，不知道为什么默认配置变成了0），经过配置该项为m1:8041后，再次启动，发现只有m1节点NM启动成功，其他节点依然失败，再次查看日志发现`Problem binding to m1:8041`，因为NM是运行在各个节点上的，所以该项配置应该对应各个节点各自的IP，所以应该配置成`${yarn.nodemanager.hostname}:8041`，问题解决。（需要注意修改所有节点的配置）
2. 问题描述：按照配置文件，在`core-site.xml`中配置了`hadoop.tmp.dir`项为`/mnt/m1/tmp,/mnt/m2/tmp,/mnt/m3/tmp`，本意是逗号做分隔，配置三个目录，但实际上逗号并没有起到分隔的作用，而是被作为目录的一部分，只有一个目录被创建（并没有影响正常运行）  
    **问题解决**：
    * 通过查看官网默认配置，发现该项是以`file://`开头的路径，所以将该项配置成`file://mnt/m1/tmp,/mnt/m2/tmp,/mnt/m3/tmp`后，通过`start-all.sh`启动，发现NM并没有被启动，通过查看日志发现，`AbstractService`报异常`Wrong FS file://mnt/m1/tmp,/mnt/m2/tmp,/mnt/m3/tmp/yarn-nm-recovery, expected：file///`（师傅说应该是因为第一个"/"被当做系统根目录处理了，所以不识别"file:/"）
    * 将该项配置成`file:///mnt/m1/tmp,/mnt/m2/tmp,/mnt/m3/tmp`后，通过`start-all.sh`启动，发现NM还是并没有被启动，通过查看日志发现，`NativeDB`报异常`IO error：/usr/hadoop/file:/mnt/m1/tmp,/mnt/m2/tmp,/mnt/m3/tmp/yarn-nm-recovery/LOCK：No such file or direcotry`，这里对比上一步可发现，`NativeDB`并不识别`file://`，而是将其当做普通路径来处理，而且由于没有根目录，所以当做相对路径处理，追加了${HADOOP_HOME}=/usr/hadoop作为根目录，但上一步报异常的`AbstractService`是识别"file://"的，所以`AbstractService`是按照多路径创建了三个目录，接着走到这里时，因为当做一个路径来`open`了，所以报异常。而且查看官网可发现该项配置并没有说明可以配置多个目录，所以应该是版本升级后不再支持多路径了
    * 将该项配置成正常的单路径（不再以`file://`的形式配多路径）：`/mnt/tmp`，问题解决（同时注意修改其他节点的配置）