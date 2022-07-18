## 三、Fork Join

Fork Join 在 JDK 1.7，并行执行任务！提高效率

适用于处理大数据：Map Reduce （把大任务拆分为小任务）

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125415196-4c6d5401aa6b556d74dbb55582286e7f-a6c5cf.png" alt="image-20220222125415196" style="zoom:67%;" />

> Fork Join 特点：工作窃取

这个里面维护的都是双端队列，B 执行完后会执行 A 的任务

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125427720-02e5bc989b8a0879a9b5ceb3e1e276c0-4fd673.png" alt="image-20220222125427720" style="zoom:80%;" />



<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125459873-78d6a09e76fc3bc5d217d88f095ddae8-b5dc5f.png" alt="image-20220222125459873" style="zoom: 67%;" />

```java
/** 
 * 求和计算的任务！
 * 如何使用 forkjoin 
 * 1、forkjoinPool 通过它来执行 
 * 2、计算任务 forkjoinPool.execute(ForkJoinTask task) 
 * 3. 计算类要继承 RecursiveTask(递归任务，有返回值的) 
 */
public class ForkJoinDemo extends RecursiveTask<Long> {
    private Long start;  // 1
    private Long end; // 1990900000
    private Long temp = 10000L; // 临界值
    
    public ForkJoinDemo(Long start, Long end) {        
        this.start = start;        
        this.end = end;    
    }
    
    // 计算方法    
    @Override    
    protected Long compute() {
        // 小于临界值则使用 for 循环执行
        if ((end - start) < temp){            
            Long sum = 0L;            
            for (Long i = start; i <= end; i++) {                
                sum += i;            
            }            
            return sum;        
        }else {     
            // forkjoin 递归            
            long middle = start + (end - start) / 2; 
            // 中间值            
            ForkJoinDemo task1 = new ForkJoinDemo(start, middle);
            task1.fork(); // 拆分任务，把任务压入线程队列
            ForkJoinDemo task2 = new ForkJoinDemo(middle+1, end);
            task2.fork(); // 拆分任务，把任务压入线程队列
            return task1.join() + task2.join();        
        }    
    }
}
```

测试代码：

```java
/** 
 * 同一个任务，别人效率高你几十倍！ 
 */
public class Test {    
   public static void main(String[] args) throws ExecutionException, InterruptedException {
        test1(); // 510
        test2(); // 255
        test3(); // 272
    }
    // 普通程序员
    public static void test1(){
        long sum = 0L;
        long start = System.currentTimeMillis();
        for (long i = 1L; i <= 10_0000_0000; i++) {
            sum += i;
        }
        long end = System.currentTimeMillis();
        System.out.println("sum="+sum+" 时间："+(end-start));
    }
    // 会使用ForkJoin
    public static void test2() throws ExecutionException, InterruptedException {
        long start = System.currentTimeMillis();
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        ForkJoinTask<Long> task = new ForkJoinDemo(0L, 10_0000_0000L);
        // 提交任务
        ForkJoinTask<Long> submit = forkJoinPool.submit(task);
        Long sum = submit.get();// 获得结果
        long end = System.currentTimeMillis();
        System.out.println("sum="+sum+" 时间："+(end-start));
    }
	// stream parallel
    public static void test3(){
        long start = System.currentTimeMillis();
        // Stream并行流
        long sum = LongStream.rangeClosed(0L, 10_0000_0000L) // 计算范围 rangeClosed -> 后端闭合
                .parallel() // 并行计算
                .reduce(0, Long::sum); // 输出结果
        long end = System.currentTimeMillis();
        System.out.println("sum="+ sum + "时间："+(end-start));
    }
}

class ForkJoinDemo extends RecursiveTask<Long> {
    private final long start;  // 1
    private final long end; // 1990900000

    public ForkJoinDemo(Long start, Long end) {
        this.start = start;
        this.end = end;
    }

    // 计算方法
    @Override
    protected Long compute() {
        // 小于临界值则使用 for 循环执行
        // 临界值
        long temp = 10000L;
        if ((end - start) < temp){
            long sum = 0L;
            for (long i = start; i <= end; i++) {
                sum += i;
            }
            return sum;
        }else {
            // forkjoin 递归
            long middle = start + (end - start) / 2;
            // 中间值
            ForkJoinDemo task1 = new ForkJoinDemo(start, middle);
            task1.fork(); // 拆分任务，把任务压入线程队列
            ForkJoinDemo task2 = new ForkJoinDemo(middle+1, end);
            task2.fork(); // 拆分任务，把任务压入线程队列
            return task1.join() + task2.join();
        }
    }
}
```

---

## 四、CompletableFuture

[引用](./CompletableFuture.md)

### 总结

|                        | Futrue                                  | FutureTask                                                   | CompletionService                                            | **CompletableFuture**                               |
| ---------------------- | --------------------------------------- | :----------------------------------------------------------- | ------------------------------------------------------------ | --------------------------------------------------- |
| **原理**               | Futrue接口                              | 接口RunnableFuture的唯一实现类，RunnableFuture接口继承自Future<V>+Runnable: | 内部通过阻塞队列+FutureTask接口                              | JDK8实现了Future<T>, CompletionStage<T>2个接口      |
| **多任务并发执行**     | 支持                                    | 支持                                                         | 支持                                                         | 支持                                                |
| **获取任务结果的顺序** | 支持任务完成先后顺序                    | 未知                                                         | 支持任务完成的先后顺序                                       | 支持任务完成的先后顺序                              |
| **异常捕捉**           | 自己捕捉                                | 自己捕捉                                                     | 自己捕捉                                                     | 源生API支持，返回每个任务的异常                     |
| **建议**               | CPU高速轮询，耗资源，可以使用，但不推荐 | 功能不对口，并发任务这一块多套一层，不推荐使用。             | 推荐使用，没有JDK8**CompletableFuture**之前最好的方案，没有质疑 | **API极端丰富，配合流式编程，速度飞起，推荐使用！** |

---

## 五、JMM

**Volatile** 是 Java 虚拟机提供**轻量级的同步机制**，类似于 **synchronized** 但是没有其强大。

1、保证可见性

**2、不保证原子性**

3、防止指令重排

### 什么是JMM

JMM ： Java内存模型，不存在的东西，概念！约定！

**关于JMM的一些同步的约定：**

1、线程解锁前，必须把共享变量**立刻**刷回主存。

2、线程加锁前，必须读取主存中的最新值到工作内存中！

3、加锁和解锁是同一把锁。

线程 **工作内存** 、**主内存**

**8 种操作：**

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220126190739728-9496284eb2158619e174236e3e03cd59-a0565e.png" alt="image-20220126190739728" style="zoom: 67%;" />

**内存交互操作有8种，虚拟机实现必须保证每一个操作都是原子的，不可在分的（对于double和long类型的变量来说，load、store、read和 write 操作在某些平台上允许例外）**

- lock （锁定）：作用于主内存的变量，把一个变量标识为线程独占状态
- unlock （解锁）：作用于主内存的变量，它把一个处于锁定状态的变量释放出来，释放后的变量才可以被其他线程锁定
- read （读取）：作用于主内存变量，它把一个变量的值从主内存传输到线程的工作内存中，以便随后的load动作使用
- load （载入）：作用于工作内存的变量，它把read操作从主存中变量放入工作内存中
- use （使用）：作用于工作内存中的变量，它把工作内存中的变量传输给执行引擎，每当虚拟机遇到一个需要使用到变量的值，就会使用到这个指令
- assign （赋值）：作用于工作内存中的变量，它把一个从执行引擎中接受到的值放入工作内存的变量副本中
- store （存储）：作用于主内存中的变量，它把一个从工作内存中一个变量的值传送到主内存中，以便后续的write使用
- write （写入）：作用于主内存中的变量，它把store操作从工作内存中得到的变量的值放入主内存的变量中

**JMM 对这八种指令的使用，制定了如下规则：**

- 不允许 read 和 load、store 和 write 操作之一单独出现。即使用了 read 必须 load，使用了 store 必须 write
- 不允许线程丢弃他最近的 assign 操作，即工作变量的数据改变了之后，必须告知主存
- 不允许一个线程将没有 assign 的数据从工作内存同步回主内存
- 一个新的变量必须在主内存中诞生，不允许工作内存直接使用一个未被初始化的变量。就是怼变量实施 use、store 操作之前，必须经过 assign 和 load 操作
- 一个变量同一时间只有一个线程能对其进行 lock。多次 lock 后，必须执行相同次数的 unlock 才能解锁
- 如果对一个变量进行 lock 操作，会清空所有工作内存中此变量的值，在执行引擎使用这个变量前，必须重新 load 或 assign 操作初始化变量的值
- 如果一个变量没有被 lock，就不能对其进行 unlock 操作。也不能 unlock 一个被其他线程锁住的变量
- 对一个变量进行 unlock 操作之前，必须把此变量同步回主内存

==问题： 程序不知道主内存的值已经被修改过了==

---

## 六、Volatile

### 1 保证可见性

```java
public class JMMDemo {    
    // 不加 volatile 程序就会死循环！    
    // 加 volatile 可以保证可见性    
    private volatile static int num = 0;
    public static void main(String[] args) {
        // main
        new Thread(()->{
            // 线程 1 对主内存的变化不知道的
            while (num==0){
            }
        }).start();
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        num = 1;
        System.out.println(num);
    }
}
```

### 2 不保证原子性

原子性：不可分割

线程 A 在执行任务的时候，不能被打扰的，也不能被分割。要么同时成功，要么同时失败。

```java
// volatile 不保证原子性
public class VDemo02 {
    // volatile 不保证原子性
    // 原子类的 Integer
    private volatile static AtomicInteger num = new AtomicInteger();
    public static void add(){
        // num++ 不是一个原子性操作
        // AtomicInteger + 1 方法，使用底层的 CAS
        num.getAndIncrement();
        
    }
    public static void main(String[] args) {
        //理论上num结果应该为 2 万
        for (int i = 1; i <= 20; i++) {
            new Thread(()->{
                for (int j = 0; j < 1000 ; j++) {
                    add();
                }
            }).start();
        }
        // 判断只要剩下的线程不大于 2 个，就说明 20 个创建的线程已经执行结束
        // Java 默认有 main gc 2 个线程
        while (Thread.activeCount() > 2){
           // 线程礼让，把自己的 CPU 调度让给其他线程
            Thread.yield();
        }
        System.out.println(Thread.currentThread().getName()+ " " + num);
    }
}
```

**如果不加** **lock** **和** **synchronized，怎么样保证原子性**

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125521757-a1b7fab39ecbe53e2fc94aa9b3fd54e9-4ad4bb.png" alt="image-20220222125521757" style="zoom: 67%;" />

**使用原子类，解决原子性问题**

```java
// volatile 不保证原子性
// 原子类的 Integer
private volatile static AtomicInteger num = new AtomicInteger();
public static void add(){
    // num++; // 不是一个原子性操作
    num.getAndIncrement(); // AtomicInteger + 1 方法，使用底层的 CAS
}
```

> 底层使用了 Unsafe 类，Unsafe 类是一个很特殊的存在，这些类的底层都直接和操作系统挂钩，在内存中修改值

### 3 禁止指令重排

什么是指令重排？：**我们写的程序，计算机并不是按照你写的那样去执行的。**

源代码 -- 编译器优化的重排 -- 指令并行也可能会重排 -- 内存系统也会重排 -- 执行

**处理器在执行指令重排的时候，会考虑：数据之间的依赖性**

```java
int x = 1; // 1
int y = 2; // 2
x = x + 5; // 3
y = x * x; // 4
```

我们所期望的：1-2-3-4 但是可能执行的时候会变成 2-1-3-4 或者 1-3-2-4，但是不可能是 4-1-2-3！

可能造成影响得到不同的结果：

前提：a b x y 这四个值默认都是 0

| 线程A | 线程B |
| :---: | :---: |
| x = a | y = b |
| b = 1 | a = 2 |

正常的结果：x = 0；y = 0，但是可能由于指令重排出现以下结果：

| 线程A | 线程B |
| :---: | :---: |
| b = 1 | a = 2 |
| x = a | y = b |

指令重排导致的诡异结果： x = 2；y = 1

**volatile** 可以避免指令重排，内存屏障。CPU指令。作用：

1. 保证特定操作的执行顺序！
2. 可以保证某些变量的内存可见性（利用这些特性 **volatile** 实现了可见性）

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125535460-bff1f166d52c45f0bbbe96f0f3215ee0-57dcb6.png" alt="image-20220222125535460" style="zoom: 67%;" />

### 总结

> volatile 是可以保证可见性。不能保证原子性，由于内存屏障，可以保证避免指令重排的现象产生！
>
> volatile 内存屏障在单例模式中使用的最多！

---

## 七、彻底玩转单例模式

**饿汉式** **DCL懒汉式**，深究！

### 1 饿汉式

```java
// 饿汉式单例
public class Hungry {
    // 可能会浪费空间
    private byte[] data1 = new byte[1024*1024];
    private byte[] data2 = new byte[1024*1024];
    private byte[] data3 = new byte[1024*1024];
    private byte[] data4 = new byte[1024*1024];
    private Hungry(){
    }
    private final static Hungry HUNGRY = new Hungry();
    public static Hungry getInstance(){
        return HUNGRY;
    }
}
```

### 2 DCL懒汉式

```java
// 懒汉式单例
// 道高一尺，魔高一丈
public class LazyMan {
    private static boolean qinjiang = false; // 标志位
    // 单例不安全，因为反射可以破坏单例，如下解决这个问题：
    // 但仍然是不安全的，可以反射获取 qinjiang 的值并改为 false
    private LazyMan(){
        synchronized (LazyMan.class){
            if (csp == false){
                csp = true;            
            }else {                
                throw new RuntimeException("不要试图使用反射破坏异常");            
            }        
        }    
    }  
    
    /**     
    * 计算机指令执行顺序：     
    * 1. 分配内存空间     
    * 2、执行构造方法，初始化对象     
    * 3、把这个对象指向这个空间     
    *     
    * 期望顺序是：123     
    * 特殊情况下实际执行：132  ===>  此时 A 线程没有问题     
    * 若 B 线程在指向空间后执行 lazyMan=null 则会 return lazyMan
    * 但此时 lazyMan 还没有完成构造
    */
    // 原子性操作：避免指令重排
    private volatile static LazyMan lazyMan;
    // 双重检测锁模式的懒汉式单例 DCL 懒汉式    
    public static LazyMan getInstance(){        
        if (lazyMan==null){            
            synchronized (LazyMan.class){                
                if (lazyMan==null){                    
                    lazyMan = new LazyMan(); // 不是一个原子性操作
                }            
            }        
        }        
        return lazyMan;    
    }
    
    // 反射！    
    public static void main(String[] args) throws Exception {        
        //LazyMan instance = LazyMan.getInstance();        
        Field qinjiang = LazyMan.class.getDeclaredField("qinjiang");        
        qinjiang.setAccessible(true);	
        Constructor<LazyMan> declaredConstructor = LazyMan.class.getDeclaredConstructor(null);        
        declaredConstructor.setAccessible(true);        
        LazyMan instance = declaredConstructor.newInstance();        
        qinjiang.set(instance,false);
        LazyMan instance2 = declaredConstructor.newInstance();        
        System.out.println(instance);        
        System.out.println(instance2);    
    }
}
```

### 3 静态内部类

```java
// 静态内部类
public class Holder {    
    private Holder(){    
    }    
    public static Holder getInstace(){        
        return InnerClass.HOLDER;
    }    
    public static class InnerClass{        
        private static final Holder HOLDER = new Holder();    
    }
}
```

> 静态内部类的优点是：外部类加载时并不需要立即加载内部类，内部类不被加载则不去初始化 INSTANCE，故而不占内存。
>
> 那么，静态内部类又是如何实现线程安全的呢？
>
> 虚拟机会保证一个类的 <clinit>() 方法在多线程环境中被正确地加锁、同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的 <clinit>() 方法，其他线程都需要阻塞等待，直到活动线程执行 <clinit>() 方法完毕。如果在一个类的 <clinit>() 方法中有耗时很长的操作，就可能造成多个进程阻塞（需要注意的是，其他线程虽然会被阻塞，但如果执行<clinit>() 方法后，其他线程唤醒之后不会再次进入<clinit>()方法。同一个加载器下，一个类型只会初始化一次），在实际应用中，这种阻塞往往是很隐蔽的。故而，可以看出 INSTANCE 在创建过程中是线程安全的，所以说静态内部类形式的单例可保证线程安全，也能保证单例的唯一性，同时也延迟了单例的实例化。
>

**但此时单例仍不安全，因为反射可以破坏单例**

### 4 枚举

```java
// enum 本身也是一个 Class 类
public enum EnumSingle {    
    INSTANCE;    
    public EnumSingle getInstance(){
        return INSTANCE;
    }
}
class Test{
    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        EnumSingle instance1 = EnumSingle.INSTANCE;
        Constructor<EnumSingle> declaredConstructor = EnumSingle.class.getDeclaredConstructor();
        declaredConstructor.setAccessible(true);
        // 没有无参构造器
        EnumSingle instance2 = declaredConstructor.newInstance();  // NoSuchMethodException: com.kuang.single.EnumSingle.<init>()
        System.out.println(instance1);
        System.out.println(instance2);
    }
}
class Test{
    public static void main(String[] args) throws NoSuchMethodException, IllegalAccessException, InvocationTargetException, InstantiationException {
        EnumSingle instance1 = EnumSingle.INSTANCE;
        // 编译时会生成 String 和 int 的有参构造器
        Constructor<EnumSingle> declaredConstructor = EnumSingle.class.getDeclaredConstructor(String.class,int.class);
        declaredConstructor.setAccessible(true);
        // 不允许通过反射获取枚举实例
        EnumSingle instance2 = declaredConstructor.newInstance();  // IllegalArgumentException: Cannot reflectively create enum objects
        System.out.println(instance1);
        System.out.println(instance2);
    }
}
```

> 枚举在 java 中与普通类一样，都能拥有字段与方法，而且枚举实例创建是线程安全的，在任何情况下，它都是一个单例

---

## 八、深入理解CAS

```java
public class CASDemo {    
    // CAS compareAndSet : 比较并交换
    public static void main(String[] args) {        
        AtomicInteger atomicInteger = new AtomicInteger(2020);         
        // 期望、更新         
        // public final boolean compareAndSet(int expect, int update)
        // 如果是期望的值就更新，否则就不更新, CAS 是 CPU 的指令，并发原语
        // 成功返回 true
        System.out.println(atomicInteger.compareAndSet(2020, 2021));         
        System.out.println(atomicInteger.get()); // 2021
        atomicInteger.getAndIncrement();
        // 看底层如何实现 ++
        System.out.println(atomicInteger.compareAndSet(2020, 2021));         
        System.out.println(atomicInteger.get());     
    } 
}
```

执行结果如图：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125547397-00f148ced6b1325260da67de56cdf2f8-c635ec.png" alt="在这里插入图片描述" style="zoom:67%;" />

**Unsafe 类**

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125603889-04d9931c587f33c3d11e703e6c2ad010-f2ebb4.png" alt="image-20220222125603889" style="zoom: 67%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125613828-2efdc1ed4fba148f348f7a8013b8cab8-919485.png" alt="image-20220222125613828" style="zoom: 67%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125623438-721117481b4828a1c734f85e5796a685-81e3df.png" alt="image-20220222125623438" style="zoom: 67%;" />

> CAS ：比较当前工作内存中的值和主内存中的值，如果这个值是期望的，那么则执行操作！如果不是就一直循环！

**缺点：**

- 循环会耗时
- 一次只能保证一个共享变量的原子性
- 存在 ABA 问题

**CAS ： ABA 问题（狸猫换太子）**

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125641262-290b8e8c544d28bebe3abbe8afe6aa8f-10a1cf.png" alt="image-20220222125641262" style="zoom: 67%;" />

```java
public class CASDemo {    
    // CAS compareAndSet : 比较并交换！     
    public static void main(String[] args) {        
        AtomicInteger atomicInteger = new AtomicInteger(2020);         
        /**
        * 类似于我们平时写的SQL：乐观锁
        * 如果某个线程在执行操作某个对象的时候，其他线程若操作了该对象
        * 即使对象内容未发生变化，也需要告诉我
        *
        */
        // ============== 捣乱的线程 ==================         
        System.out.println(atomicInteger.compareAndSet(2020, 2021));         
        System.out.println(atomicInteger.get());         
        System.out.println(atomicInteger.compareAndSet(2021, 2020));         
        System.out.println(atomicInteger.get());         
        // ============== 期望的线程 ==================         
        System.out.println(atomicInteger.compareAndSet(2020, 6666));         
        System.out.println(atomicInteger.get());     
    }
}
```

输出结果如图：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125651928-d957cee282ed68bb8bf74ddcf8bf04ad-4e9107.png" alt="image-20220222125651928" style="zoom: 80%;" />

---

## 九、原子引用

> 解决 ABA 问题，引入原子引用！对应的思想：乐观锁

带版本号的原子操作！

```java
public class CASDemo {        
    /**         
     * AtomicStampedReference 注意，         
     * 如果泛型是一个包装类，注意对象的引用问题，传入数字会被转为 Integer 类型，-127 ~ 128 会从缓存中获取，超过则会被创建为新的对象
     * 一般这里使用的是实体类对象
     */
    // 可以有一个初始对应的版本号时间戳 1        
    static AtomicStampedReference<Integer> atomicStampedReference = new AtomicStampedReference<>(111, 1);
    public static void main(String[] args) {
        new Thread(()->{
            // 获得版本号
            int stamp = atomicStampedReference.getStamp();
            System.out.println("a1=>"+stamp);
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            atomicStampedReference.compareAndSet(111, 112,
                    atomicStampedReference.getStamp(), // 最新版本号
                    atomicStampedReference.getStamp() + 1); // 更新版本号
            System.out.println("a2=>"+atomicStampedReference.getStamp());
            System.out.println(atomicStampedReference.compareAndSet(112, 111, atomicStampedReference.getStamp(), atomicStampedReference.getStamp() + 1));
            System.out.println("a3=>"+atomicStampedReference.getStamp());
        },"a").start();
        // 乐观锁的原理相同！
        new Thread(()->{
            // 获得版本号
            int stamp = atomicStampedReference.getStamp();
            System.out.println("b1=>" + stamp);
            try {
                TimeUnit.SECONDS.sleep(3);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(
                    atomicStampedReference.compareAndSet(111, 123, stamp, stamp + 1));
            System.out.println("b2=>"+atomicStampedReference.getStamp());
        },"b").start();
    }
}
```

结果如图：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125701172-8bd6349bf497ce1626ad1a93cba16690-913f1c.png" alt="image-20220222125701172" style="zoom: 67%;" />

**注意：**

**Integer 使用了对象缓存机制，默认范围是 -128 ~ 127 ，推荐使用静态工厂方法 valueOf 获取对象实例，而不是 new，因为 valueOf 使用缓存，而 new 一定会创建新的对象分配新的内存空间；**

下面是阿里巴巴开发手册的规范点：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125709204-d61c8fd2a5068bbb6a8973e1f7808912-a6c200.png" alt="image-20220222125709204" style="zoom: 67%;" />

---

## 十、各种锁的理解

### 1 公平锁、非公平锁

公平锁： 竞争锁资源的线程严格按照请求的顺序来分配锁。

非公平锁：竞争锁资源的线程可以插队来竞争锁资源 （ReentrantLock 默认非公平）

```java
public ReentrantLock() {    
    sync = new NonfairSync(); 
}
public ReentrantLock(boolean fair) {    
    sync = fair ? new FairSync() : new NonfairSync(); 
}
```

```java
/**
 * Fair version of Sync
 */
static final class FairSync extends Sync {
    private static final long serialVersionUID = -2274990926593161451L;
    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();
    }
    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
}
```

非公平锁（NonfairSync）源码：


```java
/**
 * Nonfair version of Sync
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -8159625535654395037L;
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }
    final boolean readerShouldBlock() {
        return apparentlyFirstQueuedIsExclusive();
    }
}
```
可以看到，对于读和写操作是否应该被阻塞，公平锁和非公平锁采取了不同的策略。

公平锁的策略是判断 AQS 队列中是否有正在等待的线程，如果有，返回 true，表明应该进入等待状态，并将该线程放入 AQS 队列尾部（一个 FIFO 的双向队列）；

> AQS的核心思想是：通过一个volatile修饰的int属性state代表同步状态，例如0是无锁状态，1是上锁状态。多线程竞争资源时，通过CAS的方式来修改state，例如从0修改为1，修改成功的线程即为资源竞争成功的线程，将其设为`exclusiveOwnerThread`，也称【工作线程】，资源竞争失败的线程会被放入一个 FIFO 的队列中并挂起休眠，当`exclusiveOwnerThread`线程释放资源后，会从队列中唤醒线程继续工作，循环往复。

非公平锁的写操作不管三七二十一，我就是要参与竞争去，会先去尝试竞争锁资源，如果抢占不到，就会再加入到 AQS 同步队列中去等待；非公平锁的读操作需要判断等待队列里的头结点是不是写操作（通过是否共享来判断锁类型，读锁是共享的，写锁是独占的）。

### 2 可重入锁

> 可重入锁，指的是以线程为单位，当一个线程获取对象锁之后，这个线程可以再次获取本对象上的锁，而其他的线程是不可以的。

#### 2.1 Synchronized 版

```java
public class Demo01 {    
    public static void main(String[] args) {        
        Phone phone = new Phone();        
        new Thread(()->{            
            phone.sms();        
        },"A").start();        
        new Thread(()->{            
            phone.sms();        
        },"B").start();    
    }
}
class Phone{    
    public synchronized void sms(){        
        System.out.println(Thread.currentThread().getName()+ "sms");        
        call(); // 这里也有锁(sms锁里面的call锁)    
    }    
    public synchronized void call(){        
        System.out.println(Thread.currentThread().getName()+ "call");    
    }
}
```

#### 2.2 ReentrantLock 版

```java
public class Demo02 {    
    public static void main(String[] args) {
        Phone2 phone = new Phone2();
        new Thread(()->{
            phone.sms();
        },"A").start();
        new Thread(()->{
            phone.sms();
        },"B").start();
    }
}
class Phone2{
    Lock lock = new ReentrantLock();
    public void sms(){
        lock.lock();
        // lock 锁和解锁必须成对，否则就会死在里面
        // 两个 lock() 就需要两次解锁
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName()+ "sms");
            call(); // 这里也有锁
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
    public void call(){
        try {
            System.out.println(Thread.currentThread().getName()+ "call");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

### 3 自旋锁

> spinlock

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125721765-d7df5f55fa5a4fe672251e4500b7f07b-ab1cae.png" alt="image-20220222125721765" style="zoom: 67%;" />

我们来自定义一个锁测试：

```java
// 自旋锁
public class SpinlockDemo {    
    /**
     * 原子引用
     * 默认值
     * int 0
     * Thread null
     */
    AtomicReference<Thread> atomicReference = new AtomicReference<>();    
    // 加锁    
    public void myLock(){        
        Thread thread = Thread.currentThread();        
        System.out.println(Thread.currentThread().getName()+ "==> mylock");        
        // 自旋锁：如果 atomicReference 的值为空，则执行 thread
        while (!atomicReference.compareAndSet(null,thread)) {

        }
    }    
    // 解锁   
    public void myUnLock(){        
        Thread thread = Thread.currentThread();        
        System.out.println(Thread.currentThread().getName() + "==> myUnlock");        
        atomicReference.compareAndSet(thread,null);// 解锁    
    }
}
```

测试

```java
public class TestSpinLock {    
    public static void main(String[] args) throws InterruptedException {
        //ReentrantLock reentrantLock = new ReentrantLock();
        //reentrantLock.lock();
        //reentrantLock.unlock();        
        // 底层使用的自旋锁CAS       
        SpinlockDemo lock = new SpinlockDemo();// 定义锁        
        new Thread(()-> {            
            lock.myLock();// 加锁            
            try {                
                TimeUnit.SECONDS.sleep(5);            
            } catch (Exception e) {                
                e.printStackTrace();            
            } finally {                
                lock.myUnLock();// 解锁            
            }        
        },"T1").start();        
        TimeUnit.SECONDS.sleep(1);        
        new Thread(()-> {            
            lock.myLock();            
            try {                
                TimeUnit.SECONDS.sleep(1);            
            } catch (Exception e) {                
                e.printStackTrace();            
            } finally {                
                lock.myUnLock();            
            }        
        },"T2").start();    
    }
}
```

> ​    当第一个线程 T1 获取锁的时候，能够成功获取到，不会进入 while 循环，如果此时线程 T1 没有释放锁，另一个线程 T2 又来获取锁，此时由于不满足 CAS，所以就会进入 while 循环，不断判断是否满足 CAS，直到 A 线程调用 unlock 方法释放了该锁。

### 4 死锁

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125833199-2a36b1e90a0c174cba566ff805b101a1-c338a2.png" alt="image-20220222125833199" style="zoom: 67%;" />

```java
public class TestLock {
    public static void main(String[] args) {
        String lockA = "lockA";
        String lockB = "lockB";
        DeadLock deadLock1 = new DeadLock(lockA, lockB);
        DeadLock deadLock2 = new DeadLock(lockB, lockA);
        FutureTask<String> task1 = new FutureTask<>(deadLock1);
        FutureTask<String> task2 = new FutureTask<>(deadLock2);
        new Thread(task1, "task1").start();
        new Thread(task2, "task2").start();

    }
}
class DeadLock implements Callable<String> {

    private String lockA;
    private String lockB;

    public DeadLock(String lockA, String lockB) {
        this.lockA = lockA;
        this.lockB = lockB;
    }

    @Override
    public String call() throws Exception {
        System.out.println(Thread.currentThread().getName() + "try to get " + lockA);
        synchronized (lockA) {
            System.out.println(Thread.currentThread().getName() + "hold " + lockA);
            TimeUnit.SECONDS.sleep(3);
            System.out.println(Thread.currentThread().getName() + "try to get " + lockB);
            synchronized (lockB) {
                System.out.println(Thread.currentThread().getName() + "hold " + lockB);
            }
        }
        return "OK";
    }
}
```

**使用 `jps -l` 定位进程号**

```bash
jps -l
```

**使用 `jstack 进程号` 查看进程信息，找到死锁信息**

```bash
jstack 9876
```

```bash
... ...
"task2":
        at DeadLock.call(TestLock.java:35)
        - waiting to lock <0x000000076b373ac0> (a java.lang.String)
        - locked <0x000000076b373af8> (a java.lang.String)
        at DeadLock.call(TestLock.java:17)
        at java.util.concurrent.FutureTask.run(FutureTask.java:266)
        at java.lang.Thread.run(Thread.java:748)
"task1":
        at DeadLock.call(TestLock.java:35)
        - waiting to lock <0x000000076b373af8> (a java.lang.String)
        - locked <0x000000076b373ac0> (a java.lang.String)
        at DeadLock.call(TestLock.java:17)
        at java.util.concurrent.FutureTask.run(FutureTask.java:266)
        at java.lang.Thread.run(Thread.java:748)

Found 1 deadlock.
```

