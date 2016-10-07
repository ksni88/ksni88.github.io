---
title: 从零爬Hadoop系列_8-Hadoop2OnLinuxCluster
date: 2016-09-26 15:35:36
categories: Hadoop
tags:
	- Hadoop
	- Linux
---
# Hadoop2OnLinuxCluster
本文主要是讲如何在Linux系统下安装部署Hadoop集群。  
## 环境说明  
1. 三台Linux机器（SUSE）  
2. JDK1.8（提前下载好对应的tar.gz）  
3. Hadoop2.7.2（提前下载好对应的tar.gz）  

*以下所有配置需要在每个主机上都进行，但按照本文配置，可以配置一个以后复制过去，不用任何修改。另外，本文是精简配置，如果想了解更多配置参数，可参考另一篇博文或查看[官网](http://hadoop.apache.org/docs/current)左下角的配置文件。*  
 
## 1. 同步时间  
集群上的机器需要进行时间同步，不然运行MR任务时会报错。一般集群机器不能联网，手动修改每台机器时间。
```shell
1. 查看本机时间和时区：`date`
2. 设置时区：
    * 执行tzselect命令查找适合于本地的时区
    * 执行cp /usr/share/zoneinfo/Aisa/Shanghai /etc/localtime
3. 修改日期：date –s 15/07/2015
4. 修改时间：date –s 16:18:52
5. 将系统时间同步到硬件时间：hwclock -w
```

## 2. 关闭防火墙  
如果机器上正在运行防火墙，需要把它关上。
```shell
停止防火墙：service iptables stop(启动防火墙：service iptables start)
```

但以上命令只会当次机器运行有效，机器重启又会无效，如需要，可使用如下命令：
```shell
chkconfig iptables on
chkconfig iptables off
```

## 3. 配置Host文件  
首先，要先给所有机器分配好IP和hostname，hadoop会根据主机名去/etc/hosts文件中查找对应的ip。**注意此处的ip和hostname，切记全文替换为自己的。**  
```shell
1. 查看/修改当前机器的主机名
   cat/vim /etc/HOSTNAME
2. 如果修改了，通过如下命令使其立即生效
   /etc/rc.d/boot.localnet start
3. 在每台机器的/etc/hosts文件末尾加上下面三行(替换相应的ip和hostname，此处假设hostname分别为m1，m2，m3)：
   {ip1} m1
   {ip2} m2
   {ip3} m3
```

## 4. 配置SSH互信  
为了使集群之间无密码访问（为了以后集群通信时不用每次都输入密码），需要在机器之间配置互信（只要确保能从master无密码访问slave就好了）。配置互信前请确保已经安装并启动了ssh服务。  
```shell
1. 生成密钥并配置ssh无密码登录主机(master主机)
   * ssh-keygen -t dsa -P '' -f ~/.ssh/id_dsa
   * cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
2. 将authorized.keys文件拷贝到其他两台slave主机
   * scp authorized_keys m2:~/.ssh
   * scp authorized_keys m3:~/.ssh
3. 验证是否可以从master无密码登录slave主机
   * ssh m2（在master主机输入）登录成功则配置成功，exit退出登录返回Master
```

## 5. 安装JDK和Hadoop  
Hadoop是用java开发的，Hadoop的编译和MR的运行都需要使用JDK，所以JDK是必须安装的。  
```shell
1. 在安装目录下（如/usr/java）解压JDK（解压后可删除tar.gz以节省空间）
   tar -zxvf java.tar.gz
2. 在安装目录下(如/usr/hadoop)解压Hadoop文件（解压后可删除tar.gz以节省空间）
   tar -zxvf hadoop.tar.gz
3. 配置环境变量（vim /etc/profile末尾添加）
   export JAVA_HOME=/usr/java/jdk1.8.0_19
   export CALSSPATH=.:$JAVA_HOME/lib/tools.jar
   export HADOOP_HOME=/usr/hadoop
   export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin
4. 使其立即生效
   source /etc/profile
5. 验证JDK是否成功
   java -version
6. 验证HADOOP是否成功
   hadoop version
```

## 6. 修改Hadoop配置文件  
配置文件都在${HADOOP_HOME}/etc/hadoop目录下。

### 6.1 配置slave文件  
`vim slave`，写入ip或hostname。
```shell
m1
m2
m3
```

### 6.2 配置hadoop-env.sh  
检查并确认该文件中有如下配置：`export JAVA_HOME=${JAVA_HOME}`，但有时`${JAVA_HOME}`并不能生效，可**选择性**修改为对应的目录。

### 6.3 配置core-site.xml
```xml
<configuration>
    <property>
        <name>fs.default.name</name>
        <value>hdfs://m1:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/mnt/tmp</value>
    </property>
</configuration>
```

### 6.4 配置hdfs-site.xml
```xml
<configuration>
    <property>
        <name>dfs.name.dir</name>
        <value>/usr/local/hadoop/name</value>
    </property>
    <property>
        <name>dfs.data.dir</name>
        <value>/mnt/m1/data,/mnt/m2/data,/mnt/m3/data</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
</configuration>
```

### 6.5 配置mapred-site.xml
将mapred-site.xml.template重命名为mapred-site.xml，然后修改。
```xml
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```
### 6.6 配置yarn-site.xml
```xml
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.address</name>
        <value>${yarn.nodemanager.hostname}:8041</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>m1</value>
    </property>
</configuration>
```

## 7. 配置其他节点
至此，master节点上必要的配置完成，这时可以复制到其他两个机器上。
```shell
1. 把所有配置文件打包，便于传输
   tar -zcvf /hadoop.tar.gz /usr/hadoop/etc/hadoop
2. 复制到m2、m3节点
   scp /hadoop.tar.gz m2:/usr/hadoop/etc/
   scp /hadoop.tar.gz m3:/usr/hadoop/etc/

3. 切换到m2主机，或直接ssh登录过去
   sh m2
4. 解压打包的配置文件到/usr/hadoop/etc/目录下(自动替换原文件)
   tar -zxvf /hadoop.tar.gz -C /usr/hadoop/etc/

6. 对m3做同样的操作
```

## 8. 启动验证
至此，所有配置完成，可以启动Hadoop了。

在第一次启动前，必须先格式化namenode：
`hadoop namenode -format`。

然后，通过`${HADDDOP_HOME}/sbin/start-all.sh`启动Hadoop。

之后，通过`jsp`在master节点上，应该可以看到以下五个进程：
```shell
ResourceManager
NodeManager
NameNode
SecondrayNameNode
DataNode
```

在slave节点上，应该可以看到以下两个进程：
```shell
NodeManager
DataNode
```
以上进程缺一不可，缺少的说明启动失败，可以通过查看日志查明失败原因进行修正。

正常启动以后，还可以通过Web UI查看相应的UI界面。
1. RM的Web UI：`http://${RM节点IP}:8088`，即Master节点
2. NM的Web UI：`http://${NM节点IP}:50070`，所有节点都有

以上列出的Web UI访问地址，是默认的配置地址，具体的配置详解、各组件的命令和UI使用，参见下一篇[博文]()。