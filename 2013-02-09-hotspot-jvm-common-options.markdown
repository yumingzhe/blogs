---
layout: post
title: "Hotspot JVM 常用选项"
date: 2013-02-09 10:00:48 +0800
comments: true
categories: [jvm, java, hotspot] 
---
翻译自一篇外文，网上也可找到其他翻译版，但因翻译的不完全，因此自己翻译了一下，以备不时之需。

Sun官方 JDK 中带的 Hotspot JVM 有不计其数的启动参数，我们没法记住每个参数以及它们的作用，但作为开发人员，我们只需重点关注下与堆、垃圾回收以及远程调试相关的参数即可。但是我们也需要熟悉一下其他一些重要的参数，以便日后出问题时能快速的找到相关参数进行排错。下面让我们看一下一些常用的JVM参数。

首先要说明的是，JVM 参数根据传入的方式，通常可以被划分为两类，一类是通过 -X 选项进行传入的，另一类是通过 -XX 选项进行指定的:

1) 以-X开头的参数都是非标准选项，不能确保所有厂商的JVM都支持此参数，并且后续JDK版本如有更改，不另行通知。

2) 以-XX开头的选项都是不稳定的，不推荐经常使用，并且如有更改，也不另行通知。

重要的JVM参数：

1)   布尔型参数可以通过-XX:+选项进行开启或通过-XX:-选项关闭

2)   数值型参数可以通过-XX:=传入。数值中可包含’m’或’M’，’k’或’K’，’g’或’G’用于表示大小，例如32k等价于32768.

3)   字符串型参数通过-XX:=选项指定，通常这种类型的参数用于指定文件名、路径或一系列命令。

运行java -help命令，可打印出标准选项(即所有厂商的JVM都遵循的选项)。也可以通过运行java -X命令打印出当前JVM所支持的非标准选项。如果想要列出当前程序的运行参数，可以通过运行以下语句得到：ManagementFactory.getRuntimeMXBean().getInputArguments();

<!--more-->
现在我们要学习一下程序运行时的一些重要的JVM参数： 
1)与堆大小相关的选项

下面3个jvm选项用于指定程序运行时初始化堆、最大堆大小以及线程栈的大小：

    -Xms  设置初始化堆的大小
    -Xmx  设置最大堆的大小
    -Xss 设置线程栈的大小

2)与打印GC信息相关的选项

    -verbose:gc 打印垃圾回收的运行状况以及耗时。通常首先用这个选项来检测GC是否是程序的瓶颈。
    -XX:+PrintGCDetails 除了能打印出 -verbose:gc选项的数据外，还额外输出新生代的大小以及更精确的时间。
    -XX:-PrintGCTimeStamps  打印出垃圾回收时的时间戳。

3) 用于指定垃圾回收器的选项

    -XX:+UseParallelGC   使用并行垃圾回收器。
    -XX:-UseConcMarkSweepGC 设置年老代为并发收集 (适用于jdk4以上)
    -XX:-UseSerialGC   使用串行垃圾回收器 (适用于jdk5.0以上)

4) 远程调试相关的选项
-Xdebug -Xnoagent -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=8000

5) profiling相关的选项
-Xprof   输出CPU运行时的诊断信息
-Xrunhprof   在运行时剖析运行情况，并将剖析结果打印到控制台,可以指定特定剖析对象，譬如cpu，堆(heap)等。

6) 与classpath相关的选项
-Xbootclasspath 设置启动类的路径，路径下的类不用验证直接加。JVM在加载类之前都会对其进行验证，避免某些类被重复加载从而占用大量的栈空间。可以说类的验证让JVM非常的稳定，但这也会导致在加载大型应用时启动速度过慢。所以当我们将自己编写的类的路径设为启动类路径后，这些类在加载时就会跳过验证极大的加速启动速度。

7) 与永久带(Perm Gen)大小相关的选项
对于解决 java.lang.OutOfMemoryError:Perm Gen Space问题还是比较有效的
-XX:PermSize   设置非堆内存初始值,默认是物理内存的1/64
-XX:NewRatio=2  设置新生代/老年代的比例
-XX:MaxPermSize=64m     设置非堆内存最大值,默认是物理内存的1/4

8) 与类装载(classloading )、卸载(unloading)相关的选项
-XX:+TraceClassLoading 和 -XX:+TraceClassUnloading 两个选项分别用来打印出类加载或卸载时的日志信息。当你怀疑类加载器有内存泄漏问题或类没有被卸载或垃圾回收掉时，这两个选项是非常有用的。

9)与调试相关的选项
-XX:HeapDumpPath=./java_pid.hprof  指定转储(dump)堆内存时的目录或文件名
-XX:-PrintCommandLineFlags   打印出命令行中使用的选项

最后来张图：
