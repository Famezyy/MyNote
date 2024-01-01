## BigDecial

1. 创建对象时不要直接传递小数到构造方法，否则精度不准确，使用以下方式：

   ```java
   BigDecimal b1 = new BigDecimal("3.1415926535");
   BigDecimal b1 = BigDecimal.valueOf(3.1415926535);
   ```

2. 使用 `equls()` 比较两个对象时会同时比较精度：

   ```java
   BigDecimal.valueOf(3.141).equals(BigDecimal.valueOf(3.1410)); // false
   ```

   此时可以使用 `compareTo()` 方法，该方法只会比较值：

   ```java
   BigDecimal.valueOf(3.141).compareTo(BigDecimal.valueOf(3.1410)); // 0
   ```

3. 进行运算时，要注意指定运算后的精度，否则如果结果是无限循环小数，则会报错：

   ```java
   BigDecimal b1 = BigDecimal.valueOf(1.00);
   BigDecimal b2 = BigDecimal.valueOf(3.00);
   BigDecimal res = b1.divide(b2, 2, RoundingMode.HALF_UP);
   ```

   - 第二个参数表示精度
   - 第三个参数表示对结果进行四舍五入

