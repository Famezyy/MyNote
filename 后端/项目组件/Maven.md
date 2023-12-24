# Maven

## 1. 简介

### 1.1 软件开发中的阶段

> 需求分析：分析项目具体完成的功能，有什么要求，具体怎么实现
>
> 设计：根据分析的结果，设计项目使用的技术，解决难点
>
> 开发：编码实现功能，编译代码，单元测试
>
> 测试：测试整个项目的功能是否符合设计要求，完成测试报告
>
> 发布：给用户安装项目

### 1.2 Maven 能做什么

> 1. **项目的自动构建**，帮助开发人员做项目代码的编译，测试，打包，安装，部署等工作
> 2. **管理依赖**（管理项目中使用的各种 jar 包）
>    - 依赖：项目中需要使用的其他资源，常见的是 jar，比如项目使用 MySQL 驱动，我们就说项目依赖 MySQL 驱动

### 1.3 没有 Maven 时

>   管理 jar，需要从网络中单独下载某个 jar，并且需要正确选择版本，手工处理 jar 文件之间的依赖

### 1.4 什么是 Maven

> - Maven 是 apache 基金会的开源项目，使用 java 语法开发，Maven 意思是专家，内行
>
> - 是项目的自动化构建工具，管理项目的依赖

### 1.5 Maven 中的概念

> 1. POM
> 2. 约定的目录结构
> 3. 坐标
> 4. 依赖管理
> 5. 仓库管理
> 6. 生命周期
> 7. 插件和目标
> 8. 继承
> 9. 聚合

### 1.6 Maven 的获取和安装

安装`maven`：https://maven.apache.org/download.cgi

```bash
wget https://dlcdn.apache.org/maven/maven-3/3.9.6/binaries/apache-maven-3.9.6-bin.tar.gz
tar -zxvf apache-maven-3.8.4-bin.tar.gz
export PATH=/path/to/apache-maven/bin:$PATH
source ~/.bashrc
mvn -v
```

### 1.7 Maven 的目录结构

> bin：Maven 可执行命令
>
> conf：Maven 自己的配置文件

---

## 2. Maven的核心概念

### 2.1 约定的目录结构

> - Maven 使用的是大多数人遵循的目录结构，叫作约定的目录结构
>
> - 一个 Maven 项目是一个文件夹
>
>   ​    Hello                          项目文件夹
>
>   ​        \src
>
>   ​            \main                  主程序目录（完成项目功能的代码和配置文件）
>
>   ​                \java              源代码（包和相关的类定理）
>
>   ​                \resources         配置文件
>
>   ​            \test                  放置测试程序代码
>
>   ​                \java              测试代码
>
>   ​                \resources         测试需要的配置文件
>
>   ​         pom.xml                   Maven 的配置文件

### 2.2 Maven 的使用

> 在 pom.xml 文件的目录下执行`mvn compile`会编译所有 java 文件

### ==2.3 POM 文件==

>   Project Object Model，项目对象模型，把一个项目的结构和内容抽象成一个模型，在 xml 文件中声明，Maven 通过 pom.xml 文件实现项目的构建和依赖的管理

```xml
<?xml version="1.0" encoding="UTF-8"?>

<!-- project 是跟标签，后面的是约束文件 -->
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    
    <!-- pom 模型的版本 -->
    <modelVersion>4.0.0</modelVersion>

    <!-- 坐标 -->
    <groupId>com.youyi</groupId>
    <artifactId>securityDemo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <!-- Maven 常用设置 -->
    <properties>
        <java.version>1.8</java.version>
    </properties>

</project>

```

#### 2.3.1 坐标

> - groupId
>   - 组织名称、公司、单位的标识，常使用公司域名的倒写
>   - 项目规模比较大时，也可以是域名倒写 + 大项目名称
> - artifactId
>   - 项目名称，如果 groupId 中有项目，此时当前的值就是子项目
>   - 项目名称唯一
> - version
>   - 版本，项目的版本号，使用数字：主版本号.次版本号.小版本号
>   - -SNAPSHOT 表示快照，表示该项目还在开发中，不是稳定的版本
> - packaging
>   - 项目打包的类型，可以是 jar，war，ear，pom 等，默认 jar
>
> **作用**：是资源的唯一标识，在 Maven 中，每个资源都是坐标，坐标值唯一，简称 gav
>
> **项目使用 gav**：
>
> 1. 每个 maven 项目都需要有一个自己的 gav
> 2. 管理依赖，需要使用其他的 jar 包时，也需要使用 gav 作为标识

#### 2.3.2 依赖 dependency

> **依赖**：项目中要使用的其他资源（jar），通过使用 dependency 和 gav 一起完成依赖的使用
>
> - Maven 使用 gav 作为标识，从互联网下载依赖的 jar 包
>
> ```xml
> <dependencies>
>     <dependency>
>         <groupId>log4j</groupId>
>         <artifactId>log4j</artifactId>
>         <version>1.2.17</version>
>     </dependency>
> </dependencies>
> ```

### 2.4 仓库

> - Maven的仓库用来存放：
>
>   1. Maven 工具自己的 jar 包
>
>   2. 第三方的其他 jar 包
>
>   3. 自己写的程序，可以打包为 jar，存放到仓库
>
> - **仓库的分类**：
>
>   1. 本地仓库：位于本地的计算机
>      - 默认路径：用户目录/.m2/repository
>
> - 修改本地仓库路径
>
>   - 修改 Maven 工具的配置文件（安装目录/conf/settings.xml）
>
>     - ```xml
>       <localRepository>H:\プログラミング学習\web\utils\maven_repository</localRepository>
>       ```

### 2.5 生命周期，插件和命令

> **Maven 的生命周期**：项目构建的各个阶段，包括清理，编译，测试，报告，打包，安装和部署
>
> **插件**：使用 Maven 的命令完成构建项目的各个阶段，执行命令的功能是通过插件完成的，插件就是 jar
>
> **命令**：执行 Maven 功能是由命令发出的，比如 mvn compile

#### 2.5.1 命令

> - `mvn clean`：清理命令，删除以前生成的数据，删除 target 目录
>   - **maven-clean-plugin**：清理插件
>
> - `mvn compile`：代码编译，把 src/main/java 目录中的 java 代码编译为 class 文件，并且拷贝到 target/classes 目录，同时把 src/main/resources 目录中的文件拷贝到 target/classes 目录中，classes 这个目录是存放类文件的根目录，也叫类路径 classpath
>
>   - **maven-compiler-plugin**：编译插件
>
>   - **maven-resources-plugin**：资源插件，用来处理文件。作用是把 src/main/resources 目录中的文件拷贝到 target/classes 目录中
>
> - `mvn test-compile`：编译 src/test/java 目录中的源文件，把生成的 class 拷贝到 target/test-classes 目录，同时把 src/test/resources 目录中的文件拷贝到 target/test-classes 目录
>
>   - **maven-compiler-plugin**
>
>   - **maven-resources-plugin**
>
> - `mvn test`：测试命令，执行 test-classes 目录的测试程序
>   - **maven-surefire-plugin**
>
> - `mvn package`：打包，把项目中的资源 class 文件和配置文件都放到一个压缩文件中，默认 jar 类型，web 应用是 war 类型
>
>   - **maven-jar-plugin**：执行打包处理，生成一个 jar 扩展文件，放在 target 目录下
>
>   - 打包的文件名：artifactId-version.packaging
>
>   - 打包的文件中包含的是 src/main 目录中所有的生成的 class 和配置文件，和 test 无关
>
> - `mvn install`：把生成的打包的文件，安装到 maven 仓库中
>
>   - **maven-install-plugin**：把生成的 jar 文件安装到本地仓库
>
>   - 文件结构
>     - groupId 中 "." 前后都是独立的文件夹：com.youyi
>     - artifactId：独立的文件夹
>     - version：独立的文件夹
> - `mvn javadoc:javadoc`：检查 javadoc

#### 2.5.2 自定义配置插件

```xml
<!-- 设置构建项目相关的内容 -->
<build>
	<plugins>
        <!-- 设置插件 -->
    	<plugin>
        	<groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.1</version>
            <configuration>
            	<source>1.8</source> <!-- 指定编译代码的 JDK 版本 -->
                <target>1.8</target> <!-- 运行 JAVA 的 JDK 版本 -->
            </configuration>
        </plugin>
    </plugins>
</build>
```

---

## 3. 在 Idea 中的应用

### 3.1 Idea 中集成 Maven

> file -> settings -> Build Tools -> Maven
>
> - 对当前项目的设置
>
> - **Maven home directory**: Maven 安装路径
>
> - **User settings file**: Maven 配置文件
>
> - **Local repository**: 读取 settings 文件内容，获取本地仓库的位置
>
> Maven -> Runner
>
> - VM Options: -DarchetypeCatalog=internal
>   - Maven 创建项目时，会从网络上下载 archetype-catalog.xml 作为项目的模板文件（网络不好时可以设置该参数取消下载）
> - JRE：JDK 信息
>
> file -> New Projects SetUp -> Settings for New Projects
>
> - 对新项目的设置

### 3.2 创建普通的 java 项目

> 使用模板创建项目：create from archetype -> maven-archetype-quickstart
>
> 添加 resources 目录：右键 -> Mark Directory as -> Resources Root

### 3.3 Idea 中执行 Maven 命令

> Maven 窗口 -> Lifecycle

3.4 Idea 导入 Maven 项目

> Project Structure -> Modules -> Add

---

## 4. 依赖管理

### 4.1 依赖范围

> 使用 scope 表示依赖的范围，指定该依赖在项目构建的哪个阶段起作用
>
> - compile: 默认，参与构建项目的所有阶段
> - test: 测试，只在测试阶段使用，不参与打包
> - provided: 提供者，项目在部署到服务器时不需要这个依赖，而是由服务器提供，例如 servlet 和 jsp，不参与打包
> - 

---

## 5.常用设置

### 5.1 Properties 的设置

```xml
<properties>
    <java.version>1.8</java.version> <!-- JAVA 版本 -->
    <project.buil.dsourceEncoding>UTF-8</project.buil.dsourceEncoding> <!-- 项目构建使用的编码 -->
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding> <!-- 生成报告的编码 -->
</properties>
```

### 5.2 全局变量

>   在 properties 中定义标签，这个标签就是一个变量，标签的文本就是变量的值，使用全局变量表示多个依赖使用的版本号
>
> ```xml
> <properties>
> 	<!-- 自定义变量 -->
>     <junit.version>5.2.5.RELEASE</junit.version>
> </properties>
> <dependencies>
> 	<dependency>
>     	<groupId>junit</groupId>
>         <artifactId>junit</artifactId>
>         <!-- ${} 取出全局变量的值 -->
>         <version>${junit.version}</version>
>         <scope>test</scope>
>     </dependency>
> </dependencies>
> ```

### 5.3 指定资源位置

> 处理配置文件的信息，Maven 默认处理配置文件方式：
>
> 1. 把 src/main/resources 目录下的文件拷贝在 target/classes 目录下
> 2. Maven 只处理 src/main/java 目录中的 java 文件，不会处理其他文件
>
> ==资源插件==
>
>   将 src/main/java 目录中的指定扩展名的文件拷贝到 target/classes 中
>
> ```xml
> <build>
> 	<resources>
>     	<resource>
>             <!-- 所在目录 -->
>             <directory>src/main/java</directory>
>             <includes>
>                 <!-- 扫描包括目录下的 properties, xml 文件 -->
>                 <include>**/*.properties</include>
>                 <include>**/*.xml</include>
>             </includes>
>             <!-- false时不启用过滤器 -->
>             <filtering>false</filtering>
>         </resource>
>     </resources>
> </build>
> ```

---

## 6. 多模块管理

### 第一种方式

>   pom 文件可以被子工程继承，Maven 多模块管理，其实就是让他的子模块的 pom 文件来继承父工程的 pom 文件，同时继承所有的依赖
>
> 1. 创建父工程（Maven 工程）
>
>    - packaging 标签文本内容必须设置为 pom
>    - 把 src 目录删除
>
> 2. 创建子工程
>
>    - 声明 parent 坐标
>
>      ```xml
>      <parent>
>          <groupId>org.springframework.boot</groupId>
>          <artifactId>spring-boot-starter-parent</artifactId>
>          <version>2.6.2</version>
>          <relativePath/> <!-- lookup parent from repository -->
>      </parent>
>      ```
>
> 3. 父工程加强管理子模块的所有依赖
>
>    - 把 dependency 放到 dependencyManagement 中，则子工程不会继承所有的父工程依赖
>
>    ```xml
>    <dependencyManagement>
>    	<dependency>
>        	...
>        </dependency>
>        <dependency>
>        	...
>        </dependency>
>        ...
>    </dependencyManagement>
>    ```
>
> 4. 在子工程中声明 dependency，不需要指定 version，会继承父工程的版本
>
>    - 若在子模块中指定依赖的版本号则不会继承父工程
>
>    ```xml
>    <dependencies>
>    	<dependency>
>            ...
>        </dependency>
>    </dependencies>
>    ```

### 第二种方式

> 1. 新建父工程
>
> 2. 在父工程中新建子工程（module），只会在父工程的 pom 文件中添加，不会在祖先工程中添加
>
>    ```xml
>    <modules>
>    	<module>child-project</module>
>    </modules>
>    ```
>
> 3. 子工程中没有 relativePath



