# Jacoco

JaCoCo 是一个 Java 代码覆盖率库，用于测量代码的执行覆盖率。它的实现原理主要包括以下几个部分：

- `On-the-fly` 插桩

  JaCoCo 使用 `On-the-fly` 模式进行代码覆盖率的收集，在这种模式下，JaCoCo 通过 Java 代理在运行时动态地修改字节码。当 Java 虚拟机加载类文件时，代理会拦截这个过程，并在需要的地方插入覆盖检测代码。

- `ASM` 库

  JaCoCo 依赖于 ASM 库，这是一个 Java 字节码操作和分析框架。ASM 允许 JaCoCo 在运行时解析和修改 Java 字节码。

使用时只需在 POM 文件中加入以下 plugin：

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.11</version>
            <executions>
                <execution>
                    <id>default-prepare-agent</id>
                    <goals>
                        <goal>prepare-agent</goal>
                    </goals>
                </execution>
                <execution>
                    <id>jacoco-site</id>
                    <phase>post-integration-test</phase>
                    <goals>
                        <goal>report</goal>
                    </goals>
                    <configuration>
                        <excludes>
                            <exclude>jp/co/jal/**/model/**/*.class</exclude>
                        </excludes>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

执行 `maven test` 即可在 `target/site/jacoco/*` 目录下生成报告。