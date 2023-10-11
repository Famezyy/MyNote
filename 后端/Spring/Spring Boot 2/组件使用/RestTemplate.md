## RestTemplate

快速入门

```java
@GetMapping("/test1")
public String test1() {
    RestTemplate restTemplate = new RestTemplate();
    UriComponentsBuilder builder1 = UriComponentsBuilder.newInstance().scheme("http").host("localhost").port("8080").path("/test2");
    builder1.queryParam("key", "va lue");
    URI uri = builder1.build().toUri();
    HttpHeaders headers = new HttpHeaders();
    headers.add("cookie", "1234");
    HttpEntity<MultiValueMap<String, String>> requestWithHeaders = new HttpEntity<>(null, new headers);
    /** POST 时可在 httpEntity 中设置 JSON Object */
    //InputRequestObject iro = new InputRequestObject();
    //HttpEntity<MultiValueMap<String, String>> requestWithHeaders = new HttpEntity<>(Wrapper.writeValueAsString(iro), new HttpHeaders());

    ResponseEntity<String> exchange = restTemplate.exchange(uri, HttpMethod.GET, requestWithHeaders, String.class);
    return exchange.getBody();
}

@GetMapping("/test2")
public String test2(@RequestParam String key) {
    return key;
}
```

设置超时时间

```java
HttpCpmponentsClientHttpRequestFactory requestFactory = new HttpCpmponentsClientHttpRequestFactory();
requestFactory.setReadTimeOut(READ_TIME_OUT);
requestFactory.setConnectTimeOut(CONNECTION_TIME_OUT);
RestTemplate restTemplate = new RestTemplate(requestFactory);
```

> 观察实现方法对比`HttpCpmponentsClientHttpRequestFactory`和`SimpleClientHttpRequestFactory`，前者是用连接池管理数据库的链接，后者是每次调用`restTemplate`时都要打开一个链接，然后关闭，性能稍差一些。

在 Springboot 中，可以使用`RestTemplateBuilder`去创建 RestTemplate：

```java
@Bean
public RestTemplate restTemplate(RestTemplateBuilder builder) {
    // 可通过构造者模式设置属性
    return builder.build();
}
```

### postForObject

```java
Map<String, Integer> map = new HashMap<>();
map.put("productId", order.getProductId());
String s = restTemplate.postForObject("http://localhost:8081/stock?productId={productId}", null, String.class, map);
```

## 实现SSL访问

```java
KeyStore keyStore = KeyStore.getInstance("jks");
FileInputStream inStream = new FileInputStream(new File("证书地址"));
keyStore.load(inStream, "password".toCharArray());
TrustManagerFactory tmf = TrustManagerFactory.getInstance("sunx509");
tmf.init(keyStore);
SSLContext context = SSLContext.getInstance("TLS");
context.init(null, tmf.getTrustManagers(), null);
SSLConnectionSlcketFactory factory = new SSLConnectionSocketFactory(context, new DefaultHostnameVerifier());
restTemplate.setRequestFactory(new HttpComponentsClientHttpRequestFactory(HttpClientBuilder.create().setSSLSocketFactory(factory).build()));
```

