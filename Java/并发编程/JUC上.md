## 一、JUC 简介

在 Java 5.0 提供了 java.util.concurrent （简称JUC ）包，在此包中增加了在并发编程中很常用的实用工具类，用于定义类似于线程的自定义子系统，包括线程池、异步 IO 和轻量级任务框架。提供可调的、灵活的线程池。还提供了设计用于多线程上下文中的 Collection 实现等。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125045713-d3d0745818a42ef8f8d18b3c0d49f8fa-98a951.png" alt="image-20220222125045713" style="zoom:50%;" />

---

## 二、进程与线程

**Java 的线程**

- Java 默认有两个线程：main、GC

- 对于Java而言提供了：`Thread、Runnable、Callable`操作线程，底层操作的是C++ ，Java 无法直接操作硬件
  - **Thread**
  - **Runnable** 没有返回值、效率相比入 Callable 相对较低！
  - **Callable** 有返回值！

- 线程有 6 个状态

  ```java
  public enum State {
      /**
       * 线程新生状态
       */
      NEW,
      /**
       * 线程运行中
       */
      RUNNABLE,
      /**
       * 线程阻塞状态
       */
      BLOCKED,
      /**
       * 线程等待状态，死等
       */
      WAITING,
      /**
       * 线程超时等待状态，超过一定时间就不再等
       */
      TIMED_WAITING,
      /**
       * 线程终止状态，代表线程执行完毕
       */
      TERMINATED;
  }
  ```

- wait/sleep 区别

  1. 二者来自不同的类

     - wait => Object

     - sleep => Thread

  2. 关于锁的释放

     - wait 会释放锁

     - sleep 睡觉了，抱着锁睡觉，不会释放！

  3. 使用的范围是不同的

     - wait 必须在同步代码块中使用

     - sleep 可以在任何地方睡眠

### 3.并发

- 多线程操作同一个资源
- 一核CPU，模拟出来多条线程，快速交替
- 并发编程的本质：**充分利用CPU的资源**

### 4.并行

- 多个人一起行走
- 多核 CPU ，多个线程可以同时执行，eg：线程池！

```java
public class Test1 {
    public static void main(String[] args) {
      // 获取cpu的核数 
     // CPU 密集型，IO密集型 
        System.out.println(Runtime.getRuntime().availableProcessors());
     // 如果电脑是8核，则结果输出8
     } 
}
```

---

## 三、Synchronized锁

```java
// @Description: 卖票例子
public class SaleTicketTDemo01 {
    /*
     * 真正的多线程开发，公司中的开发，降低耦合性
     * 线程就是一个单独的资源类，没有任何附属的操作！
     * 包含：属性、方法
     */
    public static void main(String[] args) {
        //并发：多个线程同时操作一个资源类，把资源类丢入线程
        Ticket ticket = new Ticket();
        new Thread(() -> {
            for (int i = 1; i < 50; i++) {
                ticket.sale();
            }
        }, "A").start();
        new Thread(() -> {
            for (int i = 1; i < 50; i++) {
                ticket.sale();
            }
        }, "B").start();
        new Thread(() -> {
            for (int i = 1; i < 50; i++) {
                ticket.sale();
            }
        }, "C").start();
    }
}

//资源类 OOP
class Ticket {
    //属性、方法
    private int number = 50;
    // 卖票的方式
    // synchronized 本质: 队列，锁
    public synchronized void sale() {
        if (number > 0) {
            System.out.println(Thread.currentThread().getName() + "卖出了" +
                    (50-(--number)) + "张票，剩余:" + number + "张票");
        }
    }
}
```

---

## 四、Lock锁（重点）

**Lock 接口**

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220124235715303-7ea0da31fec1fcc128ee373a0de451fc-a90dd5.png" alt="image-20220124235715303" style="zoom: 80%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125107097-d561d16e8d9a15b6663b632c3cf39831-0ac078.png" alt="image-20220222125107097" style="zoom:80%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125116202-4bbcdcb669c26e224a4370e12e114a23-4226b2.png" alt="image-20220222125116202" style="zoom:80%;" />

- 公平锁：十分公平，线程执行顺序按照先来后到顺序
- 非公平锁：十分不公平：可以插队 （默认锁）

将上面的卖票例子用 lock 锁替换 synchronized：

```java
public class SaleTicketTDemo02 {
    public static void main(String[] args) {
        //并发：多个线程同时操作一个资源类，把资源类丢入线程
        Ticket2 ticket = new Ticket2();
        new Thread(() -> {
            for (int i = 1; i < 50; i++) {
                ticket.sale();
            }
        }, "A").start();
        new Thread(() -> {
            for (int i = 1; i < 50; i++) {
                ticket.sale();
            }
        }, "B").start();
        new Thread(() -> {
            for (int i = 1; i < 50; i++) {
                ticket.sale();
            }
        }, "C").start();
    }
}
// Lock 3步骤
// 1. new ReentrantLock();
// 2. lock.lock()  加锁
// 3. lock.unlock() 解锁
class Ticket2 {
    //属性、方法
    private int number = 50;
    Lock lock = new ReentrantLock();
    // 卖票方式
    public void sale() {
        lock.lock();// 加锁
        try {
            // 业务代码
            if (number > 0) {
                System.out.println(Thread.currentThread().getName() + "卖出了" +
                        (50 - (--number)) + "张票，剩余:" + number + "张票");
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();// 解锁
        }
    }
}
```

Synchronized 和 Lock 区别：

- Synchronized 内置的 Java 关键字， Lock 是一个 Java 类
- Synchronized 无法判断获取锁的状态，Lock 可以判断是否获取到了锁
- Synchronized 会自动释放锁，lock 必须要手动释放锁！如果不释放锁，**死锁**
- Synchronized 线程 1（获得锁，如果线程1阻塞）、线程2（等待，傻傻的等）；Lock锁就不一定会等待下去
- Synchronized **可重入锁，不可以中断的，非公平**；Lock ，**可重入锁，可以判断锁，非公平**（可以自己设置）
  - 可重入：对象获取锁后还可以继续获取该锁
- Synchronized 适合锁少量的代码同步问题，Lock 适合锁大量的同步代码！

---

## 五、生产者和消费者问题

### 1.Synchronized版

```java
/**
 * 线程之间的通信问题：生产者和消费者问题！等待唤醒，通知唤醒
 * 线程交替执行 A、B 操作同一个变量 num = 0
 * A num+1
 * B num-1
 */
public class Demo {
    public static void main(String[] args) {
        Data data = new Data();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.increment();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "A").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.decrement();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "B").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.increment();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "C").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.decrement();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "D").start();
    }
}
// 判断等待，业务，通知
class Data {
     // 数字 资源类
    private int number = 0;
    //+1
    public synchronized void increment() throws InterruptedException {
        /*
        假设 number此时等于1，即已经被生产了产品
        如果这里用的是if判断，如果此时A,C两个生产者线程争夺increment()方法执行权
        假设A拿到执行权，经过判断number!=0成立，则A.wait()开始等待（wait()会释放锁），然后C试图去执行
        生产方法，但依然判断number!=0成立，则B.wait()开始等待（wait()会释放锁）
        碰巧这时候消费者线程线程B/D去消费了一个产品，使number=0然后，B/D消费完后调用this.notifyAll();
        这时候2个等待中的生产者线程继续生产产品，而此时number++ 执行了2次
        同理，重复上述过程，生产者线程继续wait()等待，消费者调用this.notifyAll();
        然后生产者继续超前生产，最终导致‘产能过剩’，即number大于1
        而 while 时，当线程被唤醒仍然会执行依次判断，若不符合条件则会继续 wait
        if(number != 0){
            // 等待
            this.wait();
        }*/
        while (number != 0) {
     // 注意这里不可以用if 否则会出现虚假唤醒问题，解决方法将 if 换成 while
            // 等待
            this.wait();
        }
        number++;
        System.out.println(Thread.currentThread().getName() + "=>" + number);
        // 通知其他线程，我+1完毕了
        this.notifyAll();
    }
    //-1
    public synchronized void decrement() throws InterruptedException {
        while (number == 0) {
            // 等待
            this.wait();
        }
        number--;
        System.out.println(Thread.currentThread().getName() + "=>" + number);
        // 通知其他线程，我-1完毕了
        this.notifyAll();
    }
}
```

#### 虚假唤醒

> 问题存在，A B C D 4 个线程时会出现虚假唤醒

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125131919-ba2212abcb7efe4d1cea431774940e82-b83275.png" alt="image-20220222125131919" style="zoom:80%;" />

因此上述代码中必须使用 **while** 判断，而不能使用 **if**

### 2.JUC

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125140474-879847332f047b775ef2c8649b6d3b22-d9dfda.png" alt="image-20220222125140474" style="zoom:80%;" />

代码实现：

```java
public class Demo {
    public static void main(String[] args) {
        Data2 data = new Data2();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.increment();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "A").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.decrement();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "B").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.increment();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "C").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                try {
                    data.decrement();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, "D").start();
    }
}
// 判断等待，业务，通知
class Data2 {
     // 数字 资源类
    private int number = 0;
    Lock lock = new ReentrantLock();
    Condition condition = lock.newCondition();
    //condition.await(); // 等待 
    //condition.signalAll(); // 唤醒全部
    //+1
    public  void increment() throws InterruptedException {
        lock.lock();
        try {
            // 业务代码
            while (number != 0) {
                // 等待
                condition.await();
            }
            number++;
            System.out.println(Thread.currentThread().getName() + "=>" + number);
            // 通知其他线程，我+1完毕了
            condition.signal();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
    //-1
    public  void decrement() throws InterruptedException {
        lock.lock();
        try {
            while (number == 0) {
                // 等待
                condition.await();
            }
            number--;
            System.out.println(Thread.currentThread().getName() + "=>" + number);
            // 通知其他线程，我-1完毕了
            condition.signal();
        }catch (Exception e){
            e.printStackTrace();
        }finally {
            lock.unlock();
        }
    }
}
```

#### **精准通知和唤醒线程**

问题：ABCD线程抢占执行的顺序是随机的，如果想让ABCD线程有序执行，该如何改进代码？

```java
/*
 * A 执行完调用B，B执行完调用C，C执行完调用A
 */
public class Demo {
    public static void main(String[] args) {
        Data3 data = new Data3();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                data.printA();
            }
        }, "A").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                data.printB();
            }
        }, "B").start();
        new Thread(() -> {
            for (int i = 0; i < 10; i++) {
                data.printC();
            }
        }, "C").start();
    }
}
class Data3 {
     // 资源类 Lock
    private Lock lock = new ReentrantLock();
    private Condition condition1 = lock.newCondition();
    private Condition condition2 = lock.newCondition();
    private Condition condition3 = lock.newCondition();
    private int number = 1; 
    // number=1 A执行  number=2 B执行 number=3 C执行
    public void printA() {
        lock.lock();
        try {
            // 业务，判断-> 执行-> 通知
            while (number != 1) {
                // A等待
                condition1.await();
            }
            System.out.println(Thread.currentThread().getName() + "=>AAAAAAA");
            // 唤醒指定的人，B
            number = 2;
            condition2.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
    public void printB() {
        lock.lock();
        try {
            // 业务，判断-> 执行-> 通知
            while (number != 2) {
                // B等待
                condition2.await();
            }
            System.out.println(Thread.currentThread().getName() + "=>BBBBBBBBB");
            // 唤醒，唤醒指定的人，c
            number = 3;
            condition3.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
    public void printC() {
        lock.lock();
        try {
            // 业务，判断-> 执行-> 通知
            // 业务，判断-> 执行-> 通知
            while (number != 3) {
                // C等待
                condition3.await();
            }
            System.out.println(Thread.currentThread().getName() + "=>CCCCC ");
            // 唤醒，唤醒指定的人，A
            number = 1;
            condition1.signal();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}
```

测试结果：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125151093-0debcde515cff64e88eb4779f416d773-918f63.png" alt="image-20220222125151093" style="zoom:80%;" />

---

## 六、synchronized的锁的对象

> synchronized的锁是方法的调用者

```java
/**
 * 8锁，就是关于锁的8个问题
 * 1、标准情况下，两个线程先打印发短信还是先打印打电话？
 * 1、sendSms延迟4秒，两个线程先打印发短信还是打电话？
 */.
public class Test1 {
    public static void main(String[] args) {
        Phone phone = new Phone();
        // 锁的存在
        new Thread(()->{
            phone.sendSms();
        },"A").start();
        // 捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(()->{
            phone.call();
        },"B").start();
    }
}
class Phone{
    // synchronized 锁的对象是方法的调用者！
    // 两个方法用的是同一个对象调用(同一个锁)，谁先拿到锁谁执行！
    public synchronized void sendSms(){
        try {
            TimeUnit.SECONDS.sleep(4);// 抱着锁睡眠
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    }
    public synchronized void call(){
        System.out.println("打电话");
    }
}
// 先执行发短信，后执行打电话
```

**普通方法没有锁！不是同步方法，就不受锁的影响，正常执行**

```java
/**
 * 3、增加了一个普通方法
 * 4、两个对象，两个同步方法
 */
public class Test2  {
    public static void main(String[] args) {
        // 两个对象，两个调用者，两把锁！
        Phone2 phone1 = new Phone2();
        Phone2 phone2 = new Phone2();
        //锁的存在
        new Thread(()->{
            phone1.sendSms();
        },"A").start();
        // 捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(()->{
            phone2.call();
        },"B").start();
        new Thread(()->{
            phone2.hello();
        },"C").start();
    }
}
class Phone2{
    // synchronized 锁的对象是方法的调用者！
    public synchronized void sendSms(){
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    }
    public synchronized void call(){
        System.out.println("打电话");
    }
    // 这里没有锁！不是同步方法，不受锁的影响
    public void hello(){
        System.out.println("hello");
    }
}
// 先执行打电话，接着执行hello，最后执行发短信
```

**不同实例对象的 Class 类模板只有一个，static 静态的同步方法，锁的是 Class **

```java
/**
 * 5、两个静态的同步方法，只有一个对象
 * 6、两个对象！两个静态的同步方法
 */
public class Test3  {
    public static void main(String[] args) {
        // 两个对象的 Class 类模板只有一个，static 方法锁的是 Class
        Phone3 phone1 = new Phone3();
        Phone3 phone2 = new Phone3();
        //锁的存在
        new Thread(()->{
            phone1.sendSms();
        },"A").start();
        // 捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(()->{
            phone2.call();
        },"B").start();
    }
}
// Phone3唯一的一个 Class 对象
class Phone3{
    // synchronized 锁的对象是方法的调用者！
    // static 静态方法
    // 类一加载就有了！锁的是Class
    public static synchronized void sendSms(){
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    } 
    public static synchronized void call(){
        System.out.println("打电话");
    }
}
// 先执行发短信，后执行打电话
```

```java
/**
 * 7、1个静态的同步方法，1个普通的同步方法 ，一个对象
 * 8、1个静态的同步方法，1个普通的同步方法 ，两个对象
 */
public class Test4  {
    public static void main(String[] args) {
        // 两个对象的Class类模板只有一个，static，锁的是Class
        Phone4 phone1 = new Phone4();
        Phone4 phone2 = new Phone4();
        //锁的存在
        new Thread(()->{
            phone1.sendSms();
        },"A").start();
        // 捕获
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(()->{
            phone2.call();
        },"B").start();
    }
}
// Phone3唯一的一个 Class 对象
class Phone4{
    // 静态的同步方法 锁的是 Class 类模板
    public static synchronized void sendSms(){
        try {
            TimeUnit.SECONDS.sleep(4);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("发短信");
    }
    // 普通的同步方法，锁的是调用者（对象），二者锁的对象不同，所以不需要等待
    public synchronized void call(){
        System.out.println("打电话");
    }
}
/**
* 7/8 两种情况下，都是先执行打电话，后执行发短信，因为二者锁的对象不同，
* 静态同步方法锁的是 Class 类模板，普通同步方法锁的是实例化的对象，
* 所以不用等待前者解锁后后者才能执行，而是两者并发执行
* 因为发短信休眠4s，所以打电话先执行
**/
```

### 小结

- 普通 synchronized 方法的锁是调用的对象
- static synchronized 方法的锁是 Class 本身

---

## 七、集合类不安全

### 1、List不安全

> List、ArrayList 等在并发多线程条件下，不能实现数据共享，多个线程同时调用一个 list 对象时候就会出现并发修改异常 ConcurrentModificationException

```java
// java.util.ConcurrentModificationException 并发修改异常！
public class ListTest {
    public static void main(String[] args) {
        /*
        * 并发下 ArrayList 不安全的吗，Synchronized；
        * 解决方案；
        * 1、List<String> list = new Vector<>();
        * 2、List<String> list = Collections.synchronizedList(new ArrayList<>());
        * 3、List<String> list = new CopyOnWriteArrayList<>()；
        *
        * CopyOnWrite 写入时复制，简称 COW，计算机程序设计领域的一种优化策略
        * 多个线程读取的时候是固定的，但写入时可能存在覆盖
        * 在写入的时候先复制一份，写完后再放回去，避免覆盖
        * 读写分离
        */    
        List<String> list = new CopyOnWriteArrayList<>();
        for (int i = 1; i <= 10; i++) {
            new Thread(()->{
                list.add(UUID.randomUUID().toString().substring(0,5));
                System.out.println(list);
            },String.valueOf(i)).start();
        }
    }
}
```

> - Vector 使用了 Synchronized 锁，效率较低
>
> [**CopyOnWriteArrayList**](./CopyOnWriteArrayList.md)
>
> - CopyOnWriteArrayList 使用ReentrantLock重入锁加锁，保证线程安全
>- CopyOnWriteArrayList 的写操作都要先拷贝一份新数组，在新数组中做修改，修改完了再用新数组替换老数组，所以空间复杂度是O(n)，性能比较低下（ArrayList中采用的数组动态扩容）
> - CopyOnWriteArrayList 的读操作支持随机访问，时间复杂度为O(1)
>- CopyOnWriteArrayList 采用读写分离的思想，**读操作不加锁，写操作加锁**，且写操作占用较大内存空间，所以适用于**读多写少**的场合
> - CopyOnWriteArrayList **只保证最终一致性，不保证实时一致性**，从 add 添加相关方法中可以看出，在修改数组结构时先获取一个快照，然后在真正修改数组结构前对比快照和当前数组结构是否发生变化

### 2、Set不安全

> Set、Hash 等在并发多线程条件下，不能实现数据共享，多个线程同时调用一个set对象时候就会出现并发修改异常ConcurrentModificationException

```java
/**
 * ConcurrentModificationException 并发修改异常
 * 1、Set<String> set = Collections.synchronizedSet(new HashSet<>());
 * 2、Set<String> set = new CopyOnWriteArraySet<>()；
 */
public class SetTest {
    public static void main(String[] args) {
        //Set<String> set = new HashSet<>();//不安全
        // Set<String> set = Collections.synchronizedSet(new HashSet<>());//安全
        Set<String> set = new CopyOnWriteArraySet<>();//安全
        for (int i = 1; i <=30 ; i++) {
           new Thread(()->{
               set.add(UUID.randomUUID().toString().substring(0,5));
               System.out.println(set);
           },String.valueOf(i)).start();
        }
    }
}
```

**扩展：hashSet 底层是什么？**

```java
public HashSet() {
    map = new HashMap<>();
}

// add set 本质就是 map， key 是无法重复的
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}

// Dummy value to associate with an Object in the backing Map
private static final Object PRESENT = new Object();
```

> 可以看出 HashSet 的底层就是一个 HashMap

### 3、Map不安全

[**HashMap 的初始容量和加载因子**](../HashMap.md)

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/**
 * The maximum capacity, used if a higher value is implicitly specified
 * by either of the constructors with arguments.
 * MUST be a power of two <= 1<<30.
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**  
 * The load factor used when none specified in constructor.
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```

```java
// ConcurrentModificationException
public class  MapTest {
    public static void main(String[] args) {
        // map 是这样用的吗？ 不是，工作中不用 HashMap
        // 默认等价于什么？  new HashMap<>(16,0.75); 初始容量、加载因子
        // Map<String, String> map = new ConcurrentHashMap<>();
        // 扩展：研究 ConcurrentHashMap 的原理
        Map<String, String> map = new ConcurrentHashMap<>();
        for (int i = 1; i <= 30; i++) {
            new Thread(()->{
                map.put(Thread.currentThread().getName(),
                       UUID.randomUUID().toString().substring(0,5));
                System.out.println(map);
            },String.valueOf(i)).start();
        }
    }
}
```

---

## 八、Callable (简单)

> Runnable 的实现类中有一个 FutureTask 类，接受 Callable 类型参数

- **Callable** 是 java.util 包下 concurrent 下的接口，有返回值，可以抛出被检查的异常
- **Runnable** 是 java.lang 包下的接口，没有返回值，不可以抛出被检查的异常
- 二者调用的方法不同，run() / call()

同样的 **Lock** 和 **Synchronized** 二者的区别，前者是java.util 下的接口 后者是 java.lang 下的关键字。

```java
/**
 * 1、探究原理
 * 2、觉自己会用
 */
public class CallableTest {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        // new Thread(new Runnable()).start();// 启动Runnable
        // new Thread(new FutureTask<V>()).start();
        // new Thread(new FutureTask<V>( Callable )).start();

        // new 一个MyThread实例
        MyThread thread = new MyThread();
        // MyThread实例放入FutureTask
        FutureTask futureTask = new FutureTask(thread); // 适配类
        new Thread(futureTask,"A").start();
        new Thread(futureTask,"B").start(); // call()方法结果会被缓存，提高效率，因此只打印 1 个 call
        
        // 这个get 方法可能会产生阻塞！把他放到最后
        // 或者使用异步通信来处理！
        Integer o = (Integer) futureTask.get(); 
        System.out.println(o);// 1024
    }
}
class MyThread implements Callable<Integer> {
    @Override
    public Integer call() {
        System.out.println("call()");
        return 1024;
    }
}
//class MyThread implements Runnable {
//
//    @Override
//    public void run() {
//        System.out.println("run()"); // 会打印几个run
//    }
//}
```

**细节：**

1、有缓存

2、结果可能需要等待，会阻塞！

---

## 九、常用的辅助类(必会)

### 1、CountDownLatch

**减法计数器： 实现调用几次线程后 再触发某一个任务**

```java
// 计数器
public class CountDownLatchDemo {
    public static void main(String[] args) throws InterruptedException {
        // 总数是6，必须要执行任务的时候，再使用！
        CountDownLatch countDownLatch = new CountDownLatch(6);
        for (int i = 1; i <= 6; i++) {
            new Thread(()->{
                System.out.println(Thread.currentThread().getName() + " Go out");
                countDownLatch.countDown(); // 数量 -1
            },String.valueOf(i)).start();
        }
        countDownLatch.await(); // 等待计数器归零，然后再向下执行
        System.out.println("Close Door");
    }
}
```

原理：

`countDownLatch.countDown();` // 数量 -1

`countDownLatch.await();` // 等待计数器归零，然后再向下执行

每次有线程调用 countDown() 数量 -1，当计数器变为 0 时，countDownLatch.await() 就会被唤醒，继续执行！

### 2、CyclicBarrier

**加法计数器：集齐7颗龙珠召唤神龙**

```java
public class  CyclicBarrierDemo {
    public static void main(String[] args) {
        /*
         * 集齐7颗龙珠召唤神龙
         */
        // 召唤龙珠的线程
        CyclicBarrier cyclicBarrier = new CyclicBarrier(7,()->{
            System.out.println("召唤神龙成功！");
        });
        for (int i = 1; i <= 7; i++) {
            // 在匿名内部类中需要访问局部变量，那么这个局部变量必须用final修饰符修饰
            final int temp = i;
            new Thread(()->{
                System.out.println(Thread.currentThread().getName()+"收集"+temp+"个龙珠");
                try {
                    cyclicBarrier.await(); // 等待
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } catch (BrokenBarrierException e) {
                    e.printStackTrace();
                }
            }).start();
        }
    }
}
```

### 3、Semaphore

> Semaphore：信号量

**限流 / 抢车位！6 车 — 3 个停车位置**

```java
public class SemaphoreDemo {
    public static void main(String[] args) {
        // 线程数量：停车位! 限流！
        // 如果已有 3 个线程执行（3个车位已满），则其他线程需要等待‘车位’释放后，才能执行！
        Semaphore semaphore = new Semaphore(3);
        for (int i = 1; i <= 6; i++) {
            new Thread(()->{
                // acquire() 得到
                try {
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName()+"抢到车位");
                    TimeUnit.SECONDS.sleep(2);
                    System.out.println(Thread.currentThread().getName()+"离开车位");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    semaphore.release(); // release() 释放 
                }
            },String.valueOf(i)).start();
        }
    }
}
```

**原理：**

`semaphore.acquire()`：获得，假设如果已经满了，等待，等待被释放为止

`semaphore.release()`： 释放，会将当前的信号量释放 + 1，然后唤醒等待的线程

作用：多个共享资源互斥的使用！并发限流，控制最大的线程数

---

## 十、读写锁ReadWriteLock

```java
/**
 * 独占锁（写锁） 一次只能被一个线程占有
 * 共享锁（读锁） 多个线程可以同时占有
 * ReadWriteLock
 * 读-读  可以共存！
 * 读-写  不能共存！
 * 写-写  不能共存！
 */
public class ReadWriteLockDemo {
    public static void main(String[] args) {
        //MyCache myCache = new MyCache();
        MyCacheLock myCacheLock = new MyCacheLock();
        // 写入
        for (int i = 1; i <= 5; i++) {
            final int temp = i;
            new Thread(()->{
                myCacheLock.put(temp + "",temp + "");
            }, String.valueOf(i)).start();
        }
        // 读取
        for (int i = 1; i <= 5; i++) {
            final int temp = i;
            new Thread(()->{
                myCacheLock.get(temp + "");
            }, String.valueOf(i)).start();
        }
    }
}
/**
 * 自定义缓存
 * 加锁的
 */
class MyCacheLock{
    private volatile Map<String,Object> map = new HashMap<>();
    // 读写锁：更加细粒度的控制，锁的对象是this
    private ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    // private Lock lock = new ReentrantLock();
    // 存，写入的时候，只希望同时只有一个线程写
    public void put(String key,Object value){
        readWriteLock.writeLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + "写入" + key);
            map.put(key,value);
            System.out.println(Thread.currentThread().getName() + "写入OK");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            readWriteLock.writeLock().unlock();
        }
    }
    // 取，读，所有人都可以读！
    public void get(String key){
        readWriteLock.readLock().lock();
        try {
            System.out.println(Thread.currentThread().getName() + "读取" + key);
            Object o = map.get(key);
            System.out.println(Thread.currentThread().getName() + "读取OK");
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            readWriteLock.readLock().unlock();
        }
    }
}
/**
 * 自定义缓存
 * 不加锁的
 */
class MyCache{
    private volatile Map<String,Object> map = new HashMap<>();
    // 存，写
    public void put(String key,Object value){
        System.out.println(Thread.currentThread().getName() + "写入" + key);
        map.put(key,value);
        System.out.println(Thread.currentThread().getName() + "写入OK");
    }
    // 取，读
    public void get(String key){
        System.out.println(Thread.currentThread().getName() + "读取" + key);
        Object o = map.get(key);
        System.out.println(Thread.currentThread().getName() + "读取OK");
    }
}
```

执行效果如图：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125240193-4df5463a2e4b9ef030fad440d6807711-04f76a.png" alt="image-20220222125240193" style="zoom: 50%;" />

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220222125228234-de29166ff51fd792413e47f4b9196893-05c754.png" alt="image-20220222125228234" style="zoom: 67%;" />
