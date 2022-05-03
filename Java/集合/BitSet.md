```java
BitSet bitSet = new BitSet();
// 参数只能为 int 数值
bitSet.set(5);
System.out.println(bitSet.get(5)); // true
System.out.println(bitSet.get(4)); // false
```

**应用场景**：手机号去重

```java
BitSet bitSet = new BitSet();
// 需要进行分桶，因为手机号长度超过了 int 数值范围
Map<String, BitSet> map = new HashMap<>();
BitSet bitSet135 = map.computeIfAbsent("135", k -> new BitSet());
bitSet135.set("37999999");
```

