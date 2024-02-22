# ArrayList

## 1.删除顺序排列中的某个值

```java
public static void main(String[] args) {
    ArrayList<String> list = new ArrayList<>(Arrays.asList("1", "2", "3", "3", "5"));
    for (int i = 0; i < list.size(); i++) {
        if ("3".equals(list.get(i))) {
            list.remove("3");
        }
    }
    System.out.println(list); // [1, 2, 3, 5]
}
```

可以发现最终只删除了第一个 3！

在`remove()`方法中调用了`fastRemove()`方法，在后者中当找到目标元素后，会将该元素后面的所有元素都向前移动一格，然后最后一个设置元素为 null：

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220527135655279-82293c10ed8a6b69d46320003a1921e8-565459.png" alt="image-20220527135655279" style="zoom:50%;" />

解决办法：

```java
// 1.手动移动下标
public static void main(String[] args) {
    ArrayList<String> list = new ArrayList<>(Arrays.asList("1", "2", "3", "3", "5"));
    for (int i = 0; i < list.size(); i++) {
        if ("3".equals(list.get(i))) {
            list.remove("3");
            // 当删除元素后将下表向前移动一位
            i--;
        }
    }
    System.out.println(list); // [1, 2, 5]
}

// 2.倒序删除
public static void main(String[] args) {
    ArrayList<String> list = new ArrayList<>(Arrays.asList("1", "2", "3", "3", "5"));
    // 倒序删除
    for (int i = list.size(); i >= 0; i--) {
        if ("3".equals(list.get(i))) {
            list.remove("3");
        }
    }
    System.out.println(list); // [1, 2, 5]
}

// 3.使用迭代器
public static void main(String[] args) {
    ArrayList<String> list = new ArrayList<>(Arrays.asList("1", "2", "3", "3", "5"));
    Iterator<String> iterator = list.iterator();
    while (iterator.hasNext()) {
        if ("3".equals(iterator.next())) {
            // remove 方法中帮我们将下标前移了
            iterator.remove();
        }
    }
    System.out.println(list); // [1, 2, 5]
}
```

## 2.Arrays.asList()

- `Arrays.asList()` 转换后的 list 默认是 `AbstractList`，不支持更改操作（抛出 `UnsupportedOperationException` 异常），可以强转成 list 的实现类后使用

## 3.删除元素

```java
// 底层逻辑是迭代器的 remove 方法
list.removeIf(item -> item.equals("invalid"));
```

## 4.扩容

底层是数组结构，默认长度 10，扩容时先创建新的数组，长度为原来的 1.5 倍，然后调用`Arrys.copyOf()`复制元素，将变量指向新数组。
