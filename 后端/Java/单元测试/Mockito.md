# Mockito

引入依赖

```xml
<dependency>
	<groupId>org.mockito</groupId>
    <artifactId>mockito-all</artifactId>
    <scope>test</scope>
</dependency>
```

## 1.快速上手

```java
@ExtendWith(MockitoJunitRunner.class)
public class AccountLoginCountrollerTest {

    private AccountDao accountDao;
    private HttpServletRequest request;
    private AccountLoginController accountLoginController;

    @BeforeAll
    public void setUp() {
        this.accountDao = Mockito.mock(AccountDao.class);
        this.request = Mockito.mock(HttpServletRequest.class);
        this.accountLoginController = Mockito.mock(AccountLoginController.class);
    }

    @Test
    void testLoginSuccess() {
        Account account = new Account();
        Mockito.when(request.getParameter("userId")).thenReturn("root");
        Mockito.when(request.getParameter("password")).thenReturn("123456");
        Mockito.when(accountDao.findAccount(anyString(), anyString())).thenReturn(account);
        
        String result = accountLoginController.login(request);
    }
    
    @Test
    void testLoginFailure() {
        Mockito.when(request.getParameter("userId")).thenReturn("root");
        Mockito.when(request.getParameter("password")).thenReturn("123456");
        Mockito.when(accountDao.findAccount(anyString(), anyString())).thenReturn(null);
        assertEquals("/login", accountLoginController.login(request));
    }
    
    @Test
    void testLogin505() {
        Mockito.when(request.getParameter("userId")).thenReturn("root");
        Mockito.when(request.getParameter("password")).thenReturn("123456");
        Mockito.when(accountDao.finAccount(anyString(), anyString())).thenThrow(UnsupportedOperationException.class);
        assertEquals("/501", accountLoginController.login(request));
    }
}
```

## 2.如何MOCK

### 2.1 @ExtendWith（MockitoJunitRunner.class）

```java
@ExtendWith(MockitoExtension.class)
public class MockDemo {
    @Test
    void test1() {
        Human human = Mockito.mock(Human.class, Mockito.RETURNS_SMART_NULLS);
        System.out.println(human.getUser()); // SmartNull returned by this unstubbed method call on a mock: human.getUser();
        System.out.println(human.getUser().getName()); // SmartNullPointerException
    }

    @Test
    void test2() {
        Human human = Mockito.mock(Human.class);
        System.out.println(human.getUser()); // null
        System.out.println(human.getUser().getName()); // NullPointerException
    }
}
```

### 2.2 MockitoAnnotations.initMocks(this)

```java
public class MockDemo {
    
    @Before
    public void init() {
        MockitoAnnotations.initMocks(this);
    }
    
    // 会 mock 方法的返回值
    @Mock(answer=Mockito.RETURNS_DEEP_STUBS)
    private Human human;
    
    @Test
    void test() {
        System.out.println(human.getUser()); // Mock for User, hashCode: 708348097
        System.out.println(human.getUser().getName()); // null
    }
}
```

### 3.常用方法

#### 3.1 stubbing

##### 1.doNothing() & verify()

```java
@Mock
List<Integer> list;

@Test
void test1() {
    doNothing().when(list).clear(); // 调用 list.clear() 时什么也不做
    list.clear();
    verify(list, times(1)).clear(); // 验证调用了1次 clear() 方法
    doThrow(RuntimeException.class).when(list).clear();
    assertThrows(RuntimeException.class, () -> list.clear());
}
```

##### 2.when() & doReturn()

```java
@Mock
List<Integer> list;

@Test
void test1() {
    when(list.get(0)).thenReturn(1);
    doReturn(2).when(list).get(1);
    assertEquals(1, list.get(0));
    assertEquals(2, list.get(1));
}
```

**每次调用返回不同的值**

```java
@Mock
List<Integer> list;

@Test
void test1() {
    // when(list.size()).thenReturn(1).thenReturn(2).thenReturn(3).thenReturn(4);
    when(list.size()).thenReturn(1, 2, 3, 4);
    assertTrue(() -> 1 == list.size());
    assertTrue(() -> 2 == list.size());
    assertTrue(() -> 3 == list.size());
    assertTrue(() -> 4 == list.size());
    assertTrue(() -> 4 == list.size());
}
```

> 在某些环境下执行可能会产生`UnnecessaryStubbingException`，有如下几种设置方法：
>
> 1. `@Mock(lenient = true)`
> 2. `List list = mock(List.class, withSettings().leninet())`
> 3. `lenient().when(list.size()).thenReturn()`
> 4. 在类上添加注解`@MockitoSettings(strictness = Strictness.LENIENT)`

**自定义处理逻辑**

```java
@Test
    void test1() {
        when(list.get(anyInt())).thenAnswer(mock -> {
            Integer index = mock.getArgument(0, Integer.class);
            return String.valueOf(index + 10);
        });
        System.out.println(list.get(0)); // 10
    }
```

**执行本来的方法**

```java
StubbingService service = mock(StubbingService.class);
when(service.getS()).thenCallRealMethod();
assertEquals("Alex", service.getS());
```

#### 3.2 InjectMocks

`@InjectMocks`：创建一个实例，其余用`@Mock`或者`@Spy`注解创建的 mock 将被注入到该实例中。

```java
@ExtendWith(MockitoExtension.class)
public class MockDemo {
    
    @InjectMocks
    private SomeHandler someHandler;
    
    @Mock
    private OneDependency oneDependency; // 这个 mock 会被注入到 someHandler 中
    
}
```

#### 3.3 Spy

会调用真的方法除非使用 stubbing。

```java
List<String> realList = new ArrayList<>();
List<String> list = spy(realList);

list.add("Mockito");
list.add("PowerMock");

assertEquals("Mockito", list.get(0));
assertEquals("PowerMock", list.get(1));
assertFalse(list.isEmpty());

when(list.size()).thenReturn(0);

assertEquals("Mockito", list.get(0));
assertEquals("PowerMock", list.get(1));
assertNotEquals(2, list.size());
```

或者通过注解创建：

```java
@Spy
List<String> list = new ArrayList<>();
```

#### 3.4 参数匹配

##### 1.isA() & any()

```java
Foo foo = mock(Foo.class);
when(foo.function(Mockito.isA(Child1.class))).thenReturn(100);
int result = foo.function(new Child1());
assertEquals(100, result); // true

result = foo.function(new Child2());
assertEquals(0, result); // fasle

reset(foo);

when(foo.function(Mockito.any())).thenReturn(100);
result = foo.function(new Child2());
assertEquals(100, result);
```

##### 2.anyInt() & anyString() & anyCollection()

##### 3.eq()

```java
when(simpleService.method(anyInt(), anyString(), anyCollection(), isA(Serializable.class))).thenReturn(100);
when(simpleService.method(anyInt(), eq("1"), anyCollection(), isA(Serializable.class))).thenReturn(100); // 会覆盖上面的执行，即当第二个参数是 1 时，会执行这里
```

## 3.常见场景

### 3.1 UnfinishedStubbingException

当我们一个 mock 方法中继续 mock 的时候就会跑出异常。比如如下例子

 ```java
 when(bbbModel.getAAAModel()).thenReturn(AAATest.mocAAAModel());
 ```

关键是你在`AAATest.mockAAAmodel()`方法中继续 mock 会的话就会跑出异常，这与 mock 的实现机制有关系。（You're nesting mocking inside of mocking）

具体问题是 mock 搞不清楚你到底是在 mock aaa.getNames() 方法还是 model.getAAAmodel() 方法。

解决办法：

```java
AAAModel aaaModel = AAATest.mockAAAModel();
when(bbbmodel.getAAAModel()).thenReturn(aaaModel);
```

### 3.2 验证方法执行

```java
@Test
public void testInvocationOfMethod() {

    Animal mockedParrot = Mockito.spy(parrot);
    mockedParrot.move(2);

    //verify the number of invocations
    verify(mockedParrot, times(1)).amIBird();
    verify(mockedParrot, times(1)).fly(2);
    verify(mockedParrot, never()).doNotFly();
}

@Test
public void testOrderOfMethods() {
    Animal mockedParrot = Mockito.spy(parrot);
    mockedParrot.move(1);

    //verify the order of invocations
    InOrder inOrder = Mockito.inOrder(mockedParrot);
    inOrder.verify(mockedParrot).amIBird();
    inOrder.verify(mockedParrot).fly(1);
    inOrder.verify(mockedParrot).flapWings();
}

@Test
public void testLeastMaxNumberOfInvocations() {
    Animal mockedParrot = Mockito.spy(parrot);
    mockedParrot.move(3);

    verify(mockedParrot, atLeast(1)).flapWings();
    verify(mockedParrot, atMost(5)).flapWings();
}

@Test
public void testNoInvocations() {
    Animal mockedParrot = Mockito.spy(parrot);
    parrot.move(3); //Note that this is parrot, not the 'mockedParrot'

    verifyNoMoreInteractions(mockedParrot);
}
```

### 3.3 泛型类型的方法返回值

对于泛型类型的返回值，可以使用`doReturn().when().method()`语句。

