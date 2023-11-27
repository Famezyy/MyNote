# 番外章_Java虚拟机参数

## 1.常用参数

### 1.1 通用参数

- `-XX:+PrintFlagsFinal`
  
  使用`java -XX:+PrintFlagsFinal`可以打印出当前版本的 JVM 可选参数及默认值。

- `-Xnoclassgc`

  是否对类型进行回收。

- `-verbose:class`、`-XX:+TraceClassLoading`、`-XX:+TraceClassUnLoading`

  查看类加载和卸载信息，其中`-verbose:-class`和`-XX:+TraceClassLoading`可以在 Product 版的虚拟机中使用，`-XX:+TranceClassUnLoading`参数需要 FastDebug 版的虚拟机支持。

- `-XX:+UseCondCardMark`

  是否开启卡表更新的条件判断，用于减少并发垃圾回收器的写屏障操作，以提高性能。这个参数一般不需要手动设置，因为它通常会被垃圾收集器和 JVM 内部根据系统和硬件情况自动启用或禁用。
  
- `-XX:+HeapDumpOnOutOfMemoryError [-XX:HeapDumpPath=文件路径[/文件名]]`

  当发生 OOM 时输出堆转储文件。
  
- `-XX:+HeapDumpOnCtrlBreak [-XX:HeapDumpPath=文件路径[/文件名]]`
  
  使用 ctrl+Break 键让虚拟机生成堆转储快照文件。

- `-Xms30m`、`-Xmx30m`

  分别用来设置初始堆内存大小和最大堆内存大小。
  
- `-XX:PermSize=30m`、`-XX:MaxPermSize=30m`

  分别用来设置永久代初始容量和最大容量，适用于 JDK 8 之前。
  
- `-XX:MaxMetaspaceSize=30m`

  设置元空间最大容量，适用于 JDK 8 之后。
  
- `-XX:MetaspaceSize=30m`

  设置元空间初始容量，适用于 JDK 8 之后。

- `-XX:+/-UseTLAB`

  是否开启 TLAB，HotSpot 中默认开启。

- `-Xss30m`

  设置栈的内存。

- `-Xoss30m`

  设置本地方法栈的内存，不适用于 HotSpot。

- `-Xmn30m`
  
  设置新生代大小。

- `-XX:MaxDirectMemorySize=10m`

  设置本地直接内存的最大容量。
  
- `-XX:+UseCompressedOops`
  
  开启压缩指针。

### 1.2 垃圾收集相关参数

|参数（`-XX:+...`）|描述|
|--|--|
|`MinMetaspaceFreeRatio`|控制垃圾收集之后至少保留的元空间剩余容量百分比，适当提高可减少因为元空间不足导致的垃圾收集的频率。类似的还有`-XX:MaxMetaspaceFreeRatio`，用于控制最大的元空间剩余容量的百分比。|
|`UseSerialGC`|虚拟机运行在 Client 模式下的默认值，打开此开关后，使用 Serial + Serial Old 的收集器组合进行内存回收|
|`UseParNewGC`|打开此开关后，使用 ParNew + Serial Old 的收集器组合进行内存回收，JDK 9 后不再支持|
|`UseConcMarkSweepGC`|打开此开关后，使用 ParNew + CMS + Serial Old 的收集器组合进行内存回收。Serial Old 收集器将作为 CMS 收集器出现 "Concurrent Mode Failure" 失败后的后备收集器使用|
|`UseParallelGC`|JDK 9 之前虚拟机运行在 Server 模式下的默认值，打开此开关后，使用 Parallel Scavenge + Serial Old（PS MarkSweep）的收集器组合进行内存回收|
|`UseParallelOldGC`|打开此开关后，使用 Parallel Scavenge + Parallel Old 的收集器组合进行内存回收|
|`SurvivorRatio`|新生代中 Eden 区域和 Survivor 区域的容量比值，默认为 8，代表 Eden:Survivor=8:1|
|`PretenureSizeThreshold=n`|直接晋升到老年代的对象大小，设置后大于这个参数的对象将直接在老年代分配，单位为字节数。这个参数只对 Serial 和 ParNew 两款新生代收集器有效，其他收集器如 Parallel Scavenge 并不支持这个参数|
|`MaxTenuringThreshold=n`|晋升到老年代的对象年龄。每个对象在坚持过一次 Minor GC 后年龄就会加 1，当超过这个参数值后就进入老年代，默认 15，最高也只能 15，因为对象头中分配的对象年龄占 2 个字节|
|`UseAdaptiveSizePolicy`|动态调整 Java 堆中各个区域的大小以及进入老年代的年龄|
|`HandlePromotionFailure`|是否允许分配担保失败，即老年代的剩余空间不足以应付新生代的整个 Eden 和 Survivor 区的所有对象都存活的极端情况|
|`ParallelGCThreads`|设置并行 GC 时进行内存回收的线程数|
|`GCTimeRatio`|GC 时间占总时间的比率，允许的值是一个大于 0 小于 100 的整数，默认 99，即允许 1% 的 GC 时间，仅在使用 Parallel Scavenge 收集器时生效|
|`MaxGCPauseMillis`|设置 GC 的最大停顿时间，仅在使用 Parallel Scavenge 和 G1 收集器时生效（G1 中默认 200）。允许的值是一个大于 0 的毫秒数，收集器将尽力保证内存回收花费的时间不超过用户设定值。|
|`CMSInitiatingOccupancyFraction`|设置 CMS 收集器在老年代空间被使用多少后触发垃圾收集。默认 68%，JDK 6 后默认 92%，注意不能设置得太高，否则预留的可用内存太少，容易导致并发失败产生，此时将不得不启用 Serial Old 收集器来重新进行老年代的垃圾收集。仅在使用 CMS 收集器时生效|
|`UseCMSCompactAtFullCollection`|设置 CMS 收集器在完成垃圾收集后是否要进行一次内存碎片整理。仅在使用 CMS 收集器时生效，从 JDK 9 开始废弃|
|`CMSFullGCsBeforeCompaction`|设置 CMS 收集器在进行若干次垃圾收集后再启动一次内存碎片整理。默认 0，即每次进入 Full GC 都会进行碎片整理。仅在使用 CMS 收集器时生效，从 JDK 9 开始废弃|
|`UseG1GC`|使用 G1 收集器，是 JDK 9 后的 Server 模式默认值|
|`G1HeapRegionSize=n`|用来设定 G1 收集器中的 Region 大小，范围是 1MB～32MB，且应为 2 的 N 次幂|
|`G1NewSizePercent`|G1 中新生代最小值，默认是 5%|
|`G1MaxNewSizePercent`|G1 中新生代最大值，默认值是 60%|
|`ParallelGCThreads`|用户线程冻结期间并行执行的收集器线程数|
|`ConcGCThreads=n`|并发标记、并发整理的执行线程数，对不同的收集器，根据其能够并发的阶段，有不同的含义|
|`InitiatingHeapOccupancyPercent`|设置触发标记周期的 Java 堆占用率阈值。默认值是 45%。这里的 Java 堆占比指的是 non_young_capacity_bytes，包括 old + humongous|
|`UseShenandoahGC`|使用 Shenandoah 收集器。这个选项在 OracleJDK 中不被支持，只能在 OpenJDK 12 或者某些支持 Shenandoah 的 Backport 发行版本使用。目前仍然要配合`-XX:+UnlockExperimentalVMOptions`使用|
|`ShenandoahGCHeuristics`|Shenandoah 何时启动一次 GC 过程，其可选值有 adaptive、static、compact、passive、aggressive|
|`UseZGC`|使用 ZGC 收集器，目前仍然要配合`-XX:+UnlockExperimentalVMOptions`使用|
|`UseNUMA`|启用 NUMA 内存分配支持，目前只有 Parallel 和 ZGC 支持，以后 G1 收集器可能也会支持|

## 2.JDK9前后日志参数变化

|JDK 9 前日志参数|JDK 9 后配置形式|
|--|--|
|`G1PrintHeapRegions`|`Xlog:gc+region=trace`|
|`G1PrintRegionLivenessInfo`|`Xlog:gc+liveness=trace`|
|`G1SummarizeConcMark`|`Xlog:gc+marking=trace`|
|`G1SummarizeRSetStats`|`Xlog:gc+remset*=trace`|
|`GCLogFileSize,NumberOfGCLogFiles,UseGCLog File Rotation`|`Xlog:gc*:file=<file>::filecount=<count>,filesize=<file size in kb>`|
|`PrintAdaptiveSizePolicy`|`Xlog:gc+ergo*=trace`|
|`PrintClassHistogramAfterFullGC`|`Xlog:classhisto*=trace`|
|`PrintClassHistogramBeforeFullGC`|`Xlog:classhisto*=trace`|
|`PrintGCApplicationConcurrentTime`|`Xlog:safepoint`|
|`PrintGCApplicationStoppedTime`|`Xlog:safepoint`|
|`PrintGCDateStamps`|使用 time 修饰器|
|`PrintGCTaskTimeStamps`|`Xlog:gc+task=trace`|
|`PrintGCTimeStamps`|使用 uptime 修饰器|
|`PrintHeapAtGC`|`Xlog:gc+heap=debug`|
|`PrintHeapAtGCExtended`|`Xlog:gc+heap=trace`|
|`PrintJNIGCStalls`|`Xlog:gc+jni=debug`|
|`PrintOldPLAB`|`Xlog:gc+plab=trace`|
|`PrintParallelOldGCPhaseTimes`|`Xlog:gc+phases=trace`|
|`PrintPLAB`|`Xlog:gc+plab=trace`|
|`PrintPromotionFailure`|`Xlog:gc+promotion=debug`|
|`PrintReferenceGC`|`Xlog:gc+ref=debug`|
|`PrintStringDeduplicationStatistics`|`Xlog:gc+stringdedup`|
|`PrintTaskqueue`|`Xlog:gc+task+stats=trace`|
|`PrintTenuringDistribution`|`Xlog:gc+age=trace`|
|`PrintTerminationStats`|`Xlog:gc+task+stats=debug`|
|`PrintTLAB`|`Xlog:gc+tlab=trace`|
|`TraceAdaptiveGCBoundary`|`Xlog:heap+ergo=debug`|
|`TraceDynamicGCThreads`|`Xlog:gc+task=trace`|
|`TraceMetadataHumongousAllocation`|`Xlog:gc+metaspace+alloc=debug`|
|`G1TraceConcRefinement`|`Xlog:gc+refine=debug`|
|`G1TraceEagerReclaimHumongousObjects`|`Xlog:gc+humongous=debug`|
|`G1TraceStringSymbolTableScrubbing`|`Xlog:gc+stringtable=trace`|