## jps

命令格式

- jps [ options ] [ hostid ]

用途

- 用来查看有访问权限的JVM进程状态信息
- 通过指定有访问权限的hostid来查看指定主机系统的JVM进程状态信息(指定协议和端口，目标主机应该运行着jstatd进程)，不指定hostid则查看的是本机JVM
- jps会列出所有有权限访问的运行着的JVM进程，一般是pid +程序类名或jar包名，默认省略包名和传入main函数的参数
- 因为jps使用Java命令启动的，查找main函数入口的类名或包名和传入的参数，只针对用Java命令启动的JVM进程，其他的会输出Unknown

选项

- -q 不显示类名或包名和传入main函数的参数，只列出pid
- -m 显示传入main函数的参数，对于embedded JVM可能显示null
- -l 显示类的完整包名或者jar包的绝对路径
- -v 显示传入给JVM的所有参数
- -V 显示通过flag file传入给JVM的参数（flag file可能是.hotspotrc或者通过-XX:Flags=<filename>参数传入的指定文件）
- -Joption 给启动jps的Java启动器传入参数，比如-J-Xms48m 来设置初始堆大小为48m

hostid格式

- [protocol:][[//]hostname][:port][/servername]

输出格式

- lvmid [ [ classname | JARfilename | "Unknown"] [ arg* ] [ jvmarg* ] ]

## jstack

命令格式

- jstack [ option ] pid
- jstack [ option ] executable core
- jstack [ option ] [server-id@]remote-hostname-or-IP

用途

- 用来查看指定JVM进程或远程可调试主机的线程虚拟机栈（Hotspot将虚拟机栈和本地方法栈合二为一）信息
- java程序崩溃生成core文件，jstack可以用来获得core文件的java stack和native stack的信息
- 对于Java方法栈帧，根据选项会输出全类名、方法名、字节码索引（byte code index）和行号
- 对于Native方法栈帧，the closest native symbol to 'pc', if available, is printed.
- 如果在64位机器上，需要指定选项"-J-d64"

选项

- -F 强制dump线程栈信息，多用于java程序呈现hung的状态
- -m 不仅会输出Java堆栈信息，还会输出C/C++堆栈信息（比如Native方法）
- -l 会打印出额外的锁信息，在发生死锁时可以用jstack -l pid来观察锁持有情况

详解

不同的JVM dump创建的方法和文件格式可能会有不同，不同版本的JVM，dump信息也会有差别；

在实际运行中，往往一次 dump的信息，不足以确认问题。建议产生三次 dump信息，如果每次 dump都指向同一个问题，基本才确定问题的典型性；

如果jstack在查看某一个JVM进程时无反应，大多是hung 进程，需要加-F选项强制dump栈信息，并且会输出[deadlock的信息](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr016.html#BABBBDIG)；

通过-l选项dump出的线程栈信息，一般如下：

 

```
"DestroyJavaVM"` `prio=10 tid=0x00030400 nid=0x2 waiting on condition [0x00000000..0xfe77fbf0]``  ``java.lang.Thread.State: RUNNABLE` `"Thread2"` `prio=10 tid=0x000d7c00 nid=0xb waiting ``for` `monitor entry [0xf36ff000..0xf36ff8c0]``  ``java.lang.Thread.State: BLOCKED (on object monitor)``    ``at Deadlock$DeadlockMakerThread.run(Deadlock.java:32)``    ``- waiting to lock <0xf819a938> (a java.lang.String)``    ``- locked <0xf819a970> (a java.lang.String)
```

 

每一个线程都通过一个空行分隔开，先打印Java方法栈，再打印出JVM 内部线程栈信息，每个线程信息的都包含一个头行，后边每行是跟着的Tread Trace信息。头行包含的信息为：线程名、如果是deamon thread则显示deamon、线程优先级、线程id（也是此线程在内存中的地址）、native线程id、[线程在dump那一刻的状态](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr034.html#BABJFBFI)、有效的地址范围。

其中线程的状态分析很重要，可以根据Java程序的执行过程遇到的问题，分析线程的状态，进而找到问题代码。比如我们可以使用ps -Lfp pid或者ps -mp pid -o THREAD, tid, time或者top -Hp pid，找出进程内最耗费CPU的线程，得到线程的pid后转换为十六进制，然后再通过jstack命令找到nid为该十六进制数的进程，查看该进程做的事情，进而找到代码得到为什么最耗费CPU的原因。

例子如下，通过top -Hp 3458找到id为3458进程的所有线程，通过按shift+T将输出按照TIME+列排序

![img](http://doc.hz.netease.com/download/attachments/80306523/image2017-6-2%2020%3A58%3A19.png?version=1&modificationDate=1496408298000&api=v2)

找到时间消耗排在前三的线程pid，将十进制pid转换为十六进制

![img](http://doc.hz.netease.com/download/attachments/80306523/image2017-6-2%2020%3A58%3A29.png?version=1&modificationDate=1496408308000&api=v2)

通过jstack -l 3458 > 3458.jstack 导出JVM栈信息

搜索nid为前边转换的十六进制数，找到该线程，可以看出，在此进程中最消耗CPU的前三甲线程名为VM Periodic Task Thread、C2 ComplierThread1、C2 ComplierThread0，状态都是waiting on condition。

![img](http://doc.hz.netease.com/download/attachments/80306523/image2017-6-2%2020%3A58%3A35.png?version=1&modificationDate=1496408314000&api=v2)

![img](http://doc.hz.netease.com/download/attachments/80306523/image2017-6-2%2020%3A58%3A44.png?version=1&modificationDate=1496408323000&api=v2)

[线程状态](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/tooldescr034.html#BABJFBFI)(显示在第二行java.lang.Thread.State: 线程状态)

| 线程状态      | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| NEW           | 线程还没开始                                                 |
| RUNNABLE      | 线程正在JVM中执行                                            |
| BLOCKED       | 因等待监视锁（monitor lock）而阻塞                           |
| WAITING       | 线程无限期地等待另一个线程执行特定的动作                     |
| TIMED_WAITING | 线程正在等待另一个线程执行操作，此操作有一个超时时间并不会无限期执行 |
| TERMINATED    | 线程已退出                                                   |

另外在头行的信息里的线程状态信息有：

   死锁，Deadlock（重点关注）

   执行中，Runnable 

   等待资源，Waiting on condition（重点关注）

   等待获取监视器，Waiting on monitor entry（重点关注）

   暂停，Suspended

   对象等待中，Object.wait() 或 TIMED_WAITING

   阻塞，Blocked（重点关注）

   停止，Parked

## jmap

命令格式

- jmap [ option ] pid
- jmap [ option ] executable core
- jmap [ option ] [server-id@]remote-hostname-or-IP

用途

- 输出共享对象内存映射或者堆内存信息
- 可查看指定可访问的进程pid、core文件或远程可调试主机
- 如果在64位机器上，需要指定选项"-J-d64”

选项

- 无选项，将输出共享对象内存映射。对于每一个加载在JVM的共享对象，将会输出起始地址、映射的大小、共享对象文件的绝对路径
- -dump:[live,]format=b,file=<filename> 将Java堆以hprof二进制格式转储为文件名,live是可选选项，如果加入此参数，只转储在堆中活着的对象。可以使用jhat工具查看转储的文件
- -finalizerinfo 输出正在等待finalization的对象信息。
- -heap 输出堆信息，包括使用何种GC算法、堆的配置、每个年代内存的使用信息
- -histo[:live] 输出堆内存中对象数目、占用内存大小、类名，如果带上live则只统计活对象。JVM内部类名前边有*号表示
- -permstat  输出永久代中类加载器的数据信息，输出：类加载器名称或地址、所加载的类的数量、已加载的类的大小、父类加载器名称或地址、是否存活（不可靠）、加载器类型
- -F 若dump时无反应，加上此选项强制dump，不支持live参数
- -Joption 给启动jps的Java启动器传入参数，比如-J-Xms48m 来设置初始堆大小为48m

详解

使用-histo[:live]选项输出时，class name列是对象类型，输出的内容有如下对应关系：

| B       | byte                |
| ------- | ------------------- |
| C       | char                |
| D       | double              |
| F       | float               |
| I       | int                 |
| J       | long                |
| Z       | boolean             |
| [       | 数组，如[I表示int[] |
| [L+类名 | 其他对象            |

## jhat

命令格式

- jhat [ options ] <heap-dump-file>

用途

- 启动一个web服务器并且解析java 转储的堆文件信息给web服务器
- 可以通过web浏览器查看转储出来的堆信息
- 支持 pre-designed 查询，可以使用OQL (Object Query Language，一种类似SQL查询语) 查询转储堆中的内容 例如：show all instances of a known class “Foo”
- 可以在启动了jhat之后，通过http://localhost:7000/oqlhelp/了解OQL

选项

- -stack false/true 关闭或打开跟踪调用栈对象的分配，默认是true
- -refs false/true 关闭或打开对象引用的跟踪。默认是true
- -port port-number 设置HTTP服务监听端口，默认是7000
- -exclude exclude-file 需要排除在外的可达对象的列表，每行一个
- -baseline baseline-dump-file  指定baseline 转储文件，对象在两个转储文件中有相同对象ID的不会被标记为“new”，其他的将标记为“new”，对于比较两个转储的堆文件很有用
- -debug int 开启debug模式，0代表不输出调试信息，数字越高输出的信息越详细
- -Joption 给启动jps的Java启动器传入参数，比如-J-Xms48m 来设置初始堆大小为48m

详解

转储堆内存到文件可以通过以下几种方式:

- jmap
- jconsole
- 给JVM加入-XX:+HeapDumpOnOutOfMemoryError参数，在OutOfMemoryError发生时生成
- hprof

## jstat

命名格式

- jstat [ generalOption | outputOptions vmid [interval[s|ms] [count]] ]

用途

- 显示Hotspot VM的性能数据

选项

  普通选项，后边不能跟其他选项

- -help 输出帮助信息
- -options 输出可以用到的输出选项

  输出选项，输出选项决定jstat输出的内容与格式

   statOption，第一个输出选项

- -class 用于查看类加载情况的统计
- -compiler 用于查看HotSpot中即时编译器编译情况的统计
- -gc 用于查看JVM中堆的垃圾收集情况的统计
- -gccapacity 用于查看HotSpot中即时编译器编译情况的统计
- -gccause 用于查看垃圾收集的统计情况（这个和-gcutil选项一样），如果有发生垃圾收集，它还会显示最后一次及当前正在发生垃圾收集的原因
- -gcnew 用于查看新生代垃圾收集的情况
- -gcnewcapacity 用于查看新生代的存储容量情况
- -gcold 用于查看老年代及持久代发生GC的情况
- -gcoldcapacity 用于查看老年代的容量
- -gcpermcapacity 用于查看持久代的容量
- -gcutil 用于查看新生代、老年代及持代垃圾收集的情况
- -printcompilation HotSpot编译方法的统计

   其他输出选项，放在statOpntion之后

- -h n 每隔n行就显示标题行
- -t在第一列加上时间戳，时间戳为JVM启动到这次采集数据时的时间
- -JjavaOption 给启动jstat的Java启动器传入参数，比如-J-Xms48m 来设置初始堆大小为48m
- 以下放在vmid之后的参数
- interval 采集时间间隔
- count 采集次数

[**不同的统计维度（****statOption****）及输出说明**](http://blog.csdn.net/fenglibing/article/details/6411951)

  **-class**

| **列名** | **说明**                 |
| -------- | ------------------------ |
| Loaded   | 加载了的类的数量         |
| Bytes    | 加载了的类的大小，单为Kb |
| Unloaded | 卸载了的类的数量         |
| Bytes    | 卸载了的类的大小，单为Kb |
| Time     | 花在类的加载及卸载的时间 |

   **-compiler**

| **列名**     | **说明**                       |
| ------------ | ------------------------------ |
| Compiled     | 编译任务执行的次数             |
| Failed       | 编译任务执行失败的次数         |
| Invalid      | 编译任务非法执行的次数         |
| Time         | 执行编译花费的时间             |
| FailedType   | 最后一次编译失败的编译类型     |
| FailedMethod | 最后一次编译失败的类名及方法名 |

   **-gc**

| **列名** | **说明**                                                     |
| -------- | ------------------------------------------------------------ |
| S0C      | 新生代中Survivor space中S0当前容量的大小（KB）               |
| S1C      | 新生代中Survivor space中S1当前容量的大小（KB）               |
| S0U      | 新生代中Survivor space中S0容量使用的大小（KB）               |
| S1U      | 新生代中Survivor space中S1容量使用的大小（KB）               |
| EC       | Eden space当前容量的大小（KB）                               |
| EU       | Eden space容量使用的大小（KB）                               |
| OC       | Old space当前容量的大小（KB）                                |
| OU       | Old space使用容量的大小（KB）                                |
| PC       | Permanent space当前容量的大小（KB）                          |
| PU       | Permanent space使用容量的大小（KB）                          |
| YGC      | 从应用程序启动到采样时发生 Young GC 的次数                   |
| YGCT     | 从应用程序启动到采样时 Young GC 所用的时间(秒)               |
| FGC      | 从应用程序启动到采样时发生 Full GC 的次数                    |
| FGCT     | 从应用程序启动到采样时 Full GC 所用的时间(秒)                |
| GCT      | T从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC |

  **-gccapacity**

| **列名** | **说明**                                     |
| -------- | -------------------------------------------- |
| NGCMN    | 新生代的最小容量大小（KB）                   |
| NGCMX    | 新生代的最大容量大小（KB）                   |
| NGC      | 当前新生代的容量大小（KB）                   |
| S0C      | 当前新生代中survivor space 0的容量大小（KB） |
| S1C      | 当前新生代中survivor space 1的容量大小（KB） |
| EC       | Eden space当前容量的大小（KB）               |
| OGCMN    | 老生代的最小容量大小（KB）                   |
| OGCMX    | 老生代的最大容量大小（KB）                   |
| OGC      | 当前老生代的容量大小（KB）                   |
| OC       | 当前老生代的空间容量大小（KB）               |
| PGCMN    | 持久代的最小容量大小（KB）                   |
| PGCMX    | 持久代的最大容量大小（KB）                   |
| PGC      | 当前持久代的容量大小（KB）                   |
| PC       | 当前持久代的空间容量大小（KB）               |
| YGC      | 从应用程序启动到采样时发生 Young GC 的次数   |
| FGC      | 从应用程序启动到采样时发生 Full GC 的次数    |

  **-gccause** **在****-gcutil****上增加的列**

 

| **列名** | **说明**                                                     |
| -------- | ------------------------------------------------------------ |
| LGCC     | 最后一次垃圾收集的原因，可能为“unknown GCCause”、“System.gc()”等 |
| GCC      | 当前垃圾收集的原因                                           |

  **-gcnew**

| **列名** | **说明**                                                     |
| -------- | ------------------------------------------------------------ |
| S0C      | 当前新生代中survivor space 0的容量大小（KB）                 |
| S1C      | 当前新生代中survivor space 1的容量大小（KB）                 |
| S0U      | S0已经使用的大小（KB）                                       |
| S1U      | S1已经使用的大小（KB）                                       |
| TT       | Tenuring threshold                                           |
| MTT      | Maximum tenuring threshold，用于表示TT的最大值。             |
| DSS      | Desired survivor size (KB).可以参考[这里](http://blog.csdn.net/yangjun2/article/details/6542357) |
| EC       | Eden space当前容量的大小（KB）                               |
| EU       | Eden space已经使用的大小（KB）                               |
| YGC      | 从应用程序启动到采样时发生 Young GC 的次数                   |
| YGCT     | 从应用程序启动到采样时 Young GC 所用的时间(单位秒)           |

  **-gcnewcapacity**

| **列名** | **说明**                                   |
| -------- | ------------------------------------------ |
| NGCMN    | 新生代的最小容量大小（KB）                 |
| NGCMX    | 新生代的最大容量大小（KB）                 |
| NGC      | 当前新生代的容量大小（KB）                 |
| S0CMX    | 新生代中SO的最大容量大小（KB）             |
| S0C      | 当前新生代中SO的容量大小（KB）             |
| S1CMX    | 新生代中S1的最大容量大小（KB）             |
| S1C      | 当前新生代中S1的容量大小（KB）             |
| ECMX     | 新生代中Eden的最大容量大小（KB）           |
| EC       | 当前新生代中Eden的容量大小（KB）           |
| YGC      | 从应用程序启动到采样时发生 Young GC 的次数 |
| FGC      | 从应用程序启动到采样时发生 Full GC 的次数  |

  **-gcold**

| **列名** | **说明**                                                     |
| -------- | ------------------------------------------------------------ |
| PC       | 当前持久代容量的大小（KB）                                   |
| PU       | 持久代使用容量的大小（KB）                                   |
| OC       | 当前老年代容量的大小（KB）                                   |
| OU       | 老年代使用容量的大小（KB）                                   |
| YGC      | 从应用程序启动到采样时发生 Young GC 的次数                   |
| FGC      | 从应用程序启动到采样时发生 Full GC 的次数                    |
| FGCT     | 从应用程序启动到采样时 Full GC 所用的时间(单位秒)            |
| GCT      | 从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC |

  **-gcoldcapacity**

| **列名** | **说明**                                                     |
| -------- | ------------------------------------------------------------ |
| OGCMN    | 老生代的最小容量大小（KB）                                   |
| OGCMX    | 老生代的最大容量大小（KB）                                   |
| OGC      | 当前老生代的容量大小（KB）                                   |
| OC       | 当前新生代的空间容量大小（KB）                               |
| YGC      | 从应用程序启动到采样时发生 Young GC 的次数                   |
| FGC      | 从应用程序启动到采样时发生 Full GC 的次数                    |
| FGCT     | 从应用程序启动到采样时 Full GC 所用的时间(单位秒)            |
| GCT      | 从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC |

  **-gcpermcapacity**

| **列名** | **说明**                                                     |
| -------- | ------------------------------------------------------------ |
| PGCMN    | 持久代的最小容量大小（KB）                                   |
| PGCMX    | 持久代的最大容量大小（KB）                                   |
| PGC      | 当前持久代的容量大小（KB）                                   |
| PC       | 当前持久代的空间容量大小（KB）                               |
| YGC      | 从应用程序启动到采样时发生 Young GC 的次数                   |
| FGC      | 从应用程序启动到采样时发生 Full GC 的次数                    |
| FGCT     | 从应用程序启动到采样时 Full GC 所用的时间(单位秒)            |
| GCT      | 从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC |

  **-gcutil**

| **列名** | **说明**                                                     |
| -------- | ------------------------------------------------------------ |
| S0       | Heap上的 Survivor space 0 区已使用空间的百分比               |
| S1       | Heap上的 Survivor space 1 区已使用空间的百分比               |
| E        | Heap上的 Eden space 区已使用空间的百分比                     |
| O        | Heap上的 Old space 区已使用空间的百分比                      |
| P        | Perm space 区已使用空间的百分比                              |
| YGC      | 从应用程序启动到采样时发生 Young GC 的次数                   |
| YGCT     | 从应用程序启动到采样时 Young GC 所用的时间(单位秒)           |
| FGC      | 从应用程序启动到采样时发生 Full GC 的次数                    |
| FGCT     | 从应用程序启动到采样时 Full GC 所用的时间(单位秒)            |
| GCT      | 从应用程序启动到采样时用于垃圾回收的总时间(单位秒)，它的值等于YGC+FGC |

  **-printcompilation**

| **列名** | **说明**                                                     |
| -------- | ------------------------------------------------------------ |
| Compiled | 编译任务执行的次数                                           |
| Size     | 方法的字节码所占的字节数                                     |
| Type     | 编译类型                                                     |
| Method   | 指定确定被编译方法的类名及方法名，类名中使名“/”而不是“.”做为命名分隔符，方法名是被指定的类中的方法，这两个字段的格式是由HotSpot中的“-**XX:+PrintComplation**”选项确定的。 |

## **hprof**

hprof是java命令的一个参数，能够展现CPU使用率，统计堆内存使用情况

命令格式

- java -agentlib:hprof[=options] ToBeProfiledClass
- java -Xrunprof[:options] ToBeProfiledClass
- javac -J-agentlib:hprof[=options] ToBeProfiledClass

选项

| 选项名和值              | 描述                           | 默认值           |
| ----------------------- | ------------------------------ | ---------------- |
| heap=dump\|sites\|all   | 堆内存信息                     | all              |
| cpu=samples\|times\|old | CPU使用情况                    | off              |
| monitor=y\|n            | 观察线程争用                   | n                |
| format=a\|b             | 文本或二进制输出               | a                |
| file=<file>             | 写入文件路径,默认当前路径下    | java.hprof[.txt] |
| net=<host>:<port>       | 通过socket发送二进制数据       | Off              |
| depth=<size>            | 栈的深度                       | 4                |
| interval=<ms>           | 采集间隔时间                   | 10ms             |
| cutoff=<value>          | 输出截断点                     | 0.0001           |
| lineno=y\|n             | 输出行号                       | y                |
| thread=y\|n             | 是否栈跟踪                     | n                |
| doe=y\|n                | dump on exit?                  | y                |
| msa=y\|n                | Solaris micro state accounting | n                |
| force=y\|n              | 强制输出到文件                 | y                |
| verbose                 | 尽可能多的输出信息             | y                |

 多个选项通过逗号隔开

例子

- java -agentlib:hprof=cpu=samples,interval=20,depth=3 Hello
- 上面每隔20毫秒采样CPU消耗信息，堆栈深度为3，生成的profile文件名称是java.hprof.txt，在当前目录。

CPU Usage Times Profiling(cpu=times)的例子，它相对于CPU Usage Sampling Profile能够获得更加细粒度的CPU消耗信息，能够细到每个方法调用的开始和结束，它的实现使用了字节码注入技术（BCI):

- javac -J-agentlib:hprof=cpu=times Hello.java

Heap Allocation Profiling(heap=sites)的例子：

- javac -J-agentlib:hprof=heap=sites Hello.java

Heap Dump(heap=dump)的例子，它比上面的Heap Allocation Profiling能生成更详细的Heap Dump信息：

- javac -J-agentlib:hprof=heap=dump Hello.java

虽然在JVM启动参数中加入-Xrunprof:heap=sites参数可以生成CPU/Heap Profile文件，但对JVM性能影响非常大，不建议在线上服务器环境使用。

 

 

参考：

[JVM性能调优监控工具jps、jstack、jmap、jhat、jstat、hprof使用详解](https://my.oschina.net/feichexia/blog/196575)

[JDK内置工具使用](http://blog.csdn.net/fenglibing/article/details/6411999)

[jstack和线程dump分析](http://jameswxx.iteye.com/blog/1041173)

[JDK Tools and Utilities](http://docs.oracle.com/javase/7/docs/technotes/tools/)