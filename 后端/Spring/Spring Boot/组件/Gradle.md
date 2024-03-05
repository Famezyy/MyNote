# Gradle

一款通用的构建工具，纯 Java 编写。面向开发者的脚本语言是 `Groovy` 和 `Kotlin`，即我们常用的 build.gradle 和 build.gradle.kts 或 plugin 等。

## 1.基础概念

### 1.1 Distribution

[官网](https://services.gradle.org/distributions/)

Gradle Distribution（安装包）目录主要由 `bin` 和 `lib` 构成，分别包含启动脚本和一些 JAR 包（lib）。主要有三种安装方式：

- `src`：源代码
- `bin`：可以运行，不包含文档、示例
- `all`：可以运行，包含文档、示例

一般会通过 **Wrapper** 的方式下载 Gradle。

### 1.2 Wrapper

#### 1.基本概念

Gradle Wrapper（Gradle 的包装器）是 Gradle 构建工具的一个特性，用于确保在项目中使用的 Gradle 版本是一致的，而不需要手动安装 Gradle。Gradle Wrapper 包含了一个用于下载和运行 Gradle 的脚本，这样就可以在项目的根目录中使用 Gradle Wrapper 而不必在系统上安装全局的 Gradle。

Gradle Wrapper的主要组成部分包括以下两个文件：

1. **gradlew（Unix）/ gradlew.bat（Windows）**：这是一个用于启动 Gradle 构建的脚本。Unix 系统使用`gradlew`，而 Windows 系统使用`gradlew.bat`。这个脚本可以下载指定版本的 Gradle，并在项目中使用它。
2. **gradle/wrapper 目录**：包含了一个配置文件 `gradle-wrapper.properties` 和一个 JAR 文件（例如，`gradle-wrapper.jar`）。`gradle-wrapper.properties` 文件用于指定 Gradle 的版本以及下载的 URL。

在使用 Gradle Wrapper 时，项目的开发者不需要手动下载或安装 Gradle。相反，他们可以使用项目中的 **Wrapper** 脚本，它会自动检查 Gradle 是否已安装，如果没有安装，则会下载并使用指定版本的 Gradle。这使得在不同的开发环境中更容易维护项目的一致性。

#### 2.使用

在 Gradle Wrapper 的目录中运行以下命令：

```bash
$ ./gradlew wrapper
```

### 1.3 GradleUserHome

默认情况下，Gradle 会在用户的家目录下创建一个名为 `.gradle` 的文件夹，用作 `GRADLE_USER_HOME`。在这个目录中，Gradle 存储了一些全局配置、插件、依赖库、下载的文件和其他与构建相关的数据。这有助于在多个项目之间共享某些配置，并且也能提高构建的性能，因为下载的依赖库可以在这里被缓存起来，不必每次构建都重新下载。

例如目录下有一个 `init.d` 目录，Gradle 在构建前先执行该目录下的脚本以完成一些初始化工作；还有个目录 `Wrapper`，使用 `gradlew wrapper` 时下载的安装包会放在这里。

### 1.4 Daemon

Maven在每次构建时会先启动一个 JVM 进程，往往需要花费一些时间，该进程构建完成后销毁。而 Gradle 3.0 开始默认使用 **Daemon** 模式，它是一个长时间运行的后台进程，可以加速构建过程。

Gradle Daemon 会在第一次构建时启动，并在后续的构建中保持运行，以便在需要时快速响应构建请求。它在构建时会启动一个轻量级的 Client JVM，Client 将查找并和后台的 Daemon JVM 通信，将参数发送给 Daemon，由 Daemon 处理后返回给 Client。Client 在构建结束后销毁，Daemon JVM 会一直存在，超过一定时间后会自动销毁。

Gradle Daemon 默认情况下是启用的，但你可以在 Gradle 的配置中进行调整。你可以在项目的 `gradle.properties` 文件中或者在用户主目录下的 `gradle.properties` 文件中添加以下配置来控制 Gradle Daemon 的行为：

```bash
# 启用或禁用Gradle Daemon
org.gradle.daemon=true

# 指定JVM的最大堆大小
org.gradle.jvmargs=-Xmx1024m
```

在上述例子中，`org.gradle.daemon=true` 表示启用 Gradle Daemon。也可以在执行 Gradle 时传入参数 `--no-daemon`。

可以使用以下命令手动停止 Gradle Daemon：

```bash
$ ./gradlew --stop
```

但在某些情况下 Gradle Daemon 的使用可能会导致问题，例如在使用某些插件或与特定的构建环境不兼容时。在这种情况下，你可以通过在构建时禁用 Daemon 来测试是否与 Daemon 相关。

### 1.5 构建流程

1. 执行 `./gradlew` 命令后会启动一个 Client JVM，它会查找当前机器上时候有安装对应版本的 Gradle
   - 如果未安装则先去下载 Gradle 到 GradleUserHome 中
2. 之后 Client 会查找对应版本相同的且和当前构建要求的参数兼容的 Daemon JVM 进程
   - 如果未找到则启动一个 Daemon
3. 将参数和环境变量发送给 Daemon，由 Daemon 处理后返回给 Client

## 2.DSL

### 2.1 DSL基础

DSL全称：Domain Specific Language，即领域特定语言，它是编程语言赋予开发者的一种特殊能力，通过它我们可以编写出一些看似脱离其原始语法结构的代码，从而构建出一种专有的语法结构。

DSL 分为两类，外部 DSL 和内部 DSL：

- 外部 DSL

  也称独立 DSL。因为它们是从零开始建立起来的独立语言，而不基于任何现有宿主语言的设施建立。外部 DSL 是从零开发的 DSL，在词法分析、解析技术、解释、编译、代码生成等方面拥有独立的设施。开发外部 DSL 近似于从零开始实现一种拥有独特语法和语义的全新语言。构建工具 make 、语法分析器生成工具 YACC、词法分析工具 LEX 等都是常见的外部 DSL。例如：正则表达式、XML、SQL、JSON、 Markdown 等。

- 内部 DSL

  也称内嵌式 DSL。因为它们的实现嵌入到宿主语言中，与之合为一体。内部 DSL 将一种现有编程语言作为宿主语言，基于其设施建立专门面向特定领域的各种语义。例如：Groovy DSL、Kotlin DSL 等；简而言之可以理解为，这种 DSL 简化了原有的语法结构（看起来有点像 lambda）。

#### 1.Gradle中的DSL

新建项目时 Gradle 默认的是 Groovy 语言，也即 `Groovy DSL`，比如 app>build.gradle：

```groovy
plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
}

android {
    namespace 'com.yechaoa.gradlex'
    compileSdk 32

    defaultConfig {
        applicationId "com.yechaoa.gradlex"
        minSdk 23
        targetSdk 32
        versionCode 1
        versionName "1.0"
    }
}

dependencies {
    implementation 'androidx.core:core-ktx:1.7.0'
}
```

- 第一段代码用于声明引入 Gradle 插件 plugin 的 DSL 代码，上面的 plugin 声明是简写版，完整版是这样的：

  ```groovy
  plugins {
      id 'com.android.application' version '7.3.0' apply false
  }
  ```

  - `id`：调用的是 `PluginDependenciesSpec` 中的 `id(String id)` 函数，返回 `PluginDependencySpec` 对象，`PluginDependencySpec` 对象可以理解为是 `PluginDependenciesSpec` 的一层封装
  - `version`：插件版本号。Gradle 默认插件是不需要指定版本号的，但是三方的插件则必须指定版本号
  - `apply`：是否将插件应用于当前项目及子项目，用于管理依赖传递，默认是 `true`

#### 2.还原DSL

再回到 `plugins` 这段 DSL 代码：

```groovy
plugins {
    id 'com.android.application'
}
```

我们可以理解 `plugins` 是一个函数，`plugins{}` 里面接收的是一个闭包 `Closure`，所以还原后的 DSL 是下面这样：

```groovy
plugins ({
    id 'com.android.application'
})
```

在 Groovy 中，当函数的最后一个参数或者只有一个参数是闭包时，是可以写在参数括号外面的：

```groovy
plugins () {
    id 'com.android.application'
}
```

在 Groovy 中，当函数只有一个参数的时候，是可以省略括号的，所以就回到了我们最初的 DSL 代码：

```groovy
plugins {
    id 'com.android.application'
}
```

#### 3.Kotlin写法

简写版：

```kotlin
plugins {
    // id 'com.android.application'
    id("com.android.application")
}
```

完整版：

```kotlin
plugins {
    // id 'com.android.application' version '7.3.0' apply false
    id("com.android.application") version "7.3.0" apply false
}
```

可以看出其实 `Kotlin DSL` 和 `Groovy DSL` 的写法差别不大，也是调用 `PluginDependenciesSpec` 中的 `id(String id)` 函数，只不过差别在调用 `id(String id)` 函数时有显式的括号而已。

#### 4.闭包

上面的还原 DSL 其实涉及到 Groovy 里面一个非常重要的概念，闭包 `Closure`。

**定义闭包**

```groovy
def myClosure = {}
println(myClosure)
```

`{}` 这个就是闭包，里面是空的什么都没做，所以打印出来什么也没有。这个闭包对象可以理解为是一个函数，函数是可以接收参数的，我们加个参数改造一下：

```groovy
def myClosure = {param -> param + 1}
println(myClosure(1))

// 打印 2
```

**Kotlin DSL**

我们再来看一下 Kotlin 的函数参数：

```kotlin
private fun setPrintln(doPrintln: (String) -> Unit) {
    doPrintln.invoke("yechaoa")
}

fun main() {
    setPrintln {param -> println(param)}
}

// 打印 yechaoa
```

定义了一个高阶函数 `setPrintln`，参数 `doPrintln` 是一个函数参数，`String` 表示函数参数 `doPrintln` 的参数类型，`Unit` 表示 `doPrintln` 不需要返回值。然后 `doPrintln` 执行并传参 "yechaoa"。

### 2.2 Groovy

Groovy 是 Apache 旗下的一种强大的、可选类型的和动态的语言，具有静态类型和静态编译功能，用于 Java 平台，旨在通过简洁、熟悉和易于学习的语法提高开发人员的生产力。它与任何 Java 程序顺利集成，并立即为您的应用程序提供强大的功能，包括脚本功能、领域特定语言创作、运行时和编译时元编程以及函数编程。并且，Groovy 可以与 Java 语言无缝衔接，可以在 Groovy 中直接写 Java 代码，也可以在 Java 中直接调用 Groovy 脚本，非常丝滑，学习成本很低。

#### 1. 注释

注释和 Java 一样，支持单行 `//`、多行 `/**/` 和文档注释 `/** */`。不同的是 Groovy 在脚本里的注释用 `#`。

#### 2.关键字

```groovy
as、assert、break、case、catch、class、const、continue、def、default、do、else、enum、extends、false、finally、for、goto、if、implements、import、in、instanceof、interface、new、null、package、return、super、switch、this、throw、throws、trait、true、try、while、as、in、permitsrecord、sealed、trait、var、yields、null、true、false、boolean、char、byte、short、int、long、float、double
```

#### 3.字符串

在 Groovy 种有两种字符串类型，普通字符串 `java.lang.String` 和插值字符串 `groovy.lang.GString`。

- 普通字符串：`println 'yechaoa'`

- 插值字符串：

  ```groovy
  def name = 'yechaoa'
  println "hello ${name}"
  // or
  println "hello $name"
  ```

#### 4.数字

与 java 类型相同：

- `byte`：8 位
- `char`：16 位，可用作数字类型，表示 UTF-16
- `short`：16 位
- `int`：32 位
- `long`：64 位
- `java.math.BigInteger`：2147483647 个 int

可以这么声明：

```groovy
byte  b = 1
char  c = 2
short s = 3
int   i = 4
long  l = 5
BigInteger bi =  6
```

类型会根据数字的大小调整：

```groovy
def a = 1
assert a instanceof Integer

// Integer.MAX_VALUE
def b = 2147483647
assert b instanceof Integer

// Integer.MAX_VALUE + 1
def c = 2147483648
assert c instanceof Long

// Long.MAX_VALUE
def d = 9223372036854775807
assert d instanceof Long

// Long.MAX_VALUE + 1
def e = 9223372036854775808
assert e instanceof BigInteger
```

#### 5.变量

Groovy 提供了 `def` 关键字，跟 Kotlin 的 `var` 一样，属于类型推导。

```groovy
def a = 123
def b = 'b'
def c = true 
boolean d = false
int e = 123
```

但 Groovy 其实是强类型语言，但是与 Kotlin 一样，也支持类型推导。下面这几种定义也都是 ok 的：

```groovy
def a1 = "yechaoa"
String a2 = "yechaoa"
a3 = "yechaoa"
```

#### 6.List

用大括号表示列表，参数用逗号分割。

```groovy
def numbers = [1, 2, 3]         

assert numbers instanceof List  
assert numbers.size() == 3
```

除了可以定义同类型之外，也可以定义不同类型值的列表

```groovy
def heterogeneous = [1, "a", true] 
```

迭代通过调用 `each` 和 `eachWithIndex` 方法，与 Kotlin 类似

```groovy
[1, 2, 3].each {
    println "Item: $it"//it 是对应于当前元素的隐式参数
}
['a', 'b', 'c'].eachWithIndex { it, i -> //it 是当前元素, i 是索引位置
    println "$i: $it"
}
```

#### 6.Arrays

跟 `List` 差不多，不一样的是需要显示声明类型

```groovy
String[] arrStr = ['Ananas', 'Banana', 'Kiwi']  

assert arrStr instanceof String[]    
assert !(arrStr instanceof List)

def numArr = [1, 2, 3] as int[]      

assert numArr instanceof int[]       
assert numArr.size() == 3
```

#### 8.Map

key-value 映射，每一对键值用冒号 `:` 表示，对与对之间用逗号分割，最外层是中括号。

```groovy
def colors = [red: '#FF0000', green: '#00FF00', blue: '#0000FF']   

assert colors['red'] == '#FF0000'    
assert colors.green  == '#00FF00'    

colors['pink'] = '#FF00FF'           
colors.yellow  = '#FFFF00'           

assert colors.pink == '#FF00FF'
assert colors['yellow'] == '#FFFF00'

assert colors instanceof java.util.LinkedHashMap
```

#### 9.闭包

Groovy 提供了闭包的支持，语法和 Lambda 表达式有些类似，简单来说就是一段可执行的代码块或函数指针。闭包在 Groovy 中是 `groovy.lang.Closure` 类的实例，这使得闭包可以赋值给变量，或者作为参数传递。Groovy 定义闭包的语法很简单：

```groovy
{ [closureParameters -> ] statements }
```

尽管闭包是代码块，但它可以作为任何其他变量分配给变量或字段：

```groovy
def listener = {
    e -> println "Clicked on $e.source" 
}
assert listener instanceof Closure

Closure callback = {
    println 'Done!'
}

Closure<Boolean> isTextFile = {
    File it -> it.name.endsWith('.txt')                     
}
```

闭包可以访问外部变量，而函数则不能。

```groovy
def str = 'yechaoa'
def closure={
    println str
}
closure() // yechaoa 
```

闭包调用的方式有两种，`闭包.call(参数)` 或者 `闭包(参数)`，在调用的时候可以省略圆括号。

```groovy
def closure = {
    param -> println param
}
 
closure('yechaoa')
closure.call('yechaoa')
closure 'yechaoa'
```

闭包的参数是可选的，如果没有参数的话可以省略 `->` 操作符。

```groovy
def closure = {println 'yechaoa'}
closure()
```

多个参数以逗号分隔，参数类型和函数一样可以显式声明也可省略。

```groovy
def closure = {
    String x, int y ->                                
    println "hey ${x} the value is ${y}"
}

```

如果只有一个参数的话，也可省略参数的定义，Groovy 提供了一个隐式的参数 `it` 来替代它。类似 Kotlin。

```groovy
def closure = { it -> println it } 
//和上面是等价的
def closure = { println it }   
closure('yechaoa')
```

闭包可以作为参数传入，闭包作为函数的唯一参数或最后一个参数时可省略括号。

```groovy
def eachLine(lines, closure) {
    for (String line : lines) {
        closure(line)
    }
}

eachLine('a'..'z',{ println it }) 
//可省略括号，与上面等价
eachLine('a'..'z') { println it }
```

#### 10.IO操作

读取文本文件并打印每一行文本

```groovy
new File(baseDir, 'yechaoa.txt').eachLine{ line ->
    println line
}

new File(baseDir,'yechaoa.txt').withInputStream { stream ->
    // do something ...
}
```

写文件

```groovy
new File(baseDir,'yechaoa.txt').withWriter('utf-8') { writer ->
    writer.writeLine 'Into the ancient pond'
    writer.writeLine 'A frog jumps'
    writer.writeLine 'Water’s sound!'
}

new File(baseDir,'data.bin').withOutputStream { stream ->
    // do something ...
}
```

## 3.构建

`build.gradle` 是构建的主要文件。

### 3.1 生命周期

Gradle 的生命周期可以被分为以下几个主要阶段：

1. **Initialization**

   - Detects the `settings.gradle(.kts)` file

   - Creates a [`Settings`](https://docs.gradle.org/current/dsl/org.gradle.api.initialization.Settings.html) instance

   - Evaluates the settings file to determine which projects (and included builds) make up the build

   - Creates a [`Project`](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html) instance for every project

2.  **Configuration**

   - Evaluates the build scripts, `build.gradle(.kts)`, of every project participating in the build（会实际执行 `gradle` 代码，例如 `println()`）

   - Creates a task graph for requested tasks

3. **Execution**

   - Schedules and executes the selected tasks

   - Dependencies between tasks determine execution order

   - Execution of tasks can occur in parallel

### 3.2 task

Gradle 中执行的最小单元是 `task`，对于下面这条语句：

```groovy
task('helloworld', {
    println('configure')

    doLast ({
        println('Executing task')
    })
})
```

1. 在执行 `gradlew helloworld` 时，首先创建了一个名为 'helloworld' 的 `task`
2. 执行传递的闭包函数配置该 `task`：会打印 'configure'，并将 `doLast` 的闭包函数添加到 `task` 执行列表的最后但并不执行，只有在执行了该任务时才会执行（执行`./gradlew helloworld` 时）

```bash
$ ./gradlew helloworld

> Configure project :
configure

> Task :helloworld
Executing task

BUILD SUCCESSFUL in 1s
1 actionable task: 1 executed
```

上面的代码可以进一步优化为：

```groovy
task('helloworld') {
    println('configure')

    doLast {
        println('Executing task')
    }
}
```

#### 1.dependsOn

当某个项目需要依赖其他项目时可以使用 `dependsOn`，例如下面的示例：

```groovy
task('first') {
    doLast { println("I'm base task") }
}

(0..<10).each { i ->
    task('task' + i) {
        // 当 i 为偶数时依赖于 first task
        if (i % 2 == 0) {
            dependsOn('first')
        }
    }
}
```

当执行 `./gradlew task7` 时不会打印 `I'm base task`，但是当执行 `./gradlew task8` 时会打印。

### 3.3 Project

上面介绍的 `task` 其实就是 `Project` 的一个方法，其他还有诸如以下的方法：

```groovy
// 项目 evaluate 执行完成后的钩子函数
void afterEvaluate(Action<? super Project> action);
void afterEvaluate(Closure closure);

// 对项目所有的 Project 实例执行该函数
void allprojects(Action<? super Project> action);
void allprojects(Closure configureClosure);
```

### 3.4 导入依赖案例

当想要导入 `StringUtils` 时需要编写下面的代码：

```groovy
apply plugin: 'java'

repositories {
    mavenCentral()
}

dependencies {
    implementation group: 'org.apache.commons', name: 'commons-lang3', version: '3.14.0'
}
```

## 4.插件

### 4.1 导入插件/依赖

编写插件时要实现 `Plugin` 接口：

```groovy
task('first') {
    doLast { println("I'm base task") }
}

class MyPlugin implements Plugin<Project> {
    @Override
    void apply(Project target) {
        (0..<10).each { i ->
            target.task('task' + i) {
                if (i % 2 == 0) {
                    dependsOn('first')
                }
            }
        }
    }
}

apply plugin: MyPlugin
```

`apply plugin: MyPlugin` 可以还原为 `apply ([plugin: MyPlugin])`，`[plugin: MyPlugin]` 是一个 `Map` 类型的参数，支持传入一个 `http` 的 URL 来实现加载远程目录里的 `Plugin`。

此外，还可以创建 `buildSrc/src/main/java` 目录并在其中创建一个 `MyPlugin.java` 的类来封装代码，然后在配置文件中直接使用 `apply plugin` 导入。

```bash
├─.gradle
├─build
├─buildSrc
│  ├─.gradle
│  └─src
│      └─main
│          └─java
├─gradle
│  └─wrapper
└─src
    ├─main
    └─test
```

```java
class MyPlugin implements Plugin<Project> {
    @Override
    public void apply(Project target) {
        for (int i = 0; i < 10; i++){
            target.task("task" + i);
        }
    }
}
```

```groovy
apply plugin: MyPlugin
```

此时就会执行 ``MyPlugin` 的 `apply` 方法，就会创建 `task0~task9` 的任务。

### 4.2 在配置文件中使用依赖

在 Java compile 时需要导入的依赖可以使用 `implementation`，如果想在构建时使用相关的依赖则需要使用 `buildscript`：

```groovy
import org.apache.commons.lang3.StringUtils;

apply plugin: 'java'

buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath group: 'org.apache.commons', name: 'commons-lang3', version: '3.14.0'
    }
}

StringUtils.isNotEmpty("3");
```

