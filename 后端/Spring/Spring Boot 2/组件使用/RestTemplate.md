## RestTemplate

快速入门

```java
@EnableConfigurationProperties(NasToolsApiProperties.class)
@Configuration
public class NasToolsApiConfig {

    @Bean
    public ObjectMapper objectMapper() {
        Jackson2ObjectMapperBuilder builder = new Jackson2ObjectMapperBuilder();
        builder.modules(new JavaTimeModule())
                .featuresToDisable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS, SerializationFeature.WRITE_DATES_WITH_CONTEXT_TIME_ZONE, SerializationFeature.FAIL_ON_EMPTY_BEANS)
                .serializationInclusion(JsonInclude.Include.NON_NULL);
        return builder.build();
    }

    @Bean
    public RestTemplate restTemplate() {
        RestTemplateBuilder restTemplateBuilder = new RestTemplateBuilder();
        // 将 ObjectMapper 注入进 restTemplate
        restTemplateBuilder.messageConverters(new MappingJackson2HttpMessageConverter(objectMapper()));
        return restTemplateBuilder.build();
    }
}
```

```java
@Autowired
NasToolsApiProperties nasToolsApiProperties;

@Autowired
// 只是组合了 ObjectMapper 并捕获 JsonParse 异常
NasToolsMapper nasToolsMapper;

public <T> T getNasToolsResponse(String endpoint, MultiValueMap<String, String> queryMap, Object requestBody, HttpHeaders httpHeaders, HttpMethod httpMethod, Class<T> responseClass) throws NasToolsException {
    RestTemplate restTemplate = new RestTemplate();
    HttpEntity<?> requestWithHeaders = new HttpEntity<>(null, httpHeaders);

    UriComponentsBuilder builder = UriComponentsBuilder.newInstance()
            .scheme(nasToolsApiProperties.getScheme())
            .host(nasToolsApiProperties.getHost())
            .port(nasToolsApiProperties.getPort()).path(endpoint);
    
    if (HttpMethod.GET.equals(httpMethod)) {
        builder.queryParams(queryMap);
    }
    try {
        if (HttpMethod.POST.equals(httpMethod)) {
            requestWithHeaders = new HttpEntity<>(nasToolsMapper.writeValueAsString(requestBody), httpHeaders);
            // 如果是 application/x-www-form-urlencoded 形式
            // requestWithHeaders = new HttpEntity<>(queryMap, httpHeaders);
        }
        RI uri = builder.build().toUri();
        // 会调用注入的 ObjectMapper
        ResponseEntity<T> response = restTemplate.exchange(uri, httpMethod, requestWithHeaders, responseClass);
        return response.getBody();
    } catch (HttpClientErrorException e) {
        // 返回错误代码时进入 HttpClientErrorException 这里
        NasToolsException.ErrorCode externalCallErrorResponse = NasToolsException.ErrorCode.EXTERNAL_CALL_ERROR_RESPONSE;
        // 会调用注入的 ObjectMapper
        externalCallErrorResponse.setErrorObject(e.getResponseBodyAs(ErrorMessage.class));
        throw new NasToolsException(externalCallErrorResponse, e);
    } catch (Exception e) {
        throw new NasToolsException(NasToolsException.ErrorCode.EXTERNAL_CALL_ERROR_RESPONSE, e);
    }
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

### postForObject

```java
Map<String, Integer> map = new HashMap<>();
map.put("productId", order.getProductId());
String s = restTemplate.postForObject("http://localhost:8081/stock?productId={productId}", null, String.class, map);
```

## 实现SSL访问

```java
KeyStore keyStore = KeyStore.getInstance("jks");
keyStore.load(getClass().getResourceAsStream(env.getProperties("证书地址")), env.getProperties("密码").toCharArray());
TrustManagerFactory tmf = TrustManagerFactory.getInstance("sunx509");
tmf.init(keyStore);
SSLContext context = SSLContext.getInstance("TLS");
context.init(null, tmf.getTrustManagers(), null);
SSLConnectionSlcketFactory factory = new SSLConnectionSocketFactory(context, new DefaultHostnameVerifier());
restTemplate.setRequestFactory(new HttpComponentsClientHttpRequestFactory(HttpClientBuilder.create().setSSLSocketFactory(factory).build()));
```

如果你有 PEM（Privacy Enhanced Mail）格式的证书文件，可以使用 `openssl` 工具来进行转换，将 PEM 格式的证书和私钥合并为 PKCS#12 文件，然后再将其转换为 JKS 文件。

以下是从 PEM 到 JKS 的大致步骤：

**步骤 1: 合并证书和私钥到 PKCS#12 文件**

使用`openssl`工具将 PEM 格式的证书和私钥合并为 PKCS#12 文件（`.p12`或`.pfx`格式）。

```bash
bashCopy code
openssl pkcs12 -export -out keystore.p12 -inkey private-key.pem -in certificate.pem
```

这将把名为`private-key.pem`的私钥和`certificate.pem`的证书合并到`keystore.p12`文件中。根据实际情况，你可能需要提供口令（密码）来保护 PKCS#12 文件。

**步骤 2: 从 PKCS#12 转换为 JKS**

使用`keytool`工具从 PKCS#12 文件转换为 JKS 格式。

```bash
bashCopy code
keytool -importkeystore -srckeystore keystore.p12 -srcstoretype PKCS12 -destkeystore keystore.jks -deststoretype JKS
```

这个命令会要求你提供 PKCS#12 文件的密码（口令），然后为 JKS 文件设置一个密码。

这样就可以将 PEM 格式的证书和私钥转换为 Java KeyStore (JKS) 格式，使其可以在 Java 应用中使用。
