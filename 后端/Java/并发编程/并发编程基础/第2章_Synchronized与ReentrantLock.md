# 第2章_Synchronized与ReentrantLock

## 1.共享问题

两个线程对初始值为 0 的静态变量一个做自增，一个做自减，各做 5000 次，结果是 0 吗？

```java
static int counter = 0;

public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(() -> {
        for (int i = 0; i < 5000; i++) {
            counter++;
        }
    }, "t1");

    Thread t2 = new Thread(() -> {
        for (int i = 0; i < 5000; i++) {
            counter--;
        }
    }, "t2");

    t1.start();
    t2.start();
    t1.join();
    t2.join();
    log.debug("{}",counter);
}
```

### 1.1 问题分析

以上的结果可能是正数、负数、零。为什么呢？因为 Java 中对静态变量的自增，自减并不是原子操作，要彻底理解，必须从字节码来进行分析，例如对于`i++`而言（i 为静态变量），实际会产生如下的 JVM 字节码指令：

```java
getstatic     i  // 获取静态变量i的值
iconst_1         // 准备常量1
iadd             // 自增
putstatic     i  // 将修改后的值存入静态变量i
```

而对应`i--`也是类似：

```java
getstatic     i  // 获取静态变量i的值
iconst_1         // 准备常量1
isub             // 自减
putstatic     i  // 将修改后的值存入静态变量i
```

而 Java 的内存模型如下，完成静态变量的自增，自减需要在主存和工作内存中进行数据交换：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815141901622-4e819132ad219c9f88481345ffbfb79a-c47a39.png" alt="image-20220815141901622" style="zoom:67%;" />

如果是单线程以上 8 行代码是顺序执行（不会交错）没有问题：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815141920195-f5ac2625d0b77d6962ad8e4fd6c1dca5-5d4dd8.png" alt="image-20220815141920195" style="zoom: 67%;" />

但多线程下这 8 行代码可能交错运行。

出现负数的情况：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815141954099-b2bb8856be75e45aaf9153dfb8a24df1-c51898.png" alt="image-20220815141954099" style="zoom:67%;" />

出现正数的情况：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815142006572-43425ce287f4c33248822e7163a95f1d-e73d65.png" alt="image-20220815142006572" style="zoom:67%;" />

### 1.2 临界区（Critical Section）

一个程序运行多个线程本身是没有问题的，问题出在多个线程访问共享资源时，对共享资源的读写操作发生指令交错。

一段代码块内如果存在对共享资源的多线程读写操作，称这段代码块为**临界区**。

例如，下面代码中的临界区：

```java
static int counter = 0;

static void increment() 

{    
    // 临界区
    counter++;
}

static void decrement() 

{   
    // 临界区
    counter--;
}
```

### 1.3 竞态条件（Race Condition）

多个线程在临界区内执行，由于代码的执行序列不同而导致结果无法预测，称之为发生了竞态条件。

## 2.Synchronized

为了避免临界区的竞态条件发生，有多种手段可以达到目的。

- 阻塞式的解决方案：`synchronized`，`Lock`

  > - synchronized 属于重量级锁，但是当代码运行速度较慢时，为了避免 CPU 的空转压力，使用 synchronized 往往是更好的选择
  > - synchronized 可以使用更细粒度的对象作为锁

- 非阻塞式的解决方案：原子变量

本次课使用阻塞式的解决方案：synchronized，来解决上述问题，即俗称的【对象锁】，它采用互斥的方式让同一时刻至多只有一个线程能持有【对象锁】，其它线程再想获取这个【对象锁】时就会阻塞住。这样就能保证拥有锁的线程可以安全的执行临界区内的代码，不用担心线程上下文切换。

> **注意**
>
> 虽然 java 中互斥和同步都可以采用 synchronized 关键字来完成，但它们还是有区别的：
>
> - 互斥是保证临界区的竞态条件发生，同一时刻只能有一个线程执行临界区代码
> - 同步是由于线程执行的先后、顺序不同、需要一个线程等待其它线程运行到某个点

**语法**

```java
synchronized(对象) // 线程1， 线程2(blocked)
{
    临界区
}
```

**解决**

```java
static int counter = 0;
static final Object room = new Object();
 
public static void main(String[] args) throws InterruptedException {
    Thread t1 = new Thread(() -> {
        for (int i = 0; i < 5000; i++) {
            synchronized (room) {
                counter++;
            }
        }
    }, "t1");
 
    Thread t2 = new Thread(() -> {
        for (int i = 0; i < 5000; i++) {
            synchronized (room) {
                counter--;
            }
        }
    }, "t2");
 
    t1.start();
    t2.start();
    t1.join();
    t2.join();
    log.debug("{}",counter);
}
```

你可以做这样的类比：

synchronized（对象）中的对象，可以想象为一个房间（room），有唯一入口（门）房间只能一次进入一人进行计算，线程 t1，t2 想象成两个人，当线程 t1 执行到`synchronized(room)`时就好比 t1 进入了这个房间，并锁住了门拿走了钥匙，在门内执行`count++`代码。这时候如果 t2 也运行到了`synchronized(room)`时，它发现门被锁住了，只能在门外等待，发生了上下文切换，阻塞住了。

这中间即使 t1 的 cpu 时间片不幸用完，被踢出了门外（不要错误理解为锁住了对象就能一直执行下去哦），这时门还是锁住的，t1 仍拿着钥匙，t2 线程还在阻塞状态进不来，只有下次轮到 t1 自己再次获得时间片时才能开门进入。

当 t1 执行完`synchronized{}`块内的代码，这时候才会从 obj 房间出来并解开门上的锁，唤醒 t2 线程把钥匙给他。t2 线程这时才可以进入 obj 房间，锁住了门拿上钥匙，执行它的`count--`代码。

用图来表示：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815142427939-4bcd5fff9a1af065711094efa20ea62e-3d4712.png" alt="image-20220815142427939" style="zoom:80%;" />

**思考**

`synchronized`实际是用**对象锁**保证了**临界区内代码的原子性**，临界区内的代码对外是不可分割的，不会被线程切换所打断。

为了加深理解，请思考下面的问题：

- 如果把 synchronized(obj) 放在 for 循环的外面，如何理解？-- for 循环作为原子的整体
- 如果 t1 synchronized(obj1) 而 t2 synchronized(obj2) 会怎样运作？-- 不是同一把锁不会造成阻塞
- 如果 t1 synchronized(obj) 而 t2 没有加会怎么样？如何理解？-- t2 不需要锁不会造成阻塞

**面向对象改进**

把需要保护的共享变量放入一个类

```java
class Room {
    int value = 0;

    public void increment() {
        synchronized (this) {
            value++;
        }
    }

    public void decrement() {
        synchronized (this) {
            value--;
        }
    }

    public int get() {
        synchronized (this) {
            return value;
        }
    }
}

@Slf4j
public class Test1 {

    public static void main(String[] args) throws InterruptedException {
        Room room = new Room();
        Thread t1 = new Thread(() -> {
            for (int j = 0; j < 5000; j++) {
                room.increment();
            }
        }, "t1");

        Thread t2 = new Thread(() -> {
            for (int j = 0; j < 5000; j++) {
                room.decrement();
            }
        }, "t2");
        t1.start();
        t2.start();

        t1.join();
        t2.join();
        log.debug("count: {}" , room.get());
    }
}
```

### 2.1 锁的对象

- 普通`synchronized`方法的锁是调用的对象
- `static synchronized`方法的锁是 Class 本身

```java
class Test{
    public synchronized void test() {
    
    }
}
// 等价于
class Test{
    public void test() {
        synchronized(this) {
        
        }
    }
}
```

```java
class Test{
    public synchronized static void test() {
 
    }
}
// 等价于
class Test{
    public static void test() {
        synchronized(Test.class) {
            
        }
    }
}
```

### 2.2 线程八锁

其实就是考察`synchronized`锁住的是哪个对象。

情况 1：依次打印1 - 2；或者依次打印 2 - 1

```java
@Slf4j(topic = "c.Number")
class Number{
    public synchronized void a() {
        log.debug("1");
    }
    public synchronized void b() {
        log.debug("2");
    }
}

public static void main(String[] args) {
    Number n1 = new Number();
    new Thread(() -> n1.a()).start();
    new Thread(() -> n1.b()).start();
}
```

情况 2：1s 后依次打印 1 - 2；或先打印 2，1s 后打印 1

```java
@Slf4j(topic = "c.Number")
class Number{
    public synchronized void a() {
        sleep(1);
        log.debug("1");
    }
    public synchronized void b() {
        log.debug("2");
    }
}
 
public static void main(String[] args) {
    Number n1 = new Number();
    new Thread(()->{ n1.a(); }).start();
    new Thread(()->{ n1.b(); }).start();
}
```

情况 3：先打印 3，1s 后打印 1 - 2；或先打印 2 - 3，1s 后打印 1；或先打印 3 - 2，1s 后打印 1

```java
class Number{
    public synchronized void a() {
        sleep(1);
        log.debug("1");
    }
    public synchronized void b() {
        log.debug("2");
    }
    public void c() {
        log.debug("3");
    }
}
 
public static void main(String[] args) {
    Number n1 = new Number();
    new Thread(()->{ n1.a(); }).start();
    new Thread(()->{ n1.b(); }).start();
    new Thread(()->{ n1.c(); }).start();
}
```

情况 4：先打印 2，1s 后打印 1

```java
@Slf4j(topic = "c.Number")
class Number{
    public synchronized void a() {
        sleep(1);
        log.debug("1");
    }
    public synchronized void b() {
        log.debug("2");
    }
}
 
public static void main(String[] args) {
    Number n1 = new Number();
    Number n2 = new Number();
    new Thread(()->{ n1.a(); }).start();
    new Thread(()->{ n2.b(); }).start();
}
```

情况 5：先打印 2，1s 后打印 1

```java
@Slf4j(topic = "c.Number")
class Number{
    public static synchronized void a() {
        sleep(1);
        log.debug("1");
    }
    public synchronized void b() {
        log.debug("2");

    }
}

public static void main(String[] args) {
    Number n1 = new Number();
    new Thread(()->{ n1.a(); }).start();
    new Thread(()->{ n1.b(); }).start();
}
```

情况 6：1s 后打印 1 - 2；或先打印 2，1s 后打印 1

```java
@Slf4j(topic = "c.Number")
class Number{
    public static synchronized void a() {
        sleep(1);
        log.debug("1");
    }
    public static synchronized void b() {
        log.debug("2");
    }
}
 
public static void main(String[] args) {
    Number n1 = new Number();
    new Thread(()->{ n1.a(); }).start();
    new Thread(()->{ n1.b(); }).start();
}
```

情况 7：先打印 2，1s 后打印 1

```java
@Slf4j(topic = "c.Number")
class Number{
    public static synchronized void a() {
        sleep(1);
        log.debug("1");
    }
    public synchronized void b() {
        log.debug("2");
    }
}
 
public static void main(String[] args) {
    Number n1 = new Number();
    Number n2 = new Number();
    new Thread(()->{ n1.a(); }).start();
    new Thread(()->{ n2.b(); }).start();
}
```

情况 8：1s 后打印 1 - 2；或先打印 2，1s 后打印 1

```java
@Slf4j(topic = "c.Number")
class Number{
    public static synchronized void a() {
        sleep(1);
        log.debug("1");
    }
    public static synchronized void b() {
        log.debug("2");
    }
}

public static void main(String[] args) {
    Number n1 = new Number();
    Number n2 = new Number();
    new Thread(()->{ n1.a(); }).start();
    new Thread(()->{ n2.b(); }).start();
}
```

### 2.3 Monitor解析

#### 1.Java对象头

以 32 位虚拟机为例：

普通对象

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815143653814-aa82d123978942b4ec6dfc218d20b379-49fb94.png" alt="image-20220815143653814" style="zoom: 80%;" />

数组对象

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815143705335-98c37fdabd851f313897c100e11d1264-77036e.png" alt="image-20220815143705335" style="zoom: 80%;" />

其中 Mark Word 结构为

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815143728679-db96e7f122ae8338e7ef46b837845c72-d5c1c6.png" alt="image-20220815143728679" style="zoom:80%;" />

> - 001：不可偏向
> - 101：可偏向
> - 00：轻量级锁
> - 10：重量级锁

64 位虚拟机 Mark Word

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815143743621-263a68629c5b4277e1e4978e22aa9143-b89fb3.png" alt="image-20220815143743621" style="zoom:80%;" />

<a href="https://stackoverﬂow.com/questions/26357186/what-is-in-java-object-header">参考</a>

#### 2.Monitor

Monitor 被翻译为**监视器**或**管程**。

每个 Java 对象都可以关联一个 Monitor 对象，如果使用 synchronized 给对象上锁（重量级）之后，该对象头的 Mark Word 中就被设置指向 Monitor 对象的指针。

Monitor 结构如下：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815172648807-e70da7c7cd5ac99b9062f7a83975e3c7-4c30d7.png" alt="image-20220815172648807" style="zoom:80%;" />

- 每个锁对象会关联一个 Monitor
- 刚开始 Monitor 中 Owner 为 null
- 当 Thread-2 执行`synchronized(obj)`就会将 Monitor 的所有者 Owner 置为 Thread-2，Monitor中只能有一个 Owner
- 在 Thread-2 上锁的过程中，如果 Thread-3，Thread-4，Thread-5 也来执行`synchronized(obj)`，就会进入 EntryList BLOCKED
- Thread-2 执行完同步代码块的内容，然后唤醒 EntryList 中等待的线程来竞争锁，竞争时是非公平的
- 图中 WaitSet 中的 Thread-0，Thread-1 是之前获得过锁，但条件不满足进入 WAITING 状态的线程，后面讲 wait-notify 时会分析

> **注意**
>
> - synchronized 必须是进入同一个对象的 monitor 才有上述的效果
> - 不加 synchronized 的对象不会关联监视器，不遵从以上规则

### 2.4 原理：synchronized

```java
static final Object lock = new Object();
static int counter = 0;
 
public static void main(String[] args) {
    synchronized (lock) {
        counter++;
    }
}
```

对应的字节码为

```java
public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=3, args_size=1
         0: getstatic     #2                  // <- lock引用 （synchronized开始）
         3: dup
         4: astore_1                          // lock引用 -> slot 1
         5: monitorenter                      // 将 lock 对象 MarkWord 置为 Monitor 指针
         6: getstatic     #3                  // <- i
         9: iconst_1                          // 准备常数 1
        10: iadd                              // +1
        11: putstatic     #3                  // -> i
        14: aload_1                           // <- 加载 lock 引用
        15: monitorexit                       // 将 lock 对象 MarkWord 重置, 唤醒 EntryList
        16: goto          24
        // 19~23 用于异常时释放锁
        19: astore_2                          // e -> slot 2 
        20: aload_1                           // <- lock引用
        21: monitorexit                       // 将 lock对象 MarkWord 重置, 唤醒 EntryList
        22: aload_2                           // <- slot 2 (e)
        23: athrow                            // throw e
        24: return
      Exception table:
         from    to  target type
             6    16    19   any
            19    22    19   any
      LineNumberTable:
        line 8: 0
        line 9: 6
        line 10: 14
        line 11: 24
	  LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      25     0  args   [Ljava/lang/String;
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 19
          locals = [ class "[Ljava/lang/String;", class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
```

> **注意**
>
> 方法级别的`synchronized`不会在字节码指令中有所体现。

### 2.5 原理：synchronized进阶

由于 Monitor 锁的成本非常高，如果每次都使用 Monitor 的话非常的耗时。因此 Java6 开始对 synchronized 获取锁的方式进行了改进，增加了**轻量级锁**和**偏向锁**。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/0cccd54042594f038dc445f081392fc3-ab80f68986eef758c3ee92aca69bedd3-7f5c16.png" alt="img" style="zoom:80%;" />

#### 1.轻量级锁

轻量级锁的使用场景：如果一个对象虽然有多线程要加锁，但**加锁的时间是错开的**（也就是没有竞争），那么可以使用轻量级锁来优化。

轻量级锁对使用者是透明的，即语法仍然是`synchronized`。

假设有两个方法同步块，利用同一个对象加锁：

```java
static final Object obj = new Object();
public static void method1() {
    synchronized( obj ) {
        // 同步块 A
        method2();
    }
}
public static void method2() {
    synchronized( obj ) {
        // 同步块 B
    }
}
```

创建锁记录（Lock Record）对象，每个线程的栈帧都会包含一个**锁记录的结构（Lock Record）**，内部可以存储锁定对象的 Mark Word

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815173742962-ec166786135068f500ac08c12b532eed-b2eb34.png" alt="image-20220815173742962" style="zoom:67%;" />

让锁记录中 Object reference 指向锁对象，并尝试用 cas 替换 Object 的 Mark Word，将 Mark Word 的值存入锁记录

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815173758123-25a3962ad01241b802942956d89cd0a3-8df9cf.png" alt="image-20220815173758123" style="zoom:67%;" />

如果 cas 替换成功，对象头中存储了**锁记录地址和状态**`00`，表示由该线程给对象加锁，这时图示如下

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815173817851-93b0804e50d5860766b8ae28ca8f34a9-bea588.png" alt="image-20220815173817851" style="zoom:67%;" />

如果 cas 失败，有两种情况

1. 如果是其它线程已经持有了该 Object 的轻量级锁，这时表明有竞争，进入锁膨胀过程
2. 如果是自己执行了 synchronized 锁重入，那么再添加一条 Lock Record 作为重入的计数

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815173845094-dd872484e8e91b61c596da9384e70566-9c636a.png" alt="image-20220815173845094" style="zoom:67%;" />

当退出 synchronized 代码块（解锁时）如果有取值为 null 的锁记录，表示有重入，这时重置锁记录，表示重入计数减一

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815173903330-30cf307ee4d08e9be7afe88ccc347204-766582.png" alt="image-20220815173903330" style="zoom:67%;" />

当退出 synchronized 代码块（解锁时）锁记录的值不为 null，这时使用 cas 将 Mark Word 的值恢复给对象头

- 成功，则解锁成功
- 失败，说明轻量级锁进行了锁膨胀或已经升级为重量级锁，进入重量级锁解锁流程

#### 2.锁膨胀

如果在尝试加轻量级锁的过程中，CAS 操作无法成功，这时一种情况就是有其它线程为此对象加上了轻量级锁（有竞争），这时需要进行锁膨胀，**将轻量级锁变为重量级锁**。

```java
static Object obj = new Object();
public static void method1() {
    synchronized( obj ) {
        // 同步块
    }
}
```

当 Thread-1 进行轻量级加锁时，Thread-0 已经对该对象加了轻量级锁。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815174322188-2f240f0be498b5ff708a011e02264783-801e65.png" alt="image-20220815174322188" style="zoom:67%;" />

这时 Thread-1 加轻量级锁失败，进入锁膨胀流程。

- 即为 Object 对象申请 Monitor 锁，让 Object 指向重量级锁地址
- 将 Monitor 的 Owner 指向 Thread-0
- 然后自己进入 Monitor 的 EntryList `BLOCKED`

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815174340089-b3d7d3d03ddfc489d8bb94fc4fde2b9c-3c6921.png" alt="image-20220815174340089" style="zoom:67%;" />

当 Thread-0 退出同步块解锁时，使用 cas 将 Mark Word 的值恢复给对象头，失败。这时会进入重量级解锁流程，即按照 Monitor 地址找到 Monitor 对象，设置 Owner 为 null，唤醒 EntryList 中 BLOCKED 线程。

#### 3.自旋优化

重量级锁竞争的时候，还可以使用自旋来进行优化，如果当前线程自旋成功（即这时候持锁线程已经退出了同步块，释放了锁），这时当前线程就可以避免阻塞（避免上下文切换）。

自旋重试成功的情况：

| 线程 1（core 1 上）      | 对象 Mark              | 线程 2（core 2 上）      |
| ------------------------ | ---------------------- | ------------------------ |
| -                        | 10（重量锁）           | -                        |
| 访问同步块，获取 monitor | 10（重量锁）重量锁指针 | -                        |
| 成功（加锁）             | 10（重量锁）重量锁指针 | -                        |
| 执行同步块               | 10（重量锁）重量锁指针 | -                        |
| 执行同步块               | 10（重量锁）重量锁指针 | 访问同步块，获取 monitor |
| 执行同步块               | 10（重量锁）重量锁指针 | 自旋重试                 |
| 执行完毕                 | 10（重量锁）重量锁指针 | 自旋重试                 |
| 成功（解锁）             | 01（无锁）             | 自旋重试                 |
| -                        | 10（重量锁）重量锁指针 | 成功（加锁）             |
| -                        | 10（重量锁）重量锁指针 | 执行同步块               |
| -                        | ...                    | ...                      |

自旋重试失败的情况：

| 线程 1（core 1 上）      | 对象 Mark              | 线程 2（core 2 上）      |
| ------------------------ | ---------------------- | ------------------------ |
| -                        | 10（重量锁）           | -                        |
| 访问同步块，获取 monitor | 10（重量锁）重量锁指针 | -                        |
| 成功（加锁）             | 10（重量锁）重量锁指针 | -                        |
| 执行同步块               | 10（重量锁）重量锁指针 | -                        |
| 执行同步块               | 10（重量锁）重量锁指针 | 访问同步块，获取 monitor |
| 执行同步块               | 10（重量锁）重量锁指针 | 自旋重试                 |
| 执行同步块               | 10（重量锁）重量锁指针 | 自旋重试                 |
| 执行同步块               | 10（重量锁）重量锁指针 | 自旋重试                 |
| 执行同步块               | 10（重量锁）重量锁指针 | 阻塞                     |
| -                        | ...                    | ...                      |

- 自旋会占用 CPU 时间，单核 CPU 自旋就是浪费，多核 CPU 自旋才能发挥优势
- 在 Java 6 之后自旋锁是自适应的，比如对象刚刚的一次自旋操作成功过，那么认为这次自旋成功的可能性会高，就多自旋几次；反之，就少自旋甚至不自旋，总之，比较智能
- Java 7 之后不能控制是否开启自旋功能

#### 4.偏向锁

##### 4.1 简介

轻量级锁在没有竞争时（就自己这个线程），每次重入仍然需要执行 CAS 操作。

Java 6 中引入了偏向锁来做进一步优化：**只有第一次**使用 CAS 将线程 ID 设置到对象的 Mark Word 头，之后发现这个线程 ID 是自己的就表示没有竞争，不用重新 CAS。以后只要不发生竞争，这个对象就归该线程所有。

> 如果线程 A 拿到了偏向锁还没有释放，线程 B 尝试获取这个锁时会膨胀为重量级锁。

**加锁流程**

1. 当 JVM 启动了偏向锁模式（Java 6和Java 7里是默认启动的），新创建对象的 Mark Word 中的 ThreadID 为 0，说明此对象处于偏向锁状态（但未偏向任何线程），也叫作匿名偏向锁状态

2. 线程 A 第一次访问同步代码块时，先检查对象头 Mark Word 中锁标志位是否为 01，依此判断此时对象是否处于无锁状态或者偏向锁状态

3. 若锁标志位是为 01，然后判断偏向锁的标识是否为 1：

   - 如果不是，则进入轻量级锁逻辑（使用 CAS 竞争锁）（注意：此时不是使用 CAS 尝试获取偏向锁，而是直接升级为轻量级锁；原因是：当偏向锁的标识为 0 时，表明偏向锁在此对象上被禁用，禁用原因可能是 JVM 关闭了偏向锁模式，或该类刚经历过 bulk revocation，等等。所以应该入轻量级锁逻辑）

   - 如果是 1，表明此对象是偏向锁状态，则进行下一步流程

4. 判断是偏向锁时，检查对象头 Mark Word 中记录的 ThreadID 是否是当前线程 A 的 ID：
   - 如果是，则表明当前线程 A 已经获得过该对象锁，以后线程 A 进入同步代码块时，不需要 CAS 进行加锁，只会往当前线程 A 的栈中添加一条 Displaced Mark Word 为空的 Lock Record，用来统计重入的次数
   - 如果不是，则进行 CAS 操作，尝试将当前线程 A 的 ID 替换进 Mark Word：
     - 如果当前对象锁的 ThreadID 为 0（匿名偏向锁状态），则会替换成功（将 Mark Word 中的 Thread id 由匿名 0 改成当前线程 A 的 ID，在当前线程 A 栈中找到内存地址最高的可用 Lock Record，将线程 A 的 ID 存入），获得到锁，执行同步代码块
     - 如果当前对象锁的 ThreadID 不为 0，即该对象锁已经被其他线程 B 占用了，则会替换失败，开始进行偏向锁撤销。这也是偏向锁的特点，一旦出现线程竞争，就会**撤销偏向锁**

**偏向锁的撤销**

1. 偏向锁的撤销需要等待全局安全点（`safe point`，代表了一个状态，在该状态下所有线程都是暂停的，stop-the-world），到达全局安全点后，持有偏向锁的线程 B 也被暂停了

2. 检查持有偏向锁的线程 B 的状态（会遍历当前 JVM 的所有线程，如果能找到线程 B，则说明偏向的线程 B 还存活着）：
   - 如果线程还存活，则检查线程是否还在执行同步代码块中的代码：
     - 如果是，则把该偏向锁升级为轻量级锁，且原持有偏向锁的线程 B 继续获得该轻量级锁
   - 如果线程未存活，或线程未在执行同步代码块中的代码，则进行校验是否允许重偏向：
     - 如果不允许重偏向，则将 Mark Word 设置为无锁状态（未锁定不可偏向状态），然后升级为轻量级锁，进行 CAS 竞争锁
     - 如果允许重偏向，设置为匿名偏向锁状态（即线程 B 释放偏向锁）。当唤醒线程后，进行 CAS 将偏向锁重新指向线程 A（在对象头和线程栈帧的锁记录中存储当前线程 ID）
3. 唤醒暂停的线程，从安全点继续执行代码

补充： 每次进入同步块（即执行 monitorenter）的时候都会以从高往低的顺序在栈中找到第一个可用的 Lock Record，并设置偏向线程 ID；每次解锁（即执行 monitorexit）的时候都会从最低的一个 Lock Record 移除。所以如果能找到对应的 Lock Record 说明偏向的线程还在执行同步代码块中的代码。

下图为当对象所处于偏向锁时，当前线程重入 3 次，线程栈帧中 Lock Record 记录：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/2c798bb24d254bac86cc89c5f33db89d-5646cdf7202f9f19daab5e1023316721-c75469.png" alt="img" style="zoom: 67%;" />

**偏向状态**

回忆一下对象头格式

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815143743621-263a68629c5b4277e1e4978e22aa9143-b89fb3.png" alt="image-20220815143743621" style="zoom: 80%;" />

一个对象创建时：

- 如果开启了偏向锁（默认开启），那么对象创建后，markword 值为 0x05 即最后 3 位为 101，这时它的 thread、epoch、age 都为 0

  > 没进入同步块不会上偏向锁，但是我们会看到低 3 位是 101。
  >
  > 因为每个新对象预置了一个**可偏向状态**，也叫做**匿名偏向状态**，是对象初始化中，JVM 帮我们做的。注意此时 markword 中高位是不存在 ThreadID 的， 都是 0，说明此时并没有线程偏向发生，因此也可以理解成是无锁。

- 偏向锁是默认是延迟的，不会在程序启动时立即生效，如果想避免延迟，可以加 VM 参数`-XX:BiasedLockingStartupDelay=0`来禁用延迟

- 如果没有开启偏向锁，那么对象创建后，markword 值为 0x01 即最后 3 位为 001，这时它的 hashcode、age 都为 0，第一次用到 hashcode 时才会赋值

**测试延迟特性**

java 进程启动的 4s 内，都会直接跳过偏向锁，有同步代码块时直接使用轻量级锁。原因是 JVM 初始化的代码有很多地方用到了 synchronized，如果直接开启偏向，产生竞争就要有锁升级，会带来额外的性能损耗。

**测试偏向锁**

```java
class Dog {}
```

利用 jol 第三方工具来查看对象头信息（注意这里我扩展了 jol 让它输出更为简洁）

```java
// 添加虚拟机参数 -XX:BiasedLockingStartupDelay=0 
public static void main(String[] args) throws IOException {
    Dog d = new Dog();
    ClassLayout classLayout = ClassLayout.parseInstance(d);

    new Thread(() -> {
        log.debug("synchronized 前");
        System.out.println(classLayout.toPrintableSimple(true));
        synchronized (d) {
            log.debug("synchronized 中");
            System.out.println(classLayout.toPrintableSimple(true));
        }
        log.debug("synchronized 后");
        System.out.println(classLayout.toPrintableSimple(true));
    }, "t1").start();
}
```

输出

```bash
11:08:58.117 c.TestBiased [t1] - synchronized 前 
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000101 
11:08:58.121 c.TestBiased [t1] - synchronized 中 
00000000 00000000 00000000 00000000 00011111 11101011 11010000 00000101 
11:08:58.121 c.TestBiased [t1] - synchronized 后 
00000000 00000000 00000000 00000000 00011111 11101011 11010000 00000101 
```

> **注意**
>
> - 处于偏向锁的对象解锁后，线程 id 仍存储于对象头中
>
> - 线程 id 是操作系统分配的，不是 java 中得到的

**测试禁用**

在上面测试代码运行时在添加 VM 参数`-XX:-UseBiasedLocking`禁用偏向锁

输出

```bash
11:13:10.018 c.TestBiased [t1] - synchronized 前 
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001 
11:13:10.021 c.TestBiased [t1] - synchronized 中 
00000000 00000000 00000000 00000000 00100000 00010100 11110011 10001000 
11:13:10.021 c.TestBiased [t1] - synchronized 后 
00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001 
```

**测试 hashCode**

正常状态对象一开始是没有 hashCode 的，第一次调用才生成。

##### 4.2 撤销

**调用锁对象默认hashCode**

- 轻量级锁会在锁记录中记录 hashCode
- 重量级锁会在 Monitor 中记录 hashCode

- 偏向锁的对象头中没有空间存储`hash`值，因此如果一个对象调用了默认的哈希算法计算了一致性哈希值，就不能使用偏向锁，而直接使用轻量级锁。如果一个锁已经是偏向锁，再调用`hashcode()`，就会直接退化成为重量级锁
- 如果覆写了`hashcode()`方法，对象头中就不会生成 hashcode，而是每次通过`hashcode()`方法调用。因为覆写`hashcode()`之后，该方法中很可能会利用被修改的成员来计算哈希值，所以 jvm 不敢将其存储到 markword 中。此时偏向锁依然生效

在调用 hashCode 后使用偏向锁，记得去掉`-XX:-UseBiasedLocking`。

输出

```bash
11:22:10.386 c.TestBiased [main] - 调用 hashCode:1778535015 
11:22:10.391 c.TestBiased [t1] - synchronized 前 
00000000 00000000 00000000 01101010 00000010 01001010 01100111 00000001 
11:22:10.393 c.TestBiased [t1] - synchronized 中 
00000000 00000000 00000000 00000000 00100000 11000011 11110011 01101000 
11:22:10.393 c.TestBiased [t1] - synchronized 后 
00000000 00000000 00000000 01101010 00000010 01001010 01100111 00000001 
```

**其它线程使用过同一锁对象**

当有其它线程使用过偏向锁对象时，会将偏向锁升级为轻量级锁（在前一线程访问之后另一线程再访问）。

```java
private static void test2() throws InterruptedException {

    Dog d = new Dog();
    Thread t1 = new Thread(() -> {
        synchronized (d) {
            log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
        }
        synchronized (TestBiased.class) {
            TestBiased.class.notify();
        }
        // 如果不用 wait/notify 使用 join 必须打开下面的注释
        // 因为：t1 线程不能结束，否则底层线程可能被 jvm 重用作为 t2 线程，底层线程 id 是一样的
        /*try {
            System.in.read();
        } catch (IOException e) {
            e.printStackTrace();
        }*/
    }, "t1");
    t1.start();


    Thread t2 = new Thread(() -> {
        synchronized (TestBiased.class) {
            try {
                TestBiased.class.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
        synchronized (d) {
            log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
        }
        log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
    }, "t2");
    t2.start();
}
```

输出

```bash
[t1] - 00000000 00000000 00000000 00000000 00011111 01000001 00010000 00000101 
[t2] - 00000000 00000000 00000000 00000000 00011111 01000001 00010000 00000101 
[t2] - 00000000 00000000 00000000 00000000 00011111 10110101 11110000 01000000 
[t2] - 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001 
```

**调用锁对象的wait/notify**

升级为重量级锁。

```java
public static void main(String[] args) throws InterruptedException {
    Dog d = new Dog();
 
    Thread t1 = new Thread(() -> {
        log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
        synchronized (d) {
            log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
            try {
                d.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.debug(ClassLayout.parseInstance(d).toPrintableSimple(true));
        }
    }, "t1");
    t1.start();
 
    new Thread(() -> {
        try {
            Thread.sleep(6000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        synchronized (d) {
            log.debug("notify");
            d.notify();
        }
    }, "t2").start();
 
}
```

输出

```bash
[t1] - 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000101 
[t1] - 00000000 00000000 00000000 00000000 00011111 10110011 11111000 00000101 
[t2] - notify 
[t1] - 00000000 00000000 00000000 00000000 00011100 11010100 00001101 11001010 
```

**批量重偏向**

如果对象虽然被多个线程访问，但没有竞争，这时偏向了线程 T1 的对象仍有机会重新偏向 T2，重偏向会重置对象的 Thread ID。

当撤销偏向锁阈值达到 20 次后，jvm 会这样觉得，我是不是偏向错了呢，于是会在给剩余对象加锁时重新使用偏向锁偏向至加锁线程。

```java
private static void test3() throws InterruptedException {
 
    Vector<Dog> list = new Vector<>();
    Thread t1 = new Thread(() -> {
        for (int i = 0; i < 30; i++) {
            Dog d = new Dog();
            list.add(d);
            synchronized (d) {
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
            }
        }
        synchronized (list) {
            list.notify();
        }        
    }, "t1");
    t1.start();
 
    
    Thread t2 = new Thread(() -> {
        synchronized (list) {
            try {
                list.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        log.debug("===============> ");
        for (int i = 0; i < 30; i++) {
            Dog d = list.get(i);
            log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
            synchronized (d) {
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
            }
            log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
        }
    }, "t2");
    t2.start();
}
```

可以发现，t1 线程首先给 d 对象加偏向锁偏向 t1 线程，t2 线程会给前 19 个 d 对象加轻量级锁，但是从 20 个后就会重新加偏向锁偏向至 t2 线程。

**批量撤销**

当撤销偏向锁阈值超过 40 次后，jvm 会这样觉得，自己确实偏向错了，根本就不该偏向。于是整个类的所有对象都会变为不可偏向的，新建的对象也是不可偏向的。

```java
static Thread t1,t2,t3;
private static void test4() throws InterruptedException {
    Vector<Dog> list = new Vector<>();

    int loopNumber = 39;
    t1 = new Thread(() -> {
        for (int i = 0; i < loopNumber; i++) {
            Dog d = new Dog();
            list.add(d);
            synchronized (d) {
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
            }
        }
        LockSupport.unpark(t2);
    }, "t1");
    t1.start();

    t2 = new Thread(() -> {
        LockSupport.park();
        log.debug("===============> ");
        for (int i = 0; i < loopNumber; i++) {
            Dog d = list.get(i);
            log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
            synchronized (d) {
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
            }
            log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
        }
        LockSupport.unpark(t3);
    }, "t2");
    t2.start();

    t3 = new Thread(() -> {
        LockSupport.park();
        log.debug("===============> ");
        for (int i = 0; i < loopNumber; i++) {
            Dog d = list.get(i);
            log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
            synchronized (d) {
                log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
            }
            log.debug(i + "\t" + ClassLayout.parseInstance(d).toPrintableSimple(true));
        }
    }, "t3");
    t3.start();

    t3.join();
    log.debug(ClassLayout.parseInstance(new Dog()).toPrintableSimple(true));
}
```

可以发现，t3 线程的前 19 个打印为：不可偏向状态 - 轻量级锁 - 不可偏向状态（t2 线程的前 19 个已经升级为轻量级锁变为不可偏向状态，从第 20 个开始重偏向）。从第 20 个开始为：偏向到 t2 - 轻量级锁 - 不可偏向状态。此时撤销操作次数为：t3 线程的 20 次 + t2 线程的 19 次。

然后重新新建一个 Dog 的实例可以发现，它的偏向状态为 001，即不可偏向。

> **参考资料**
>
> - https://github.com/farmerjohngit/myblog/issues/12
>
> - https://www.cnblogs.com/LemonFive/p/11246086.html
>
> - https://www.cnblogs.com/LemonFive/p/11248248.html
>
> - <a href="https://www.oracle.com/technetwork/java/biasedlocking-oopsla2006-wp-149958.pdf">偏向锁论文</a>

#### 5.锁消除

```java
@Fork(1)
@BenchmarkMode(Mode.AverageTime)
@Warmup(iterations=3)
@Measurement(iterations=5)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
public class MyBenchmark {
    static int x = 0;
    @Benchmark
    public void a() throws Exception {
        x++;
    }
    @Benchmark
    public void b() throws Exception {
        Object o = new Object();
        synchronized (o) {
            x++;
        }
    }
}
```

`java -jar benchmarks.jar`

可以发现两者性能几乎一样。因为 JIT 发现对象 o 是局部变量，加锁毫无意义，便会将其优化掉，真正执行是只执行`x++`操作。

```java
Benchmark            Mode  Samples  Score  Score error  Units 
c.i.MyBenchmark.a    avgt        5  1.542        0.056  ns/op 
c.i.MyBenchmark.b    avgt        5  1.518        0.091  ns/op 
```

`java -XX:-EliminateLocks -jar benchmarks.jar`

当关掉锁消除时就可以发现加锁时的性能非常差。

```java
Benchmark            Mode  Samples   Score  Score error  Units 
c.i.MyBenchmark.a    avgt        5   1.507        0.108  ns/op 
c.i.MyBenchmark.b    avgt        5  16.976        1.572  ns/op 
```

**锁粗化**

在遇到一连串地对同一锁不断进行请求和释放的操作时，把所有的锁操作整合成锁的一次请求，从而减少对锁的请求同步次数，这个操作叫做锁的粗化。

## 3.变量的线程安全分析

**成员变量和静态变量是否线程安全？** 

- 如果它们没有共享，则线程安全
- 如果它们被共享了，根据它们的状态是否能够改变，又分两种情况
  - 如果只有读操作，则线程安全
  - 如果有读写操作，则这段代码是临界区，需要考虑线程安全

**局部变量是否线程安全？**

- 局部变量是线程安全的
- 但局部变量引用的对象则未必
  - 如果该对象没有逃离方法的作用访问，它是线程安全的
  - 如果该对象逃离方法的作用范围，需要考虑线程安全

### 3.1 局部变量线程安全分析

```java
public static void test1() {
    int i = 10;
    i++;
}
```

每个线程调用`test1()`方法时局部变量`i`，会在每个线程的栈帧内存中被创建多份，因此不存在共享

```java
public static void test1();
    descriptor: ()V
        flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=1, args_size=0
         0: bipush        10
         2: istore_0
         3: iinc          0, 1
         6: return
      LineNumberTable:
        line 10: 0
        line 11: 3
        line 12: 6
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            3       4     0     i   I
```

如图

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815142849707-e6647477e30ae9a5c037c0ad1b191fe4-b0f8b9.png" alt="image-20220815142849707" style="zoom:80%;" />

局部变量的引用稍有不同。

先看一个成员变量的例子：

```java
class ThreadUnsafe {
    ArrayList<String> list = new ArrayList<>();
    
    public void method1(int loopNumber) {
        for (int i = 0; i < loopNumber; i++) {
            // { 临界区, 会产生竞态条件
            method2();
            method3();
            // } 临界区
        }
    }

    private void method2() {
        list.add("1");
    }

    private void method3() {
        list.remove(0);
    }
}
```

执行

```java
static final int THREAD_NUMBER = 2;
static final int LOOP_NUMBER = 200;

public static void main(String[] args) {
    ThreadUnsafe test = new ThreadUnsafe();
    for (int i = 0; i < THREAD_NUMBER; i++) {
        new Thread(() -> {
            test.method1(LOOP_NUMBER);
        }, "Thread" + i).start();
    }
}
```

其中一种情况是，如果线程 2 还未 add，线程 1 remove 就会报错：

```java
Exception in thread "Thread1" java.lang.IndexOutOfBoundsException: Index: 0, Size: 0 
 at java.util.ArrayList.rangeCheck(ArrayList.java:657) 
 at java.util.ArrayList.remove(ArrayList.java:496) 
 at cn.itcast.n6.ThreadUnsafe.method3(TestThreadSafe.java:35) 
 at cn.itcast.n6.ThreadUnsafe.method1(TestThreadSafe.java:26) 
 at cn.itcast.n6.TestThreadSafe.lambda$main$0(TestThreadSafe.java:14) 
 at java.lang.Thread.run(Thread.java:748) 
```

分析：

- 无论哪个线程中的 method2 引用的都是同一个对象中的 list 成员变量
- method3 与 method2 分析相同

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815142954560-519c1c19dca2a0e3cf1ae1619506a73f-01cdbf.png" alt="image-20220815142954560" style="zoom: 67%;" />

将 list 修改为局部变量就不会有上述问题了。

```java
class ThreadSafe {
    public final void method1(int loopNumber) {
        ArrayList<String> list = new ArrayList<>();
        for (int i = 0; i < loopNumber; i++) {
            method2(list);
            method3(list);
        }
    }
 
    private void method2(ArrayList<String> list) {
        list.add("1");
    }
 
    private void method3(ArrayList<String> list) {
        list.remove(0);
    }
}
```

分析：

- list 是局部变量，每个线程调用时会创建其不同实例，没有共享
- 而 method2 的参数是从 method1 中传递过来的，与 method1 中引用同一个对象
- method3 的参数分析与 method2 相同

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815143029635-debe679cfd01fa2304a167b7ed793afd-8839b0.png" alt="image-20220815143029635" style="zoom:67%;" />

方法访问修饰符带来的思考，如果把 method2 和 method3 的方法修改为 public 会不会代理线程安全问题？

- 情况1：有其它线程调用 method2 和 method3
- 情况2：在 情况1 的基础上，为 ThreadSafe 类添加子类，子类覆盖 method2 或 method3 方法，即

```java
class ThreadSafe {
    public final void method1(int loopNumber) {
        ArrayList<String> list = new ArrayList<>();
        for (int i = 0; i < loopNumber; i++) {
            method2(list);
            method3(list);
        }
    }

    private void method2(ArrayList<String> list) {
        list.add("1");
    }

    private void method3(ArrayList<String> list) {
        list.remove(0);
    }
}

class ThreadSafeSubClass extends ThreadSafe{
    @Override
    public void method3(ArrayList<String> list) {
        new Thread(() -> {
            list.remove(0);
        }).start();
    }
}
```

> 从这个例子可以看出 private 或 ﬁnal 提供【安全】的意义所在，请体会开闭原则中的【闭】

### 3.2 常见线程安全类

- String
- Integer
- StringBuﬀer
- Random
- Vector
- Hashtable
- java.util.concurrent 包下的类

这里说它们是线程安全的是指，多个线程调用它们同一个实例的某个方法时，是线程安全的。也可以理解为

```java
Hashtable table = new Hashtable();
 
new Thread(()->{
    table.put("key", "value1");
}).start();
 
new Thread(()->{
    table.put("key", "value2");
}).start();
```

- 它们的每个方法是原子的
- 但注意它们多个方法的组合不是原子的，见后面分析

**线程安全类方法的组合**

对**同一个变量**执行多个线程安全方法可能造成线程不安全。

```java
Hashtable table = new Hashtable();
// 线程1，线程2
if( table.get("key") == null) {
    table.put("key", value);
}
```

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815143217377-94aa33b2f501f342abafa2770474e821-9d074a.png" alt="image-20220815143217377" style="zoom: 80%;" />

**不可变类线程安全性**

`String`、`Integer`等都是不可变类，因为其内部的状态不可以改变，因此它们的方法都是**线程安全**的。

String 执行`replace`，`substring`等方法时会创建一个新的 String 对象，可以仿照 String 的实现方式：

```java
public class Immutable{
    private int value = 0;

    public Immutable(int value){
        this.value = value;
    }

    public int getValue(){
        return this.value;
    }
}
```

如果想增加一个增加的方法呢？

```java
public class Immutable{
    private int value = 0;

    public Immutable(int value){
        this.value = value;
    }

    public int getValue(){
        return this.value;
    }

    public Immutable add(int v){
        return new Immutable(this.value + v);
    }  
}
```

### 3.3 实例分析

例 1：

```java
public class MyServlet extends HttpServlet {
    // 不安全
    Map<String,Object> map = new HashMap<>();
    // 安全
    String S1 = "...";
    // 安全
    final String S2 = "...";
    // 不安全
    Date D1 = new Date();
    // 不安全：final 只是限定了引用地址不变，但是 Date 的属性状态可能发生变化
    final Date D2 = new Date();

    public void doGet(HttpServletRequest request, HttpServletResponse response) {
        // 使用上述变量
    }
}
```

例 2：

```java
public class MyServlet extends HttpServlet { 
    private UserService userService = new UserServiceImpl();
    
    public void doGet(HttpServletRequest request, HttpServletResponse response) {
        userService.update(...);
    }
}
 
public class UserServiceImpl implements UserService {
    // 记录调用次数，不安全
    private int count = 0;
    
    public void update() {
        // ...
        count++;
    }
}
```

例 3：

```java
@Aspect
@Component
public class MyAspect {
    // 不安全
    private long start = 0L;

    @Before("execution(* *(..))")
    public void before() {
        start = System.nanoTime();
    }

    @After("execution(* *(..))")
    public void after() {
        long end = System.nanoTime();
        System.out.println("cost time:" + (end-start));
    }
}
```

例 4：

```java
public class MyServlet extends HttpServlet {

    private UserService userService = new UserServiceImpl();

    public void doGet(HttpServletRequest request, HttpServletResponse response) {
        userService.update(...);
    }
}

public class UserServiceImpl implements UserService {

    private UserDao userDao = new UserDaoImpl();

    public void update() {
        userDao.update();
    }
}

public class UserDaoImpl implements UserDao { 
    public void update() {
        String sql = "update user set password = ? where username = ?";
        // 安全
        try (Connection conn = DriverManager.getConnection("","","")){
            // ...
        } catch (Exception e) {
            // ...
        }
    }
}
```

例 5：

```java
public class MyServlet extends HttpServlet {

    private UserService userService = new UserServiceImpl();

    public void doGet(HttpServletRequest request, HttpServletResponse response) {
        userService.update(...);
    }
}

public class UserServiceImpl implements UserService {
    
    private UserDao userDao = new UserDaoImpl();

    public void update() {
        userDao.update();
    }
}
public class UserDaoImpl implements UserDao {
    // 不安全
    private Connection conn = null;
    
    public void update() throws SQLException {
        String sql = "update user set password = ? where username = ?";
        conn = DriverManager.getConnection("","","");
        // ...
        conn.close();
    }
}
```

例 6：

```java
public class MyServlet extends HttpServlet {

    private UserService userService = new UserServiceImpl();
    
    public void doGet(HttpServletRequest request, HttpServletResponse response) {
        userService.update(...);
    }
}
 
public class UserServiceImpl implements UserService {    
    public void update() {
        // 安全
        UserDao userDao = new UserDaoImpl();
        userDao.update();
    }
}
 
public class UserDaoImpl implements UserDao {

    private Connection = null;
    public void update() throws SQLException {
        String sql = "update user set password = ? where username = ?";
        conn = DriverManager.getConnection("","","");
        // ...
        conn.close();
    }
}
```

例 7：

```java
public abstract class Test {

    public void bar() {
        // 是否安全
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        foo(sdf);
    }

    public abstract foo(SimpleDateFormat sdf);


    public static void main(String[] args) {
        new Test().bar();
    }
}
```

其中 foo 的行为是不确定的，可能导致不安全的发生，被称之为外星方法

```java
public void foo(SimpleDateFormat sdf) {
    String dateStr = "1999-10-11 00:00:00";
    for (int i = 0; i < 20; i++) {
        new Thread(() -> {
            try {
                sdf.parse(dateStr);
            } catch (ParseException e) {
                e.printStackTrace();
            }
        }).start();
    }
}
```

请比较 JDK 中 String 类的实现，类是`final`的，避免了子类重写方法导致线程安全问题。

例 8：

```java
private static Integer i = 0;
public static void main(String[] args) throws InterruptedException {
    List<Thread> list = new ArrayList<>();
    for (int j = 0; j < 2; j++) {
        Thread thread = new Thread(() -> {
            for (int k = 0; k < 5000; k++) {
                synchronized (i) {
                    i++;
                }
            }
        }, "" + j);
        list.add(thread);
    }
    list.stream().forEach(t -> t.start());
    list.stream().forEach(t -> {
        try {
            t.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });
    log.debug("{}", i);
}
```

## 4.wait notify

**原理**

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220816145431113-f30efb0052e4b01ffdd3692e4dcc46dc-42a2d7.png" alt="image-20220816145431113" style="zoom:67%;" />

- 获得了锁之后，Owner 线程发现条件不满足，调用 wait 方法，即可进入 WaitSet 变为 WAITING 状态
- BLOCKED 和 WAITING 的线程都处于阻塞状态，不占用 CPU 时间片
- BLOCKED 线程会在 Owner 线程释放锁时唤醒
- WAITING 线程会在 Owner 线程调用 notify 或 notifyAll 时唤醒，但**唤醒后并不意味者立刻获得锁**，仍需进入 
- EntryList 重新竞争

**wait/sleep 区别**

1. 二者来自不同的类

   - wait => Object

   - sleep => Thread

2. 关于锁的释放

   - wait 会释放锁

   - sleep 不会释放

3. 使用的范围是不同的

   - wait 必须在同步代码块中使用

   - sleep 可以在任何地方睡眠

**API**

- `obj.wait()`让进入 object 监视器的线程到 waitSet 等待
- `obj.notify()`在 object 上正在 waitSet 等待的线程中挑一个唤醒 
- `obj.notifyAll()`让 object 上正在 waitSet 等待的线程全部唤醒

它们都是线程之间进行协作的手段，都属于 Object 对象的方法。必须获得此对象的锁，才能调用这几个方法。直接调用会产生`IllegalMonitorStateException`异常。

```java
final static Object obj = new Object();
 
public static void main(String[] args) {
 
    new Thread(() -> {
        synchronized (obj) {
            log.debug("执行....");
            try {
                obj.wait(); // 让线程在obj上一直等待下去
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.debug("其它代码....");
        }
    }).start();
 
    new Thread(() -> {
        synchronized (obj) {
            log.debug("执行....");
            try {
                obj.wait(); // 让线程在obj上一直等待下去
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.debug("其它代码....");
        }
    }).start();
 
    // 主线程两秒后执行
    sleep(2);
    log.debug("唤醒 obj 上其它线程");
    synchronized (obj) {
        obj.notify(); // 唤醒obj上一个线程
        // obj.notifyAll(); // 唤醒obj上所有等待线程
    }
}
```

notify 的一种结果

```bash
20:00:53.096 [Thread-0] c.TestWaitNotify - 执行.... 
20:00:53.099 [Thread-1] c.TestWaitNotify - 执行.... 
20:00:55.096 [main] c.TestWaitNotify - 唤醒 obj 上其它线程 
20:00:55.096 [Thread-0] c.TestWaitNotify - 其它代码.... 
```

notifyAll 的结果

```bash
19:58:15.457 [Thread-0] c.TestWaitNotify - 执行.... 
19:58:15.460 [Thread-1] c.TestWaitNotify - 执行.... 
19:58:17.456 [main] c.TestWaitNotify - 唤醒 obj 上其它线程 
19:58:17.456 [Thread-1] c.TestWaitNotify - 其它代码.... 
19:58:17.456 [Thread-0] c.TestWaitNotify - 其它代码.... 
```

- `wait()`方法会释放对象的锁，进入 WaitSet 等待区，从而让其他线程就机会获取对象的锁。无限制等待，直到`notify`为止

- `wait(long n)`有时限的等待，到 n 毫秒后结束等待，或是被`notify`

> **`sleep(long n)`和`wait(long n)`的区别**
>
> 1. `sleep`是 Thread 方法，而`wait`是 Object 的方法
>
> 2. `sleep`不需要强制和`synchronized`配合使用，但`wait`需要和`synchronized`一起用
> 3. `sleep`在睡眠的同时，不会释放对象锁的，但`wait`在等待的时候会释放对象锁
> 4. 它们状态都是`TIMED_WAITING`

### 4.1 wait notify的应用

**step 1**

```java
static final Object room = new Object();
static boolean hasCigarette = false;
static boolean hasTakeout = false;
```

思考下面的解决方案好不好，为什么？

```java
new Thread(() -> {
    synchronized (room) {
        log.debug("有烟没？[{}]", hasCigarette);
        if (!hasCigarette) {
            log.debug("没烟，先歇会！");
            sleep(2);
        }
        log.debug("有烟没？[{}]", hasCigarette);
        if (hasCigarette) {
            log.debug("可以开始干活了");
        }
    }
}, "小南").start();

for (int i = 0; i < 5; i++) {
    new Thread(() -> {
        synchronized (room) {
            log.debug("可以开始干活了");
        }
    }, "其它人").start();
}

sleep(1);
new Thread(() -> {
    // 这里能不能加 synchronized (room)？
    hasCigarette = true;
    log.debug("烟到了噢！");
}, "送烟的").start();
```

输出

```bash
20:49:49.883 [小南] c.TestCorrectPosture - 有烟没？[false] 
20:49:49.887 [小南] c.TestCorrectPosture - 没烟，先歇会！ 
20:49:50.882 [送烟的] c.TestCorrectPosture - 烟到了噢！ 
20:49:51.887 [小南] c.TestCorrectPosture - 有烟没？[true] 
20:49:51.887 [小南] c.TestCorrectPosture - 可以开始干活了 
20:49:51.887 [其它人] c.TestCorrectPosture - 可以开始干活了 
20:49:51.887 [其它人] c.TestCorrectPosture - 可以开始干活了 
20:49:51.888 [其它人] c.TestCorrectPosture - 可以开始干活了 
20:49:51.888 [其它人] c.TestCorrectPosture - 可以开始干活了 
20:49:51.888 [其它人] c.TestCorrectPosture - 可以开始干活了 
```

- 其它干活的线程，都要一直阻塞，效率太低
- 小南线程必须睡足 2s 后才能醒来，就算烟提前送到，也无法立刻醒来
- 加了`synchronized (room)`后，就好比小南在里面反锁了门睡觉，烟根本没法送进门，main 没加 synchronized 就好像 main 线程是翻窗户进来的
- 解决方法，使用`wait - notify`机制

**step 2**

思考下面的实现行吗，为什么？

```java
new Thread(() -> {
    synchronized (room) {
        log.debug("有烟没？[{}]", hasCigarette);
        if (!hasCigarette) {
            log.debug("没烟，先歇会！");
            try {
                room.wait(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        log.debug("有烟没？[{}]", hasCigarette);
        if (hasCigarette) {
            log.debug("可以开始干活了");
        }
    }
}, "小南").start();
for (int i = 0; i < 5; i++) {
    new Thread(() -> {
        synchronized (room) {
            log.debug("可以开始干活了");
        }
    }, "其它人").start();
}

sleep(1);
new Thread(() -> {
    synchronized (room) {
        hasCigarette = true;
        log.debug("烟到了噢！");
        room.notify();
    }
}, "送烟的").start();
```

输出

```bash
20:51:42.489 [小南] c.TestCorrectPosture - 有烟没？[false] 
20:51:42.493 [小南] c.TestCorrectPosture - 没烟，先歇会！ 
20:51:42.493 [其它人] c.TestCorrectPosture - 可以开始干活了 
20:51:42.493 [其它人] c.TestCorrectPosture - 可以开始干活了 
20:51:42.494 [其它人] c.TestCorrectPosture - 可以开始干活了 
20:51:42.494 [其它人] c.TestCorrectPosture - 可以开始干活了 
20:51:42.494 [其它人] c.TestCorrectPosture - 可以开始干活了 
20:51:43.490 [送烟的] c.TestCorrectPosture - 烟到了噢！ 
20:51:43.490 [小南] c.TestCorrectPosture - 有烟没？[true] 
20:51:43.490 [小南] c.TestCorrectPosture - 可以开始干活了 
```

解决了其它干活的线程阻塞的问题，但如果有其它线程也在等待条件呢？

**step 3**

```java
new Thread(() -> {
    synchronized (room) {
        log.debug("有烟没？[{}]", hasCigarette);
        if (!hasCigarette) {
            log.debug("没烟，先歇会！");
            try {
                room.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        log.debug("有烟没？[{}]", hasCigarette);
        if (hasCigarette) {
            log.debug("可以开始干活了");
        } else {
            log.debug("没干成活...");
        }
    }
}, "小南").start();

new Thread(() -> {
    synchronized (room) {
        Thread thread = Thread.currentThread();
        log.debug("外卖送到没？[{}]", hasTakeout);
        if (!hasTakeout) {
            log.debug("没外卖，先歇会！");
            try {
                room.wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        log.debug("外卖送到没？[{}]", hasTakeout);
        if (hasTakeout) {
            log.debug("可以开始干活了");
        } else {
            log.debug("没干成活...");
        }
    }
}, "小女").start();

sleep(1);
new Thread(() -> {
    synchronized (room) {
        hasTakeout = true;
        log.debug("外卖到了噢！");
        room.notify();
    }
}, "送外卖的").start();
```

输出

```bash
20:53:12.173 [小南] c.TestCorrectPosture - 有烟没？[false] 
20:53:12.176 [小南] c.TestCorrectPosture - 没烟，先歇会！ 
20:53:12.176 [小女] c.TestCorrectPosture - 外卖送到没？[false] 
20:53:12.176 [小女] c.TestCorrectPosture - 没外卖，先歇会！ 
20:53:13.174 [送外卖的] c.TestCorrectPosture - 外卖到了噢！ 
20:53:13.174 [小南] c.TestCorrectPosture - 有烟没？[false] 
20:53:13.174 [小南] c.TestCorrectPosture - 没干成活... 
```

`notify`只能随机唤醒一个 WaitSet 中的线程，这时如果有其它线程也在等待，那么就可能唤醒不了正确的线程，称之为【虚假唤醒】，解决方法，改为`notifyAll`。

**step 4**

```java
new Thread(() -> {
    synchronized (room) {
        hasTakeout = true;
        log.debug("外卖到了噢！");
        room.notifyAll();
    }
}, "送外卖的").start();
```

输出

```bash
20:55:23.978 [小南] c.TestCorrectPosture - 有烟没？[false] 
20:55:23.982 [小南] c.TestCorrectPosture - 没烟，先歇会！ 
20:55:23.982 [小女] c.TestCorrectPosture - 外卖送到没？[false] 
20:55:23.982 [小女] c.TestCorrectPosture - 没外卖，先歇会！ 
20:55:24.979 [送外卖的] c.TestCorrectPosture - 外卖到了噢！ 
20:55:24.979 [小女] c.TestCorrectPosture - 外卖送到没？[true] 
20:55:24.980 [小女] c.TestCorrectPosture - 可以开始干活了 
20:55:24.980 [小南] c.TestCorrectPosture - 有烟没？[false] 
20:55:24.980 [小南] c.TestCorrectPosture - 没干成活... 
```

用`notifyAll`仅解决某个线程的唤醒问题，但使用`if + wait`判断仅有一次机会，一旦条件不成立，就没有重新判断的机会了，解决方法，用`while + wait`，当条件不成立，再次`wait`。

**step 5**

将 if 改为 while

```java
while (!hasCigarette) {
    log.debug("没烟，先歇会！");
    try {
        room.wait();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

输出

```bash
20:58:34.322 [小南] c.TestCorrectPosture - 有烟没？[false] 
20:58:34.326 [小南] c.TestCorrectPosture - 没烟，先歇会！ 
20:58:34.326 [小女] c.TestCorrectPosture - 外卖送到没？[false] 
20:58:34.326 [小女] c.TestCorrectPosture - 没外卖，先歇会！ 
20:58:35.323 [送外卖的] c.TestCorrectPosture - 外卖到了噢！ 
20:58:35.324 [小女] c.TestCorrectPosture - 外卖送到没？[true] 
20:58:35.324 [小女] c.TestCorrectPosture - 可以开始干活了 
20:58:35.324 [小南] c.TestCorrectPosture - 没烟，先歇会！ 
```

```java
synchronized(lock) {
    while(条件不成立) {
        lock.wait();
    }
    // 干活
}
 
//另一个线程
synchronized(lock) {
    lock.notifyAll();
}
```

### 扩展：模式之保护性暂停 

#### 1.定义

即 Guarded Suspension，用在一个线程等待另一个线程的执行结果。

**要点**

- 有一个结果需要从一个线程传递到另一个线程，让他们关联同一个 GuardedObject
- 如果有结果不断从一个线程到另一个线程那么可以使用消息队列（见生产者/消费者）
- JDK 中，join 的实现、Future 的实现，采用的就是此模式
- 因为要等待另一方的结果，因此归类到同步模式

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220816202612067-8ee1148e787010bb6a1999f7daff9264-70f441.png" alt="image-20220816202612067" style="zoom:67%;" />

#### 2.实现

```java
class GuardedObject {
    private Object response;
    private final Object lock = new Object();

    public Object get() {
        synchronized (lock) {
            // 条件不满足则等待
            while (response == null) {
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            return response;
        }
    }

    public void complete(Object response) {
        synchronized (lock) {
            // 条件满足，通知等待线程
            this.response = response;
            lock.notifyAll();
        }
    }
}
```

#### 3.应用

一个线程等待另一个线程的执行结果

```java
public static void main(String[] args) {
    GuardedObject guardedObject = new GuardedObject();
    new Thread(() -> {
        try {
            // 子线程执行下载
            List<String> response = download();
            log.debug("download complete...");
            guardedObject.complete(response);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }).start();

    log.debug("waiting...");
    // 主线程阻塞等待
    Object response = guardedObject.get();
    log.debug("get response: [{}] lines", ((List<String>) response).size());

}
```

执行结果

```bash
08:42:18.568 [main] c.TestGuardedObject - waiting...
08:42:23.312 [Thread-0] c.TestGuardedObject - download complete...
08:42:23.312 [main] c.TestGuardedObject - get response: [3] lines
```

#### 4.带超时版GuardedObject

如果要控制超时时间呢

```java
class GuardedObjectV2 {

    private Object response;
    private final Object lock = new Object();
    public Object get(long millis) {
        synchronized (lock) {
            // 1) 记录最初时间
            long begin = System.currentTimeMillis();
            // 2) 已经经历的时间
            long timePassed = 0;
            while (response == null) {
                // 4) 假设 millis 是 1000，结果在 400 时唤醒了，那么还有 600 要等
                long waitTime = millis - timePassed;
                log.debug("waitTime: {}", waitTime);
                if (waitTime <= 0) {
                    log.debug("break...");
                    break;
                }
                try {
                    lock.wait(waitTime);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                // 3) 如果提前被唤醒，这时已经经历的时间假设为 400
                timePassed = System.currentTimeMillis() - begin;
                log.debug("timePassed: {}, object is null {}", 
                          timePassed, response == null);
            }
            return response;
        }
    }
    public void complete(Object response) {
        synchronized (lock) {
            // 条件满足，通知等待线程
            this.response = response;
            log.debug("notify...");
            lock.notifyAll();
        }
    }
}
```

测试，没有超时

```java
public static void main(String[] args) {
    GuardedObjectV2 v2 = new GuardedObjectV2();
    new Thread(() -> {
        sleep(1);
        v2.complete(null);
        sleep(1);
        v2.complete(Arrays.asList("a", "b", "c"));
    }).start();

    Object response = v2.get(2500);
    if (response != null) {

        log.debug("get response: [{}] lines", ((List<String>) response).size());
    } else {
        log.debug("can't get response");
    }
}
```

输出

```bash
08:49:39.917 [main] c.GuardedObjectV2 - waitTime: 2500
08:49:40.917 [Thread-0] c.GuardedObjectV2 - notify...
08:49:40.917 [main] c.GuardedObjectV2 - timePassed: 1003, object is null true
08:49:40.917 [main] c.GuardedObjectV2 - waitTime: 1497
08:49:41.918 [Thread-0] c.GuardedObjectV2 - notify...
08:49:41.918 [main] c.GuardedObjectV2 - timePassed: 2004, object is null false
08:49:41.918 [main] c.TestGuardedObjectV2 - get response: [3] lines
```

测试，超时

```java
// 等待时间不足
List<String> lines = v2.get(1500);
```

输出

```bash
08:47:54.963 [main] c.GuardedObjectV2 - waitTime: 1500
08:47:55.963 [Thread-0] c.GuardedObjectV2 - notify...
08:47:55.963 [main] c.GuardedObjectV2 - timePassed: 1002, object is null true
08:47:55.963 [main] c.GuardedObjectV2 - waitTime: 498
08:47:56.461 [main] c.GuardedObjectV2 - timePassed: 1500, object is null true
08:47:56.461 [main] c.GuardedObjectV2 - waitTime: 0
08:47:56.461 [main] c.GuardedObjectV2 - break...
08:47:56.461 [main] c.TestGuardedObjectV2 - can't get response
08:47:56.963 [Thread-0] c.GuardedObjectV2 - notify...
```

#### 5.原理：join

 调用者轮询检查线程 alive 状态

```java
t1.join();
```

等价于下面的代码

```java
synchronized (t1) {
    // 调用者线程进入 t1 的 waitSet 等待, 直到 t1 运行结束
    while (t1.isAlive()) {
        t1.wait(0);
    }
}
```

#### 6.多任务版GuardedObject

图中 Futures 就好比居民楼一层的信箱（每个信箱有房间编号），左侧的 t0，t2，t4 就好比等待邮件的居民，右侧的 t1，t3，t5 就好比邮递员。

如果需要在多个类之间使用 GuardedObject 对象，作为参数传递不是很方便，因此设计一个用来解耦的中间类，这样不仅能够解耦【结果等待者】和【结果生产者】，还能够同时支持多个任务的管理。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220816204220359-1864263447afc7d6c845ac24026844b8-94e86c.png" alt="image-20220816204220359" style="zoom:67%;" />

新增 id 用来标识 Guarded Object

```java
class GuardedObject {

    // 标识 Guarded Object
    private int id;

    public GuardedObject(int id) {
        this.id = id;
    }

    public int getId() {
        return id;
    }

    // 结果
    private Object response;

    // 获取结果
    // timeout 表示要等待多久 2000
    public Object get(long timeout) {
        synchronized (this) {
            // 开始时间 15:00:00
            long begin = System.currentTimeMillis();
            // 经历的时间
            long passedTime = 0;
            while (response == null) {
                // 这一轮循环应该等待的时间
                long waitTime = timeout - passedTime;
                // 经历的时间超过了最大等待时间时，退出循环
                if (timeout - passedTime <= 0) {
                    break;
                }
                try {
                    this.wait(waitTime); // 虚假唤醒 15:00:01
                } catch (InterruptedException e) {

                    e.printStackTrace();
                }
                // 求得经历时间
                passedTime = System.currentTimeMillis() - begin; // 15:00:02  1s
            }
            return response;
        }
    }

    // 产生结果
    public void complete(Object response) {
        synchronized (this) {
            // 给结果成员变量赋值
            this.response = response;
            this.notifyAll();
        }
    }
}
```

中间解耦类

```java
class Mailboxes {
    private static Map<Integer, GuardedObject> boxes = new Hashtable<>();
 
    private static int id = 1;
    // 产生唯一 id
    private static synchronized int generateId() {
        return id++;
    }
 
    public static GuardedObject getGuardedObject(int id) {
        return boxes.remove(id);
    }
 
    public static GuardedObject createGuardedObject() {
        GuardedObject go = new GuardedObject(generateId());
        boxes.put(go.getId(), go);
        return go;
    }
 
    public static Set<Integer> getIds() {
        return boxes.keySet();
    }
}
```

业务相关类

```java
class People extends Thread{
    @Override
    public void run() {
        // 收信
        GuardedObject guardedObject = Mailboxes.createGuardedObject();
        log.debug("开始收信 id:{}", guardedObject.getId());
        Object mail = guardedObject.get(5000);
        log.debug("收到信 id:{}, 内容:{}", guardedObject.getId(), mail);
    }
}
```

```java
class Postman extends Thread {
    private int id;
    private String mail;
 
    public Postman(int id, String mail) {
        this.id = id;
        this.mail = mail;
    }
 
    @Override
    public void run() {
        GuardedObject guardedObject = Mailboxes.getGuardedObject(id);
        log.debug("送信 id:{}, 内容:{}", id, mail);
        guardedObject.complete(mail);
    }
}
```

测试

```java
public static void main(String[] args) throws InterruptedException {
    for (int i = 0; i < 3; i++) {
        new People().start();
    }
    Sleeper.sleep(1);
    for (Integer id : Mailboxes.getIds()) {
        new Postman(id, "内容" + id).start();
    }
}
```

某次运行结果

```bash
10:35:05.689 c.People [Thread-1] - 开始收信 id:3
10:35:05.689 c.People [Thread-2] - 开始收信 id:1
10:35:05.689 c.People [Thread-0] - 开始收信 id:2
10:35:06.688 c.Postman [Thread-4] - 送信 id:2, 内容:内容2
10:35:06.688 c.Postman [Thread-5] - 送信 id:1, 内容:内容1
10:35:06.688 c.People [Thread-0] - 收到信 id:2, 内容:内容2
10:35:06.688 c.People [Thread-2] - 收到信 id:1, 内容:内容1
10:35:06.688 c.Postman [Thread-3] - 送信 id:3, 内容:内容3
10:35:06.689 c.People [Thread-1] - 收到信 id:3, 内容:内容3
```

### 扩展：异步模式之生产者消费者

#### 1.定义

 **要点**

- 与前面的保护性暂停中的 GuardObject 不同，不需要产生结果和消费结果的线程一一对应
- 消费队列可以用来平衡生产和消费的线程资源
- 生产者仅负责产生结果数据，不关心数据该如何处理，而消费者专心处理结果数据
- 消息队列是有容量限制的，满时不会再加入数据，空时不会再消耗数据
- JDK 中各种阻塞队列，采用的就是这种模式

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220816214426931-bcc6ddf8e0e87568e6c7aa6512ecea0a-5c802e.png" alt="image-20220816214426931" style="zoom:67%;" />

#### 2.实现

```java
final class Message {
    private int id;
    private Object message;

    public Message(int id, Object message) {
        this.id = id;
        this.message = message;
    }

    public int getId() {
        return id;
    }

    public Object getMessage() {
        return message;
    }
}

class MessageQueue {
    private LinkedList<Message> queue;
    private int capacity;

    public MessageQueue(int capacity) {
        this.capacity = capacity;
        queue = new LinkedList<>();
    }

    public Message take() {
        synchronized (queue) {
            while (queue.isEmpty()) {
                log.debug("没货了, wait");
                try {
                    queue.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            Message message = queue.removeFirst();
            queue.notifyAll();
            return message;
        }
    }
    public void put(Message message) {
        synchronized (queue) {
            while (queue.size() == capacity) {
                log.debug("库存已达上限, wait");
                try {
                    queue.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            queue.addLast(message);
            queue.notifyAll();
        }
    }
}
```

#### 3.应用

```java
MessageQueue messageQueue = new MessageQueue(2);
// 4 个生产者线程, 下载任务
for (int i = 0; i < 4; i++) {
    int id = i;
    new Thread(() -> {
        try {
            log.debug("download...");
            List<String> response = Downloader.download();
            log.debug("try put message({})", id);
            messageQueue.put(new Message(id, response));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }, "生产者" + i).start();
}

// 1 个消费者线程, 处理结果
new Thread(() -> {
    while (true) {
        Message message = messageQueue.take();
        List<String> response = (List<String>) message.getMessage();
        log.debug("take message({}): [{}] lines", message.getId(), response.size());
    }

}, "消费者").start();
```

某次运行结果

```bash
10:48:38.070 [生产者3] c.TestProducerConsumer - download...
10:48:38.070 [生产者0] c.TestProducerConsumer - download...
10:48:38.070 [消费者] c.MessageQueue - 没货了, wait
10:48:38.070 [生产者1] c.TestProducerConsumer - download...
10:48:38.070 [生产者2] c.TestProducerConsumer - download...
10:48:41.236 [生产者1] c.TestProducerConsumer - try put message(1)
10:48:41.237 [生产者2] c.TestProducerConsumer - try put message(2)
10:48:41.236 [生产者0] c.TestProducerConsumer - try put message(0)
10:48:41.237 [生产者3] c.TestProducerConsumer - try put message(3)
10:48:41.239 [生产者2] c.MessageQueue - 库存已达上限, wait
10:48:41.240 [生产者1] c.MessageQueue - 库存已达上限, wait
10:48:41.240 [消费者] c.TestProducerConsumer - take message(0): [3] lines
10:48:41.240 [生产者2] c.MessageQueue - 库存已达上限, wait
10:48:41.240 [消费者] c.TestProducerConsumer - take message(3): [3] lines
10:48:41.240 [消费者] c.TestProducerConsumer - take message(1): [3] lines
10:48:41.240 [消费者] c.TestProducerConsumer - take message(2): [3] lines
10:48:41.240 [消费者] c.MessageQueue - 没货了, wait
```

## 5.Park & Unpark

**基本使用**

它们是 LockSupport 类中的方法

```java
// 暂停当前线程，状态为 wait
LockSupport.park(); 
 
// 恢复某个线程的运行
LockSupport.unpark(暂停线程对象)
```

先 park 再 unpark

```java
Thread t1 = new Thread(() -> {
    log.debug("start...");
    sleep(1);
    log.debug("park...");
    LockSupport.park();
    log.debug("resume...");
},"t1");
t1.start();
 
sleep(2);
log.debug("unpark...");
LockSupport.unpark(t1);
```

输出

```java
18:42:52.585 c.TestParkUnpark [t1] - start... 
18:42:53.589 c.TestParkUnpark [t1] - park... 
18:42:54.583 c.TestParkUnpark [main] - unpark... 
18:42:54.583 c.TestParkUnpark [t1] - resume... 
```

先 unpark 再 park

```java
Thread t1 = new Thread(() -> {
    log.debug("start...");
    sleep(2);
    log.debug("park...");
    LockSupport.park();
    log.debug("resume...");
}, "t1");
t1.start();
 
sleep(1);
log.debug("unpark...");
LockSupport.unpark(t1);
```

输出

```java
18:43:50.765 c.TestParkUnpark [t1] - start... 
18:43:51.764 c.TestParkUnpark [main] - unpark... 
18:43:52.769 c.TestParkUnpark [t1] - park... 
18:43:52.769 c.TestParkUnpark [t1] - resume... 
```

**特点**

与 Object 的`wait & notify`相比

- `wait`，`notify`和`notifyAll`必须配合 Object Monitor 一起使用，而`park`，`unpark`不必
- `park & unpark`是以线程为单位来【阻塞】和【唤醒】线程，而`notify`只能随机唤醒一个等待线程，`notifyAll`是唤醒所有等待线程，就不那么【精确】
- `park & unpark`可以先`unpark`，而`wait & notify`不能先`notify`

**原理**

每个线程都有自己的一个 Parker 对象，由三部分组成`_counter`，`_cond`和`_mutex`。

打个比喻

- 线程就像一个旅人，Parker 就像他随身携带的背包，条件变量就好比背包中的帐篷。`_counter`就好比背包中的备用干粮（0 为耗尽，1 为充足）
- 调用`park`就是要看需不需要停下来歇息
  - 如果备用干粮耗尽，那么钻进帐篷歇息
  - 如果备用干粮充足，那么不需停留，继续前进
- 调用`unpark`，就好比令干粮充足
  - 如果这时线程还在帐篷，就唤醒让他继续前进
  - 如果这时线程还在运行，那么下次他调用`park`时，仅是消耗掉备用干粮，不需停留继续前进
    - 因为背包空间有限，多次调用`unpark`仅会补充一份备用干粮

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220817115105612-3f39239f8e4b54257135167e3ac4ec8e-4ea52e.png" alt="image-20220817115105612" style="zoom:67%;" />

1. 当前线程调用`Unsafe.park()`方法
2. 检查 _counter ，本情况为 0，这时，获得 _mutex 互斥锁
3. 线程进入 _cond 条件变量阻塞
4. 设置 _counter = 0

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220817115123210-8b0d89ed4714db9880d624c19e14cf16-f61262.png" alt="image-20220817115123210" style="zoom:67%;" />

1. 调用`Unsafe.unpark(Thread_0)`方法，设置 _counter 为 1
2. 唤醒 _cond 条件变量中的 Thread_0
3. Thread_0 恢复运行
4. 设置 _counter 为 0

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220817115134064-ef828f96b222f5b753c7e334f36ef76d-275fbe.png" alt="image-20220817115134064" style="zoom:67%;" />

1. 调用`Unsafe.unpark(Thread_0)`方法，设置 _counter 为 1
2. 当前线程调用`Unsafe.park()`方法
3. 检查 _counter ，本情况为 1，这时线程无需阻塞，继续运行
4. 设置 _counter 为 0

## 6.重新理解线程状态转换

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815145225633-73b1ffb388892e6722fe59673637ab18-9fb613.png" alt="image-20220815145225633" style="zoom:67%;" />

假设有线程`Thread t`

**情况 1：`NEW --> RUNNABLE`** 

当调用`t.start()`方法时，由`NEW --> RUNNABLE `

**情况 2：`RUNNABLE <--> WAITING`** 

t 线程用`synchronized(obj)`获取了对象锁后：

- 调用`obj.wait()`方法时，t 线程从`RUNNABLE --> WAITING`
- 调用`obj.notify()`，`obj.notifyAll()`，`t.interrupt()`时
  - 竞争锁成功，t 线程从`WAITING --> RUNNABLE`，进入 waitSet
  - 竞争锁失败，t 线程从`WAITING --> BLOCKED`，进入 entryList

```java
public class TestWaitNotify {
    final static Object obj = new Object();

    public static void main(String[] args) {
        new Thread(() -> {
            synchronized (obj) {
                log.debug("执行....");
                try {
                    obj.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.debug("其它代码...."); // 断点
            }
        },"t1").start();

        new Thread(() -> {
            synchronized (obj) {
                log.debug("执行....");
                try {
                    obj.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.debug("其它代码...."); // 断点
            }
        },"t2").start();

        sleep(0.5);
        log.debug("唤醒 obj 上其它线程");
        synchronized (obj) {
            obj.notifyAll(); // 唤醒obj上所有等待线程  断点
        }
    }
}
```

**情况 3：`RUNNABLE <--> WAITING`**

- 当前线程调用`t.join()`方法时，当前线程从`RUNNABLE --> WAITING`
  - 注意是当前线程在 t 线程对象的监视器上等待
- t 线程运行结束，或调用了当前线程的`interrupt()`时，当前线程从`WAITING --> RUNNABLE`

**情况 4：`RUNNABLE <--> WAITING`**

- 当前线程调用`LockSupport.park()`方法会让当前线程从`RUNNABLE --> WAITING`
- 调用`LockSupport.unpark(目标线程)`或调用了线程的`interrupt()`，会让目标线程从`WAITING --> RUNNABLE`

**情况 5：`RUNNABLE <--> TIMED_WAITING`**

t 线程用`synchronized(obj)`获取了对象锁后

- 调用`obj.wait(long n)`方法时，t 线程从`RUNNABLE --> TIMED_WAITING`
- t 线程等待时间超过了 n 毫秒，或调用`obj.notify()`，`obj.notifyAll()`，`t.interrupt()`时
  - 竞争锁成功，t 线程从`TIMED_WAITING --> RUNNABLE`
  - 竞争锁失败，t 线程从`TIMED_WAITING --> BLOCKED`

**情况 6：`RUNNABLE <--> TIMED_WAITING`**

- 当前线程调用`t.join(long n)`方法时，当前线程从`RUNNABLE --> TIMED_WAITING`
  - 注意是当前线程在 t 线程对象的监视器上等待
- 当前线程等待时间超过了 n 毫秒，或 t 线程运行结束，或调用了当前线程的`interrupt()`时，当前线程从`TIMED_WAITING --> RUNNABLE`

**情况 7：`RUNNABLE <--> TIMED_WAITING`**

- 当前线程调用`Thread.sleep(long n)`，当前线程从`RUNNABLE --> TIMED_WAITING`
- 当前线程等待时间超过了 n 毫秒，当前线程从`TIMED_WAITING --> RUNNABLE`

**情况 8：`RUNNABLE <--> TIMED_WAITING`**

- 当前线程调用`LockSupport.parkNanos(long nanos)`或`LockSupport.parkUntil(long millis)`时，当前线程从`RUNNABLE --> TIMED_WAITING`
- 调用`LockSupport.unpark(目标线程)`或调用了线程的`interrupt()`，或是等待超时，会让目标线程从`TIMED_WAITING--> RUNNABLE`

**情况 9：`RUNNABLE <--> BLOCKED`**

- t 线程用`synchronized(obj)`获取了对象锁时如果竞争失败，从`RUNNABLE --> BLOCKED`
- 持 obj 锁线程的同步代码块执行完毕，会唤醒该对象上所有 BLOCKED 的线程重新竞争，如果其中 t 线程竞争成功，从`BLOCKED --> RUNNABLE`，其它失败的线程仍然 BLOCKED 

**情况 10：`RUNNABLE <--> TERMINATED`**

当前线程所有代码运行完毕，进入 TERMINATED

## 7.死锁

- 互斥 - 使用线程私有变量
- 占有、等待 - 一次性申请所有的资源
- 不可抢夺 - 申请不到需要的资源则释放已持有的资源
- 相互等待 - 顺序申请资源

### 7.1 案例

一间大屋子有两个功能：睡觉、学习，互不相干。

现在小南要学习，小女要睡觉，但如果只用一间屋子（一个对象锁）的话，那么并发度很低，解决方法是准备多个房间（多个对象锁）。

例如

```java
class BigRoom {
 
    public void sleep() {
        synchronized (this) {
            log.debug("sleeping 2 小时");
            Sleeper.sleep(2);
        }
    }
 
    public void study() {
        synchronized (this) {
            log.debug("study 1 小时");
            Sleeper.sleep(1);
        }
    }
 
}
```

执行

```java
BigRoom bigRoom = new BigRoom();
new Thread(() -> {
    bigRoom.compute();
},"小南").start();
new Thread(() -> {
    bigRoom.sleep();
},"小女").start();
```

某次结果

```bash
12:13:54.471 [小南] c.BigRoom - study 1 小时 
12:13:55.476 [小女] c.BigRoom - sleeping 2 小时
```

改进

```java
class BigRoom {

    private final Object studyRoom = new Object();
    private final Object bedRoom = new Object();

    public void sleep() {
        synchronized (bedRoom) {
            log.debug("sleeping 2 小时");
            Sleeper.sleep(2);
        }
    }

    public void study() {
        synchronized (studyRoom) {
            log.debug("study 1 小时");
            Sleeper.sleep(1);
        }
    }

}
```

某次执行结果

```bash
12:15:35.069 [小南] c.BigRoom - study 1 小时 
12:15:35.069 [小女] c.BigRoom - sleeping 2 小时
```

将锁的粒度细分

- 好处，是可以增强并发度
- 坏处，如果一个线程需要同时获得多把锁，就容易发生死锁

### 7.2 死锁

有这样的情况：一个线程需要同时获取多把锁，这时就容易发生死锁。

`t1线程`获得`A对象`锁，接下来想获取`B对象`的锁；`t2线程`获得`B对象`锁，接下来想获取`A对象`的锁：

```java
Object A = new Object();
Object B = new Object();
Thread t1 = new Thread(() -> {
    synchronized (A) {
        log.debug("lock A");
        sleep(1);
        synchronized (B) {
            log.debug("lock B");
            log.debug("操作...");
        }
    }
}, "t1");

Thread t2 = new Thread(() -> {
    synchronized (B) {
        log.debug("lock B");
        sleep(0.5);
        synchronized (A) {
            log.debug("lock A");
            log.debug("操作...");
        }
    }
}, "t2");
t1.start();
t2.start();
```

结果

```bash
12:22:06.962 [t2] c.TestDeadLock - lock B 
12:22:06.962 [t1] c.TestDeadLock - lock A 
```

### 7.3 定位死锁

检测死锁可以使用`jconsole`工具，或者使用`jps`定位进程 id，再用`jstack`定位死锁：

```bash
cmd > jps
Picked up JAVA_TOOL_OPTIONS: -Dfile.encoding=UTF-8
12320 Jps
22816 KotlinCompileDaemon
33200 TestDeadLock              // JVM 进程
11508 Main
28468 Launcher
```

```bash
cmd > jstack 33200
Picked up JAVA_TOOL_OPTIONS: -Dfile.encoding=UTF-8
2018-12-29 05:51:40
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.91-b14 mixed mode):
 
"DestroyJavaVM" #13 prio=5 os_prio=0 tid=0x0000000003525000 nid=0x2f60 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
 
"Thread-1" #12 prio=5 os_prio=0 tid=0x000000001eb69000 nid=0xd40 waiting for monitor entry [0x000000001f54f000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at thread.TestDeadLock.lambda$main$1(TestDeadLock.java:28)
        - waiting to lock <0x000000076b5bf1c0> (a java.lang.Object)
        - locked <0x000000076b5bf1d0> (a java.lang.Object)
        at thread.TestDeadLock$$Lambda$2/883049899.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)
 
"Thread-0" #11 prio=5 os_prio=0 tid=0x000000001eb68800 nid=0x1b28 waiting for monitor entry [0x000000001f44f000]
   java.lang.Thread.State: BLOCKED (on object monitor)
        at thread.TestDeadLock.lambda$main$0(TestDeadLock.java:15)
        - waiting to lock <0x000000076b5bf1d0> (a java.lang.Object)
        - locked <0x000000076b5bf1c0> (a java.lang.Object)
        at thread.TestDeadLock$$Lambda$1/495053715.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)
        
// 略去部分输出
 
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x000000000361d378 (object 0x000000076b5bf1c0, a java.lang.Object),
  which is held by "Thread-0"
"Thread-0":
  waiting to lock monitor 0x000000000361e768 (object 0x000000076b5bf1d0, a java.lang.Object),
  which is held by "Thread-1"
 
Java stack information for the threads listed above:
===================================================
"Thread-1":
        at thread.TestDeadLock.lambda$main$1(TestDeadLock.java:28)
        - waiting to lock <0x000000076b5bf1c0> (a java.lang.Object)
        - locked <0x000000076b5bf1d0> (a java.lang.Object)
        at thread.TestDeadLock$$Lambda$2/883049899.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)
"Thread-0":
        at thread.TestDeadLock.lambda$main$0(TestDeadLock.java:15)
        - waiting to lock <0x000000076b5bf1d0> (a java.lang.Object)
        - locked <0x000000076b5bf1c0> (a java.lang.Object)
        at thread.TestDeadLock$$Lambda$1/495053715.run(Unknown Source)
        at java.lang.Thread.run(Thread.java:745)
 
Found 1 deadlock.
```

- 避免死锁要注意加锁顺序
- 另外如果由于某个线程进入了死循环，导致其它线程一直等待，对于这种情况 linux 下可以通过 top 先定位到 CPU 占用高的 Java 进程，再利用`top -Hp`进程 id 来定位是哪个线程，最后再用`jstack`排查

### 7.4 哲学家就餐问题

有五位哲学家，围坐在圆桌旁。

- 他们只做两件事，思考和吃饭，思考一会吃口饭，吃完饭后接着思考
- 吃饭时要用两根筷子吃，桌上共有 5 根筷子，每位哲学家左右手边各有一根筷子
- 如果筷子被身边的人拿着，自己就得等待

筷子类

```java
class Chopstick {
    String name;
 
    public Chopstick(String name) {
        this.name = name;
    }
 
    @Override
    public String toString() {
        return "筷子{" + name + '}';
    }
}
```

哲学家类

```java
class Philosopher extends Thread {
    Chopstick left;
    Chopstick right;
    public Philosopher(String name, Chopstick left, Chopstick right) {
        super(name);
        this.left = left;
        this.right = right;
    }

    private void eat() {
        log.debug("eating...");
        Sleeper.sleep(1);
    }

    @Override
    public void run() {
        while (true) {
            // 获得左手筷子
            synchronized (left) {
                // 获得右手筷子
                synchronized (right) {
                    // 吃饭
                    eat();
                }
                // 放下右手筷子
            }
            // 放下左手筷子
        }
    }
}
```

就餐

```java
Chopstick c1 = new Chopstick("1");
Chopstick c2 = new Chopstick("2");
Chopstick c3 = new Chopstick("3");
Chopstick c4 = new Chopstick("4");
Chopstick c5 = new Chopstick("5");
new Philosopher("苏格拉底", c1, c2).start();
new Philosopher("柏拉图", c2, c3).start();
new Philosopher("亚里士多德", c3, c4).start();
new Philosopher("赫拉克利特", c4, c5).start();
new Philosopher("阿基米德", c5, c1).start();
```

执行不多会，就执行不下去了

```bash
12:33:15.575 [苏格拉底] c.Philosopher - eating... 
12:33:15.575 [亚里士多德] c.Philosopher - eating... 
12:33:16.580 [阿基米德] c.Philosopher - eating... 
12:33:17.580 [阿基米德] c.Philosopher - eating... 
// 卡在这里, 不向下运行 
```

使用`jconsole`检测死锁，发现

```bash
-------------------------------------------------------------------------
名称: 阿基米德
状态: cn.itcast.Chopstick@1540e19d (筷子1) 上的BLOCKED, 拥有者: 苏格拉底
总阻止数: 2, 总等待数: 1
 
堆栈跟踪: 
cn.itcast.Philosopher.run(TestDinner.java:48)
   - 已锁定 cn.itcast.Chopstick@6d6f6e28 (筷子5)
-------------------------------------------------------------------------
名称: 苏格拉底
状态: cn.itcast.Chopstick@677327b6 (筷子2) 上的BLOCKED, 拥有者: 柏拉图
总阻止数: 2, 总等待数: 1
 
堆栈跟踪: 
cn.itcast.Philosopher.run(TestDinner.java:48)
   - 已锁定 cn.itcast.Chopstick@1540e19d (筷子1)
-------------------------------------------------------------------------
名称: 柏拉图
状态: cn.itcast.Chopstick@14ae5a5 (筷子3) 上的BLOCKED, 拥有者: 亚里士多德
总阻止数: 2, 总等待数: 0
 
堆栈跟踪: 
cn.itcast.Philosopher.run(TestDinner.java:48)
   - 已锁定 cn.itcast.Chopstick@677327b6 (筷子2)
-------------------------------------------------------------------------
名称: 亚里士多德
状态: cn.itcast.Chopstick@7f31245a (筷子4) 上的BLOCKED, 拥有者: 赫拉克利特
总阻止数: 1, 总等待数: 1
 
堆栈跟踪: 
cn.itcast.Philosopher.run(TestDinner.java:48)
   - 已锁定 cn.itcast.Chopstick@14ae5a5 (筷子3)
-------------------------------------------------------------------------
名称: 赫拉克利特
状态: cn.itcast.Chopstick@6d6f6e28 (筷子5) 上的BLOCKED, 拥有者: 阿基米德
总阻止数: 2, 总等待数: 0
 
堆栈跟踪: 
cn.itcast.Philosopher.run(TestDinner.java:48)
   - 已锁定 cn.itcast.Chopstick@7f31245a (筷子4)
```

这种线程没有按预期结束，执行不下去的情况，归类为【活跃性】问题，除了死锁以外，还有活锁和饥饿者两种情况。

### 7.5 活锁

活锁出现在两个线程互相改变对方的结束条件，最后谁也无法结束，例如

```java
public class TestLiveLock {
    static volatile int count = 10;
    static final Object lock = new Object();
 
    public static void main(String[] args) {
        new Thread(() -> {
            // 期望减到 0 退出循环
            while (count > 0) {
                sleep(0.2);
                count--;
                log.debug("count: {}", count);
            }
        }, "t1").start();
        new Thread(() -> {
            // 期望超过 20 退出循环
            while (count < 20) {
                sleep(0.2);
                count++;
                log.debug("count: {}", count);
            }
        }, "t2").start();
    }
}
```

可通过**增加睡眠时间**解决。

### 7.6 饥饿

线程饥饿的可能原因之一是不同线程或线程组之间的线程优先级不正确。另一个可能的原因可能是使用非终止循环（无限循环）或在特定资源上等待过多时间，同时保留了其他线程所需的关键锁。

通常建议尽量避免修改线程优先级（`Thread.setPriority（）`），因为这是导致线程饥饿的主要原因。一旦开始使用线程优先级调整应用程序，它就会与特定平台紧密耦合，并且还会带来线程匮乏的风险。

先来看看使用顺序加锁的方式解决之前的死锁问题：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815150609361-3d0b25dd3a4b33bfc22490afb4e57e55-0df29b.png" alt="image-20220815150609361" style="zoom:67%;" />

顺序加锁的解决方案

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220815150624182-7fcb88a151f2903f91c7c9c2c9a1a974-e0d079.png" alt="image-20220815150624182" style="zoom:67%;" />

但是这样会导致有的线程一直获取不到锁，造成饥饿现象。

> **注意**
>
> Windows 实现了一种线程回退机制，通过该机制，可以使长时间没有机会运行的线程获得临时的优先级提升，因此几乎无法实现完全的饥饿。

## 8.ReentrantLock

相对于`synchronized`它具备如下特点

- 可被其他线程中断
- 可以设置超时时间
- 可以设置为公平锁
- 支持多个条件变量

与`synchronized`一样，都支持可重入。

基本语法

```java
// 获取锁
reentrantLock.lock();
try {
    // 临界区
} finally {
    // 释放锁
    reentrantLock.unlock();
}
```

### 8.1 可重入

可重入是指同一个线程如果首次获得了这把锁，那么因为它是这把锁的拥有者，因此有权利再次获取这把锁，如果是不可重入锁，那么第二次获得锁时，自己也会被锁挡住。

```java
static ReentrantLock lock = new ReentrantLock();
 
public static void main(String[] args) {
    method1();
}
 
public static void method1() {
    lock.lock();
    try {
        log.debug("execute method1");
        method2();
    } finally {
        lock.unlock();
    }
}
 
public static void method2() {
    lock.lock();
    try {
        log.debug("execute method2");
        method3();
    } finally {
        lock.unlock();
    }
}
 
public static void method3() {
    lock.lock();
    try {
        log.debug("execute method3");
    } finally {
        lock.unlock();
    }
}
```

输出

```bash
17:59:11.862 [main] c.TestReentrant - execute method1 
17:59:11.865 [main] c.TestReentrant - execute method2 
17:59:11.865 [main] c.TestReentrant - execute method3 
```

### 8.2 可打断

**示例**

```java
ReentrantLock lock = new ReentrantLock();
 
Thread t1 = new Thread(() -> {
    log.debug("启动...");
    try {
        // 如果没有竞争那么此方法就会获取 lock 对象锁
        // 如果有竞争就进入阻塞队列，但可以被其他线程用 interrupt 打断
        lock.lockInterruptibly();
    } catch (InterruptedException e) {
        e.printStackTrace();
        log.debug("等锁的过程中被打断");
        return;
    }
    try {
        log.debug("获得了锁");
    } finally {
        lock.unlock();
    }
}, "t1");
 
 
lock.lock();
log.debug("获得了锁");
t1.start();
try {
    sleep(1);
    t1.interrupt();
    log.debug("执行打断");
} finally {
    lock.unlock();
}
```

输出

```bash
18:02:40.520 [main] c.TestInterrupt - 获得了锁 
18:02:40.524 [t1] c.TestInterrupt - 启动... 
18:02:41.530 [main] c.TestInterrupt - 执行打断 
java.lang.InterruptedException 
 at 
java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchr
onizer.java:898) 
 at 
java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchron
izer.java:1222) 
 at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335) 
 at cn.itcast.n4.reentrant.TestInterrupt.lambda$main$0(TestInterrupt.java:17) 
 at java.lang.Thread.run(Thread.java:748) 
18:02:41.532 [t1] c.TestInterrupt - 等锁的过程中被打断 
```

注意如果是不可中断模式，那么即使使用了`interrupt`也不会让等待中断

```java
ReentrantLock lock = new ReentrantLock();

Thread t1 = new Thread(() -> {
    log.debug("启动...");
    lock.lock();
    try {
        log.debug("获得了锁");
    } finally {
        lock.unlock();
    }
}, "t1");


lock.lock();
log.debug("获得了锁");
t1.start();
try {
    sleep(1);
    t1.interrupt();
    log.debug("执行打断");
    sleep(1);
} finally {
    log.debug("释放了锁");
    lock.unlock();
}
```

输出

```bash
18:06:56.261 [main] c.TestInterrupt - 获得了锁 
18:06:56.265 [t1] c.TestInterrupt - 启动... 
18:06:57.266 [main] c.TestInterrupt - 执行打断 // 这时 t1 并没有被真正打断, 而是仍继续等待锁 
18:06:58.267 [main] c.TestInterrupt - 释放了锁 
18:06:58.267 [t1] c.TestInterrupt - 获得了锁 
```

### 8.3 锁超时

立刻失败

```java
ReentrantLock lock = new ReentrantLock();
Thread t1 = new Thread(() -> {
    log.debug("启动...");
    if (!lock.tryLock()) {
        log.debug("获取立刻失败，返回");
        return;
    }
    try {
        log.debug("获得了锁");
    } finally {
        lock.unlock();
    }
}, "t1");

lock.lock();
log.debug("获得了锁");
t1.start();
try {
    sleep(2);
} finally {
    lock.unlock();
}
```

输出

```bash
18:15:02.918 [main] c.TestTimeout - 获得了锁 
18:15:02.921 [t1] c.TestTimeout - 启动... 
18:15:02.921 [t1] c.TestTimeout - 获取立刻失败，返回
```

超时失败

```java
ReentrantLock lock = new ReentrantLock();
Thread t1 = new Thread(() -> {
    log.debug("启动...");
    try {
        if (!lock.tryLock(1, TimeUnit.SECONDS)) {
            log.debug("获取等待 1s 后失败，返回");
            return;
        }
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    try {
        log.debug("获得了锁");
    } finally {
        lock.unlock();
    }
}, "t1");
 
lock.lock();
log.debug("获得了锁");
t1.start();
try {
    sleep(2);
} finally {
    lock.unlock();
}
```

输出

```bash
18:19:40.537 [main] c.TestTimeout - 获得了锁 
18:19:40.544 [t1] c.TestTimeout - 启动... 
18:19:41.547 [t1] c.TestTimeout - 获取等待 1s 后失败，返回
```

### 扩展：使用 tryLock 解决哲学家就餐问题

```java
class Chopstick extends ReentrantLock {
    String name;
 
    public Chopstick(String name) {
        this.name = name;
    }
 
    @Override
    public String toString() {
        return "筷子{" + name + '}';
    }
}
```

```java
class Philosopher extends Thread {
    Chopstick left;
    Chopstick right;

    public Philosopher(String name, Chopstick left, Chopstick right) {
        super(name);
        this.left = left;
        this.right = right;
    }

    @Override
    public void run() {
        while (true) {
            // 尝试获得左手筷子
            if (left.tryLock()) {
                try {
                    // 尝试获得右手筷子
                    if (right.tryLock()) {
                        try {
                            eat();
                        } finally {
                            right.unlock();
                        }
                    }
                } finally {
                    left.unlock();
                }
            }
        }
    }

    private void eat() {
        log.debug("eating...");
        Sleeper.sleep(1);
    }
}
```

### 8.4 公平锁

- 公平锁：竞争锁资源的线程严格按照请求的顺序来分配锁。

- 非公平锁：竞争锁资源的线程可以插队来竞争锁资源 （ReentrantLock 默认非公平）

```java
ReentrantLock lock = new ReentrantLock(false);
 
lock.lock();
for (int i = 0; i < 500; i++) {
    new Thread(() -> {
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " running...");
        } finally {
            lock.unlock();
        }
    }, "t" + i).start();
}
 
// 1s 之后去争抢锁
Thread.sleep(1000);
new Thread(() -> {
    System.out.println(Thread.currentThread().getName() + " start...");
    lock.lock();
    try {
        System.out.println(Thread.currentThread().getName() + " running...");
    } finally {
        lock.unlock();
    }
}, "强行插入").start();
lock.unlock();
```

强行插入，有机会在中间输出。

```bash
t39 running... 
t40 running... 
t41 running... 
t42 running... 
t43 running... 
强行插入 start... 
强行插入 running... 
t44 running... 
t45 running... 
t46 running... 
t47 running... 
t49 running... 
```

改为公平锁后

```java
ReentrantLock lock = new ReentrantLock(true);
```

强行插入，总是在最后输出

```bash
t465 running... 
t464 running... 
t477 running... 
t442 running... 
t468 running... 
t493 running... 
t482 running... 
t485 running... 
t481 running... 
强行插入 running... 
```

公平锁一般没有必要，会**降低并发度**，后面分析原理时会讲解。

**原理**

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

非公平锁的写操作不管三七二十一，我就是要参与竞争去，会先去尝试竞争锁资源，如果抢占不到，就会再加入到 AQS 同步队列中去等待；非公平锁的读操作需要判断等待队列里的头结点是不是写操作（通过是否共享来判断锁类型，读锁是共享的，写锁是独占的）。

### 8.5 条件变量

`synchronized`中也有条件变量，就是我们讲原理时那个 waitSet 休息室，当条件不满足时进入 waitSet 等待

`ReentrantLock`的条件变量比`synchronized`强大之处在于，它是支持多个条件变量的，这就好比

- `synchronized`是那些不满足条件的线程都在一间休息室等消息
- `ReentrantLock`支持多间休息室，有专门等烟的休息室、专门等早餐的休息室、唤醒时也是按休息室来唤醒

使用要点：

- await 前需要获得锁
- await 执行后，会**释放锁**，进入 conditionObject 等待
- await 的线程被唤醒（或打断、或超时）取重新竞争 lock 锁
- 竞争 lock 锁成功后，从 await 后继续执行

例子

```java
static ReentrantLock lock = new ReentrantLock();
// 创建条件变量（休息室）
static Condition waitCigaretteQueue = lock.newCondition();
static Condition waitbreakfastQueue = lock.newCondition();
static volatile boolean hasCigrette = false;
static volatile boolean hasBreakfast = false;

public static void main(String[] args) {
    new Thread(() -> {
        try {
            lock.lock();
            while (!hasCigrette) {
                try {
                    waitCigaretteQueue.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            log.debug("等到了它的烟");
        } finally {
            lock.unlock();
        }
    }).start();

    new Thread(() -> {
        try {
            lock.lock();
            while (!hasBreakfast) {
                try {
                    waitbreakfastQueue.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            log.debug("等到了它的早餐");
        } finally {
            lock.unlock();
        }
    }).start();

    sleep(1);
    sendBreakfast();
    sleep(1);
    sendCigarette();
}
private static void sendCigarette() {
    lock.lock();
    try {
        log.debug("送烟来了");
        hasCigrette = true;
        // 随机唤醒一个线程
        waitCigaretteQueue.signal();
    } finally {
        lock.unlock();
    }
}

private static void sendBreakfast() {
    lock.lock();
    try {
        log.debug("送早餐来了");
        hasBreakfast = true;
        waitbreakfastQueue.signal();
    } finally {

        lock.unlock();
    }
}
```

输出

```bash
18:52:27.680 [main] c.TestCondition - 送早餐来了 
18:52:27.682 [Thread-1] c.TestCondition - 等到了它的早餐 
18:52:28.683 [main] c.TestCondition - 送烟来了 
18:52:28.683 [Thread-0] c.TestCondition - 等到了它的烟
```

### 8.6 与 Lock 区别

- Synchronized 内置的 Java 关键字， Lock 是一个 Java 类
- Synchronized 无法判断获取锁的状态，Lock 可以判断是否获取到了锁
- Synchronized 会自动释放锁，lock 必须要手动释放锁
- Synchronized **可重入锁，不可以中断的，非公平**；Lock ，**可重入锁，可打断，可设置公平**
- Synchronized 适合锁少量的代码同步问题，Lock 适合锁大量的同步代码

## 扩展：同步模式之顺序控制

### 1.固定运行顺序

比如，必须先 2 后 1 打印。

#### 1.1 wait notify 版

```java
// 用来同步的对象
static Object obj = new Object();
// t2 运行标记， 代表 t2 是否执行过
static boolean t2runed = false;
 
public static void main(String[] args) {
 
    Thread t1 = new Thread(() -> {
        synchronized (obj) {
            // 如果 t2 没有执行过
            while (!t2runed) { 
                try {
                    // t1 先等一会
                    obj.wait(); 
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    }
            }
        }
        System.out.println(1);
    });
 
    Thread t2 = new Thread(() -> {
        System.out.println(2);
        synchronized (obj) {
            // 修改运行标记
            t2runed = true;
            // 通知 obj 上等待的线程（可能有多个，因此需要用 notifyAll）
            obj.notifyAll();
        }
    });
 
    t1.start();
    t2.start();
}
```

可以看到，实现上很麻烦：

- 首先，需要保证先 wait 再 notify，否则 wait 线程永远得不到唤醒。因此使用了『运行标记』来判断该不该 wait
- 第二，如果有些干扰线程错误地 notify 了 wait 线程，条件不满足时还要重新等待，使用了 while 循环来解决此问题
- 最后，唤醒对象上的 wait 线程需要使用 notifyAll，因为『同步对象』上的等待线程可能不止一个

#### 1.2 Park Unpark 版

可以使用 LockSupport 类的 park 和 unpark 来简化上面的题目：

```java
Thread t1 = new Thread(() -> {
    // 当没有『许可』时，当前线程暂停运行；有『许可』时，用掉这个『许可』，当前线程恢复运行
    LockSupport.park();
    System.out.println("1");
});

Thread t2 = new Thread(() -> {
    System.out.println("2");
    // 给线程 t1 发放『许可』（多次连续调用 unpark 只会发放一个『许可』）
    LockSupport.unpark(t1);
});

t1.start();
t2.start();
```

`park`和`unpark`方法比较灵活，他俩谁先调用，谁后调用无所谓。并且是以线程为单位进行『暂停』和『恢复』，不需要『同步对象』和『运行标记』。

### 2.交替输出

线程 1 输出 a 5 次，线程 2 输出 b 5 次，线程 3 输出 c 5 次。现在要求输出 abcabcabcabcabc 怎么实现。

#### 2.1 wait notify 版

```java
class SyncWaitNotify {
    private int flag;
    private int loopNumber;
 
    public SyncWaitNotify(int flag, int loopNumber) {
        this.flag = flag;
        this.loopNumber = loopNumber;
    }
 
    public void print(int waitFlag, int nextFlag, String str) {
        for (int i = 0; i < loopNumber; i++) {
            synchronized (this) {
                while (this.flag != waitFlag) {
                    try {
                        this.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.print(str);
                flag = nextFlag;
                this.notifyAll();
            }
        }
    }
}
```

```java
SyncWaitNotify syncWaitNotify = new SyncWaitNotify(1, 5);
new Thread(() -> {
    syncWaitNotify.print(1, 2, "a");
}).start();
new Thread(() -> {
    syncWaitNotify.print(2, 3, "b");
}).start();
new Thread(() -> {
    syncWaitNotify.print(3, 1, "c");
}).start();
```

#### 2.2 Lock 条件变量版

```java
class AwaitSignal extends ReentrantLock {
    public void start(Condition first) {
        this.lock();
        try {
            log.debug("start");
            first.signal();
        } finally {
            this.unlock();
        }
    }
    public void print(String str, Condition current, Condition next) {
        for (int i = 0; i < loopNumber; i++) {
            this.lock();
            try {
                current.await();
                log.debug(str);
                next.signal();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                this.unlock();
            }
        }
    }

    // 循环次数
    private int loopNumber;

    public AwaitSignal(int loopNumber) {
        this.loopNumber = loopNumber;
    }
}
```

```java
AwaitSignal as = new AwaitSignal(5);
Condition aWaitSet = as.newCondition();
Condition bWaitSet = as.newCondition();
Condition cWaitSet = as.newCondition();
 
 
new Thread(() -> {
    as.print("a", aWaitSet, bWaitSet);
}).start();
new Thread(() -> {
    as.print("b", bWaitSet, cWaitSet);
}).start();
new Thread(() -> {
    as.print("c", cWaitSet, aWaitSet);
}).start();
 
as.start(aWaitSet);
```

#### 2.3 Park Unpark 版

```java
 class SyncPark {

    private int loopNumber;
    private Thread[] threads;
 
    public SyncPark(int loopNumber) {
        this.loopNumber = loopNumber;
    }
 
    public void setThreads(Thread... threads) {
        this.threads = threads;
    }
 
    public void print(String str) {
        for (int i = 0; i < loopNumber; i++) {
            LockSupport.park();
            System.out.print(str);
            LockSupport.unpark(nextThread());
        }
    }
 
    private Thread nextThread() {
        Thread current = Thread.currentThread();
        int index = 0;
        for (int i = 0; i < threads.length; i++) {
            if(threads[i] == current) {
                index = i;
                break;
            }
        }
        if(index < threads.length - 1) {
            return threads[index+1];
        } else {
            return threads[0];
        }
    }
 
    public void start() {
        for (Thread thread : threads) {
            thread.start();
        }
        LockSupport.unpark(threads[0]);
    }
}
```

```java
SyncPark syncPark = new SyncPark(5);
Thread t1 = new Thread(() -> {
    syncPark.print("a");
});
Thread t2 = new Thread(() -> {
    syncPark.print("b");
});
Thread t3 = new Thread(() -> {
    syncPark.print("c\n");
});
syncPark.setThreads(t1, t2, t3);
syncPark.start();
```
