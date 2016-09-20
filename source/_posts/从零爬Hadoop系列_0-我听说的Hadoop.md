---
title: 从零爬Hadoop系列_0-我听说的Hadoop
date: 2016-07-23 14:54:29
categories: Hadoop
tags:
	- Hadoop
	- 大数据
	- 读书笔记
---
## 前言
在两年前入学就听舍友说这些高大上的东西，然而两年过去了，前一段阿里面试问及依然一问三不知。最近在华为实习，有幸借此机会接触学习大数据，三周过去获益匪浅。不仅仅是学到了知识，而且还学到并深刻体会到学习能力和学习方法的重要性。从看书到写笔记，再把笔记串联起来整理成博文，不仅记录学习历程，方便以后回头复习，而且使自己理解得更透彻，而不是蒙混过关。所以我把最近学习的笔记，结合自己的理解整理成一系列博文，主要包括读书笔记、环境搭建、使用说明、以及一些源码理解，旨在二次理解、记录存档和分享。
## 什么是Hadoop
近几年大数据火得不行，平时我们可能多多少少会听到一些相关的东西，比如Hadoop、MapReduce、Spark什么的，但还真没仔细了解过都是些什么东西，有什么用。所以在开始学习之前，先来看一下我们要学的是什么。
>[**What Is Apache Hadoop?**](http://hadoop.apache.org/)  
The Apache™ Hadoop® project develops open-source software for reliable, scalable, distributed computing.  
The Apache Hadoop software library is a framework that allows for the distributed processing of large data sets across clusters of computers using simple programming models. It is designed to scale up from single servers to thousands of machines, each offering local computation and storage. Rather than rely on hardware to deliver high-availability, the library itself is designed to detect and handle failures at the application layer, so delivering a highly-available service on top of a cluster of computers, each of which may be prone to failures.

简单地总结一下就是借助Hadoop，我们可以很方便的把廉价普通的机器组成集群，然后通过简单的编程模型实现分布式地大数据计算处理，而且有容错机制保证高可靠性。不懂吗？接着往下看。
## 主要模块
Hadoop有很多版本，其中从Hadoop1升级到Hadoop2的变化最大。可以说，Hadoop1只是一个以MapReduce为主的计算框架，而Hadoop2则进化成一个以Yarn为主的弹性计算平台，侧重于集群的资源管理，同时可以集成包括MapReduce在内的众多计算框架。Hadoop2主要包括以下四个模块：
>The project includes these modules:  
**Hadoop Common**: The common utilities that support the other Hadoop modules.  
**Hadoop Distributed File System (HDFS™)**: A distributed file system that provides high-throughput access to application data.  
**Hadoop YARN**: A framework for job scheduling and cluster resource management.  
**Hadoop MapReduce**: A YARN-based system for parallel processing of large data sets.  

如果你和我一样对大数据和Hadoop是个小白，那你可能需要先弄明白集群是怎么运作的。  
什么是集群呢？其实很简单，通俗点讲，就是由一群机器组成的一个整体，而且这个整体和其他所有团体一样，拥有自己的内部架构，根据责任划分“部门”，自然也就有Boss和干活的小喽喽。Boss作为集群整体对外的任务承接人，拿到任务以后，再根据内部规则有条不紊地分配下发，具体到各个小喽喽都有自己的一小部分任务做，大家都完成以后整个任务也就完成了，人多力量大嘛。听起来是不是很像我们码农在公司搬砖一样呢？其实就是一样的，这种架构就是我们常听到的Master-Slave（你有没有觉得我们曾经学过的很多东西其实都不是什么新东西，都是从我们现实世界已有的概念和规则搬过去了而已，不过也不能说不是创新）。  
现在我们再来解释一下上边四个组件的作用：
1. **Hadoop Common**：顾名思义，是一些公共的基础组件，主要包括RPC、事件库和状态机三部分。RPC是什么呢？Remote Process Call，远程过程调用，主要用于集群间各个机器通过网络进行通信的，是不是才反应过来？我们平时写代码顶多也就是线程通信、进程通信，那集群之间当然是网络通信了。至于事件库和状态机，可以先不细究，只用记着它们两个让整个集群的并发量上了天就可以了。  
2. **HDFS**：一个分布式文件系统，什么用呢？文件系统可以理解吧？存储在硬盘上的二进制数据，就是靠文件系统变成了一个个有组织的、方便操作的文件。那对于集群才说，当然需要一个分布式的文件系统，把整个集群的所有硬盘数据当成一个整体来管理使用了。怎么做到的？当然也是靠Master-Slave啦。
3. **YARN**：这个是Hadoop2中最重要的角色，后续的读书笔记也主要是讲它的。前边说可以很方便的把廉价机器组成集群，通过简单的编程模型就能够实现分布式大数据处理，就是靠它实现的。
4. **MapReduce**：这个不用再多说了吧，网上解释的例子一大堆，反正通过它，你可以只用写很少量的代码实现分布式计算。  

作为开场白，就介绍这么多吧，后续就要开始读书笔记了，看的是董西成的《Hadoop技术内幕：深入解析YARN架构设计和实现原理》。读完整个书再看源码，才意识到如果有人写得代码可以让人专门出书来解读，而且不是瞎吹，硬生生解读成一本书，那是要有多厉害。不过话说回来，我们学的所有代码好像都有对应的书在解读，所以大牛还是很多的，随便看一个都能获益匪浅。