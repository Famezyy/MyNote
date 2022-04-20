## StringBuilder、StringBuffer、StringJoiner

| 类型          | 特点                                                | 使用场景                           |
| ------------- | --------------------------------------------------- | ---------------------------------- |
| String        | 不可变，线程安全                                    | 操作少量数据或不需要操作数据       |
| StringBuilder | 可变，线程不安全                                    | 需要频繁操作数据且不用考虑线程安全 |
| StringBuffer  | 可变，线程安全（添加了Synchronize关键字），性能较差 | 需要频繁操作数据且需要考虑线程安全 |

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

> `char`占用两个字节，`byte`占用一个字节

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

