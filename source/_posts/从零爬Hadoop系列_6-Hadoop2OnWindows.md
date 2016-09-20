---
title: 从零爬Hadoop系列_6-Hadoop2OnWindows
date: 2016-07-31 15:29:36
categories: Hadoop
tags:
	- 编译Hadoop
	- Windows SDK
	- 安装部署Hadoop单机模式
---
# Hadoop2OnWindows
本文主要是讲如何在Windows系统下编译、安装部署单机模式的Hadoop。
## 下载Hadoop源码
当前比较流行的Hadoop源代码版本有两个：Apache Hadoop和Cloudera Distributed
Hadoop（简称 CDH） 。Apache Hadoop是由雅虎、Cloudera、Facebook等公司组成的Hadoop社区共同研发的， 它属于最原始的开源版本，在该版本基础上，很多公司进行了封装和优化，
推出了自己的开源版本，其中，最有名的一个是Cloudera公司发布的CDH版本。

YARN属于Hadoop 2.0的一个分支，此处我使用的是[Apache版本](http://hadoop.apache.org/)的2.7.2。

## 环境说明
为什么把环境说明放在第二步呢？因为所需环境在源码文件里有明确说明。把刚才下载的源码解压到合适的路径下，然后在源码根目录下有一个**BUILDING.txt**文件，该文件依次列出了`Unix`和`Windows`的所需环境。  
```shell
----------------------------------------------------------------------------------

Building on Windows

----------------------------------------------------------------------------------
Requirements:

* Windows System
* JDK 1.7+
* Maven 3.0 or later
* Findbugs 1.3.9 (if running findbugs)
* ProtocolBuffer 2.5.0
* CMake 2.6 or newer
* Windows SDK 7.1 or Visual Studio 2010 Professional
* Windows SDK 8.1 (if building CPU rate control for the container executor)
* zlib headers (if building native code bindings for zlib)
* Internet connection for first build (to fetch all Maven and Hadoop dependencies)
* Unix command-line tools from GnuWin32: sh, mkdir, rm, cp, tar, gzip. These
  tools must be present on your PATH.
```
按照上边的说明(如果你下的版本和我不一样，就看你的BUILDING文件)，依次下载配置好环境变量，其中Protocol只是一个exe编译器，把路径配到PATH中即可，其他环境的安装配置不再详述(对于`if`的可选项，我都没配，按需选配)。

### Windows SDK安装错误
如果你和我一样，在安装Windows SDK 7.1时出现如下图的错误：  
![image](http://cevxd.img48.wal8.com/img48/542077_20160404152451/146994994572.png)  
具体错误原因我也没查到，我的解决方法是：
1. 卸载当前系统中非4.0的`.Net`
2. 下载安装`.Net 4.0`
3. 重启电脑，再次安装Windows SDK

如果不灵，那我也没办法了-_-!


## 使用Maven编译
之所以不直接在IDEA中导入，是因为如果直接把源码导入为Maven项目，在下载好对应的依赖包后，你依然会发现部分类或函数无法找到。这是因为自Hadoop 2.0开始使用了Protocol Buffers定义了RPC协议，而这些Protocol Buffers文件直到在Maven编译源码时才会生成对应的Java类，因此如果源码中引用了这些类，自然就无法找到了。所以倒不如先用Maven编译好了再导入来得省事。

### 编译步骤
1. 找到刚才安装好的Windows SDK，打开`Windows SDK command prompt`
2. 进入刚才下载好解压过的Hadoop源码根目录
3. 如果是32位系统，执行命令：`set Platform=Win32`(注意大小写)
4. 如果是64位系统，执行命令：`set Platform=x64`(注意大小写)
5. 然后执行命令：`mvn package -Pdist,native-win -DskipTests -Dtar`  

![image](http://cevxd.img48.wal8.com/img48/542077_20160404152451/146994994661.png)  
然后就等着编译和下载需要的依赖包吧（`-DskipTests`是为了省去对Test的编译）。

### 编译错误
1. 如果编译失败，首先检查上边说的环境是否都配好了，都已经写进PATH环境变量；其次检查命令参数，注意不要想当然的把Win32写成x32，或者把x64写成Win64，set Platform要一字不差，大小写也不能错
2. 如果在编译期间出现如下图的错误，这是因为没有准备Protocol Buffers环境，下载加进`PATH`即可解决（记得重新打开`Windows SDK command prompt`命令行，不然不会使用新的`PATH`变量）。  
![image](http://cevxd.img48.wal8.com/img48/542077_20160404152451/146994994625.png)  
3. 还有一种错误，当时没有截图，记得是在编译hadoop-common包时出现的，具体错误信息不记得了，原因是因为CMake版本太低，重新下载最新版本配好环境即可解决

### 免编译走捷径
如果你在以上步骤中步履维艰，出错不断，你也可以参考[这个网址](http://toodey.com/2015/08/10/hadoop-installation-on-windows-without-cygwin-in-10-mints/)中的“Step 3”步骤中的说明。其做法是下载他已经编译好的，Windows系统下需要的文件，然后直接替换掉官网上的Unix发布版中的`bin`文件即可。所以捷径只需两步：
1. 下载官网上的binary文件
2. 下载上述网址中的文件，并替换掉官网文件中的bin文件

## 源码学习环境
等编译好后，就可以直接导入IDE了，我这里使用的是IDEA，直接导入即可，如果你使用的是Eclipse，还需要在编译完成后执行命令`mvn eclipse:eclipse -DskipTests`将其转为Eclipse项目。

其中我在系列开篇时提到的Hadoop四大模块对应的项目依次如下：
1. hadoop-common-project：Hadoop 基础库所在目录，该目录中包含了其他所有模块可能会用到的基础库，包括RPC、Metrics、Counter等
2. hadoop-mapreduce-project：MapReduce框架的实现，在MRv1中，MapReduce 由编程模型（map/reduce）、调度系统（JobTracker 和 TaskTracker）和数据处理引擎（MapTask和ReduceTask）等模块组成，而此处的MapReduce则不同于MRv1中的实现，它的资源调度功能由新增的YARN完成（编程模型和数据处理引擎不变），自身仅包含非常简单的任务分配功能
3. hadoop-hdfs-project：Hadoop分布式文件系统实现，不同于Hadoop 1.0中单NameNode实现，Hadoop 2.0支持多NameNode，同时解决了NameNode单点故障问题
4. hadoop-yarn-project：Hadoop资源管理系统YARN实现。这是Hadoop 2.0 新引入的分支，该系统能够统一管理系统中的资源，并按照一定的策略分配给各个应用程序

这时我们就可以结合之前看书的理解，对着代码再梳理一下，加深理解。

## Hadoop安装部署
如果刚才编译成功的话，我们可以在Hadoop源码根目录下的target文件中得到一个二进制的hadoop-2.7.2.tar.gz（机智如你可能会问，干嘛费这么大劲，直接去官网下载不得了？因为官网的是Unix的发行版，不能在Windows上直接部署安装）。

解压这个二进制文件，我们将得到如下目录：
1. `bin`：Hadoop最基本的管理脚本和使用脚本所在目录，这些脚本是sbin目录下管理脚本的基础实现，用户可以直接使用这些脚本管理和使用Hadoop
2. `etc`：Hadoop配置文件所在的目录，包括 core-site.xml、hdfs-site.xml、mapred-site.xml等从Hadoop 1.0继承而来的配置文件和yarn-site.xml等Hadoop 2.0 新增的配置文件
3. `include`：对外提供的编程库头文件（具体动态库和静态库在lib 目录中），这些头文件均是用C++定义的，通常用于C++语言访问HDFS或者编写MapReduce程序
4. `lib`：该目录包含了Hadoop对外提供的编程动态库和静态库，与include 目录中的头文件结合使用
5. `libexec`：各个服务对应的Shell 配置文件所在目录，可用于配置日志输出目录、启动参数（比如 JVM 参数）等基本信息
6. `sbin`：Hadoop管理脚本所在目录，主要包含HDFS和YARN中各类服务的启动/ 关闭脚本
7. `share`：Hadoop各个模块编译后的JAR包所在目录

对于我们呢，只会用到etc里边的配置文件和sbin里边的管理脚本。下边我们开始进行主题——Hadoop2OnWindows，单节点(伪分布式)集群安装部署。

### 1. 配置文件
这里所提的配置文件都在/etc/hadoop目录下。
#### slaves
```shell
localhost
```
#### hadoop-env.cmd
注意把`HADOOP_PREFIX`的值换成自己的解压路径。
```shell
set HADOOP_PREFIX=D:\Project\hadoop-2.7.2-src
set HADOOP_CONF_DIR=%HADOOP_PREFIX%\etc\hadoop
set YARN_CONF_DIR=%HADOOP_CONF_DIR%
set PATH=%PATH%;%HADOOP_PREFIX%\bin
```
#### core-site.xml
```shell
<configuration>
  <property>
    <name>fs.default.name</name>
    <value>hdfs://0.0.0.0:19000</value>
  </property>
</configuration>
```
#### hdfs-site.xml
```shell
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
</configuration>
```

#### mapred-site.xml
先将文件`mapred-site.xml.template`重命名然后编辑。
```shell
<configuration>
   <property>
     <name>mapreduce.framework.name</name>
     <value>yarn</value>
   </property>
</configuration>
```

#### yarn-site.xml
```shell
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>

  <property>
    <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
    <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>
</configuration>
```
### 2. 环境变量
执行脚本命令文件`%HADOOP_PREFIX%\etc\hadoop\hadoop-env.cmd`，使所需的环境变量生效。
### 3. 格式化HDFS文件系统
对于第一次启动，首先要格式化HDFS的namenode，通过执行如下命令：
`%HADOOP_PREFIX%\bin\hdfs namenode -format`
### 4. 启动Hadoop
这里有两种方法：
1. 第一种，依次执行如下两个脚本命令文件：`%HADOOP_PREFIX%\sbin\start-dfs.cmd`、`%HADOOP_PREFIX%\sbin\start-yarn.cmd`
2. 第二种，执行这个脚本命令文件：`%HADOOP_PREFIX%\sbin\start-all.cmd`

其实两种方法本质是一样的，后者只不过是顺序调用前者。
### 5. 验证
启动以后，执行`jps`命令，如果看到如下4个进程，说明启动成功，否则查看启动日志查明原因。
```shell
ResourceManager
NodeManager
NameNode
DataNode
```

## 参考
在Windows系统下编译安装Hadoop简直是一种折磨，步履维艰，这个过程中遇到各种错误，本文只是列出了我记载的几个，大致步骤主要是参考这个链接：[Hadoop2OnWindows](https://wiki.apache.org/hadoop/Hadoop2OnWindows)