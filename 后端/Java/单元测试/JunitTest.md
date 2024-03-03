# JunitTest

## 1.Junit5的变化

Junit5 与之前版本的 Junit 框架有很大不同，现在由三个不同子项目的几个不同模块组成。

`Junit5 = Junit platform + Junit jupiter + Junit vintage`

- `Junit platform`：是在 JVM 上启动测试框架的基础，不仅支持 Junit 自制的测试引擎，其他测试引擎也都可以接入
- `Junit jupiter`：提供了 Junit5 的新的编程模型，是 Junit5 新特性的核心，内部包含了一个测试引擎，用于在 Junit platform 上运行
- `Junit vintage`：为了兼容老的版本，`Junit vintage`提供了兼容`Junit4.x`、`Junit3.x`的测试引擎

## 2.常用注解

- `@Test`：表示该方法是测试方法

- `@ParameterizedTest`：表示方法是参数化测试

- `@RepeatedTest`：表示方法可重复执行

- `@DisplayName`：为测试类或者测试方法设置展示名称

- `@BeforeEach`：在每个单元测试之前执行

- `@AfterEach`：在每个单元测试之后执行

- `@BeforeAll`：在所有单元测试之前执行一次

- `@AfterAll`：在所有单元测试执行之后执行一次

- `@Tag`：表示单元测试类别

- `@Disabled`：表示测试类或测试方法不执行

- `@Timeout`：表示测试方法运行如果超过了指定时间则返回错误

  ```java
  @Timeout(value=500, unit = TimeUnit.MILLISECONDS)
  ```

- `ExtendWith`：为测试类或测试方法提供扩展类引用

## 3.断言

断言是测试方法中的核心部分，用来对测试需要满足的条件进行验证。这些断言方法都是`org.junit.jupiter.api.Assertions`的静态方法。

### 3.1 简单断言

| 方法              | 说明                                 |
| ----------------- | ------------------------------------ |
| assertEquals      | 判断两个对象或两个原始类型是否相等   |
| assertNotEquals   | 判断两个对象或两个原始类型是否不相等 |
| assertSame        | 判断两个对象引用是否指向同一个对象   |
| assertNotSame     | 判断两个对象引用是否指向不同的对象   |
| assertTrue        | 判断给定的布尔值是否为 true          |
| assertFalse       | 判断给定的布尔值是否为 false         |
| assertNull        | 判断给定的对象引用是否为 null        |
| assertNotNull     | 判断给定的对象引用是否不为 null      |
| assertArrayEquals | 数组断言                             |
| assertAll         | 组合断言                             |
| assertThrows      | 异常断言                             |
| assertTimeout     | 超时断言                             |
| fail              | 快速失败                             |

### 3.2 数组断言

```java
import static org.junit.jupiter.api.Assertions.assertArrayEquals;
// 判断数组中的元素是否相等
assertArrayEquals(new int[] {1, 2}, new int[] {1, 2}); // true
```

### 3.3 组合断言

`assertAll()`接收多个`org.junit.jupiter.api.Executable`函数式接口作为要验证的断言。

```java
assertAll(
    "math",
	() -> assertEquals(2, 1 + 1),
    () -> assertTrue(1 > 0)
);
```

### 3.4 异常断言

```java
assertThrows(AirthmeticException.class, () -> System.out.println(1 % 0));
assertThrows(AirthmeticException.class, () -> System.out.println(1 % 0), "failure message");
```

### 3.5 超时断言

```java
// 如果测试方法时间超过 1s 则会抛出异常
assertTimeout(Duration.ifMillis(1000), () -> Thread.sleep(500));
```

### 3.6 快速失败

```java
fail("This should fail");
```

## 4.前置条件

类似于断言，不同之处在于不满足条件的断言会导致测试方法失败，而不满足的前置条件只会使得测试方法的执行终止。前置条件可以看成是测试方法执行的前提，当该前提不满足是，就没有继续执行测试的必要。

```java
assumeTrue(Objects.equals(a, 1));
assumeFalse(() -> Objects.equals(a, 1));
// 如果第一个参数为 true，则执行第二个参数
assumeingThat (
	Objects.equals(this.environment, "DEV"),
    () -> System.out.println("In DEV")
);
```

## 5.嵌套测试

Junit5 可以通过 Java 中的内部类和 `@Nested` 注解实现嵌套测试，从而可以更好地把相关的测试方法组织在一起。在内部类中可以使用 `@BeforeEach` 和 `@AfterEach`，而且嵌套的层次没有限制。执行内层的 test 会驱动外层的 `BeforeEach` 和 `AfterEach`，但是外层不会驱动执行内层。

```java
class TestDemo {
    
    @Test
    void test1() {}
    
    @Nested
    class NestedTestDemo {
        @BeforeEach
        void before() {}
        
        @Test
        void test2() {}
        
        @Test
        void test3() {}
    }
}
```

## 6.参数化测试

| 注解             | 说明                                                      |
| ---------------- | --------------------------------------------------------- |
| `@ValueSource`   | 指定入参来源，支持八大基础类以及 String 类型或 Class 类型 |
| `@NullSource`    | 提供一个 null 的入参                                      |
| `@EnumSource`    | ·提供一个枚举入参                                         |
| `@CsvFileSource` | ·读取指定 CSV 文件内容作为入参                            |
| `@MethodSource`  | 读取指定方法的返回值作为入参，需返回一个流，使用静态方法  |

**例：**

```java
@ParameterizedTest
@ValueSource(strings = {"one", "two"})
public void parameterizedTest1(String string) {
    Assertions.assertTrue(StringUtils.isNotBlank(string));
}

@ParameterizedTest
@MethodSource("method")
public void parameterizedTest1(String string) {
    Assertions.assertNotNull(name);
}

static Stream<String> method() {
    return Stream.of("apple", "banna");
}

// 传递两个参数
static Stream<Arguments> provideStringsForIsBlank() {
    return Stream.of(
      Arguments.of(null, true),
      Arguments.of("", true),
      Arguments.of("  ", true),
      Arguments.of("not blank", false)
    );
}
```

## 7.迁移指南

- `@Before`、`@After` 替换为 `@BeforeEach`、`@AfterEach`
- `@BeforeClass`、`@AfterClass` 替换为 `@BeforeAll`、`@AfterAll`
- `@Ignore` 替换为 `@Disabled`
- `@Category` 替换为 `@Tag`
- `@Runwith`、`@Rule` 和 `@ClassRule` 替换为 `@ExtendWith`

## 8.整合SpringBoot

```java
@SpringBootTest
@ExtendWith(MockitoExtension.class)
@MockitoSettings(strictness = Strictness.LENIENT)
public class ParameterValidationUtilsTest {
    
    @InjectMocks
    ParameterValidationUtils mockParameterValidationUtils;
    
    @Mock
    InternationalMilesPostRegistrationProperties mockInternationalMilesPostRegistrationProperties;
    
    @Mock
    JmbCoreLogger mockJmbCoreLogger;
    
    @BeforeEach
    void setUp() {
        when(mockInternationalMilesPostRegistrationProperties.getProperty(CoreConstants.JMBCMNE001)).thenReturn("The request parameter is invalid. ");
        when(mockInternationalMilesPostRegistrationProperties.getProperty(InternationalMilesPostRegistrationConstant.DATE_FORMAT)).thenReturn("uuuuMMdd");
        when(client.getResponse()).thenAnswer(
        	invo -> {
                Thread.sleep(1000);
                return new Response();
            }
        );
    }
    
    @ParameterizedTest
    @MethodSource("getDateSources")
    void validateActivityDateInvalidDateTest(String invalidDate) {
        // throws customException if the date is invalid
        CustomException customException = Assertions.assertThrows(CustomException.class, () -> mockParameterValidationUtils.validateActivityDate(true, invalidDate));
        Assertions.assertEquals(CoreConstants.JMBCMNE001, customException.getErrorCode());
        Assertions.assertEquals("The request parameter is invalid. ", customException.getMessage());
    }
    
    static Stream<String> getDateSources() {
        return Stream.of("20210229", "20210431", "20210532");
    }
    
    @Test
    void validateActivityDateConditionsTest() {
        LocalDate now = LocalDate.now();
        String dateFormat = "yyyyMMdd";
        
        // throws customException if get a future date 
        String futureDateStr = now.plusDays(1).toString(dateFormat);
        CustomException customException1 = Assertions.assertThrows(CustomException.class, () -> mockParameterValidationUtils.validateActivityDate(true, futureDateStr));
        Assertions.assertEquals(CoreConstants.JMBCMNE001, customException1.getErrorCode());
        Assertions.assertEquals("The request parameter is invalid. ", customException1.getMessage());
        // normal case
        Assertions.assertDoesNotThrow(() -> mockParameterValidationUtils.validateActivityDate(true, now.toString(dateFormat)));
        
        // throws customException if it is JAL group and the date is older than 6 months from current date
        String sixMonthsAgo = now.plusMonths(-6).plusDays(-1).toString(dateFormat);
        CustomException customException2 = Assertions.assertThrows(CustomException.class, () -> mockParameterValidationUtils.validateActivityDate(true, sixMonthsAgo));
        Assertions.assertEquals(CoreConstants.JMBCMNE001, customException2.getErrorCode());
        Assertions.assertEquals("The request parameter is invalid. ", customException2.getMessage());
        // normal case
        Assertions.assertDoesNotThrow(() -> mockParameterValidationUtils.validateActivityDate(true, now.plusMonths(-6).toString(dateFormat)));
        
        // throws customException if it is partnership group and the date is older than 6 months plus 14 days
        String sixMonthsPlus14DaysAgo = now.plusMonths(-6).plusDays(-15).toString(dateFormat);
        CustomException customException3= Assertions.assertThrows(CustomException.class, () -> mockParameterValidationUtils.validateActivityDate(false, sixMonthsPlus14DaysAgo));
        Assertions.assertEquals(CoreConstants.JMBCMNE001, customException3.getErrorCode());
        Assertions.assertEquals("The request parameter is invalid. ", customException3.getMessage());
        // normal case
        Assertions.assertDoesNotThrow(() -> mockParameterValidationUtils.validateActivityDate(false, now.plusMonths(-6).plusDays(-14).toString(dateFormat)));
    }
    
    @ParameterizedTest
    @MethodSource("getValidCarrierCodeResources")
    void validateCarrierCodePositiveTest(String carrierCode) {
        Assertions.assertDoesNotThrow(() -> mockParameterValidationUtils.validateCarrierCode(carrierCode));
    }
    
    static Stream<String> getValidCarrierCodeResources() {
        return Stream.of("JAL", "GKF");
    }
    
    @ParameterizedTest
    @MethodSource("getInvalidCarrierCodeResources")
    void validateCarrierCodeNegativeTest(String carrierCode) {
        CustomException customException = Assertions.assertThrows(CustomException.class, () -> mockParameterValidationUtils.validateCarrierCode(carrierCode));
        Assertions.assertEquals(CoreConstants.JMBCMNE001, customException.getErrorCode());
        Assertions.assertEquals("The request parameter is invalid. ", customException.getMessage());
    }
    
    static Stream<String> getInvalidCarrierCodeResources() {
        return Stream.of(null, "", " ", "  ");
    }
```

