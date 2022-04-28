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

