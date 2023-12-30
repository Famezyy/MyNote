# ThreadLocal

在多线程并发执行过程中，为了保证多个线程对变量的安全访问，可以将变量放到 `ThreadLocal` 类型的对象中，使变量在每个线程间独立。

## 1.基本使用

`ThreadLocal` 位于 JDK 的 java.lang 核心包。如果程序创建了一个 `ThreadLocal` 实例，那么在访问这个变量值时，每个线程都会拥有一个独立的、自己的本地值，避免了线程安全问题。当线程结束后，每个线程拥有的本地值会被释放。

`ThreadLocal` 提供了以下三种方法用于进行本地值的操作：

```java
public T get();
public void set(T value);
public void remove();
```

下面的例子展示了 `ThreadLocal` 的值无法在两个线程间共享： 

```java
public class ThreadLocalTest {
    static ThreadLocal<Person> tl = new ThreadLocal<>();
    
    public static void main(String[] args) {
        new Thread(() ->{
            try {
                // 休息 1 秒
                TimeUnit.SECONDS.sleep(1);
            } catch (Exception e) {
                e.printStackTrace();
            }
            tl.set(new Person());
        }).start();
        
        new Thread(() ->{
            try {
                // 休息 2 秒
                TimeUnit.SECONDS.sleep(2);
            } catch (Exception e) {
                e.printStackTrace();
            }
            System.out.println(tl.get()); // null
        }).start();
    }
    
    static class Person {
        String name = "zhangsan";
    }
}
```
如果线程尚未在本地变量中绑定一个值，直接通过调用 `get()` 方法获取本地值会获取一个空值，此时可以通过 `ThreadLocal.withInitial()` 静态工厂方法，在定义 `ThreadLocal` 对象时设置一个获取初始值的回调函数：

```java
ThreadLocal<String> localVariable = ThreadLocal.withInitial(() -> {"initial value"});
```

## 2.适用场景

`ThreadLocal` 的常用案例有：为每个线程绑定一个会话信息、数据库连接、HTTP 请求等。

### 2.1 线程隔离

`ThreadLocal` 的主要价值在于线程隔离，`ThreadLocal` 中的数据只属于当前线程，对其他线程不可见；同时由于各个线程之间的数据相互隔离，避免了同步加锁带来的性能损失，提升了并发的性能。

下面的代码来自 Hibernate，代码中通过 `ThreadLocal` 进行数据库连接 `Session` 的线程本地化存储：

```java
private static final ThreadLocal threadSession = new ThreadLocal();

public static Session getSession() throws InfrastructureException {
    // 一个 Session 代表一个数据库连接
    Session s = (Session) threadSession.get();
    try {
        if (s == null) {
            s = getSessionFactory().openSession();
            threadSession.set(s);
        }
    } catch (HibernateException ex) {
        throw new InfrastructureException(ex);
    }
    return s;
}
```

### 2.2 跨函数传递数据

在同一个线程内跨类、跨方法传递数据时，如果不用 `ThreadLocal`，则需要靠传参和返回值，会增加类和方法之间的耦合度。而线程执行过程中所执行到的函数都能读写 `ThreadLocal` 变量，从而可以方便地实现跨函数的数据传递。

例如如下获取 `Session` 和保存 `Session` 的示例代码：

```java
private static final ThreadLocal<HttpSession> sessionLocal = new ThreadLocal<>();

public static void setSession(HttpSession session) {
    sessionLocal.set(session);
}

public static HttpSession getSession() {
    HttpSession session = sessionLocal.get();
    return session;
}
```

## 3.内部结构演进

在早期的 JDK 版本中，ThreadLocal 的内部结构是一个 `Map`，其中每一个线程实例作为 Key，线程在“线程本地变量”中绑定的值为 Value。这个版本中的  `Map` 的拥有者是 `ThreadLocal`，每一个 `ThreadLocal` 实例拥有一个 `Map` 实例。

到了 JDK 8 版本中，ThreadLocal 的内部结构发生了演进，虽然还是使用 `Map` 结构，但是其拥有者变成了 `Thread` 实例，每一个 `Thread` 实例拥有一个 `map` 实例。同时 `Map` 的 Key 也变成了 `ThreadLocal` 实例。

新版本的 `ThreadLocalMap` 还是由 `ThreadLocal` 类维护的，由 `ThreadLocal` 负责 `ThreadLocalMap` 实例的获取和创建，并从中设置本地值、获取本地值。所以 `ThreadLocalMap` 还寄存于 `ThreadLocal` 内部，没有迁移到 `Thread` 内部。

与早期的实现相比，新版本的主要优势为：

- 每个 `ThreadLocalMap` 存储的 "Key-Value" 数量变少。早期版本的数量与线程个数强相关，这意味着每个线程都有一个槽位，无论是否实际上使用 `ThreadLocal` 存储变量；而新版本的 Key 为 `ThreadLocal` 实例，即每个 `ThreadLocal` 实例对应一个槽位，而不是每个线程一个槽位，因此相比较下 `TreadLocal` 实例会少一些。
- 早期版本 `ThreadLocalMap` 的拥有者为 `ThreadLocal`，在 `Thread` 实例销毁后，`ThreadLocalMap` 还是存在的；而新版本中 `ThreadLocalMap` 的拥有者为 `Thread`，当 `Thread` 销毁后，`ThreadLocalMap` 也会随之销毁，在一定程度上减少了内存的消耗。

## 4.ThreadLocal源码分析

### 4.1 set(T value)

```java
// Thread 的 threadLocals (threadLocalMap) 成员变量，每个线程管理一个，初始为 null
ThreadLocal.ThreadLocalMap threadLocals = null;

public void set(T value) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取当前线程的 ThreadLocalMap 成员属性
    ThreadLocalMap map = getMap(t);
    // 如果 ThreadLocalMap 存在
    if (map != null) {
        // 将 value 放到 map 中，key 是当前的 ThreadLocal 对象
        map.set(this, value);
    } else {
        // 如果 ThreadLocalMap 不存在则创建一个实例，然后作为成员属性关联到 t 实例
        // 只有当赋值或获取值时才会创建 ThreadLocalMap
        createMap(t, value);
    }
}

// 获取线程 t 的 ThreadLocalMap 成员属性
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

// 为线程 t 创建一个 ThreadLocalMap 成员
// 并为新的 map 成员设置第一个 Key-Value 对，Key 为当前 ThreadLocal 对象
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
###  4.2 get()

```java
public T get() {
    // 获得当前线程
    Thread t = Thread.currentThread();
    // 获得线程的 ThreadLocalMap 成员变量
    ThreadLocalMap map = getMap(t);
    // 如果 ThreadLocalMap 存在
    if (map != null) {
        // 以 ThreadLocal 为 Key 尝试获取
        ThreadLocalMap.Entry e = map.getEntry(this);
        // 如果 entry 存在
        if (e != null) {
            @SuppressWarnings("unchecked")
            // 返回 entry 的 value
            T result = (T)e.value;
            return result;
        }
    }
    // 如果当前线程对应的 map 不存在
    // 或者 map 存在但是没有对应的键值对，返回初始值
    return setInitialValue();
}

// 设置 ThreadLocal 关联的初始值并返回
private T setInitialValue() {
    // 调用初始化钩子函数，获取初始值
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
    return value;
}
```

### 4.3 remove()



### 4.4 initialValue()

## 5.ThreadLocalMap源码分析

```java
// ThreadLocal 的静态内部类
static class ThreadLocalMap {

    // Entry 继承了弱引用
    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            // 相当于 new WeakReference<ThreadLocal<?>>(k);
            super(k);
            value = v;
        }
    }
    // ----------------------------- 存入 -----------------------------

    // ThreadLocal.ThreadLocalMap 的 set() 方法，最终存储到了继承了弱引用的 Entry 数组
    private void set(ThreadLocal<?> key, Object value) {
        Entry[] tab = table;
        int len = tab.length;
        int i = key.threadLocalHashCode & (len-1);
        // 先看数组中是否已经存储了该键值对
        // ThreadLocalMap 的存放策略为：当出现键冲突时，会依次向后查找第一个为 null 的键然后存入
        for (Entry e = tab[i];
             e != null;
             e = tab[i = nextIndex(i, len)]) {
            ThreadLocal<?> k = e.get();

            if (k == key) {
                e.value = value;
                return;
            }

            // 如果 key 为 null 说明该 key 已经被回收，则会重复利用该位置
            if (k == null) {
                replaceStaleEntry(key, value, i);
                return;
            }
        }
        // 如果数组中没有该键值对，将 entry 存放到 tab（Entry数组）
        tab[i] = new Entry (key, value);

        int sz = ++size;
        // 如果长度大于 threshold（默认 Entry 数组长度的 2/3），进行扩容
        if (!cleanSomeSlots(i, sz) && sz >= threshold)
            rehash();
    }

    private void rehash() {
        // 先删除 key 为 null 的 Entry
        expungeStaleEntries();
        // Use lower threshold for doubling to avoid hysteresis
        if (size >= threshold - threshold / 4)
            resize();
    }
    // ----------------------------- 查找 -----------------------------

    // 查找 key 对应的 Entry
    private Entry getEntry(ThreadLocal<?> key) {
        int i = key.threadLocalHashCode & (table.length - 1);
        Entry e = table[i];
        if (e != null && e.get() == key)
            return e;
        else
            return getEntryAfterMiss(key, i, e);
    }

    // 在第一次索引时没找到（存在键冲突或该 Entry 已被删除）
    private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
        Entry[] tab = table;
        int len = tab.length;

        while (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == key)
                return e;
            if (k == null)
                // 当存在 key 为 null 时，删除该 Entry 直到出现为 null 的 Entry 为止
                expungeStaleEntry(i);
            	// 然后继续遍历查找索引
            else
                i = nextIndex(i, len);
            e = tab[i];
        }
        return null;
    }
}
```
## 2.为什么要用弱引用？
如果使用了强引用作为 key 去创建一个线程私有的变量，那么这个私有变量的生命周期就与`ThreadLocals`这个 map 绑定了，也就是与该线程绑定了，直到线程结束前都不会被回收。如果是个弱引用，一旦`tl`与`new`出来的`ThreadLocal`切断了联系，当下次 GC 时该`ThreadLocal`对象就会被回收。
<img src="img/ThreadLocal/image-20220525224019756-394004747a056767951740f09d9845a7-aa6090.png" alt="image-20220525224019756" style="zoom:80%;" />

## 3.内存泄漏
但是此时只有`key`被回收了，`Entry`对象中的`value`却永远被保存下来了，这就是内存泄漏的问题。设想如果这发生在**线程池**中：一个线程被使用并创建了`ThreadLocal`变量，但是发生了内存泄露并返回给了线程池。甚者`key`也没有被回收并回到了线程池中。
所以线程池在回收线程后，会首先清理`threadLocals`。

> 调用`get()`方法和`set()`方法都会重复利用 key 为 null 的 Entry。
