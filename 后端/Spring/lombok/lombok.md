# Lombok

## 引入依赖

```xml
<dependency>
	<groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
</dependency>
```

> eclipse 需要运行 lombok.jar 进行安装，选择 eclipse 目录。

## 功能 

Lombok 是做什么呢？其实很简单，一个最简单的例子就是能够通过添加注解自动生成一些方法，使我们代码更加简洁易懂。例如下面一个类。

```java
@Data
public class TestLombok {
    private String name;
    private Integer age;

    public static void main(String[] args) {
        TestLombok testLombok = new TestLombok();
        testLombok.setAge(12);
        testLombok.setName("zs");
    }
}
```

我们使用 Lombok 提供的`Data`注解，在没有写`get、set`方法的时候也能够使用其`get、set`方法。我们看它编译过后的`class`文件，可以看到它给我们自动生成了`get、set`方法。

当然 Lombok 的功能不止如此，还有很多其他的注解帮助我们简便开发，网上有许多的关于 Lombok 的使用方法，这里就不再啰嗦了。正常情况下我们在项目中自定义注解，或者使用`Spring`框架中`@Controller、@Service`等等这类注解都是`运行时注解`，运行时注解大部分都是通过反射来实现的。而`Lombok`是使用编译时注解实现的。那么编译时注解是什么呢？

## 编译时注解

> 注解（也被成为元数据）为我们在代码中添加信息提供了一种形式化的方法，使我们可以在稍后某个时刻非常方便地使用这些数据。 ——————摘自《Thinking in Java》

Java 中的注解分为**运行时注解**和**编译时注解**，运行时注解就是我们经常使用的在程序运行时通过反射得到我们注解的信息，然后再做一些操作。而编译时注解是什么呢？就是在程序在编译期间通过注解处理器进行处理。

- 编译期：Java 语言的编译期是一段不确定的操作过程，因为它可能是将`*.java`文件转化成`*.class`文件的过程；也可能是指将字节码转变成机器码的过程；还可能是直接将`*.java`编译成本地机器代码的过程
- 运行期：从 JVM 加载字节码文件到内存中，到最后使用完毕以后卸载的过程都属于运行期的范畴

## 注解处理工具apt

> 注解处理工具 apt(Annotation Processing Tool)，这是 Sun 为了帮助注解的处理过程而提供的工具，apt 被设计为操作 Java 源文件，而不是编译后的类。

它是 javac 的一个工具，中文意思为编译时注解处理器。APT 可以用来在编译时扫描和处理注解。通过 APT 可以获取到注解和被注解对象的相关信息，在拿到这些信息后我们可以根据需求来自动的生成一些代码，省去了手动编写。注意，获取注解及生成代码都是在代码**编译**时候完成的，相比反射在运行时处理注解大大提高了程序性能。APT 的核心是`AbstractProcessor`类。

正常情况下使用 APT 工具只是能够生成一些文件（**不仅仅是我们想象的class文件，还包括xml文件等等之类的**），并不能修改原有的文件信息。

但是此时估计会有疑问，那么`Lombok`不就是在我们原有的文件中新增了一些信息吗？我在后面会有详细的解释，这里简单介绍一下，其实`Lombok`是修改了Java中的**抽象语法树`AST`**才做到了修改其原有类的信息。

接下来我们演示一下如何用`APT`工具生成一个 class 文件，然后我们再说`Lombok`是如何修改已存在的类中的属性的。

### 定义注解 

首先当然我们需要定义自己的注解了

```java
@Retention(RetentionPolicy.SOURCE) // 注解只在源码中保留
@Target(ElementType.TYPE) // 用于修饰类
public @interface MyGetter {
    String value() default "";
}
```

`Retention`注解上面有一个属性value，它是`RetentionPolicy`类型的枚举类，`RetentionPolicy`枚举类中有三个值。

```java
public enum RetentionPolicy {
    SOURCE,
    CLASS,
    RUNTIME
}
```

- `SOURCE`修饰的注解：修饰的注解,表示注解的信息会被编译器抛弃，不会留在 class 文件中，注解的信息只会留在源文件中
- `CLASS`修饰的注解：表示注解的信息被保留在 class 文件（字节码文件）中当程序编译时，但不会被虚拟机读取在运行的时候
- `RUNTIME`修饰的注解：表示注解的信息被保留在 class 文件（字节码文件）中当程序编译时，会被虚拟机保留在运行时。**所以它能够通过反射调用，所以正常运行时注解都是使用的这个参数**

`Target`注解上面也有个属性 value，它是`ElementType`类型的枚举。是用来修饰此注解作用在哪的。

```java
public enum ElementType { 
    TYPE, 
    FIELD, 
    METHOD, 
    PARAMETER,
    CONSTRUCTOR,
    LOCAL_VARIABLE,
    ANNOTATION_TYPE,
    PACKAGE,
    TYPE_PARAMETER,
    TYPE_USE21
}
```

### 定义注解处理器

我们要定义注解处理器的话，那么就需要继承`AbstractProcessor`类。继承完以后基本的框架类型如下

```java
@SupportedSourceVersion(SourceVersion.RELEASE_8) 
@SupportedAnnotationTypes("aboutjava.annotion.MyGetter") 
public class MyGetterProcessor extends AbstractProcessor { 
    @Override 
    public synchronized void init(ProcessingEnvironment processingEnv) { 
        super.init(processingEnv); 
    } 
    @Override    
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        return true;
    }
}
```

我们可以看到在子类中上面有两个注解，注解描述如下

- `@SupportedSourceVersion`：表示所支持的 Java 版本
- `@SupportedAnnotationTypes`：表示该处理器要处理的注解

继承了父类的两个方法，方法描述如下

- init 方法：主要是获得编译时期的一些环境信息
- process 方法：在编译时，编译器执行的方法。也就是我们写具体逻辑的地方

我们是演示一下如何通过继承`AbstractProcessor`类来实现在编译时生成类，所以我们在`process`方法中书写我们生成类的代码。如下所示。

```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    StringBuilder builder = new StringBuilder()
        .append("package aboutjava.annotion;\n\n")
        .append("public class GeneratedClass {\n\n") // open class
        .append("\tpublic String getMessage() {\n") // open method
        .append("\t\treturn \"");
    // for each javax.lang.model.element.Element annotated with the CustomAnnotation
    for (Element element : roundEnv.getElementsAnnotatedWith(MyGetter.class)) {
        String objectType = element.getSimpleName().toString();
        // this is appending to the return statement
        builder.append(objectType).append(" says hello!\\n");
    }
    builder.append("\";\n") // end return
        .append("\t}\n") // close method
        .append("}\n"); // close class
    try { // write the file
        JavaFileObject source = processingEnv.getFiler().createSourceFile("aboutjava.annotion.GeneratedClass");
        Writer writer = source.openWriter();
        writer.write(builder.toString());
        writer.flush();
        writer.close();
    } catch (IOException e) {
        // Note: calling e.printStackTrace() will print IO errors
        // that occur from the file already existing after its first run, this is normal
    }
    return true;
}
```

### 定义使用注解的类（测试类） 

上面的两个类就是基本的工具类了，一个是定义了注解，一个是定义了注解处理器，接下来我们来定义一个测试类（TestAno.java）。我们在类上面加上我们自定的注解类。

```java
@MyGetter
public class TestAno {
    public static void main(String[] args) {
        System.out.printf("1");
    }
}
```

这样我们在编译期就能生成文件了，接下来演示一下在编译时生成文件，此时不要着急直接进行 javac 编译，`MyGetter`类是注解类没错，而`MyGetterProcessor`是注解类的处理器，那么我们在编译`TestAno` Java 文件的时候就会触发处理器。因此这两个类是无法一起编译的。

先给大家看一下我的目录结构

```
aboutjava    
	-- annotion        
		-- MyGetter.java
		-- MyGetterProcessor.java
		-- TestAno.java
```

所以我们先将注解类和注解处理器类进行编译

```bash
javac aboutjava/annotion/MyGett*
```

接下来进行编译我们的测试类，此时在编译时需要加上`processor`参数，用来指定相关的注解处理类。

```bash
javac -processor aboutjava.annotion.MyGetterProcessor aboutjava/annotion/TestAno.java
```

可以看到自动生成了 Java 文件。

## Javac原理

既然我们是在编译期对类进行操作了，那么我们就需要了解在Java中Javac到底对程序做了什么。Javac对代码编译的过程其实就是用Java来写的，我们可以查看其源码对其简单的分析，如何下载源码，Debug源码这里我就不进行分析了，推荐一篇文章写的挺好的。[Javac 源码调试教程](https://juejin.cn/post/6844903882166894605)。

编译过程大致分为了三个阶段

- 解析与填充符号表
- 注解处理
- 分析与字节码生成

这三个阶段的交互过程如下图所示。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220515182242645-777ec3bfc5745d22f58993a528af6eba-b05d2b.png" alt="image-20220515182242645" style="zoom: 67%;" />

### 解析与填充符号表

这一步骤是两个步骤，包括了解析和填充符号，其中解析是分为**词法分析**和**语法分析**两个步骤。

#### 词法分析和语法分析

词法分析就是将源代码的字符流转变为　Java　中的标记（Token）集合，单个字符是程序编写过程中最小的元素，而标记（Token）则是编译过程中最小的元素，关键字、变量名、字面量、运算符都可以成为标记（Token）。比如在 Java 中`int a = b+2`，这段代码则表示了 6 个标记`Token`，分别是`int、a、=、b、+、2`。虽然关键字 int 是由三个字符构成的，但是它只是一个 Token，不可以再拆分了。

语法分析是根据 Token 序列构造抽象对象树的过程，抽象语法树（Abstract syntax tree），是一种用来描述代码语法结构的树形表示方法，语法树的每一个节点都代表着程序代码中的一个语法结构，例如包、类型、修饰符、运算符、接口、返回值甚至是代码注释都是一个语法结构。

语法分析分析出来的树结构是由`JCTree`来表示的，我们可以看一下它的子类有哪些。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220515182405857-5b9e4f99134270bd753c6ef0276c830a-509118.png" alt="image-20220515182405857" style="zoom:50%;" />

我们自己建一个类，可以观察它在编译过程中用树结构表示是一种怎样的结构。

```java
public class HelloJvm {

    private String a;
    private String b;

    public static void main(String[] args) {
        int c = 1+2;
        System.out.println(c);
        print();
    }

    private static void print(){

    }
}
```

大家注意我划红线的地方，可以看到这些都是 JCTree 的子类。我们可以知道编译期的树是以`JCCompilationUnit`为根节点，然后作为类的构成元素例如方法、私有变量、class 类，这些都是作为树的构成一种。

![image-20220515182513057](https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220515182513057-7ce3633582d3babbd1ebf832a050ed93-587b18.png)

#### 填充符号表

> 填充符号表和我们的 Lombok 原理关联不大，这里了解即可。

完成了语法分析和词法分析以后，下一步就是填充符号表的过程，符号表是由一组符号地址和符号信息构成的表格，可以将它想象成哈希表中的 K-V 值对的形式（符号表不一定是哈希表实现，可以使有序符号表，树状符号表、栈结构符号表等）。符号表中所登记的信息在编译的不同阶段都要用到，在语义分析中，符号表所登记的内容将用于语义检查（如检查一个名字的使用和原先的说明是否一致）和产生中间代码。在目标代码生成阶段，当对符号名进行地址分配时，符号表是地址分配的依据。

### 注解处理器 

第一步的解析和填充符号表完成以后，接下来就是我们的重头戏注解处理器了。因为在这一步就是 Lombok 实现原理的关键。

在 JDK1.5 之后，Java 语言提供了对注解的支持，这些注解与普通的 Java 代码一样，是在运行期间发挥作用的。在 JDK1.6 中实现了对 JSR-269 的规范，提供了一组插入式注解处理器的标准 API 在编译期间对注解进行处理，我们可以把它看作是一组编译器的插件，在这些插件里面，可以读取，修改，添加抽象语法树中的任意元素。

如果这些插件在处理注解期间对语法树进行了修改，那么编译器将回到解析及填充符号表的过程重新处理，直到所有的插入式注解处理器都没有了再对语法树进行修改为止。每一次循环成为一个 Round。

有了编译器注解处理的标准 API 后，我们的代码才有可能干涉编译器的行为，由于语法树中的任意元素，甚至包括代码注释都可以在插件之中访问到，所以通过插入式注解处理器实现的插件在功能上有很大的发挥空间。只要有足够多的创意，程序员可以使用插入式注解处理器来实现许多原本只能在编码中完成的事情。

### 语义分析与字节码生成 

语法分析之后，编译器获得了程序代码的抽象语法树表示，语法树能表示一个结构正确的源程序的抽象，但是无法保证源程序是符合逻辑的。而语义分析的主要任务就是对结构上正确的源程序进行上下文有关性质的审查，如进行类型检查。

比如我们有以下代码

```java
int a = 1;
boolean b = false;
char c = 2;
```

下面我们有可能出现如下运算

```java
int d = b+c;
```

其实上面的代码在结构上能构成准确的语法树，但是在语义上下面的运算是错误的。所以如果运行的话就会出现编译不通过，无法编译。

## 自己实现一个简单的Lombok

上面我们了解了 javac 的过程，那么我们直接来自己写一个简单的在已有类中添加代码的小工具，我们就只生成 set 方法。首先写一个自定义的注解类。

```java
@Retention(RetentionPolicy.SOURCE) // 注解只在源码中保留
@Target(ElementType.TYPE) // 用于修饰类
public @interface MySetter {
}
```

然后写对于此注解类的注解处理器类

```java
@SupportedSourceVersion(SourceVersion.RELEASE_8) 
@SupportedAnnotationTypes("aboutjava.annotion.MySetter") 
public class MySetterProcessor extends AbstractProcessor { 
    private Messager messager; 
    private JavacTrees javacTrees; 
    private TreeMaker treeMaker; 
    private Names names; 
    /**
    * @Description: 1. Message 主要是用来在编译时期打log用的
    *               2. JavacTrees 提供了待处理的抽象语法树
    *               3. TreeMaker 封装了创建AST节点的一些方法
    *               4. Names 提供了创建标识符的方法
    */
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        this.messager = processingEnv.getMessager();
        this.javacTrees = JavacTrees.instance(processingEnv);
        Context context = ((JavacProcessingEnvironment)processingEnv).getContext();
        this.treeMaker = TreeMaker.instance(context);
        this.names = Names.instance(context);
    }
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        return false;
    }
}
```

此处我们注意我们在 init 方法中获得一些编译阶段的一些环境信息。我们从环境中提取出一些关键的类，描述如下。

- `JavacTrees`：提供了待处理的抽象语法树
- `TreeMaker`：封装了操作 AST 抽象语法树的一些方法
- `Names`：提供了创建标识符的方法
- `Messager`：主要是在编译器打日志用的

然后接下来我们利用所提供的工具类对已存在的 AST 抽象语法树进行修改。主要的修改逻辑存在于`process`方法中，如果返回是 true 的话，那么 javac 过程会再次重新从解析与填充符号表处开始进行。`process`方法的逻辑主要如下

```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    Set<? extends Element> elementsAnnotatedWith = roundEnv.getElementsAnnotatedWith(MySetter.class);
    elementsAnnotatedWith.forEach(e->{
        JCTree tree = javacTrees.getTree(e);
        tree.accept(new TreeTranslator(){
            @Override
            public void visitClassDef(JCTree.JCClassDecl jcClassDecl) {
                List<JCTree.JCVariableDecl> jcVariableDeclList = List.nil();
                // 在抽象树中找出所有的变量
                for (JCTree jcTree : jcClassDecl.defs){
                    if (jcTree.getKind().equals(Tree.Kind.VARIABLE)){
                        JCTree.JCVariableDecl jcVariableDecl = (JCTree.JCVariableDecl) jcTree;
                        jcVariableDeclList = jcVariableDeclList.append(jcVariableDecl);
                    }
                }
                // 对于变量进行生成方法的操作
                jcVariableDeclList.forEach(jcVariableDecl -> {
                    messager.printMessage(Diagnostic.Kind.NOTE, jcVariableDecl.getName()+"has been processed");
                    jcClassDecl.defs = jcClassDecl.defs.prepend(makeSetterMethodDecl(jcVariableDecl));
                });
                super.visitClassDef(jcClassDecl);
            }
        });
    });
    return true;
}
```

其实看起来比较难，原理比较简单，主要是我们对于 API 的不熟悉所以看起来不好懂，但是主要意思就是如下

1. 找到`@MySetter`注解所标注的类，获得其语法树
2. 遍历其语法树，找到其参数节点
3. 自己建一个方法节点，并添加到语法树中

用图表示的话，我们建了一个测试类`TestMySetter`，我们知道其语法树的大致结构如下图所示。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220515183445784-5c1314c61ce4acbe23c1a3b8d907374f-ac359d.png" alt="image-20220515183445784" style="zoom:80%;" />

那么我们的目标就是将其语法树变成下图所示，因为最终生成字节码是根据语法树来生成的，所以我们在语法树中添加了方法的节点，那么在生成字节码的时候就会生成对应方法的字节码。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20220515183500785-8ff8591b3abbf04fabeffb6a8986655b-dd5eb6.png" alt="image-20220515183500785" style="zoom: 50%;" />

其中生成方法节点的代码如下

```java
private JCTree.JCMethodDecl makeSetterMethodDecl(JCTree.JCVariableDecl jcVariableDecl){

    ListBuffer<JCTree.JCStatement> statements = new ListBuffer<>();
    // 生成表达式 例如 this.a = a;
    JCTree.JCExpressionStatement aThis = makeAssignment(treeMaker.Select(treeMaker.Ident(names.fromString("this")), jcVariableDecl.getName()), treeMaker.Ident(jcVariableDecl.getName()));
    statements.append(aThis);
    JCTree.JCBlock block = treeMaker.Block(0, statements.toList());

    // 生成入参
    JCTree.JCVariableDecl param = treeMaker.VarDef(treeMaker.Modifiers(Flags.PARAMETER), jcVariableDecl.getName(), jcVariableDecl.vartype, null);
    List<JCTree.JCVariableDecl> parameters = List.of(param);

    // 生成返回对象
    JCTree.JCExpression methodType = treeMaker.Type(new Type.JCVoidType());
    
    return treeMaker.MethodDef(treeMaker.Modifiers(Flags.PUBLIC), getNewMethodName(jcVariableDecl.getName()), methodType, List.nil(), parameters,List.nil(), block, null);
}

private Name getNewMethodName(Name name){
    String s = name.toString();
    return names.fromString("set"+s.substring(0,1).toUpperCase()+s.substring(1,name.length()));
}

private JCTree.JCExpressionStatement makeAssignment(JCTree.JCExpression lhs, JCTree.JCExpression rhs) {
    return treeMaker.Exec(
        treeMaker.Assign(
            lhs,
            rhs
        )
    );
}
```

最后我们执行下面三个命令

```bash
javac -cp $JAVA_HOME/lib/tools.jar aboutjava/annotion/MySetter* -d
javac -processor aboutjava.annotion.MySetterProcessor aboutjava/annotion//TestMySetter.java
javap -p aboutjava/annotion/TestMySetter.class
```

可以看到输出的内容如下

```java
Compiled from "TestMySetter.java"
public class aboutjava.annotion.TestMySetter {
  private java.lang.String name;
  public void setName(java.lang.String);
  public aboutjava.annotion.TestMySetter();
}
```

可以看到字节码中已经生成了我们需要的`setName`方法。

