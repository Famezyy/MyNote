# 第1章_JavaScript基础

## 1.简介

- 诞生于 1995 年，用于处理网页的前端验证，即检查用户输入的内容是否符合要求
- 由网景公司发明，起初命名为 LiveScript，后来由 SUN 公司介入更名为 JavaScript
- ECMAScript 是 JavaScript 的标准，不同浏览器对该标准实现有所不同，Chrome 使用 V8 引擎
- 一个 JavaScript 由三个部分组成：ECMAScript，DOM，BOM
- JS 的特点
  - 解释型语言，从上往下一行一行执行
  - 类似于 C 和 JAVA 的语法结构
  - 动态语言
  - 基于原型的面向对象

常用函数：

```js
console.log("msg");
console.time("timerName");
console.timeEnd("timerName");
document.write("msg");
```

### 1.1 JS的编写位置

1. 可以编写到标签的指定属性中（耦合性太高）

   ```html
   <button onclick="alert('hello');">我是按钮</button>  
   <a href="javascript:alert('aaa');">超链接</a>
   ```

2. 可以编写到 script 标签中

   ```html
   <script type="text/javascript">  
       /* 弹出窗口 */
       alert("要输出的内容");
       /* 向 body 中写入内容 */
   	document.write("test");
       /* 向控制台输出内容 */
       console.log("test");
   </script>
   ```

3. 可以将代码编写到外部的 js 文件中，然后通过标签将其引入；script 标签一旦用于引入外部文件了，就不能在编写代码了，即使编写了浏览器也会忽略，如果需要则可以在创建一个新的 script 标签用于编写内部代码

```html
<script type="text/javascript" src="文件路径">【这里不能写代码】</script>
<script>
	alert("new tag");
</script>
```

### 1.2 基本的语法

**分号**

JS 函数声明不需要分号`；`，但是赋值语句要加分号

```js
function functionName(arg0,arg1,arg2) {  
// 函数声明  
}  
var functionName = function(arg0,arg1,arg2) {  
// 函数表达式  
};
```

**注释**

1. 单行注释

   ```js
   // 注释内容
   ```

2. 多行注释

   ```js
   /*  
     注释内容  
   */  
   ```

**其他规则**

- JS 严格区分大小写
- JS 中每条语句以分号结尾，如果不写分号，浏览器会自动添加，但是会消耗一些系统资源，而且有些时候浏览器会加错分号，所以在开发中分号**必须写**

- JS 中会自动忽略多个空格和换行，所以我们可以利用空格和换行对代码进行格式化

### 1.3 字面量和变量

**字面量**

字面量实际上就是一些固定的值，比如 1 2 3 4 true false null NaN "hello"，**字面量都是不可以改变的。**

**变量**

变量可以用来保存字面量，并且可以保存任意的字面量。一般都是通过变量来使用字面量，而不直接使用字面量，而且也可以通过变量名来对字面量的含义进行描述。

**声明变量**

使用`var`关键字来声明一个变量：

```js
var a;  
```

为变量赋值（不赋值则是`undefined`）：

```
a = 1; 
```

声明和赋值同时进行：

```js
var a = 1;   
```

**标识符**

在 JS 中所有的可以自主命名的内容，都可以认为是一个标识符，是标识符就应该遵守标识符的规范，比如：变量名、函数名、属性名。

规范：

- 标识符中可以含有字母、数字、`_`、`$`
- 标识符不能以数字开头
- 标识符不能是 JS 中的关键字和保留字，例如：`var`、`else`、`class`
- 标识符一般采用驼峰命名法：xxxYyyZzz

### 1.4 数据类型

#### 1.六种数据类型

JS 中一共分成六种数据类型 5 个基本数据类型 + object

- String：字符串
- Number：数值
- Boolean：布尔值
- Null：空值
- Undefined：未定义
- Object：对象

用`typeof`运算符可以检查数据类型。

1. **String 字符串**

   JS 中的字符串需要使用引号引起来，双引号或单引号都行。

   ```js
   var str;
   str = "hello";
   str = 'hello';
   str = "hell'o'";
   str = 'hell"o"';
   str = ""hello""; ×
   str = ''hello''; ×
   ```

   在字符串中使用`\`作为转义字符：

   ```bash
   \'  ==> '
   \"  ==> "
   \n  ==> 换行  
   \t  ==> 制表符  
   \\  ==> \
   \u  ==> 转义 unicode
   ```

   使用`typeof`运算符检查字符串时，会返回"string"。

   > **扩展**
   >
   > 在网页中使用 unicode 编码时使用`&#`加上十进制的数字。例如`\u2620`是骷髅头，十进制是`9760`：
   >
   > ```html
   > <h1>
   >     &#9760;
   > </h1>
   > ```

2. **Number 数值**

   JS 中所有的整数和浮点数都是 Number 类型。

   能表示的最大值：`Number.MAX_VALUE = 1.7976931348623157e+308`。如果表示的数字超过最大值，则返回`Infinity`表示无穷大；

   但是`Number.MIN_VALUE`表示的是最小的正值，是一个小数，`Number.MIN_VALUE=5e-324`。

   一些特殊的数字，能赋值给变量：

   - `Infinity`：正无穷

   - `-Infinity`：负无穷

   - `NaN`：非法数字（Not A Number），如果使用字符串运算则会返回 NaN，例如`a = "abc" * "bcd"`

   其他进制数字的表示方式：

   - 以`0b`开头表示二进制，但是不是所有的浏览器都支持

   - 以`0`开头表示八进制

   - 以`0x`开头表示十六进制

   使用`typeof`检查一个 Number 类型的数据时会返回"number"（包括`NaN`和`Infinity`）。

   > **注意**
   >
   > JS 进行浮点数计算式可能会得到不精确的结果，不要使用 JS 进行精确度要求高的计算。

3. **Boolean 布尔值**

   布尔值主要用来进行逻辑判断，布尔值只有两个：

   - `true`：逻辑的真
   - `false`：逻辑的假

   使用`typeof`检查一个布尔值时，会返回"boolean"。

4. **Null 空值**

   空值专门用来表示为空的对象，Null 类型的值只有一个：null，使用`typeof`检查一个 Null 类型的值时会返回"object"。

5. **Undefined 未定义**

   如果声明一个变量但是没有为变量赋值此时变量的值就是 undefined，该类型的值只有一个 undefined。使用`typeof`检查一个 Undefined 类型的值时，会返回"undefined"。

6. **引用数据类型**

   Object 对象

#### 2.强制类型转换

类型转换就是指将其他的数据类型转换为`String`、`Number`或`Boolean`

##### 2.1 转换为String

- 方式 1：调用被转换数据的`toString()`方法，例：
  ```js
  var a = 123;
  a = a.toString();
  ```

  > **注意**
  >
  > 这个方法不适用于`null`和`undefined`，由于这两个类型的数据中没有方法，所以调用`toString()`时会报错。

- 方式 2：调用`String()`函数，例：

  ```js
  var a = 123;  
  a = String(a);
  ```

  对于`Number`、`Boolean`、`String`，都会调用他们的`toString()`方法来将其转换为字符串；对于`null`值，直接转换为字符串"null"；对于`undefined`直接转换为字符串"undefined"。

- 方式 3：隐式的类型转换，为任意的数据类型 + `""`，例：

  ```js
  true + ""; // "true"
  ```

  原理和`String()`函数一样。

##### 2.2 转换为Number

- 方式 1：调用`Number()`函数，例：

  ```js
  Number("123"); // 123
  ```

  转换的情况：

  - 字符串 > 数字
    - 如果字符串是一个合法的数字，则直接转换为对应的数字
    - 如果字符串是一个非法的数字，则转换为`NaN`
    - 如果是一个空串或纯空格的字符串，则转换为 0
  - 布尔值 > 数字
    - `true`转换为 1
    - `false`转换为 0

  - 空值 > 数字
    - `null`转换为 0

  - 未定义 > 数字
    - `undefined`转换为`NaN`

- 方式 2：调用`parseInt()`或`parseFloat()`，这两个函数专门用来将一个字符串转换为数字的。

  如果对非 String 使用`parseInt()`或`parseFloat()`，它会先将其转换为`String`然后再转换，可以将一个字符串中的**以数字开始的有效的整数位**提取出来，并转换为`Number`。

  可以在`parseInt()`中传入第二个参数，来指定进制，该值介于 2 ~ 36 之间，但是`parseInt()`会有一些默认的规则：

  - 如果 string 以 "0x" 或者 0x 开头，`parseInt()`会把 string 的其余部分解析为十六进制的整数
  - 如果 string 以 0 开头，那么 ECMAScript v3 允许`parseInt()`的一个实现把其后的字符解析为八进制数字
  - 如果 string 以 0b 开头，则会将其解析为 2 进制（不是所有浏览器支持）
  - 如果 string 以 1~9 的数字开头，`parseInt()`将把它解析为十进制的整数

  例：

  ```js
  parseInt("123.456px"); //　123
  parseInt("a123"); // NaN
  parseInt(true); // NaN
  
  // 进制转换
  parseInt(0x11); // 17
  Number(0x11); // 17
  parseInt("0x11"); // 17
  Number("0x11"); // 17
  parseInt("0x11", 16); // 17
  
  // 八进制字符串不支持
  parseInt(011); // 9
  Number(011); // 9
  parseInt("011"); // 11
  Number("011"); // 11
  parseInt("011", 8); // 9
  
  // 0b 支持不是很好
  parseInt(0b11); // 3
  Number(0b11); // 3
  parseInt("0b11"); // 0
  Number("0b11"); // 3
  parseInt("0b11", 2); // 0
  ```

  `parseFloat()`可以将一个**字符串中的有效的小数位**提取出来，并转换为`Number`。例：

  ```js
  a = parseFloat(123.456px); //　123.456
  ```

- 方式 3：使用一元的`+`来进行隐式的类型转换，例：

  ```js
  +"123"; // 123
  ```

  原理：和`Number()`函数一样。

##### 2.3 转换为布尔值

- 方式 1：使用`Boolean()`函数，例：

  ```js
  Boolean("false"); //　true
  Boolean("hellow"); // true
  ```

  转换的情况：

  - 字符串 > 布尔
    - 除了空串其余全是`true`（空格也是 true。表示含有内容）
  - 数值 > 布尔
    - 除了`0`和`NaN`其余的全是`true`
  - `null`、`undefined` > 布尔
    - 都是`false`
  - 对象 > 布尔
    - 都是`true`

- 方式 2：隐式转换，使用两次非运算符`!!`，原理与`Boolean()`一致

  ```js
  !!"hellow"; //true
  ```

## 2.基础语法

### 2.1 运算符

运算符也称为操作符，通过运算符可以对一个或多个值进行运算或操作。

- **typeof 运算符**

  用来检查一个变量的数据类型，语法：`typeof 变量`，它会返回一个用于描述类型的字符串作为结果。

- **算数运算符**

  - `+`对两个值进行加法运算并返回结果
  - `-`对两个值进行减法运算并返回结果
  - `*`对两个值进行乘法运算并返回结果
  - `/`对两个值进行除法运算并返回结果
  - `%`对两个值进行取余运算并返回结果

  除了加法以外，对非`Number`类型的值进行运算时，都会先转换为`Number`然后在做运算。而做加法运算时，如果是两个字符串进行相加，则会做拼串操作，将两个字符连接为一个字符串。

  任何值和字符串做加法，都会先转换为字符串，然后再拼串。任何值和`NaN`运算都是`NaN`。

  **做运算时是从左向右依次进行**。

  ```js
  true + 1; // 2
  true + false; // 1
  2 + null; // 2
  2 + NaN; // NaN
  "2" + "2"; // "22"
  "2" + 2; // "22"
  1 + 2 + "3"; // "33"
  "1" + 2 + 3; // "123"
  ```

- **一元运算符**

  一元运算符只需要一个操作数。

  - `+`

    就是正号，不会对值产生任何影响，但是可以将一个非数字转换为数字，例：

    ```js
    +true; // 1
    ```

  - `-`

    就是负号，可以对一个数字进行符号位取反，例：

    ```js
    -10; // -10
    ```

  - 自增

    自增可以使变量在原值的基础上自增 1，自增使用`++`，自增可以使用前++（++a）和后++(a++)，无论是`++a`还是`a++`都会立即使原变量自增 1，不同的是`++a`和`a++`的值是不同的，`++a`的值是变量的新值（自增后的值），`a++`的值是变量的原值（自增前的值）。

    ```js
    1 + +"2" + 3; // 6
    
    var d = 20;
    d++ + ++d + d; // 64
    
    var d = 20;
    d = d++; // 20
    ```

  - 自减

    自减可以使变量在原值的基础上自减 1，自减使用`--`，自减可以使用前--（--a）和后--(a--)，无论是`--a`还是`a--`都会立即使原变量自减 1，不同的是`--a`和`a--`的值是不同的，`--a`的值是变量的新值（自减后的值），`a--`的值是变量的原值（自减前的值）。

- **逻辑运算符**

  - `!`

    非运算可以对一个布尔值进行取反，`true`变`false`、`false`变`true`，但是不会修改原数据的值。

    当对非布尔值使用`!`时，会先将其转换为布尔值然后再取反，我们可以利用`!`来将其他的数据类型转换为布尔值。

  - `&&`

    `&&`可以对符号两侧的值进行与运算。只有两端的值都为`true`时，才会返回`true`。只要有一个`false`就会返回`false`。

    与是一个短路的与，如果第一个值是`false`，则不再检查第二个值。

    对于非布尔值，它会将其转换为布尔值然后做运算，并返回原值。

    规则：

    - 如果第一个值为`true`，则返回第二个值

    - 如果第一个值为`false`，则返回第一个值

  - `||`
    `||`可以对符号两侧的值进行或运算。只有两端都是`false`时，才会返回`false`。只要有一个`true`，就会返回`true`。

    或是一个短路的或，如果第一个值是`true`，则不再检查第二个值。

    对于非布尔值，它会将其转换为布尔值然后做运算，并返回原值。

    规则：

    - 如果第一个值为`true`，则返回第一个值
    - 如果第一个值为`false`，则返回第二个值

- **赋值运算符**

  - `=`
    可以将符号右侧的值赋值给左侧变量。

  - `+=`

    ```js
    a += 5 相当于 a = a + 5    
    var str = "hello";
    str += "world";  // "helloworld"
    ```

  - `-=`

    ```js
    a -= 5 相当于 a = a - 5  
    ```

  - `*=`

    ```js
    a *= 5 相当于 a = a * 5
    ```

  - `/=`

    ```js
    a /= 5 相当于 a = a / 5	  
    ```

  - `%=`

    ```js
    a %= 5 相当于 a = a % 5
    ```

- **关系运算符**

  关系运算符用来比较两个值之间的大小关系的：`>`、`>=`、`<`、`<=`。

  关系运算符的规则和数学中一致，用来比较两个值之间的关系，如果关系成立则返回`true`，关系不成立则返回`false`。

  如果比较的两个值是非数值，会将其转换为`Number`然后再比较。

  任何值和`NaN`做任何比较都是`false`。

  如果比较的两个值都是字符串，此时会比较字符串的`Unicode`编码，而不会转换为`Number`。比较时是一位一位进行比较，如果两位一样，则比较下一位。

  ```js
  1 > "0"; // true
  1 >= true; // true
  10 >= null; // true
  10 > "hello"; // false，此时比较的是 10 和 NaN
  "1" < "5"; // true
  "11" < "5"; // true，此时比较的是编码
  11 < "5"; // false
  "abc" < "b"; // true
  "bbc" < "b"; // false
  ```

- **相等运算符**

  - `==`

    判断左右两个值是否相等，如果相等返回`true`，如果不等返回`false`。

    相等会自动对两个值进行类型转换，如果**对不同的类型进行比较，会将其转换为相同的类型然后再比较**，转换后相等它也会返回`true`。

    ```js
    1 == "1"; // true
    true == "1"; // true
    true == 1; // true
    true == "hello"; // false
    null == 0; // false
    ```

  - `!=`
    判断左右两个值是否不等，如果不等则返回`true`，如果相等则返回`false`，不等也会做自动的类型转换。

  - `===`
    判断左右两个值是否全等，它和相等类似，只不过它不会进行自动的类型转换，如果两个值的类型不同，则直接返回`false`。

  - `!==`

    和不等类似，但是它不会进行自动的类型转换，如果两个值的类型不同，它会直接返回`true`。

  特殊的值：

  - `null`和`undefined`

    由于`undefined`衍生自`null`，所以`null == undefined`会返回`true`，但是`null === undefined`会返回`false`。

  - `NaN`

    `NaN`不与任何值相等，包含它自身`NaN == NaN`为`false`。

    `isNaN()`函数用于检查其参数是否是非数字值。如果参数值为`NaN`或字符串（数字、空串、空格则为`false`）、对象、`undefined`等非数字值则返回`true`, 否则返回`false`。

- **三元运算符**

  语法：`条件表达式?语句1:语句2`

  执行流程：先对条件表达式求值判断，如果判断结果为`true`，则执行语句 1，并返回执行结果，如果判断结果为`false`，则执行语句 2，并返回执行结果。

- **逗号运算符**

  使用`,`可以同时声明多个变量并赋值。

  ```js
  var a, b, c;
  var a = 1, b = 2, c = 3;
  ```

**优先级**
和数学一样，比如先乘除、后加减、先与后或；具体的优先级可以参考优先级的表格，在表格中越靠上的优先级越高，优先级越高的越优先计算，优先级相同的，从左往右计算。

优先级不需要记忆，如果越到拿不准的，使用`()`来改变优先级。

| **优先级** |                      **运算符**                       |                        **说明**                        |         **结合性**         |
| :--------: | :---------------------------------------------------: | :----------------------------------------------------: | :------------------------: |
|     1      |                       []、.、()                       |        字段访问、数组索引、函数调用和表达式分组        |          从左向右          |
|     2      |      ++、--、-、~、!、delete、new、typeof、void       |     一元运算符、返回数据类型、对象创建、未定义的值     |          从右向左          |
|     3      |                        *、/、%                        |                   相乘、相除、求余数                   |          从左向右          |
|     4      |                         +、-                          |                 相加、相减、字符串串联                 |          从左向右          |
|     5      |                      <<、>>、>>>                      |               左位移、右位移、无符号右移               |          从左向右          |
|     6      |               <、<=、>、>=、instanceof                | 小于、小于或等于、大于、大于或等于、是否为特定类的实例 |          从左向右          |
|     7      |                  \==、!=、\===、!\==                  |               相等、不相等、全等，不全等               |          从左向右          |
|     8      |                           &                           |                        按位“与”                        |          从左向右          |
|     9      |                           ^                           |                       按位“异或”                       |          从左向右          |
|     10     |                          \|                           |                        按位“或”                        |          从左向右          |
|     11     |                          &&                           |                   短路与（逻辑“与”）                   |          从左向右          |
|     12     |                         \|\|                          |                   短路或（逻辑“或”）                   |          从左向右          |
|     13     |                          ?:                           |                       条件运算符                       |          从右向左          |
|     14     | =、+=、-=、*=、/=、%=、&=、\|=、^=、<、<=、>、>=、>>= |                     混合赋值运算符                     |          从右向左          |
|     15     |                           ,                           |                        多个计算                        | 按优先级计算，然后从右向左 |

### 2.2 流程控制语句

程序都是自上向下的顺序执行的，通过流程控制语句可以改变程序执行的顺序，或者反复的执行某一段的程序。

JS 中可以使用`{}`代码块，仅起到分组的作用，不会隔离内部和外部的数据访问。

#### 1.条件判断语句

条件判断语句也称为`if`语句。

- 语法一：

  ```bash
  if(条件表达式){  
  	语句...  
  }  
  ```

  执行流程：`if`语句执行时，会先对条件表达式进行求值判断，如果值为`true`，则执行`if`后的语句；如果值为`false`，则不执行。

- 语法二：

  ```bash
  if(条件表达式){  
  	语句...  
  }else{  
  	语句...  
  } 
  ```

  执行流程：`if...else`语句执行时，会对条件表达式进行求值判断，如果值为`true`，则执行`if`后的语句；如果值为`false`，则执行`else`后的语句。

- 语法三：

  ```bash
  if(条件表达式){  
  	语句...  
  }else if(条件表达式){  
  	语句...  
  }else if(条件表达式){  
  	语句...  
  }else{  
  	语句...  
  }	
  ```

  执行流程：`if...else if...else`语句执行时，会自上至下依次对条件表达式进行求值判断，如果判断结果为`true`，则执行当前`if`后的语句，执行完成后语句结束。如果判断结果为`false`，则继续向下判断，直到找到为`true`的为止。如果所有的条件表达式都是`false`，则执行`else`后的语句。

#### 2.条件分支语句

`switch`语句。

语法:

```bash
switch(条件表达式){  
	case 表达式:  
		语句...  
		break;  
	case 表达式:  
		语句...  
		break;  
	case 表达式:  
		语句...  
		break;  
	default:  
		语句...  
		break;  
}  
```

执行流程：
`switch...case...`语句在执行时，会依次将`case`后的表达式的值和`switch`后的表达式的值进行**全等**比较，如果比较结果为`false`，则继续向下比较。如果比较结果为`true`，则从当前`case`处开始向下执行代码。如果所有的`case`判断结果都为`false`，则从`default`处开始执行代码。

#### 3.循环语句

- **while 循环**

  语法：

  ```bash
  while(条件表达式){  
      语句...  
  }  
  ```

  执行流程：`while`语句在执行时，会先对条件表达式进行求值判断，如果判断结果为`false`，则终止循环；如果判断结果为`true`，则执行循环体。循环体执行完毕，继续对条件表达式进行求值判断，依此类推。

- **do...while 循环**

  语法:

  ```bash
  do{  
  语句...  
  }while(条件表达式)  
  ```

  执行流程：`do...while`在执行时，会先执行`do`后的循环体，然后在对条件表达式进行判断，如果判断判断结果为`false`，则终止循环。如果判断结果为`true`，则继续执行循环体，依此类推。

  和`while`的区别：

  - `while`先判断后执行
  - `do...while`先执行后判断
  - `do...while`可以确保循环体至少执行一次

- **for 循环**

  语法：

  ```bash
  for(初始化表达式; 条件表达式; 更新表达式){  
      语句...  
  } 
  ```

**死循环**

```bash
while(true){
}  

for(;;){
}
```

## 3.对象（Object）

对象是 JS 中的引用数据类型，**对象是一种复合数据类型，在对象中可以保存多个不同数据类型的属性**。使用`typeof`检查一个对象时，会返回"object"。

### 3.1 分类

1. 内建对象
   - 由 ES 标准中定义的对象，在任何的 ES 的实现中都可以使用
   - 比如：`Math`、`String`、`Number`、`Boolean`、`Function`、`Object`....

2. 宿主对象
   - 由 JS 的运行环境提供的对象，目前来讲主要指由浏览器提供的对象，比如`BOM`、`DOM`

3. 自定义对象

### 3.2 创建对象

- 方式 1：`var obj = new Object();`

- 方式 2：使用对象字面量，`var obj = {};`

- 方式 3：使用对象字面量时可以在创建对象时直接向对象中添加属性

  语法：

  ```js
  var obj = {  
      name:"zhang3",  // 属性名可以加引号也可以不加
      age:26,  
      gender:"男",  
      "!%#&":"test" // 对于一些特殊的名字需要加引号
      test: {name:"li4"}
  } 
  ```

### 3.3 向对象中添加属性

语法：

- `对象.属性名 = 属性值;`

- `对象["属性名"] = 属性值;` // 这种方式能够使用特殊的属性名

  ```js
  obj.123 = 123; // SyntaxError
  obj[123] = 123;
  obj[123]; // 123
  ```

> **注意**
>
> 对象的属性名没有任何要求，不需要遵守标识符的规范，但是在开发中，尽量按照标识符的要求去写。

属性值也可以任意的数据类型。

### 3.4 读取对象中的属性

语法：

- `对象.属性名;`

- `对象["属性名"];` // "属性名"可以使字符串常量，也可以是字符串变量

  ```js
  var obj = new Object();
  var a = "test";
  obj[a] = "test";
  obj.test; // "test"
  ```

如果读取一个对象中没有的属性，它不会报错，而是返回一个`undefined`。

### 3.5 删除对象中的属性

语法：

- `delete 对象.属性名;`
- `delete 对象["属性名"];`

### 3.6 遍历

可以使用`in`检查对象中是否含有指定属性，语法：`"属性名" in 对象`。如果在对象中含有该属性，则返回`true`；如果没有则返回`false`。

```js
var obj = new Object();
obj.name = "zhang3";
"test" in obj; // false
"name" in obj; // true
```

循环遍历时会遍历对象自身的和继承的可枚举属性（不含`Symbol`属性）：

```js
var obj = {'0':'a','1':'b','2':'c'};  
for(var i in obj) {  
	console.log(i, ":", obj[i]);  
}
/*
0 : a
1 : b
2 : c
*/
```

```js
var obj = new Object();
obj.name = "zhang3";
obj.age = 38;
for(var key in obj){  
	document.write("property：name = " + key + " ;value = " + obj[key] + "<br/>");  
}
```

### 3.7 基本数据类型和引用数据类型

**基本数据类型**：`String`、`Number`、`Boolean`、`Null`、`Undefined`

- 基本数据类型的数据，变量是直接保存的它的值
- 基本数据类型的值保存在栈内存中
- 变量与变量之间是互相独立的，修改一个变量不会影响其他的变量
- 比较两个基本数据类型变量时比较的就是值

**引用数据类型**：`Object`

- 引用数据类型的数据，变量是保存的对象的引用（内存地址）
- 对象是保存在堆内存中，每创建一个新的对象，就在堆内存中开辟一块新的空间
- 如果多个变量指向的是同一个对象，此时修改一个变量的属性，会影响其他变量
- 比较两个引用数据类型变量时比较的是地址，地址相同才相同

### 3.8 垃圾回收（GC）

在 JS 中拥有自动的垃圾回收机制，会自动将这些垃圾对象从内存中销毁，我们不需要也不能进行垃圾回收的操作，需要做的只是要将不再使用的变量设置`null`即可。

## 4.函数（Function）

函数也是一个对象，也具有普通对象的功能（能有属性）。函数中可以封装一些代码，在需要的时候可以去调用函数来执行这些代码。只有在调用时才会执行函数中的代码。

使用`typeof`检查一个函数时会返回"function"。

### 4.1 创建函数

- **函数对象**（了解，基本不使用）

  可以将要封装的代码以字符串的形式传递给函数对象。

  ```js
  var fun = new Function("console.log('hello world');");
  fun(); // hello world
  ```

- **函数声明**

  ```js
  function 函数名([形参1,形参2...形参N]){  
  	语句...  
  } 
  ```

- **函数表达式**

  ```js
  var 函数名 = function([形参1,形参2...形参N]){  
  	语句...  
  };
  ```

### 4.2 调用函数

语法：`函数对象([实参1,实参2...实参N]);`

当我们调用函数时，函数中封装的代码会按照编写的顺序执行。

**嵌套函数**

```js
function fun1() {
    function fun2() {
        return "fun2";
    }
    return fun2;
}
var a = fun1();
console.log(a()); // "fun2"
console.log(fun1()()); // "fun2"
```

**立即执行函数**
函数定义完，立即被调用，这种函数叫做立即执行函数，立即执行函数往往只会执行一次。

```js
(function(a, b){  
    console.log("a = " + a);  
    console.log("b = " + b);  
}(123, 456)); 
```

同时函数中声明的变量无法被外部访问。

```js
var tmp = 234;
(function() {
    var tmp = 123;
    console.log(tmp); // 123
}());
console.log("第二次:" + tmp); // 234
```

### 4.3 形参和实参

- **形参：形式参数**

  定义函数时，可以在`()`中定义一个或多个形参，形参之间使用`,`隔开。

  **定义形参就相当于在函数内声明了对应的变量但是并不赋值**，形参会在调用时才赋值。

  ```js
  var a = 123;
  function fun(a) {
      console.log(a); // undefined
      a = 456; // 这里改变的是函数作用域的 a 的值
  }
  fun();
  console.log(a); // 123
  ```

- **实参：实际参数**

  调用函数时，可以在`()`传递实参，传递的实参会赋值给对应的形参。

  调用函数时 JS 解析器不会检查实参的类型和个数，可以传递任意数据类型的值，也可以是一个函数。

  ```js
  function sum (a, b, c) {
      return c(a, b);
  }
  console.log(sum(1, 2, function(a, b){return a + b;})); // 3
  ```

  如果实参的数量大于形参，多余实参将不会赋值；如果实参的数量小于形参，则没有对应实参的形参将会赋值`undefined`。

### 4.4 返回值：函数执行的结果

使用`return`来设置函数的返回值，该值就会成为函数的返回值，可以通过一个变量来接收返回值。

`return`后边的代码都不会执行，一旦执行到`return`语句时，函数将会立刻退出。

`return`后可以跟任意类型的值，可以是基本数据类型，也可以是一个对象。

如果`return`后不跟值，或者是不写`return`则函数默认返回`undefined`。

> **`break`、`continue`和`return`的区别**
>
> - `break`：退出循环
> - `continue`：跳过当次循环
> - `return`：退出函数

### 4.5 方法（method）

可以将一个函数设置为一个对象的属性。当一个对象的属性是一个函数时，我们称这个函数是该对象的**方法**。
```js
var obj = new Object();
obj.name = "zhang3";
obj.sayName = function() {
    console.log(this.name);
}
obj.sayName(); // "zhang3"
```

```js
var obj = {
    name: "li4",
    sayName: function() {
        console.log(this.name);
    }
}
obj.sayName(); // "li4"
```

### 4.6 call()、apply()、arguments

`call()`和`apply()`这两个方法都是函数对象的方法需要通过函数对象来调用，通过两个方法可以直接调用函数，并且可以通过第一个实参来指定函数中`this`。不同的是`call`是直接传递函数的实参而`apply`需要将实参封装到一个数组中传递。

```js
// case 1
function fun1() {
    console.log("test");
}

fun1(); // test
fun1.call(); // test
fun1.apply(); // test

var obj1 = {
    name: "obj1",
    sayName: function() {
        console.log(this.name);
    }
}

var obj2 = {
    name: "obj2"
}

// case 2
function fun2() {
    console.log(this);
}
fun2(); // Window
fun2.call(obj1); // {obj1}
fun2.apply(obj1); // {obj2}

// case 3
// 使用 apply 时就把 obj2 指定为了 this
obj1.sayName.apply(obj2); // obj2

// case 4
function fun3(a, b) {
    console.log(this.name);
    console.log(a);
    console.log(b);
}
fun3.call(obj1, 1, 2);
fun3.apply(obj1, [1, 2]);
```

使用`call`和`apply`调用时，`this`是指定的那个对象；在全局作用域中`this`代表`window`。

**arguments**

`arguments`和`this`类似，都是函数中的隐含的参数，在调用函数时浏览器会传递这两个参数。不论定义不定义形参，所有实参都会传递给`arguments`。

`arguments`是类数组元素，虽然不是一个数组，但是可以通过索引获取数据。它用来封装函数执行过程中的实参，所以即使不定义形参，也可以通过`arguments`来使用实参（不推荐）。

```js
function fun() {
    console.log(arguments[0]);
}
fun(1, 2); // 1
```

`arguments`中有一个属性`callee`表示当前执行的函数对象。

```js
function fun() {
    console.log(arguments.callee);
}
fun(); // fun()
```

### 4.7 作用域

作用域简单来说就是一个变量的作用范围。在 JS 中作用域分成两种：

1. **全局作用域**

   直接在`script`标签中编写的代码都运行在全局作用域中，**全局作用域在打开页面时创建，在页面关闭时销毁。**

   全局作用域中有一个全局对象`window`，它代表的是整个的浏览器的窗口。`window`对象由浏览器提供，可以在页面中直接使用。

   在全局作用域中创建的变量都会作为`window`对象的属性保存，在全局作用域中创建的函数都会作为`window`对象的方法保存。

   ```js
   var a = 10;
   console.log(window.a); // 10
   
   function fun() {
       return 1;
   }
   console.log(window.fun()); // 1
   ```

   在全局作用域中创建的变量和函数可以在页面的任意位置访问，在函数作用域中也可以访问到全局作用域的变量。

   ```js
   var a = 123;
   function fun1() {
       var a = 456;
       console.log(window.a);
   }
   fun1(); // 123
   ```

2. **函数作用域**

   函数作用域是函数执行时创建的作用域，每次调用函数都会创建一个新的函数作用域。**函数作用域在函数执行时创建，在函数执行结束时销毁。**

   在函数作用域中创建的变量，不能在全局中访问。当在函数作用域中使用一个变量时，它会先在自身作用域中寻找，如果找到了则直接使用，如果没有找到则到上一级作用域中寻找，如果找到了则使用，找不到则继续向上找，直到全局作用域都找不到则报错 ReferenceError。

3. **变量的声明提前**

   在全局作用域中，使用`var`关键字声明的变量会在所有的代码执行之前被声明，但是不会赋值。所以我们可以在变量声明前使用变量。

   ```js
   console.log(a); // undefined
   var a = 123;
   // 相当于
   var a;
   ...
   console.log(a);
   a = 123;
   ```

   但是不使用`var`关键字声明的变量不会被声明提前。

   ```js
   console.log(a); // ReferenceError: a is not defined
   a = 123;
   ```

   > **注意**
   >
   > 如果先赋值 a 但是不使用 var 也可以，此时就是创建了一个全局变量，相当于使用了`window.a`，在函数作用域中也是如此。
   >
   > ```js
   > a = 123;
   > console.log(a); // 123
   > ```

   在函数作用域中，也具有该特性（变量的声明提前），使用`var`关键字声明的变量会在函数所有的代码执行前被声明。

   ```js
   var a = 123;
   function fun() {
       console.log(a);
       var a = 456;
   }
   fun(); // undefined
   ```

   如果没有使用`var`关键字声明变量，则变量会变成全局变量。

   ```js
   var a;
   function fun() {
       a = 123;
   }
   fun();
   console.log(a); // 123
   ```

   ```js
   var a;
   function fun() {
       var a = 123;
   }
   fun();
   console.log(a); // undefined
   ```

4. **函数的声明提前**

   在全局作用域中，使用函数声明创建的函数`function fun(){}`，会在所有的代码执行之前被创建，也就是我们可以在函数声明前去调用函数，但是使用函数表达式`var fun = function(){}`创建的函数没有该特性。

   ```js
   console.log(fun1()); // 1
   console.log(fun2()); // 2
   
   function fun1() {
       return 1;
   }
   
   function fun2() {
       return 2;
   }
   ```

   ```js
   console.log(fun1()); // TypeError: fun1 is not a function
   var fun1 = function() {
       return 1;
   }
   ```

   函数作用域中一致。

### 4.8 this（上下文对象）

我们每次调用函数时，解析器都会将一个上下文对象作为隐含的参数传递进函数。使用`this`来引用上下文对象，根据函数的调用形式不同，`this`的值也不同。

1. 以函数的形式调用时，`this`是`window`

   ```js
   function fun() {
       console.log(this);
   }
   fun(); // Window{...}
   ```

2. 以方法的形式调用时，`this`就是调用方法的对象

   ```js
   function fun() {
       console.log(this);
   }
   var obj = {
       name: "li4",
       function: fun
   }
   obj.function(); // {name: 'li4', function: ƒ}
   ```

   如果在对象的方法中不使用 this 则会默认从全局查找

   ```js
   var a = 123;
   function fun() {
       console.log(a);
   }
   
   var obj = {
       a: 456,
       printA: fun
   }
   
   fun(); // 123
   obj.printA(); // 123
   ```

   ```js
   var a = 123;
   function fun() {
       console.log(this.a);
   }
   
   var obj = {
       a: 456,
       printA: fun
   }
   
   fun(); // 123
   obj.printA(); // 456
   ```

3. 以构造函数的形式调用时，`this`就是新创建的对象

4. 使用`call`和`apply`调用时，`this`是指定的那个对象

5. 在全局作用域中`this`代表`window`

6. 在事件的响应函数中，响应函数给谁绑定的`this`就是谁

### 4.9 构造函数

构造函数是专门用来创建对象的函数，**一个构造函数我们也可以称为一个类**。

通过一个构造函数创建的对象，我们称该对象时这个构造函数的**实例**；通过同一个构造函数创建的对象，我们称为一类对象。
构造函数就是一个普通的函数，只是他的首字母要大写，并且调用时要使用`new`。不使用`new`则就是一个普通函数，没有返回值则为`undefined`。

例：

```js
function Person(name, age, gender) {  
    this.name = name;  
    this.age = age;  
    this.gender = gender;  
    this.sayName = function() {  
        alert(this.name);  
    };  
}
var person = new Person("zhang3", 29, "男");
```

构造函数的执行流程：

1. 创建一个新的对象
2. 将新的对象设置为函数的上下文对象（this）
3. 逐行执行函数中的代码
4. 将新建的对象返回

`instanceof`可以用来检查一个对象是否是一个类的实例，语法：`对象 instanceof 构造函数`。如果该对象时构造函数的实例，则返回`true`，否则返回`false`。

`Object`是所有对象的祖先，所以任何对象和`Object`做`instanceof`都会返回`true`。

在 Person 构造函数中为每个实例都创建了一个`sayName`方法：

```js
var person1 = new Person("zhang3", 29, "男");
var person2 = new Person("li4", 25, "男");
person1.sayName == person2.sayName; // false
```

可以让示例共享一个方法：

```js
var sayName = function() {  
    alert(this.name);  
};
function Person(name , age , gender) {  
    this.name = name;  
    this.age = age;  
    this.gender = gender;  
    this.sayName = sayName;
}
var person1 = new Person("zhang3", 29, "男");
var person2 = new Person("li4", 25, "男");
person1.sayName == person2.sayName; // true
```

但是将方法定义在全局作用域中就污染了全局命名空间，而且也不安全。

### 4.10 原型（prototype）

创建一个函数（也是一个对象）以后，解析器都会默认在函数中添加一个属性`prototype`，`prototype`属性指向的是一个对象，这个对象我们称为原型对象。当函数作为构造函数使用，**它所创建的对象中都会有一个隐含的属性指向该原型对象**，这个隐含的属性可以通过对象`.__proto__`来访问。

<img src="https://raw.githubusercontent.com/Famezyy/picture/master/notePictureBed/image-20230506174937616.png" alt="image-20230506174937616" style="zoom:80%;" />

原型对象就相当于一个公共的区域，凡是通过同一个构造函数创建的实例都可以访问到相同的原型对象。我们可以将对象中共有的属性和方法统一添加到原型对象中，这样我们只需要添加一次，就可以使所有的对象都可以使用。

```js
function MyClass(name){
    this.name = name;
}
MyClass.prototype.sayName = function() {  
    alert(this.name);  
};
var myClass = new MyClass("zhang3");
myClass.sayName(); // "zhang3"
```

当我们去访问对象的一个属性或调用对象的一个方法时，它会先在自身中寻找，如果在自身中找到了，则直接使用。如果没有找到，则去原型对象中寻找，如果找到了则使用，如果没有找到，则去原型的原型中寻找，依此类推直到找到`Object`对象的原型为止（`myClass.__proto__.__proto__ == Object.__proto__`、`Object.__proto__.__proto__=null`），`Object`对象的原型的原型为`null`，如果依然没有找到则返回`undefined`。

使用`in`检查对象中是否含有该属性时，当对象没有而原型有依然会返回`true`。可以使用`hasOwnProperty()`检查对象自身中是否含有某个属性，语法：`对象.hasOwnProperty("属性名")`。

### 4.11 toString方法

当我们直接在页面中打印一个对象时，事件上是输出的对象的`toString()`方法的返回值，即`console.log(person) `相当于`console.log(person.toString())`，该方法在`Object`的原型中。如果我们希望在输出对象时不输出`[object Object]`，可以为对象添加一个`toString()`方法：

```js
// 修改 Person 原型的 toString() 
Person.prototype.toString = function(){  
	return "Person[name = " + this.name + ", age = " + this.age + ", gender = " + this.gender + "]";  
};
```

## 5.数组（Array）

数组也是一个对象，是一个用来存储数据的对象和`Object`类似，但是它的存储效率比普通对象要高。数组中保存的内容我们称为元素，数组使用索引来操作元素，索引由`0`开始。

### 5.1 数组的操作

- **创建数组**

  元素可以是任意类型。

  ```js
  var arr = new Array();
  var arr = new Array(10); // 指定初始长度
  var arr = [];
  var arr = new Array(123, "hello", undefined, {"name": "zhang3"});
  var arr = [123, "hello", true, null, function(){alert(1)}];  
  ```

- **向数组中添加元素**

  ```js
  arr[0] = 123;  
  arr[1] = "hello";
  ```

- **读取元素**

  读取不存在的索引返回`undefined`。

  ```js
  arr[1]; // "hello"
  arr[2]; // undefined
  ```

- **获取数组的长度**

  ```js
  arr.length;
  ```

  `length`获取到的是数组的最大索引`+1`，对于连续的数组，`length`获取到的就是数组中元素的个数。

- **修改数组的长度**

  ```js
  arr.length = 5;
  ```

  如果修改后的`length`大于原长度，则多出的部分会空出来，如果修改后的`length`小于原长度，则原数组中多出的元素会被删除。

- **向数组的最后添加元素**

  ```js
  arr[arr.length] = 10;
  ```

### 5.2 常用方法

| functionName |                           function                           |                      usage                       |
| :----------: | :----------------------------------------------------------: | :----------------------------------------------: |
|   `push()`   |      向数组的末尾添加一个或多个元素，并返回数组新的长度      |         `arr.push(元素1, 元素2, 元素N)`          |
|   `pop()`    |          删除数组的最后一个元素，并返回被删除的元素          |                   `arr.pop()`                    |
| `unshift()`  |     向数组的开头添加一个或多个元素，并返回数组的新的长度     |        `arr.unshift(元素1, 元素2, 元素N)`        |
|  `shift()`   |         删除数组开头的第一个元素，并返回被删除的元素         |                  `arr.shift()`                   |
| `reverse()`  |              反转一个数组，它会对原数组产生影响              |                 `arr.reverse()`                  |
|  `concat()`  | 连接两个或多个数组或元素，它不会影响原数组，并把新数组作为返回值返回；不传参时相当于浅复制原数组 | `var arr3 = arr1.concat(arr2, "item1", "item2")` |

- `slice(sart, [end])`

  可以从一个数组中截取指定的元素，该方法不会影响原数组，而是将截取到的内容封装为一个新的数组并返回。
  参数：

  1. 截取开始位置的索引（包括开始位置）
  2. 截取结束位置的索引（不包括结束位置）

  第二个参数可以省略不写，如果不写则一直截取到最后。参数可以传递一个负值，如果是负值，则表示从后往前计算索引。

- `splice()`

  可以用来删除数组中指定元素，并使用新的元素替换，该方法直接影响原数组，并将删除的元素封装到新数组中并返回。

  参数：

  1. 删除开始位置的索引
  2. 删除的个数
  3. 三个以后，都是替换的元素，这些元素将会插入到第一个参数指定的索引的前边，插入元素个数不限

- `join([splitor])`

  可以将一个数组转换为一个字符串，不会对原数组产生影响。

  参数：

  需要一个字符串作为参数，这个字符串将会作为连接符来连接数组中的元素，如果不指定连接符则默认使用逗号`,`连接。

- `sort()`

  可以对一个数组中的内容进行排序，调用以后会直接修改原数组，默认是按照`Unicode`编码进行排序。

  可以自己指定排序的规则，需要传入一个回调函数作为参数，回调函数中需要定义两个形参。浏览器将会分别使用数组中的两个元素作为实参去调用回调函数，根据回调函数的返回值来决定元素的顺序，如果返回一个大于 0 的值，则元素会交换位置；如果返回一个小于 0 的值，则元素位置不变；如果返回一个 0 ，则认为两个元素相等，也不交换位置。

  一般使用：

  - 如果需要升序排列，则返回`a-b`
  - 如果需要降序排列，则返回`b-a`

```js
function(a, b){  
	//升序排列  
	//return a-b;  
	//降序排列  
	return b-a;  
}  

arr.sort(function(a, b) {return a - b;});
```

### 5.3 遍历数组

一般情况我们都是使用`for`循环来遍历数组：

```js
for(var i = 0; i < arr.length; i++){
    console.log(arr[i]);
}  
```

使用`forEach()`方法来遍历数组（不兼容 IE8）：

```js
arr.forEach(function(value, index, obj){
    
});  
```

`forEach()`方法需要一个回调函数作为参数，数组中有几个元素，回调函数就会被调用几次，每次调用时，都会将遍历到的信息以实参的形式传递进来，我们可以定义形参来获取这些信息。

- `value`：正在遍历的元素
- `index`：正在遍历元素的索引
- `obj`：正在被遍历的数组对象

**练习：去重**

```js
for (var i = 0; i < arr.length; i++) {
    console.log(arr.length);
    for (var j = i + 1; j < arr.length; j++) {
        if (arr[i] == arr[j]) {
            arr.splice(j, 1);
            // 当删除元素后，后一个元素的索引会往前移，因此需要将 j--
            j--;
        }
    }
}
```

## 6.常用类和方法

### 6.1 包装类

在 JS 中为我们提供了三个包装类：

- `String()`
- `Boolean()`
- `Number()`

通过这三个包装类可以创建基本数据类型的对象。例：

```js
var num = new Number(2);  
var str = new String("hello");  
var bool = new Boolean(true); 

typeof num; // object
typeof str; // object
typeof bool; // object
```

但是在实际应用中千万不要这么干，这些类型主要是浏览器来使用。当我们去操作一个基本数据类型的属性和方法时，解析器会临时将其转换为对应的包装类，然后再去操作属性和方法，操作完成以后再将这个临时对象进行销毁。例如以下代码：

```js
var num = 3;
var str = num.toString(); // 会转换为包装类调用方法
typeof str; // string

var s = "test";
s.hello = "hello"; // 自动转换包装类添加属性，然后删除这个包装类
console.log(s.hello); // undefined - 自动转换包装类读取属性，然后删除这个包装类
```

### 6.2 Date

日期的对象，在 JS 中通过`Date`对象来表示一个时间。

- 创建一个当前的时间对象

  ```js
  var d = new Date();
  ```

- 创建一个指定的时间对象

  ```js
  // 格式：Month/Date/Year Hour:Min:Sec
  var d = new Date("03/12/2020 18:25:45");
  ```

**常用方法**

|        方法         |                             用途                             |
| :-----------------: | :----------------------------------------------------------: |
|     `getDate()`     |                  当前日期对象是几日（1-31）                  |
|     `getDay()`      |  返回当前日期对象的星期信息（0-6） 0：周日、1：周一 ... ...  |
|    `getMonth()`     |    返回当前日期对象的月份（0-11） 0：一月、1：二月... ...    |
|   `getFullYear()`   |              返回当前日期对象的年份（4 位数字）              |
|    `getHours()`     |                返回 Date 对象的小时（0 ~ 23）                |
|   `getMinutes()`    |                返回 Date 对象的分钟（0 ~ 59）                |
|   `getSeconds()`    |                返回 Date 对象的秒数（0 ~ 59）                |
| `getMilliseconds()` |               返回 Date 对象的毫秒（0 ~ 999）                |
|     `getTime()`     | 返回当前日期对象的时间戳，时间戳指的是从格林威治标准时间 1970 年 1 月 1 日 0 时 0 分 0 秒，到现在时间的毫秒数，计算机底层保存时间都是以时间戳的形式保存的 |
|    `Date.now()`     |                可以获取当前代码执行时的时间戳                |
|    `setHours()`     |               设置 Date 对象中的小时（0 ~ 23）               |

### 6.3 Math

`Math`属于一个工具类，它不需要我们创建对象，封装了属性运算相关的常量和方法，我们可以直接使用它来进行数学运算相关的操作。

- `Math.PI`：常量，圆周率

- `Math.abs(n)`：绝对值

- `Math.ceil(n)`：向上取整，小数有值则进 1

- `Math.floor(n)`：向下取整，无视小数

- `Math.round(n)`：四舍五入取整

- `Math.random()`：生成一个 01 之间的随机数

  生成一个 x y 之间的随机数：`Math.round(Math.random()*(y-x)+x)`

- `Math.pow(x,y)`：求 x 的 y 次幂

- `Math.sqrt(n)`：对一个数进行开方

- `Math.max(n1, n2, n3)`：求多个数中最大值

- `Math.min(n1, n2, n3)`：求多个数中的最小值

### 6.4 字符串

字符串在底层是以字符数组的形式保存的。

```js
var str = "hello world";
str.length; // 11
str[1]; // e
str.charAt(4); // o
str.charCodeAt(4); // 111，返回指定位置的字符编码
String.fromCharCode(111); // o，根据字符编码获取字符，默认 10 进制，16 进制需要加上 0x
str.concat("!"); // hello world!
str.split(" "); // ["hello", "world"]，传入空字符串则表示拆分每个字符
str.indexOf("o"); // 4，返回第一次出现的索引，没有则返回 -1；可以指定一个第二个参数，来表示开始查找的位置
str.lastIndexOf("o"); // 7，返回最后一次出现的索引；可以指定一个第二个参数，表示开始查找的位置
str.slice(6, 11); // world，返回截取的字符串，第一个参数指定开始的索引（in），第二个参数指定结束的索引（out）；可以省略第二个参数，如果省略则一直截取到最后；可以传负数，如果是负数则从后往前的索引
str.substring(11, 6); // 和 slice() 基本一致，不同的是它不能接受负值作为参数，如果设置一个负值，则会自动修正为0；substring() 中如果第二个参数小于第一个，自动调整交换位置
str.substr(6, 5); // 和 slice() 基本一致，不同的是传入第二个参数时表示截取的数量
str.toUpperCase(); // HELLO WORLD
str.toLowerCase(); // hello world
str.padStart("15", "1"); // 1111hello world
str.padEnd("15", "1"); // "hello world1111"
```

### 6.5 正则表达式

#### 1.创建正则表达式

```js
var reg = new RegExp("正则表达式","匹配模式");
var reg = /正则表达式/匹配模式; // 匹配模式可以多个一起写：/gi
```

由于第一种创建方式可以传递变量，因此更加灵活。

匹配模式：

- `i`：忽略大小写（ignore）
- `g`：全局匹配模式（默认为 1 次）

设置匹配模式时，可以都不设置，也可以设置 1 个，也可以全设置，设置时没有顺序要求。

#### 2.检查字符串是否符合正则表达式

```js
reg.test(str); // true or false
```

例：

```js
var reg = new RegExp("a", "i");
reg.test("Abc"); // true

reg = /a/i;
reg.test("Abc"); true
```

#### 3.常用字符串方法

- `split()`

  可以传递一个正则表达式，此时会根据正则表达式去拆分数组。

  `split()`即使不指定全局匹配也会全部拆分。

  ```js
  var str = "1q2w3e4r5";
  str.split(/[A-z]/); // ["1", "2", "3", "4", "5"]
  ```

- `search()`

  可以搜索字符串中是否含有指定内容，如果搜索到指定内容，则会返回第一次出现的索引，如果没有搜索到返回 -1；它可以接受一个正则表达式作为参数，然后会根据正则表达式去检索字符串。

  `serach()`只会查找第一个，即使设置全局匹配也没用。

  ```js
  var str = "abc def ghi";
  str.search("def"); // 4
  str.search(/[ghi]/); // 8
  
  str = "11.2";
  str.search("\\."); // 2
  str.search(/\./); // 2
  ```

- `match()`

  可以根据正则表达式，从一个字符串中将符合条件的内容提取出来。

  默认情况下只会找到第一个符合要求的内容，找到以后就停止检索，我们可以设置正则表达式为**全局匹配模式**，这样就会匹配到所有的内容。`match()`会将匹配到的内容封装到一个数组中返回，即使只查询到一个结果。

  ```js
  var str = "1q2w3e4r5F6G";
  str.match(/[A-z]/); // q
  str.match(/[A-z]/g); // ["q", "w", "e", "r", "F", "G"]
  ```

- `replace()`

  可以将字符串中指定内容替换为新的内容。第一个参数是被替换的内容，可以接受一个正则表达式作为参数；第二个参数是新的内容，空串`""`表示删除。

  默认只会替换第一个，可以设置为**全局匹配模式**来替换所有。

  ```js
  var str = 1q2w3e4r;
  str.replace(/[A-z]/, ""); // 12w3e4r
  str.replace(/[A-z]/g, "") // 1234
  
  str = "       ab      c         ";
  str.replace(/^\s*|\s*$/g, ""); // ab      c
  ```

