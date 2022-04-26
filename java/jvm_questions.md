## JVM常见问题

### JVM主要组成部分

- 类加载器（ClassLoader）
- 运行时数据区（Runtime Data Area）
- 执行引擎（Execution Engine）
- 本地库接口（Native Interface）

组件的作用： 首先通过类加载器（ClassLoader）会把 Java 代码转换成字节码，运行时数据区（Runtime Data Area）再把字节码加载到内存中，而字节码文件只是 JVM 的一套指令集规范，并不能直接交给底层操作系统去执行，因此需要特定的命令解析器执行引擎（Execution Engine），将字节码翻译成底层系统指令，再交由 CPU 去执行，而这个过程中需要调用其他语言的本地库接口（Native Interface）来实现整个程序的功能。

### JVM 运行时数据区？详细介绍下每个区域的作用

### 有哪些类加载器

### 哪些情况会触发类加载

### 什么是双亲委派模型？

### 类装载的执行过程？

### 怎么判断对象是否可以被回收？

### 哪些变量可以作为GC Roots

### Java 中都有哪些引用类型？

### JVM 有哪些垃圾回收算法？

### JVM 有哪些垃圾回收器？

### 新生代垃圾回收器和老生代垃圾回收器都有哪些？有什么区别？

### Minor GC与Full GC分别在什么时候发生？

### 简述分代垃圾回收器是怎么工作的？

### 生产环境下排错过程？

有上线的话，先回滚恢复线上环境，然后根据日志、监控等还原现场定位原因。

[大厂的线上生产问题排查指南](https://cloud.tencent.com/developer/article/1791840)

### JVM 性能监控的工具？

### 常用的 JVM 调优的参数都有哪些？


## Linux排查问题命令

- CPU相关问题，可以使用top、vmstat、pidstat、ps等工具排查；
- 内存相关问题，可以使用free、top、ps、vmstat、cachestat、sar等工具排查；
- IO相关问题，可以使用lsof、iostat、pidstat、sar、iotop、df、du等工具排查；
- 网络相关问题，可以使用ifconfig、ip、nslookup、dig、ping、tcpdump、iptables等工具排查。

### top!!

实时显示当前系统正在执行的进程的相关信息，包括进程 ID、内存占用率、CPU 占用率等。

![](https://raw.githubusercontent.com/rainsbaby/notebook/master/imgs/jvm/linux_cmd_top.png)

	top -c // 显示命令完全模式
	
交互命令：

- P：按%CPU使用率排行
- T：按MITE+排行
- M：按%MEM排行

### vmstat

对操作系统的虚拟内存、进程、CPU活动进行监控。他是对系统的整体情况进行统计，不足之处是无法对某个进程进行深入分析。

### pidstat

sysstat工具的一个命令，用于监控全部或指定进程的cpu、内存、线程、设备IO等系统资源的占用情况。

### ps

ps(process status)，用来查看当前运行的进程状态，一次性查看，如果需要动态连续结果使用 top。


### free

显示Linux系统中空闲的、已用的物理内存及swap内存,及被内核使用的buffer。

### cachestat

### sar

### lsof

lsof(list open files)是一个查看进程打开的文件的工具。 在linux 系统中，一切皆文件。 通过文件不仅仅可以访问常规数据，还可以访问网络连接和硬件。 所以lsof 命令不仅可以查看进程打开的文件、目录，还可以查看进程监听的端口等socket 相关的信息。

### iostat

对系统的磁盘操作活动进行监视。它的特点是汇报磁盘活动统计情况，同时也会汇报出CPU使用情况。它不能对某个进程进行深入分析，仅对系统的整体情况进行分析。


### iotop

### df

### du

### netstat

显示与IP、TCP、UDP和ICMP协议相关的统计数据，一般用于检验本机各端口的网络连接情况。

###ifconfig

### ip

### nslookup

### dig

### ping

### tcpdump

### iptables
