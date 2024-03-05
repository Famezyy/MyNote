# Swagger

## 1. 简介

### 前后端分离

Vue + SpringBoot

- 后端时代：前端只管理静态页面，后端管理模板引擎 JSP

- **前后端分离时代**：

  - 后端：控制层，服务层，数据访问层

  - 前端：控制层，视图层

  - 前后端交互：API

  - 前后端相对独立，甚至可以部署在不同服务器

- 问题：前后端协商
- 解决方案：
  - 制定 schema，实时更新最新的 API，降低集成的风险
  - 前端测试后端接口，后端提供接口

### Swagger

- 世界上最流行的 API 框架
- RestFul Api 文档在线自动生成
- 直接运行，在线测试 API 接口

## 2. SpringBoot 集成 Swagger

- 在项目中使用 Swagger 需要 SpringBox，**Spring 版本不能高于 2.5.6**

  - Swagger2
  - ui

  ```xml
  <!-- https://mvnrepository.com/artifact/io.springfox/springfox-swagger-ui -->
  <dependency>
      <groupId>io.springfox</groupId>
      <artifactId>springfox-swagger-ui</artifactId>
      <version>2.9.2</version>
  </dependency>
  <!-- https://mvnrepository.com/artifact/io.springfox/springfox-swagger2 -->
  <dependency>
      <groupId>io.springfox</groupId>
      <artifactId>springfox-swagger2</artifactId>
      <version>2.9.2</version>
  </dependency>
  ```

- 编写 swagger 配置类

  ```java
  @Configuration
  @EnableSwagger2 // 开启 Swagger
  public class SwaggerConfig {
  }
  ```

- 访问 http://localhost:8080/swagger-ui.html

## 3. 配置 Swagger

```java
@Configuration
@EnableSwagger2 // 开启 Swagger
@Profile({"dev", "test"}) // 只在 dev 和 test 环境下生效
public class SwaggerConfig {

    // 配置 Swagger 的 Docket 的 bean 实例
    @Bean
    public Docket docket(Environment env) {

        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                // .enable(false) 关闭 swagger
                // 配置扫描接口的方式
                .select()
                // 指定扫描的包
                // any(), none(), withClassAnnotation(注解的反射对象), withMethodAnnotation(注解的反射对象)
                .apis(RequestHandlerSelectors.basePackage("swaggerdemo.youyi.controller"))
                // 过滤路径
                .paths(PathSelectors.ant("/**"))
                .build();
    }

    private ApiInfo apiInfo() {
        // 作者信息
        Contact DEFAULT_CONTACT = new Contact("youyi", "", "");
        return new ApiInfo(
                "My Swagger API Document",
                "Api Documentation",
                "1.0",
                "https://www.baidu.com",
                DEFAULT_CONTACT,
                "Apache 2.0",
                "http://www.apache.org/licenses/LICENSE-2.0",
                new ArrayList());

    }
}
```

## 4. 接口注释

```java
// model 注释
@ApiModel("user")
public class User {

    // 属性注释
    @ApiModelProperty("name")
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```

```java
@RestController
// controller 注释
@Api(tags="hello 控制")
public class HelloController {

    @RequestMapping(value="/hello")
    // 端点注释
    @ApiOperation("hello 端点")
    // 参数注释
    public User hello(@ApiParam("用户名") String username) {
        return new User();
    }
}
```

## 5.根据Swagger生成Model

导入以下依赖和 plugin

```xml
<dependencies>
	<dependency>
        <groupId>io.swagger.core.v3</groupId>
        <artifactId>swagger-annotations</artifactId>
        <version>2.2.19</version>
    </dependency>
    <dependency>
        <groupId>org.openapitools</groupId>
        <artifactId>jackson-databind-nullable</artifactId>
        <version>0.2.6</version>
    </dependency>
    <dependency>
        <groupId>javax.validation</groupId>
        <artifactId>validation-api</artifactId>
        <version>2.0.1.Final</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.openapitools</groupId>
            <artifactId>openapi-generator-maven-plugin</artifactId>
            <version>7.2.0</version>
            <executions>
                <execution>
                    <id>api-call-jal-jmbSwagger</id>
                    <goals>
                        <goal>generate</goal>
                    </goals>
                    <configuration>
                        <inputSpec>${basedir}/src/main/resources/swagger.yaml</inputSpec>
                        <generatorName>spring</generatorName>
                        <addCompileSourceRoot>false</addCompileSourceRoot>
                        <configOptions>
                            <sourceFolder>src/main/java</sourceFolder>
                            <!-- java 版本 -->
                            <dateLibrary>java17</dateLibrary>
                            <hideGenerationTimestamp>true</hideGenerationTimestamp>
                            <recursiveBeanValidation>true</recursiveBeanValidation>
                            <useBeanValidation>true</useBeanValidation>
                            <serializableModel>true</serializableModel>
                            <delegatePattern>true</delegatePattern>
                            <interfaceOnly>true</interfaceOnly>
                            <objectMapper />
                        </configOptions>
                        <output>${project.basedir}</output>
                        <!-- 生成的 model 路径 -->
                        <modelPackage>jp.co.jal.jmb.performance.info.model.jmb</modelPackage>
                        <additionalProperties>
                            <java17>true</java17>
                            <additionalProperty>jackson=true</additionalProperty>
                        </additionalProperties>
                        <generateApis>false</generateApis>
                        <generateApiDocumentation>false</generateApiDocumentation>
                        <generateApiTests>false</generateApiTests>
                        <generateModelDocumentation>false</generateModelDocumentation>
                        <generateSupportingFiles>false</generateSupportingFiles>
                        <generateModelDocumentation>false</generateModelDocumentation>
                        <!-- 是否跳过 -->
                        <skip>false</skip>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

