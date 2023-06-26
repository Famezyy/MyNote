## StringBuilder、StringBuffer、StringJoiner

| 类型          | 特点                                                         | 使用场景                           |
| ------------- | ------------------------------------------------------------ | ---------------------------------- |
| String        | value 值由`final`修饰，不可变，线程安全，每次修改产生新的对象，性能最差 | 操作少量数据或不需要操作数据       |
| StringBuilder | 可变，线程不安全，每次修改不产生新的对象，性能最高           | 需要频繁操作数据且不用考虑线程安全 |
| StringBuffer  | 可变，线程安全（添加了Synchronize关键字），每次修改不产生新的对象，性能较差 | 需要频繁操作数据且需要考虑线程安全 |

- `StringBuilder`适合**大量字符串的拼接**，不适用于==少量字符串拼接==和==循环体内使用==，后者推荐直接使用`+`

- `concat`方法底层使用了`System.ArrayCopy()`方法，适用于**大量的字符串拼接**，但不适用==少量字符串的拼接==

- `StringJoiner`用于字符串分隔符拼接

  ```java
  String[] names = {"A", "B", "C", "D"};
  StringJoiner sj = new StringJoiner(",", "[", "]");
  for (String name : names) {
      sj.add(name);
  }
  System.out.println(sj); // [A,B,C,D]
  ```

## String底层数组结构

在`JAVA8`及以前是`char数组`，因为英文只占一个字节，用`char数组`太浪费空间了，所以`JAVA9`之后改为`byte数组`。

> `char`占用两个字节（UTF-16），`byte`占用一个字节

为了兼容英文之外的其他语言，`JAVA9`还新增了一个`code`字段，用来描述使用哪种编码格式。

```java
/**
 * The supported values in this implementation are
 *
 * LATIN1
 * UTF16
 */
private final byte coder;

static final byte LATIN1 = 0;
static final byte UTF16 = 1;
```

求字符串长度的代码也发生了改变：

```java
public int length() {
    /**
     * 使用 LATIN 字符时，一个字节存储一个字符，所以长度等于字节长度
     * 使用 UTF16 时，两个字节存储一个字符，所以长度为字节长度一半，使用带符号位运算 >> 1
     *
     */
    return value.length >> coder();
}
```

使用`UTF-16`时，由于两个字节最多存储 6 万多个字符，对于其他字符，该如何处理呢？

其实`UTF-16`的两个字节称为一个`代码单元`，基本字符通常由`一个代码`单元组成，对于需要用两个以上字节表示的字符，可以使用`辅助字符`来表示，该字符用`两个代码单元`构成。

> 基本字符：U+0000 ~ U+FFFF
>
> 辅助字符：U+100000 ~ U+10FFFF

当获取字符长度时，如果存在辅助字符，则使用`length()`方法获取不到实际长度：

```java
String s1 = "zhao";
String s2 = "zhao😊";

System.out.println(s1.length()); // 4
System.out.println(s2.length()); // 6
System.out.println(s2.charAt(4)); // ?
System.out.println(s2.charAt(5)); // ?
```

因为`length()`的作用是获取`代码单元`的数量，而辅助字符需要两个代码单元。此时可以使用`码点`的方式来获取：

> 字符底层都是用二进制表示的，一个二进制对应一个字符，这个二进制就是码点

```java
String s1 = "zhao";
String s2 = "zhao😊";

System.out.println(s1.codePointCount(0, s1.length())); // 4
System.out.println(s2.codePointCount(0, s2.length())); // 5
```

判断`辅助字符`的方法：

```java
String s1 = "zhao";
String s2 = "zhao😊";

/// 1.查看码点是否是辅助字符
// 获取码点
System.out.println(Arrays.toString(s1.codePoints().toArray())); // [122, 104, 97, 111]
System.out.println(Arrays.toString(s2.codePoints().toArray())); // [122, 104, 97, 111, 128522]

// 判断码点是否是辅助字符
System.out.println(Character.isSupplementaryCodePoint(122)); // false
System.out.println(Character.isSupplementaryCodePoint(128522)); // true

/// 2.查看char是否是辅助字符
System.out.println(Character.isSurrogate(s2.charAt(3))); // false
System.out.println(Character.isSurrogate(s2.charAt(4))); // true
System.out.println(Character.isSurrogate(s2.charAt(5))); // true
```

##  最大长度

在**编译期**，java 编译器要求字符串字面量的长度小于`Poll.MAX_STRING_LENGTH = 2^16-1 = 65535`，也就是最大`65534`。

在**运行期**，字符串是以`char[]`形式存在，而数组长度是 int 类型，`Integer.MAX_VALUE = 2^31-1 = 2147483647`，约为 21 亿，一个 char 占用两个字节，共需要大约 4G 存储空间。
