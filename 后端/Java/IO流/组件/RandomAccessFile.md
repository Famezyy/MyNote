# RandomAccessFile

## 1.简介

- RandomAccessFile 既可以读取文件内容，也可以向文件输出数据。同时，RandomAccessFile 支持“随机访问”的方式，程序快可以直接跳转到文件的任意地方来读写数据

- 由于 RandomAccessFile 可以自由访问文件的任意位置，所以如果需要**访问文件的部分内容**，而不是把文件从头读到尾，使用 RandomAccessFile 将是更好的选择

- 与 OutputStream、Writer 等输出流不同的是，RandomAccessFile 允许自由定义文件记录指针，RandomAccessFile 可以不从开始的地方开始输出，因此 RandomAccessFile 可以向已存在的文件后追加内容，如果程序需要向已存在的文件后追加内容，则应该使用 RandomAccessFile

- RandomAccessFile 的方法虽然多，但它有一个最大的局限，就是只能读写文件，不能读写其他 IO 节点

- RandomAccessFile 的一个重要使用场景就是网络请求中的多线程下载及断点续传

## 2.常用方法

### 2.1 构造函数

RandomAccessFile 类有两个构造函数，一个需要使用 String 参数来指定文件名，一个使用 File 参数来指定文件本身。除此之外，创建 RandomAccessFile 对象时还需要指定一个 mode 参数，该参数指定 RandomAccessFile 的访问模式，一共有 4 种模式。

- `r`：以只读方式打开。调用结果对象的任何 write 方法都将导致抛出 IOException
- `rw`：打开以便读取和写入
- `rws`：打开以便读取和写入。相对于 "rw"，"rws" 还要求对“文件的内容”或“元数据”的每个更新都同步写入到基础存储设备
- `rwd`：打开以便读取和写入，相对于 "rw"，"rwd" 还要求对“文件的内容”的每个更新都同步写入到基础存储设备

### 2.2 重要方法

RandomAccessFile 既可以读文件，也可以写文件，所以类似于 InputStream 的 read() 方法，以及类似于 OutputStream 的 write() 方法，RandomAccessFile 都具备。除此之外，RandomAccessFile 具备两个特有的方法，来支持其随机访问的特性。

RandomAccessFile 对象包含了一个记录指针，用以标识当前读写处的位置，当程序新创建一个 RandomAccessFile 对象时，该对象的文件指针记录位于文件头（也就是 0 处），当读/写了 n 个字节后，文件记录指针将会后移 n 个字节。除此之外，RandomAccessFile 还可以自由移动该记录指针。下面就是 RandomAccessFile 具有的两个特殊方法，来操作记录指针，实现随机访问：

`long getFilePointer()`：返回文件记录指针的当前位置

`void seek(long pos)`：将文件指针定位到 pos 位置

## 3.RandomAccessFile的使用

利用 RandomAccessFile 实现文件的多线程下载，即多线程下载一个文件时，将文件分成几块，每块用不同的线程进行下载。下面是一个利用多线程在写文件时的例子，其中预先分配文件所需要的空间，然后在所分配的空间中进行分块，然后写入：

```java
/** 
 * 测试利用多线程进行文件的写操作 
 */  
public class Test {  

  public static void main(String[] args) throws Exception {  
    // 预分配文件所占的磁盘空间，磁盘中会创建一个指定大小的文件  
    RandomAccessFile raf = new RandomAccessFile("D://abc.txt", "rw");  
    raf.setLength(1024*1024); // 预分配 1M 的文件空间  
    raf.close();  

    // 所要写入的文件内容  
    String s1 = "第一个字符串";  
    String s2 = "第二个字符串";  
    String s3 = "第三个字符串";  
    String s4 = "第四个字符串";  
    String s5 = "第五个字符串";  

    // 利用多线程同时写入一个文件，可分配较大的缓冲区
    // 虽然分配的缓存区远超过实际写入的大小，但是未写入的地方由于不会显示，最终文件内容依然是紧凑的
    new FileWriteThread(1024*1,s1.getBytes()).start();
    new FileWriteThread(1024*2,s2.getBytes()).start();
    new FileWriteThread(1024*3,s3.getBytes()).start();
    new FileWriteThread(1024*4,s4.getBytes()).start(); 
    new FileWriteThread(1024*5,s5.getBytes()).start();
  }  

  // 利用线程在文件的指定位置写入指定数据  
  static class FileWriteThread extends Thread{  
    private int skip;  
    private byte[] content;  

    public FileWriteThread(int skip, byte[] content){  
      this.skip = skip;  
      this.content = content;  
    }  

    public void run(){  
      RandomAccessFile raf = null;  
      try {  
        raf = new RandomAccessFile("D://abc.txt", "rw");  
        raf.seek(skip);  
        raf.write(content);  
      } catch (FileNotFoundException e) {  
        e.printStackTrace();  
      } catch (IOException e) {  
        e.printStackTrace();  
      } finally {  
        try {  
          raf.close();  
        } catch (Exception e) {  
        }  
      }  
    }  
  }  
}  
```

当 RandomAccessFile 向指定文件中插入内容时，将会覆盖掉原有内容。如果不想覆盖掉，则需要将原有内容的长度读取出来，然后先把插入内容插入后再把原有内容追加到插入内容后。

```java
// 声明一个RandomAccessFile对象
RandomAccessFile r = new RandomAccessFile("F:/tmp/xx.xx", "rw");
//把指针移动到结尾
r.seek(r.length());
//进行写入文件
r.write(" this".getBytes());
//关闭文件
r.close();
```

