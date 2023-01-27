## UnfinishedStubbingException

当我们一个 mock 方法中继续 mock 的时候就会跑出异常。比如如下例子

 ```java
 when(bbbModel.getAAAModel()).thenReturn(AAATest.mocAAAModel());
 ```

关键是你在 AAATest.mockAAAmodel() 方法中继续 mock 会的话 就会跑出异常，这与 mock 的实现机制有关系。（You're nesting mocking inside of mocking）

具体问题是 mock 搞不清楚你到底是在 mock aaa.getNames() 方法还是 model.getAAAmodel() 方法。

解决办法：

```java
AAAModel aaaModel = AAATest.mockAAAModel();
when(bbbmodel.getAAAModel()).thenReturn(aaaModel);
```

## 验证方法执行

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

## 泛型类型的方法返回值

对于泛型类型的返回值，可以使用`doReturn().when().method()`语句。

