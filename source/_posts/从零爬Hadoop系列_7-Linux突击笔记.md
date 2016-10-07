---
title: 从零爬Hadoop系列_7-Linux突击笔记
date: 2016-09-26 15:34:36
categories: Linux
tags:
	- Linux
---
虽然在Windows上也能跑Hadoop，但总不是那个味，而且学计算机的没接触过Linux，更不是那个味，所以借此机会学一些简单的Linux基础。但并不是系统性的学习Linux，只是为了能在Linux环境下跑Hadoop，所以只看了两天，现学现用。本文主要是一些基本命令的练习和《鸟哥的Linux私房菜》中的基础知识，类似于学习笔记，好记性不如烂笔头嘛。

## Linux简介
Linux简介遍地都是，想详细了解的可以深入查询了解，此处只是写一下凭自己看书留下的主要印象。

1. **诞生**：在早期电脑还只是大型机用于军事、科研等时，只能同时支撑30个左右的终端，贝尔实验室要开发一个操作系统，目标是支持300个终端，开发了四年宣告失败，实验室内部的一个员工为了方便个人档案管理，抽取失败的项目开发成Unix，慢慢在实验室内部流行开来，且有更多人来开发维护。
2. **发展**：到了90年代个人电脑流行开时，芬兰的大学生Linus通过GNU接触到Unix，把当时只能用于大型机的Unix改写移植到个人x86电脑上使用，借助GNU流行至今。
3. **理解**：操作系统从上到下依次为：用户、应用程序、内核、硬件，四者中只有内核不好理解。说得简单点，操作系统就相当于一个介于用户和硬件的中介，只有通过操作系统，用户才能使用比较统一的操作或命令来使用五花八门的硬件为我们工作，而完成这项任务的重要角色其实是内核，一个很小的东西(Windows之所以那么大是因为有那么繁杂的界面和功能)。所以在Windows下，我们和内核之间还有一个中介，而在Linux下使用命令行可以和内核直接交互，越过了层层封装，可以完成更多底层操作和功能，当然也需要相应的知识学习。
4. **优点**：Linux之所以可以流行至今，并且作为服务器的首选，主要因为以下优点：硬件要求低、开源、免费、稳定、安全、真正的多人多任务。简单点讲，就是硬件成本够低，够安全(因为开源)，够稳定，反正服务器主要用来常年跑服务，不需要GUI，这不就是为服务器量身定做的吗？
5. **缺点**：命令行操作需要学习成本。虽然有GUI，但其本质还是Linux上的一个应用，不如直接操作Linux系统。

虽然当前有很多版本，但内核是一样的，所以本质上是一样的。

## 档案
档案相当于Linux下我们理解的文件和文件夹的统称，只由一位属性做区分。
### 档案属性
档案属性共十位，如drwxrwxrwx，若下标记为0-9，则：
1. 0位：档案类型
    * d：目录
    * -：常规档案
    * l：连结档，link file
    * b：区块（block）设备档，装置文件里面的可供存储的接口设备；
    * c：字符（character）设备档，装置文件里面的串行端口设备，例如键盘、鼠标；
2. 123位：拥有人的权限
    * r：可读，数值4
    * w：可写，数值2
    * x：可执行，数值1（同win下的.exe等扩展名的标示作用，在Linux下是否可执行仅靠该标注。另外当0位为d时，表示是否可以在此目录下执行命令）
3. 456位：与拥有人同群组的权限：同上
4. 789位：非同群组的权限：同上
5. 注意点：当文档类型为d即目录时，要特别注意权限x，例如drwxr--r--，此时除了拥有者外，其他人均不能查看该目录，为什么呢？虽然有r权限，但没有x权限，即不能在该目录下执行任何命令，如ls，也就无法查看，相当于没有进入该目录。


### 连结档link file
1. `inode table`：inode中存储着档案的属性，及该档案放置在哪一个block中等信息
2. `block area`：真正存储数据的地方，所以访问档案时，先查inode table，再到对应的block操作数据
3. 对于【目录】文件本身，只有对应的inode，没有对应的block
4. `Hard Links`：新建立一个inode指向档案的block区块，即它允许一个档案有多个不同的路径名，可以防止误删（因为删除操作只是删除对应的inode，类似于JVM的GC）
    * 缺点1：不能跨filesystem，因为不同的filesystem有不同的inode table
    * 缺点2：不能link目录
5. `Symbolic Links`：新建立一个特殊的档案文件，通过保存真正的档案位置从而把操作导向真正的档案，类似于Windows的快捷方式，如果源文档被删除，该link会打不开
6. `ln [-s] [src] [des]`：创建连结档，`s`：symbolic，即默认为hard link
7. `ll`和连结档
    * `ll`共7列，分别是：`属性`、`link个数`、`拥有者`、`群组`、`修改时间`、`文件名`；
    * 假设当前目录下有一个test.txt档案，在分别建立其一个硬链接hard.txt和软连接soft.txt之后，`ll`命令结果如下：
        ```shell
        total 8
        -rw-r--r-- 2 root root 77 Jul 1 15:15 hard.txt
        lrwxrwxrwx 2 root root  8 Jul 1 15:21 soft.txt -> test.txt
        -rw-r--r-- 2 root root 77 Jul 1 15:15 test.txt
        ```
    * 可以发现有一下几个注意点：
        1. 硬链接的属性首位并非l，而是普通的-，软连接才是l
        2. 源文件和硬链接的link数都为2，而且此时两文件出了命名不同外，其他全部一模一样，已经不分“真假”了，都是真的
        3. 软连接和源文件还是能分辨出“真假”的，软连接的link数为1，而且软连接的命名有指向源文件的说明
        4. 软连接的访问权限全开了

### 常见目录的大致内容
1. `/bin`：存放一般用户常用的执行档，如`ls`，`rm`，`mkdir`，`rmdir`等
2. `/boot`：存放Linux核心，以及和开机相关的档案，重要！
3. `/dev`：即device，Linux将设备视为档案，如硬盘、软盘、光驱等
4. `/etc`：存放系统在开机过程中需要读取的档案，重要！
5. `/home`：系统使用者的家目录
6. `/lib`：存放Linux执行或编译一些程序时使用到的一些库
7. `/list+found`：系统不正常产生错误时，会将一些遗失的片段存在此目录下，通常会自动出现在dev目录下，如加装一个硬盘于/disk中，则会产生/disk/lost+found目录
8. `/mnt`：软盘和光盘预设挂载点，通常软盘挂在/mnt/floppy下，光盘挂在/mnt/cdrom下，不过不是定死的
9. `/proc`：系统核心及执行程序的一些信息。该目录在系统启动时自动被挂上，且不占用硬盘空间，因为里面都是内存内的数据
10. `/root`：系统管理员的家目录
11. `/sbin`：存放系统管理员常用的执行档，如：fdisk等
12. `/tmp`：暂时存放档案的地方，要定期清理，不存重要数据
13. `/usr`：存放相当多的系统信息，内有许多目录，存放程序和指令等等，类似windows下的program files，重要!
14. `/var`：重要！所有服务的登录文件或log files都在/var/log目录下，数据库如MySQL的数据库则在/var/lib目录下，其他目录如邮件等也在这里

## 常用命令
### 档案
1. `df -h`：
    * df：**D**isk **F**ilesystem
    * h：**H**uman readable
2. `du -sh`：
    * du：**D**isk **U**sage
    * s：**S**ummarize
    * h：**H**uman readable
3. `ll`：ls -l的别名
    * ls：**L**ist directory contents
    * l：**L**ong Listing format
4. `ls [dir]`：**l**i**s**t，列出指定目录下所有文件(默认当前路径)
5. `mv`：**m**o**v**e
6. `cp`：**c**o**p**y
7. `scp`：**s**ecure **c**o**p**y（remote file copy program）
8. `rm`：**r**e**m**ove
9. `find -name`
10. `ls [-l]`：
11. `cd [dir]`：**C**hange **D**irectory，切换到指定路径
    * 目录符号
        * `.`：当前目录；
        * `..`：上级目录；
        * `~`：当前用户的家目录；
        * `~user`：user这个用户的家目录
12. `pwd`：**P**rint **W**orking **D**irectory，显示当前目录
13. `mkdir [-m(权限)/p(递归全建)] [name]`：新建一个目录
14. `rmdir [-p(递归全删)] [name]`：删除一个空目录
15. 查看档案内容
    * `cat`：con**cat**，打印档案的全部内容
    * `tac`：从最后一行往前显示，和cat相反
    * `more`：分页显示
    * `less`：分页显示，可以往前翻页
    * `head`：头10行
    * `tail`：最后10行
    * `tailf`：同`tail --follow=[name]`，常用于跟踪日志文件
16. `vi`：使用vim编辑
17. `grep [text]`：**G**lobally search a **R**egular **E**xpression and **P**rint，文本搜索
18. `find [filename]`：相当于Windows下文件管理器的搜索

### 进程
1. `ps -ef | grep <pid>`：
    * ps：**P**rocesses **S**napshot
    * e：同A，All
    * f：**F**ull-Format list
    * grep：**G**lobally search a **R**egular **E**xpression and **P**rint
2. `top -p <pid>`：
    * top：display Linux Tasks
    * p：**P**rocess
3. `kill <pid>`

### 网络
1. `lsof -i:<port>`：
    * lsof：**L**ist **O**pen **F**ile
    * i：**I**nternet，最多100个参数
2. ifconfig：network **I**nter**f**ace **config**ure
3. netstat -anp | grep <port>：
    * netstat：**net**work **stat**istics
    * a：All，both listening and non-listening sockets
    * n：Numerical addresses
    * p：show the PID and name of the Program to which each socket belongs
4. ethtool <ethX>：网卡工具，查看或设置网卡参数。可以配合ifconfig使用。
5. ping
