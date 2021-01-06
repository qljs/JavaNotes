



## 1. Jmap

### 1.1 Jmap -histo：查看实例数量和占用内存。

- num：序号
- instaces：实例数量
- bytes：占用空间大小
- class name：类名称，[C：数组char[]，[I：数组int[]，[B：数组byte[]

![image-20201230105230157](..\images\jvm\jmap.png)



### 1.2 Jmap -heap：查看堆信息

![image-20201230111741803](..\images\jvm\jmap_heap.png)



### 1.3 Jmap -dump：查看堆内存

通过`jmap -dump:format=b,file= filename`可以到处堆内存dump文件，通过参数`-XX:HeapDumpOnOutOfMemoryError`和`-XX:HeapDumpPath=filepath`参数可以指定发生OOM时到处dump文件，通过**jvisualvm**命令工具，可以分析dump文件。

![image-20201230113128197](..\images\jvm\jmap-dump.png) 



## 2. Jstack

通过jstack 可以查看死锁问题。

- Thread-1：线程名；
- prio = 5：线程优先级；
- tid=0x000002453aff4000：线程id；
- nid=0x2088：线程对应的本地线程 nid；
- java.lang.Thread.State: BLOCKED：线程状态。

![image-20201230114229975](..\images\jvm\jstack.png)



> #### jstack找出占用cpu最高的堆栈信息

1. 使用命令top -p  ，显示你的java进程的内存情况，pid是你的java进程号，比如4977；
2. 按H，获取每个线程的内存情况；
3. 找到内存和cpu占用最高的线程tid，比如4977；
4. 转为十六进制得到 0x1371 ,此为线程id的十六进制表示；
5. 执行 jstack 4977|grep -A 10 1371，得到线程堆栈信息中1371这个线程所在行的后面10行；
6. 查看对应的堆栈信息找出可能存在问题的代码。



## 3. JInfo

查看正在运行的Java应用程序扩展参数。

`jinfo -flags`：查看JVM参数

![image-20201230141424173](..\images\jvm\jinfo_flags.png)



## 4. Jstat

jstat命令可查看堆内存各部分使用量，以及加载类的数量。

命令格式：jstat [-命令选项] [vmid] [时间间隔(毫秒)] [查询次数]

**垃圾回收统计：jstat -gc**

![image-20201230145829147](..\images\jvm\jstat.png)

- S0C：第一个幸存区（survivor）大小
- S1C：第二个幸存区大小
- S0U：第一个幸存区使用大小
- S1U：第二个幸存区使用大小
- EC：eden区大小
- EU：eden区使用大小
- OC：老年代大小
- OU：老年代使用大小
- MC：元空间大小
- MU：元空间使用大小
- CCSC：压缩类空间大小
- CCSU：压缩类空间使用大小
- YGC：年轻代垃圾回收次数
- YGCT：年轻代垃圾回收总耗时，单位s
- FGC：Full GC 次数
- FGCT：Full GC 耗时
- GCT：垃圾回收总耗时



**堆内存统计：jstat -gccapacity**

![](..\images\jvm\jstat-gcc.png)

- NGCMN：新生代最小容量
- NGCMX：新生代最大容量 
- NGC：当前新生代容量 
- S0C：第一个幸存区大小 
- S1C：第二个幸存区的大小 
- EC：eden区的大小 
- OGCMN：老年代最小容量 
- OGCMX：老年代最大容量 
- OGC：当前老年代大小 
- OC：当前老年代大小 
- MCMN：最小元数据容量 
- MCMX：最大元数据容量 
- MC：当前元数据空间大小
- CCSMN：最小压缩类空间大小 
- CCSMX：最大压缩类空间大小 
- CCSC：当前压缩类空间大小 
- YGC：年轻代gc次数 
- FGC：Full GC次数



[orcale 工具命令及解释](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/index.html)



