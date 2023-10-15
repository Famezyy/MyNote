# 番外章_Java虚拟机参数

- `-Xnoclassgc`

  是否对类型进行回收。

- `-verbose:class`、`-XX:+TraceClassLoading`、`-XX:+TraceClassUnLoading`

  查看类加载和卸载信息，其中`-verbose:-class`和`-XX:+TraceClassLoading`可以在 Product 版的虚拟机中使用，`-XX:+TranceClassUnLoading`参数需要 FastDebug 版的虚拟机支持。

- `-XX:+UseCondCardMark`

  是否开启卡表更新的条件判断，用于减少并发垃圾回收器的写屏障操作，以提高性能。这个参数一般不需要手动设置，因为它通常会被垃圾收集器和 JVM 内部根据系统和硬件情况自动启用或禁用。
  
- `-XX:+HeapDumpOnOutOfMemoryError [-XX:HeapDumpPath=文件路径[/文件名]]`

  当发生 OOM 时输出堆转储文件。
  
- `-Xmx30m`、`-Xmx30m`

  分别用来设置堆的最小内存和最大内存。
  
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
  
- `-XX:MaxDirectMemorySize=10m`

  设置本地直接内存的最大容量。
  
- `-XX:MinMetaspaceFreeRatio`

  作用是在垃圾收集之后控制最小的元空间剩余容量的百分比，可减少因为元空间不足导致的垃圾收集的频率。类似的还有`-XX:MaxMetaspaceFreeRatio`，用于控制最大的元空间剩余容量的百分比。
  
  
