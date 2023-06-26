# 第1章_ES基础

ECMAScript 6.0（以下简称 ES6）是 JavaScript 语言的下一代标准，已经在 2015 年 6 月正式发布了。它的目标，是使得 JavaScript 语言可以用来编写复杂的大型应用程序，成为企业级开发语言。

## 1.背景

**ECMAScript 和 JavaScript 的关系**

一个常见的问题是，ECMAScript 和 JavaScript 到底是什么关系？

要讲清楚这个问题，需要回顾历史。1996 年 11 月，JavaScript 的创造者 Netscape 公司，决定将 JavaScript 提交给标准化组织 ECMA，希望这种语言能够成为国际标准。次年，ECMA 发布 262 号标准文件（ECMA-262）的第一版，规定了浏览器脚本语言的标准，并将这种语言称为 ECMAScript，这个版本就是 1.0 版。

该标准从一开始就是针对 JavaScript 语言制定的，但是之所以不叫 JavaScript，有两个原因。一是商标，Java 是 Sun 公司的商标，根据授权协议，只有 Netscape 公司可以合法地使用 JavaScript 这个名字，且 JavaScript 本身也已经被 Netscape 公司注册为商标。二是想体现这门语言的制定者是 ECMA，不是 Netscape，这样有利于保证这门语言的开放性和中立性。

因此，ECMAScript 和 JavaScript 的关系是，前者是后者的规格，后者是前者的一种实现（另外的 ECMAScript 方言还有 JScript 和 ActionScript）。日常场合，这两个词是可以互换的。

**ES6 与 ECMAScript 2015 的关系**

2011 年，ECMAScript 5.1 版发布后，就开始制定 6.0 版了。因此，ES6 这个词的原意就是指 JavaScript 语言的下一个版本。

但是，因为这个版本引入的语法功能太多，而且制定过程当中，还有很多组织和个人不断提交新功能。事情很快就变得清楚了，不可能在一个版本里面包括所有将要引入的功能。常规的做法是先发布 6.0 版，过一段时间再发 6.1 版，然后是 6.2 版、6.3 版等等。

但是，标准的制定者不想这样做。他们想让标准的升级成为常规流程：任何人在任何时候，都可以向标准委员会提交新语法的提案，然后标准委员会每个月开一次会，评估这些提案是否可以接受，需要哪些改进。如果经过多次会议以后，一个提案足够成熟了，就可以正式进入标准了。这就是说，标准的版本升级成为了一个不断滚动的流程，每个月都会有变动。

标准委员会最终决定，标准在每年的 6 月份正式发布一次，作为当年的正式版本。接下来的时间，就在这个版本的基础上做改动，直到下一年的 6 月份，草案就自然变成了新一年的版本。这样一来，就不需要以前的版本号了，只要用年份标记就可以了。

ES6 的第一个版本，就这样在 2015 年 6 月发布了，正式名称就是《ECMAScript 2015 标准》（简称 ES2015）。2016 年 6 月，小幅修订的《ECMAScript 2016 标准》（简称 ES2016）如期发布，这个版本可以看作是 ES6.1 版，因为两者的差异非常小（只新增了数组实例的`includes`方法和指数运算符），基本上是同一个标准。根据计划，2017 年 6 月发布 ES2017 标准。

因此，ES6 既是一个历史名词，也是一个泛指，含义是 5.1 版以后的 JavaScript 的下一代标准，涵盖了 ES2015、ES2016、ES2017 等等，而 ES2015 则是正式名称，特指该年发布的正式版本的语言标准。本书中提到 ES6 的地方，一般是指 ES2015 标准，但有时也是泛指“下一代 JavaScript 语言”。

## 2.let与const

### 2.1 let

#### 1.基础使用

ES6 新增了`let`命令，用来声明变量。它的用法类似于`var`，但是所声明的变量，只在`let`命令所在的代码块内有效。

```js
{
  let a = 10;
  var b = 1;
}

a // ReferenceError: a is not defined.
b // 1
```

上面代码在代码块之中，分别用`let`和`var`声明了两个变量。然后在代码块之外调用这两个变量，结果`let`声明的变量报错，`var`声明的变量返回了正确的值。这表明，`let`声明的变量只在它所在的代码块有效。

`for`循环的计数器，就很合适使用`let`命令。

```js
for (let i = 0; i < 10; i++) {
  // ...
}

console.log(i);
// ReferenceError: i is not defined
```

上面代码中，计数器`i`只在`for`循环体内有效，在循环体外引用就会报错。

下面的代码如果使用`var`，最后输出的是`10`。

```js
var a = [];
for (var i = 0; i < 10; i++) {
  a[i] = function () {
    console.log(i);
  };
}
a[6](); // 10
```

上面代码中，变量`i`是`var`命令声明的，在全局范围内都有效，所以全局只有一个变量`i`。每一次循环，变量`i`的值都会发生改变，而循环内被赋给数组`a`的函数内部的`console.log(i)`，里面的`i`指向的就是全局的`i`。也就是说，所有数组`a`的成员里面的`i`，指向的都是同一个`i`，导致运行时输出的是最后一轮的`i`的值，也就是 10。

如果使用`let`，声明的变量仅在块级作用域内有效，最后输出的是 6。

```js
var a = [];
for (let i = 0; i < 10; i++) {
  a[i] = function () {
    console.log(i);
  };
}
a[6](); // 6
```

上面代码中，变量`i`是`let`声明的，当前的`i`只在本轮循环有效，所以每一次循环的`i`其实都是一个新的变量，所以最后输出的是`6`。

另外，`for`循环设置循环变量的那部分是一个父作用域，而循环体内部是一个单独的子作用域。

```js
for (let i = 0; i < 3; i++) {
  let i = 'abc';
  console.log(i);
}
// abc
// abc
// abc
```

上面代码正确运行，输出了 3 次`abc`。这表明函数内部的变量`i`与循环变量`i`不在同一个作用域，有各自单独的作用域（同一个作用域不可使用 `let` 重复声明同一个变量）。

#### 2.不存在声明提前

`var`命令会发生“声明提前”现象，即变量可以在声明之前使用，值为`undefined`。为了纠正这种现象，`let`命令改变了语法行为，它所声明的变量一定要在声明后使用，否则报错。

```js
// var 的情况
console.log(foo); // 输出 undefined
var foo = 2;

// let 的情况
console.log(bar); // 报错 ReferenceError
let bar = 2;
```

上面代码中，变量`foo`用`var`命令声明，会发生变量提升，即脚本开始运行时，变量`foo`已经存在了，但是没有值，所以会输出`undefined`。变量`bar`用`let`命令声明，不会发生变量提升。这表示在声明它之前，变量`bar`是不存在的，这时如果用到它，就会抛出一个错误。

#### 3.暂时性死区

只要作用域内存在`let`、`const`，它们所声明的变量或常量就自动 “绑定” 这个区域，不再受到外部作用域的影响。

```js
var a = 2;
function func() {
    console.log(a);
    var a = 3;
}
func(); // undefined

let a = 2;
function func() {
    console.log(a);
    let a = 1;
}
func(); // 报错

let a = 2;
function func() {
    console.log(a);
}
func(); // 2
```

**即：只要作用域内出现了同名的 let 或 const，那么就会去找这个量（向前找），如果找不到也不会跳去外部找，只会直接报错！**

> 只要我们遵守 “**先声明后使用**”，那么其实就基本不会遇到声明提前及暂时性死区问题。

#### 4.不允许重复声明

`let`不允许在相同作用域内，重复声明同一个变量。

```js
// 报错
function func() {
  let a = 10;
  var a = 1;
}

// 报错
function func() {
  let a = 10;
  let a = 1;
}
```

因此，不能在函数内部重新声明参数。

```js
function func(arg) {
  let arg;
}
func() // 报错

function func(arg) {
  {
    let arg;
  }
}
func() // 不报错
```

#### 5.不与顶层对象挂钩

顶层对象，在浏览器环境指的是`window`对象，在 Node 指的是`global`对象。ES5 之中，顶层对象的属性与全局变量是等价的。

```js
window.a = 1;
a // 1

a = 2;
window.a // 2
```

上面代码中，顶层对象的属性赋值与全局变量的赋值，是同一件事。

顶层对象的属性与全局变量挂钩，被认为是 JavaScript 语言最大的设计败笔之一。这样的设计带来了几个很大的问题，首先是没法在编译时就报出变量未声明的错误，只有运行时才能知道（因为全局变量可能是顶层对象的属性创造的，而属性的创造是动态的）；其次，程序员很容易不知不觉地就创建了全局变量（比如打字出错）；最后，顶层对象的属性是到处可以读写的，这非常不利于模块化编程。另一方面，`window`对象有实体含义，指的是浏览器的窗口对象，顶层对象是一个有实体含义的对象，也是不合适的。

ES6 为了改变这一点，一方面规定，为了保持兼容性，`var`命令和`function`命令声明的全局变量，依旧是顶层对象的属性；另一方面规定，`let`命令、`const`命令、`class`命令声明的全局变量，不属于顶层对象的属性。也就是说，从 ES6 开始，全局变量将逐步与顶层对象的属性脱钩。

```js
var a = 1;
// 如果在 Node 的 REPL 环境，可以写成 global.a
// 或者采用通用方法，写成 this.a
window.a // 1

let b = 1;
window.b // undefined
```

上面代码中，全局变量`a`由`var`命令声明，所以它是顶层对象的属性；全局变量`b`由`let`命令声明，所以它不是顶层对象的属性，返回`undefined`。

全局作用域中，`var` 声明的变量，`function` 声明的函数，都会自动变成 window 对象的属性或方法。

```js
var age = 18;
function add() {}
console.log(window.age);            // 18
console.log(window.add === add);     // true
let age = 18;
const add = function() {}
console.log(window.age);            // undefined
console.log(window.add === add);     // false
```

#### 6.块级作用域

ES5 只有全局作用域和函数作用域，没有块级作用域，这带来很多不合理的场景。

第一种场景，内层变量可能会覆盖外层变量。

```js
var tmp = new Date();

function f() {
  console.log(tmp);
  if (false) {
    var tmp = 'hello world';
  }
}

f(); // undefined
```

上面代码的原意是，`if`代码块的外部使用外层的`tmp`变量，内部使用内层的`tmp`变量。但是，函数`f`执行后，输出结果为`undefined`，原因在于变量提升，导致内层的`tmp`变量覆盖了外层的`tmp`变量。

第二种场景，用来计数的循环变量泄露为全局变量。

```js
var s = 'hello';

for (var i = 0; i < s.length; i++) {
  console.log(s[i]);
}

console.log(i); // 5
```

上面代码中，变量`i`只用来控制循环，但是循环结束后，它并没有消失，泄露成了全局变量。

ES6 中的`let`实际上为 JavaScript 新增了块级作用域：

- 作用域链：内层作用域 ——> 外层作用域 ——> 全局作用域
- 块级作用域：除了对象 `{}`，函数 `{}`（函数作用域）之外的一切 `{}` 都属于块级作用域。

```js
function f1() {
  let n = 5;
  if (true) {
    let n = 10;
  }
  console.log(n); // 5
}
```

上面的函数有两个代码块，都声明了变量`n`，运行后输出 5。这表示外层代码块不受内层代码块的影响。如果两次都使用`var`定义变量`n`，最后输出的值才是 10。

ES6 允许块级作用域的任意嵌套。

```js
{{{{
  {let insane = 'Hello World'}
  console.log(insane); // 报错
}}}};
```

上面代码使用了一个五层的块级作用域，每一层都是一个单独的作用域。第四层作用域无法读取第五层作用域的内部变量。

内层作用域可以定义外层作用域的同名变量。

```js
{{{{
  let insane = 'Hello World';
  {let insane = 'Hello World'}
}}}};
```

块级作用域的出现，实际上使得获得广泛应用的匿名立即执行函数表达式（匿名 IIFE）不再必要了。

```js
// IIFE 写法
(function () {
  var tmp = ...;
  ...
}());

// 块级作用域写法
{
  let tmp = ...;
  ...
}
```

#### 7.函数声明

ES5 规定，函数只能在顶层作用域和函数作用域之中声明，不能在块级作用域声明。

```js
// 情况一
if (true) {
  function f() {}
}

// 情况二
try {
  function f() {}
} catch(e) {
  // ...
}
```

上面两种函数声明，根据 ES5 的规定都是非法的。

但是，浏览器没有遵守这个规定，为了兼容以前的旧代码，还是支持在块级作用域之中声明函数，因此上面两种情况实际都能运行，不会报错。

ES6 引入了块级作用域，明确允许在块级作用域之中声明函数。ES6 规定，块级作用域之中，函数声明语句的行为类似于`let`，在块级作用域之外不可引用。

```js
function f() { console.log('I am outside!'); }

(function () {
  if (false) {
    // 重复声明一次函数 f
    function f() { console.log('I am inside!'); }
  }
  f();
}());
```

上面代码在 ES5 中运行，会得到“I am inside!”，因为在`if`内声明的函数`f`会被提升到函数头部，实际运行的代码如下。

```js
// ES5 环境
function f() { console.log('I am outside!'); }

(function () {
  function f() { console.log('I am inside!'); }
  if (false) {
  }
  f();
}());
```

ES6 就完全不一样了，理论上会得到“I am outside!”。因为块级作用域内声明的函数类似于`let`，对作用域之外没有影响。但是，如果你真的在 ES6 浏览器中运行一下上面的代码，是会报错的，这是为什么呢？

```js
// 浏览器的 ES6 环境
function f() { console.log('I am outside!'); }

(function () {
  if (false) {
    // 重复声明一次函数 f
    function f() { console.log('I am inside!'); }
  }
  f();
}());
// Uncaught TypeError: f is not a function
```

上面的代码在 ES6 浏览器中会报错， ES6 中函数声明有以下规则：

- 允许在块级作用域内声明函数
- 同时，函数声明还会提升到所在的块级作用域的头部

注意，上面规则只对 ES6 的浏览器实现有效，其他环境的实现不用遵守，还是将块级作用域的函数声明当作`let`处理。

据此，在浏览器的 ES6 环境中，块级作用域内声明的函数的行为类似于`var`声明的变量。上面的例子实际运行的代码如下。

```js
// 浏览器的 ES6 环境
function f() { console.log('I am outside!'); }
(function () {
  var f = undefined;
  if (false) {
    function f() { console.log('I am inside!'); }
  }
  f();
}());
// Uncaught TypeError: f is not a function
```

考虑到环境导致的行为差异太大，应该避免在块级作用域内声明函数。如果确实需要，也应该写成函数表达式，而不是函数声明语句。

```js
// 块级作用域内部的函数声明语句，建议不要使用
{
  let a = 'secret';
  function f() {
    return a;
  }
}

// 块级作用域内部，优先使用函数表达式
{
  let a = 'secret';
  let f = function () {
    return a;
  };
}
```

另外，还有一个需要注意的地方。ES6 的块级作用域必须有大括号，如果没有大括号，JavaScript 引擎就认为不存在块级作用域。

```js
// 第一种写法，报错
if (true) let x = 1;

// 第二种写法，不报错
if (true) {
  let x = 1;
}
```

函数声明也是如此

```js
// 不报错
if (true) {
  function f() {}
}

// 报错
if (true)
  function f() {}
```

### 2.2 const

#### 1.基本用法

`const`声明一个只读的常量。一旦声明，常量的值就不能改变。

```js
const PI = 3.1415;
PI // 3.1415

PI = 3;
// TypeError: Assignment to constant variable.
```

`const`声明的变量不得改变值，这意味着`const`一旦声明变量，就必须**立即初始化**，不能留到以后赋值。

```js
const foo;
// SyntaxError: Missing initializer in const declaration
```

`const`的作用域与`let`命令相同：只在声明所在的块级作用域内有效。

```js
if (true) {
  const MAX = 5;
}

MAX; // Uncaught ReferenceError: MAX is not defined
```

`const`命令声明的常量也是不提升，同样存在暂时性死区，只能在声明的位置后面使用。

```js
if (true) {
  console.log(MAX); // ReferenceError
  const MAX = 5;
}
```

`const`声明的常量，也与`let`一样不可重复声明。

```js
var message = "Hello!";
let age = 25;

// 以下两行都会报错
const message = "Goodbye!";
const age = 30;
```

不与顶层对象挂钩。

```js
const age = 30;
console.log(window.age); // undefined
```

#### 2. 本质

`const`实际上保证的并不是变量的值不得改动，而是变量指向的那个内存地址所保存的数据不得改动。对于简单类型的数据（数值、字符串、布尔值），值就保存在变量指向的那个内存地址，因此等同于常量。但对于复合类型的数据（主要是对象和数组），变量指向的内存地址，保存的只是一个指向实际数据的指针，`const`只能保证这个指针是固定的（即总是指向另一个固定的地址），至于它指向的数据结构是不是可变的，就完全不能控制了。因此，将一个对象声明为常量必须非常小心。

```js
const foo = {};

// 为 foo 添加一个属性，可以成功
foo.prop = 123;
foo.prop // 123

// 将 foo 指向另一个对象，就会报错
foo = {}; // TypeError: "foo" is read-only
```

上面代码中，常量`foo`储存的是一个地址，这个地址指向一个对象。不可变的只是这个地址，即不能把`foo`指向另一个地址，但对象本身是可变的，所以依然可以为其添加新属性。

下面是另一个例子。

```js
const a = [];
a.push('Hello'); // 可执行
a.length = 0;    // 可执行
a = ['Dave'];    // 报错
```

上面代码中，常量`a`是一个数组，这个数组本身是可写的，但是如果将另一个数组赋值给`a`，就会报错。

如果想使用属性不可变的对象，可以使用`freeze`方法，注意该方法只能冻住对象的直接属性，即不保证属性对象的属性不可变。

```js
const obj = Object.freeze({
    name: "zhang3",
    age: 30
});
obj.name = "li4";
console.log(obj);
```

> **提示：let 与 const**
>
> 如果不知道用什么的时候，就用 `const`。
>
> 原因：如果应该是常量，那么刚好符合需求。如果应该是变量，那么后来报错时，再来改为变量也为时不晚。同时，一开始就设置为常量还会避免真的需要为常量时，该值在后来被意外修改的情况。

**let 和 const 总结**

1. `let`声明的变量会产生块作用域，`var`不会产生块作用域
2. `const`声明的常量也会产生块作用域
3. 不同代码块之间的变量无法互相访问
4. 注意: 对象属性修改和数组元素变化不会出发`const`错误 （数组和对象存的是引用地址）
5. 应用场景：声明对象类型使用`const`，非对象类型声明选择`let`
6. `cosnt`声明必须赋初始值，标识符一般为大写，值不允许修改

## 2.解构赋值

ES6 允许按照一定模式从数组和对象中提取值，对变量进行赋值，这被称为解构（Destructuring）。

### 2.1 数组的解构赋值

#### 1.基础使用

以前，为变量赋值只能直接指定值。

```js
let a = 1;
let b = 2;
let c = 3;
```

ES6 允许写成下面这样。

```js
let [a, b, c] = [1, 2, 3];
```

上面代码表示，可以从数组中提取值，按照对应位置，对变量赋值。

1. 模式（结构）匹配 `[] = [1, 2, 3];`
2. 索引值相同的完成赋值 `const [a, b, c] = [1, 2, 3];`

举例：

```js
const [a, [, , b], c] = [1, [2, 3, 4], 5];
console.log(a, b, c);    // 1 4 5
```

#### 2.数组解构赋值的默认值

**（1）默认值的基本用法**

```js
const [a, b] = [];
console.log(a, b);    // undefined undefined

// ---------------------------------------
const [a = 1, b = 2] = [];
console.log(a, b);    // 1 2
```

**（2）默认值的生效条件**

只有当一个数组成员严格等于（===）`undefined`时，对应的默认值才会生效。

```js
const [a = 1, b = 2] = [3, 0];        // 3 0
const [a = 1, b = 2] = [3, null];    // 3 null
const [a = 1, b = 2] = [3];            // 3 2
```

**（3）默认值表达式**

如果默认值是表达式，默认值表达式是惰性求值的（即：当无需用到默认值时，表达式是不会求值的）

```js
const func = () => {
    return 24;
};

const [a = func()] = [1];    // 1
const [b = func()] = [];    // 24
```

#### 3.数组解构赋值的应用

**（1）arguments**

```js
function func() {
    const [a, b] = arguments;
    console.log(a, b);    // 1 2
}
func(1, 2);
```

**（2）NodeList**

```js
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>NodeList</title>
</head>
<body>
    <p>1</p>
    <p>2</p>
    <p>3</p>
    <script>
        const [p1, p2, p3] = document.querySelectorAll('p');
        console.log(p1, p2, p3);
        /*
        <p>1</p>
        <p>2</p>
        <p>3</p>
        */
    </script>
</body>
</html>
```

**（3）函数参数的解构赋值**

```js
const array = [1, 1];
// const add = arr => arr[0] + arr[1];
const add = ([x = 0, y = 0]) => x + y;
console.log(add(array));    // 2
console.log(add([]));        // 0
```

**（4）交换变量的值**

```js
let x = 2, y = 1;

// 原来
let tmp = x;
x = y;
y = tmp;

// 现在
[x, y] = [y, x];
console.log(x, y); // 1 2
```

**（5）跳过某项值使用逗号隔开**

在解构数组时，可以忽略不需要解构的值，可以使用逗号对解构的数组进行忽略操作，这样就不需要声明更多的变量去存值了：

```js
var [a, , , b] = [10, 20, 30, 40];
console.log(a);   // 10
console.log(b);   // 40
```

上面的例子中，在 a、b 中间用逗号隔开了两个值，这里怎么判断间隔几个值呢，可以看出逗号之间组成了多少间隔，就是间隔了多少个值。如果取值很少的情况下可以使用下标索引的方式来获取值。

**（6）剩余参数中的使用**

通常情况下，需要把剩余的数组项作为一个单独的数组，这个时候我们可以借助展开语法把剩下的数组中的值，作为一个单独的数组，如下：

```js
var [a, b, ...rest] = [10, 20, 30, 40, 50];
console.log(a);     // 10
console.log(b);     // 20
console.log(rest);  // [30, 40, 50]
```

在 rest 的后面不能有逗号`,`不然会报错，程序会认出你后面还有值。`...rest`是剩余参数的解构，所以只能放在数组的最后，在它之后不能再有变量，否则会报错。

#### 4.必须要分号的两种情况

```js
// 1. 立即执行函数
// ;(function () { })();
// (function () { })();

// 2. 使用数组解构的时候
// const arr = [1, 2, 3]
const str = 'pink';
[1, 2, 3].map(function (item) {
  console.log(item)
})

let a = 1
let b = 2
;[b, a] = [a, b]
console.log(a, b)
```

> **扩展**
>
> `map(item)`是数组的一个方法，接受两个参数：
>
> - 参数 1：回调函数，参数为每个元素，结果为处理后的元素
> - 参数 2：索引

### 2.2对象的解构赋值

#### 1.基础使用

对象的解构和数组基本类似，对象解构的变量是在`{}`中定义的。对象没有索引，但对象有更明确的键，通过键可以很方便地去对象中取值。在 ES6 之前直接使用键取值已经很方便了：

```js
var obj = {name: 'imooc', age: 7};
var name = obj.name;  // imooc
var age = obj.age;    // 7
```

但是在 ES6 中通过解构的方式，更加简洁地对取值做了简化，不需要通过点操作增加额外的取值操作。

```js
let obj = {name: 'imooc', age: 7};
let { name, age } = obj;  // name: imooc, age: 7
let { age } = obj; // age: 7
```

在`{}`直接声明 name 和 age 用逗号隔开即可得到目标对象上的值，完成声明赋值操作。

1. 模式（结构）匹配`{} = {};`
2. 属性名相同的完成赋值`const {name, age} = {name: 'jerry', age: 18};` 或 `const {age, name} = {name: 'jerry', age: 18};`

#### 2.对象解构赋值的默认值

1. 对象的属性值严格等于`undefined`时，对应的默认值才会生效
2. 如果默认值是表达式，默认值表达式是惰性求值的

对象的默认值和数组的默认值一样，只能通过严格相等运算符`===`来进行判断，只有当一个对象的属性值严格等于`undefined`，默认值才会生效。

```js
var {a = 10, b = 5} = {a: 3};                 // a = 3, b = 5
var {a = 10, b = 5} = {a: 3, b: undefined};   // a = 3, b = 5
var {a = 10, b = 5} = {a: 3, b: null};        // a = 3, b = null
```

所以这里的第二项 b 的值是默认值，第三项的`null === undefined`的值为 false，所以 b 的值为 null。

#### 3.重命名属性

在对象解构出来的变量不是我们想要的变量命名，这时我们需要对它进行重命名。

```js
var {a:x = 8, b:y = 3} = {a: 2};

console.log(x); // 2
console.log(y); // 3
```

这里把 a 和 b 的变量名重新命名为 x 和 y。

#### 4.对象解构赋值的应用

**（1）对象作为函数参数**

```js
// 之前
const logPersonInfo = user => console.log(user.name, user.age);
logPersonInfo({name: 'jerry', age: 18});

// 之后
const logPersonInfo = ({age = 21, name = 'ZJR'}) => console.log(name, age);
logPersonInfo({name: 'jerry', age: 18});    // jerry 18
logPersonInfo({});    // ZJR 21
```

**（2）复杂的嵌套（主要是缕清逻辑关系即可）**

```js
const obj = {
    x: 1,
    y: [2, 3, 4],
    z: {
        a: 5,
        b: 6
    }
};

// ----------------------------------------------------
const {x, y, z} = obj;
console.log(x, y, z);    // 1 [ 2, 3, 4 ] { a: 5, b: 6 }

// ----------------------------------------------------
const {y: [, y2]} = obj;
console.log(y2);    // 3
console.log(y);     // 报错

// ----------------------------------------------------
const {y: y, y: [, y2]} = obj;
console.log(y2);    // 3
console.log(y);     // [ 2, 3, 4 ]

// ----------------------------------------------------
const {y, y: [, y2], z, z: {b}} = obj;
console.log(y2);    // 3
console.log(y);     // [ 2, 3, 4 ]
console.log(z);     // { a: 5, b: 6 }
console.log(b);     // 6
```

**（3）剩余参数中的使用**

在对象的解构中也可以使用剩余参数，对对象中没有解构的剩余属性做聚合操作，生成一个新的对象。

```js
var {a, c, ...rest} = {a: 1, b: 2, c: 3, d: 4}
console.log(a);     // 1
console.log(c);     // 3
console.log(rest);  // { b: 2, d: 4 }
```

对象中的 b、d 没有被解构，通过剩余参数语法把没有解构的对象属性聚合到一起形成新的对象。

#### 5.注意点

（1）如果要将一个已经声明的变量用于解构赋值，必须非常小心。

```js
// 错误的写法
let x;
{x} = {x: 1};
// SyntaxError: syntax error
```

上面代码的写法会报错，因为 JavaScript 引擎会将`{x}`理解成一个代码块，从而发生语法错误。只有不将大括号写在行首，避免 JavaScript 将其解释为代码块，才能解决这个问题。

```js
// 正确的写法
let x;
({x} = {x: 1});
```

上面代码将整个解构赋值语句，放在一个圆括号里面，就可以正确执行。关于圆括号与解构赋值的关系，参见下文。

（2）解构赋值允许等号左边的模式之中不放置任何变量名。因此，可以写出非常古怪的赋值表达式。

```js
({} = [true, false]);
({} = 'abc');
({} = []);
```

上面的表达式虽然毫无意义，但是语法是合法的，可以执行。

（3）由于数组本质是特殊的对象，因此可以对数组进行对象属性的解构。

```js
let arr = [1, 2, 3];
let {0 : first, [1] : second, [arr.length - 1] : last} = arr;
first // 1
second // 2
last // 3
```

上面代码对数组进行对象解构。数组`arr`的`0`键对应的值是`1`，`[arr.length - 1]`就是`2`键，对应的值是`3`。

### 2.3 字符串的解构赋值

既可以用数组的形式来解构赋值，也可以用对象的形式来解构赋值。

```js
// 数组形式解构赋值
const [a, b, , , c] = 'hello';
console.log(a, b, c);    // h e o

// 对象形式解构赋值
const {0: a, 1: b, 4: o, length} = 'hello';
console.log(a, b, o, length);    // h e o 5
```

### 2.4 数值和布尔值的解构赋值

只能按照对象的形式来解构赋值（会先自动将等号右边的值转为对象）。

```js
// 先来复习一下将数值和布尔值转化为对象
console.log(new Number(123));
console.log(new Boolean(true));
// 转化后的对象里没有任何的属性（没有 123 这个属性，也没有 true 这个属性）和方法，
// 所有的属性和方法都在它的继承 __proto__ 中，比如 toString 方法就是继承来的。

// 里面的值只能是默认值，继承的方法倒是可以取到
const {a = 1, toString} = 123;
console.log(a, toString);    // 1 [Function: toString]

const {b = 1, toString} = true;
console.log(b, toString);    // 1 [Function: toString]
```

> 知道有这回事即可，一般都用不到，因为没太大意义。

### 2.5 undefined 和 null 没有解构赋值

由于 undefined 和 null 无法转为对象，所以对它们进行解构赋值，都会报错。

## 3.字符串扩展

### 3.1 模板字符串

普通字符串：

```js
'字符串'
"字符串"
```

模板字符串：

```js
`字符串`
```

#### 1.与一般字符串的区别

对于普通用法没有区别

```js
const name1 = 'zjr';
const name2 = `zjr`;
console.log(name1, name2, name1 === name2);
// zjr zjr true
```

**字符串拼接**时有巨大区别

```js
const person = {
    name: 'zjr',
    age: 18,
    sex: '男'
};
const info =
    '我的名字是：' + person.name +
    '，性别是：' + person.sex +
    '，今年：' + person.age + '岁';
console.log(info); // 我的名字是：zjr，性别是：男，今年：18岁


const person = {
    name: `zjr`,
    age: 18,
    sex: `男`
};
const info = `我的名字是：${person.name}，性别是：${person.sex}，今年：${person.age}岁`;
console.log(info);

// 我的名字是：zjr，性别是：male，今年：18岁
```

> **提示**
>
> 模板字符串最大的优势：方便注入！

#### 2.模板字符串的注意事项

##### 2.1 输出多行字符串

```js
// 一般字符串
const info = "第一行\n第二行";
/*
第一行
第二行
*/


// 模板字符串
const info = `第一行
第二行`;	// 注意不能有缩进
console.log(info);
/*
第一行
第二行
*/
```

> **提示**
>
> 模板字符串中，所有的空格、换行或缩进都会被保存在输出中。

##### 2.2 输出特殊字符

```js
const info = `\``;	// ```
const info = `\\`;	// `\`
const info = `""`;	// `""`
const info = `''`;	// `''`
```

#### 3.模板字符串的注入

```js
const username = 'alex';
const person = {
    age: 18,
    sex: `male`
};
const getSex = function (sex) {
    return sex === `male` ? '男' : '女';
};

const info = `${username},${person.age + 2},${getSex(person.sex)}`;
console.log(info); // alex,20,男
```

> **提示**
>
> 模板字符串的 `${}` 可以写入几乎所有的表达式！例如字符串、数值、布尔值、表达式、函数……（只要结果是个 “值” 即可）。

#### 4.模板字符串的应用

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8"/>
    <title>模板字符串的应用</title>
    <style>
        body {
            padding: 50px 0 0 300px;
            font-size: 22px;
        }

        ul {
            padding: 0;
        }

        p {
            margin-bottom: 10px;
        }
        
        .active {
            background-color: yellow;
        }
    </style>
</head>
<body>
<p>学生信息表</p>
<ul id="list">
    <li style="list-style: none;">信息加载中……</li>
</ul>

<script>
    // 数据（此处只是模拟数据，后期是通过 Ajax 从后台获取）
    const students = [
        {
            username: 'Alex',
            age: 18,
            sex: 'male'
        },
        {
            username: 'ZhangSan',
            age: 28,
            sex: 'male'
        },
        {
            username: 'LiSi',
            age: 20,
            sex: 'female'
        }
    ];

    const list = document.getElementById('list');

    let html = '';

    students.map((item, index) => {
        html += `<li class = "${index == 0 ? 'active' : ''}">我的名字是：${item.username},${item.sex},${item.age}</li>`;
    });

    list.innerHTML = html;
</script>
</body>
</html>
```

学生信息表

- ==我的名字是：Alex,male,18==
- 我的名字是：ZhangSan,male,28
- 我的名字是：LiSi,female,20

### 3.2 方法

#### 1.includes()、startsWith()、endsWith()

传统上，JavaScript 只有`indexOf`方法，可以用来确定一个字符串是否包含在另一个字符串中。ES6 又提供了三种新方法。

- `includes()`：返回布尔值，表示是否找到了参数字符串
- `startsWith()`：返回布尔值，表示参数字符串是否在原字符串的头部
- `endsWith()`：返回布尔值，表示参数字符串是否在原字符串的尾部

```js
let s = 'Hello world!';

s.startsWith('Hello') // true
s.endsWith('!') // true
s.includes('o') // true
```

这三个方法都支持第二个参数，`startsWith`和`includes`表示开始搜索的索引位置（include），`endsWith`表示结束位置的后一个索引（exclude）。

```js
let s = 'Hello world!';

s.startsWith('world', 6) // true
s.endsWith('Hello', 5) // true
s.includes('Hello', 6) // false
```

#### 2.repeat()

`repeat`方法返回一个新字符串，表示将原字符串重复`n`次。

```js
'x'.repeat(3) // "xxx"
'hello'.repeat(2) // "hellohello"
'na'.repeat(0) // ""
```

参数如果是小数会移除小数部分。

```js
'na'.repeat(2.9) // "nana"
```

如果`repeat`的参数是负数或者`Infinity`，会报错。

```js
'na'.repeat(Infinity)
// RangeError
'na'.repeat(-1)
// RangeError
```

但是，如果参数是 0 到 -1 之间的小数，则等同于 0，这是因为会先进行取整运算。0 到 -1 之间的小数，取整以后等于`-0`，`repeat`视同为 0。

```js
'na'.repeat(-0.9) // ""
```

参数`NaN`等同于 0。

```js
'na'.repeat(NaN) // ""
```

如果`repeat`的参数是字符串，则会先转换成数字。

```js
'na'.repeat('na') // ""
'na'.repeat('3') // "nanana"
```

#### 3.padStart()，padEnd()

ES2017 引入了字符串补全长度的功能。如果某个字符串不够指定长度，会在头部或尾部补全。`padStart()`用于头部补全，`padEnd()`用于尾部补全。

```js
'x'.padStart(5, 'ab') // 'ababx'
'x'.padStart(4, 'ab') // 'abax'

'x'.padEnd(5, 'ab') // 'xabab'
'x'.padEnd(4, 'ab') // 'xaba'
```

上面代码中，`padStart()`和`padEnd()`一共接受两个参数，第一个参数是字符串补全生效的最大长度，第二个参数是用来补全的字符串。

如果原字符串的长度，等于或大于最大长度，则字符串补全不生效，返回原字符串。

```js
'xxx'.padStart(2, 'ab') // 'xxx'
'xxx'.padEnd(2, 'ab') // 'xxx'
```

如果用来补全的字符串与原字符串，两者的长度之和超过了最大长度，则会截去超出位数的补全字符串。

```js
'abc'.padStart(10, '0123456789')
// '0123456abc'
```

如果省略第二个参数，默认使用空格补全长度。

```js
'x'.padStart(4) // '   x'
'x'.padEnd(4) // 'x   '
```

`padStart()`的常见用途是为数值补全指定位数。下面代码生成 10 位的数值字符串。

```js
'1'.padStart(10, '0') // "0000000001"
'12'.padStart(10, '0') // "0000000012"
'123456'.padStart(10, '0') // "0000123456"
```

另一个用途是提示字符串格式。

```js
'12'.padStart(10, 'YYYY-MM-DD') // "YYYY-MM-12"
'09-12'.padStart(10, 'YYYY-MM-DD') // "YYYY-09-12"
```

#### 4.trimStart()，trimEnd()

`trimStart()`和`trimEnd()`这两个方法，它们的行为与`trim()`一致，`trimStart()`消除字符串头部的空格，`trimEnd()`消除尾部的空格。它们返回的都是新字符串，不会修改原始字符串。

```js
const s = '  abc  ';

s.trim() // "abc"
s.trimStart() // "abc  "
s.trimEnd() // "  abc"
```

上面代码中，`trimStart()`只消除头部的空格，保留尾部的空格。`trimEnd()`也是类似行为。

除了空格键，这两个方法对字符串头部（或尾部）的 tab 键、换行符等不可见的空白符号也有效。

浏览器还部署了额外的两个方法，`trimLeft()`是`trimStart()`的别名，`trimRight()`是`trimEnd()`的别名。

#### 5.at()

`at()`方法接受一个整数作为参数，返回参数指定位置的字符，支持负索引（即倒数的位置）。

```js
const str = 'hello';
str.at(1) // "e"
str.at(-1) // "o"
```

如果参数位置超出了字符串范围，`at()`返回`undefined`。

## 4.数值扩展

### 4.1 方法属性

#### 1.声明进制

以下写法`var`也可以。

```js
let num1 = 100  // 100
let num2 = 0x100 // 256
let num3 = 0b100 // 4
let num4 = 0o100 // 64
```

#### 2.isFinite()，isNaN()

`Number`的方法与全局的区别是不会对非数值进行转换，而全局的方法会先尝试对非数值进行转换，再进行判断。

```js
isFinite(100)  // true
isFinite(100/0) // false
isFinite(Infinity) // false
isFinite("100") // true，会先转换为数值 100

Number.isFinite(100)  // true
Number.isFinite(100/0) // false
Number.isFinite(Infinity) // false
Number.isFinite("100") // false，直接判断，不会转换为数值
```

```js
Number.isNaN(100) // false
Number.isNaN(NaN) // true
Number.isNaN("100") // false
Number.isNaN("test") // false，直接判断

isNaN(100) // false
isNaN(NaN) // true
isNaN("100") // false
isNaN("test") // true，先尝试转换为数值，为 NaN
```

#### 3.isInteger()

用来判断一个数值是否是整数。

```js
Number.isInteger(100) // true
Number.isInteger(100.0) // true
Number.isInteger(100.1) // false
Number.isInteger("test") // false
Number.isInteger("100") // false
```

#### 4.最小常量

`Number.EPSILON`表示 1 与大于 1 的最小浮点数之间的差 2.220446049250313e^-16^，常用于判断浮点运算结果是否在误差范围内。

```js
function isEqual(a, b) {
    return Math.abs(a - b) < Number.EPSILON;
}

console.log(isEqual(0.1 + 0.2, 0.3)); // false
console.log(0.1 + 0.2 === 0.3); // false
```

#### 5.trunc()

舍弃小数部分直接返回整数。

```js
Math.trunc(1.2) // 1
Math.trunc(1.8) // 1
Math.trunc(-1.2) // -1
Math.trunc(-1.8) // -1
```

#### 6.sign()

判断一个数是正数、负数还是 0，对于非数值会先将其转换为数值。

```js
Math.sign(-100) // -1
Math.sign(100) // +1
Math.sign(0) // +0
Math.sign(-0) // -0
Math.sign("test") // NaN
```

## 5.数组扩展

### 5.1 扩展运算符

#### 1.含义

扩展运算符是三个点`...`。它好比`rest`参数的逆运算，将一个数组转为用逗号分隔的参数序列。

```js
console.log(...[1, 2, 3])
// 1 2 3

console.log(1, ...[2, 3, 4], 5)
// 1 2 3 4 5

[...document.querySelectorAll('div')]
// [<div>, <div>, <div>]
```

该运算符主要用于函数调用。

```js
function push(array, ...items) {
  array.push(...items);
}

function add(x, y) {
  return x + y;
}

const numbers = [4, 38];
add(...numbers) // 42
```

上面代码中，`array.push(...items)`和`add(...numbers)`这两行，都是函数的调用，它们都使用了扩展运算符。该运算符将一个数组，变为参数序列。

扩展运算符与正常的函数参数可以结合使用，非常灵活。

```js
function f(v, w, x, y, z) { }
const args = [0, 1];
f(-1, ...args, 2, ...[3]);
```

扩展运算符后面还可以放置表达式。

```js
const arr = [
  ...(x > 0 ? ['a'] : []),
  'b',
];
```

如果扩展运算符后面是一个空数组，则不产生任何效果。

```js
[...[], 1]
// [1]
```

注意，只有函数调用时，扩展运算符才可以放在圆括号中，否则会报错。

```js
(...[1, 2])
// Uncaught SyntaxError: Unexpected number

console.log((...[1, 2]))
// Uncaught SyntaxError: Unexpected number

console.log(...[1, 2])
// 1 2
```

上面三种情况，扩展运算符都放在圆括号里面，但是前两种情况会报错，因为扩展运算符所在的括号不是函数调用。

#### 1.2 替代函数的 apply()

由于扩展运算符可以展开数组，所以不再需要`apply()`方法将数组转为函数的参数了。

```js
// ES5 的写法
function f(x, y, z) {
  // ...
}
var args = [0, 1, 2];
f.apply(null, args);

// ES6 的写法
function f(x, y, z) {
  // ...
}
let args = [0, 1, 2];
f(...args);
```

下面是扩展运算符取代`apply()`方法的一个实际的例子，应用`Math.max()`方法，简化求出一个数组最大元素的写法。

```js
// ES5 的写法
Math.max.apply(null, [14, 3, 77])

// ES6 的写法
Math.max(...[14, 3, 77])

// 等同于
Math.max(14, 3, 77);
```

上面代码中，由于 JavaScript 不提供求数组最大元素的函数，所以只能套用`Math.max()`函数，将数组转为一个参数序列，然后求最大值。有了扩展运算符以后，就可以直接用`Math.max()`了。

另一个例子是通过`push()`函数，将一个数组添加到另一个数组的尾部。

```js
// ES5 的写法
var arr1 = [0, 1, 2];
var arr2 = [3, 4, 5];
Array.prototype.push.apply(arr1, arr2);

// ES6 的写法
let arr1 = [0, 1, 2];
let arr2 = [3, 4, 5];
arr1.push(...arr2);
```

上面代码的 ES5 写法中，`push()`方法的参数不能是数组，所以只好通过`apply()`方法变通使用`push()`方法。有了扩展运算符，就可以直接将数组传入`push()`方法。

下面是另外一个例子。

```js
// ES5
new (Date.bind.apply(Date, [null, 2015, 1, 1]))

// ES6
new Date(...[2015, 1, 1]);
```

#### 1.3 扩展运算符的应用

**（1）复制数组**

数组是复合的数据类型，直接复制的话，只是复制了指向底层数据结构的指针，而不是克隆一个全新的数组。

```js
const a1 = [1, 2];
const a2 = a1;

a2[0] = 2;
a1 // [2, 2]
```

上面代码中，`a2`并不是`a1`的克隆，而是指向同一份数据的另一个指针。修改`a2`，会直接导致`a1`的变化。

ES5 只能用变通方法来复制数组。

```js
const a1 = [1, 2];
const a2 = a1.concat();

a2[0] = 2;
a1 // [1, 2]
```

上面代码中，`a1`会返回原数组的克隆，再修改`a2`就不会对`a1`产生影响。

扩展运算符提供了复制数组的简便写法。

```js
const a1 = [1, 2];
// 写法
const a2 = [...a1];
```

上面的两种写法，`a2`都是`a1`的克隆。

**（2）合并数组**

扩展运算符提供了数组合并的新写法。

```js
const arr1 = ['a', 'b'];
const arr2 = ['c'];
const arr3 = ['d', 'e'];

// ES5 的合并数组
arr1.concat(arr2, arr3);
// [ 'a', 'b', 'c', 'd', 'e' ]

// ES6 的合并数组
[...arr1, ...arr2, ...arr3]
// [ 'a', 'b', 'c', 'd', 'e' ]
```

不过，这两种方法都是浅拷贝，使用的时候需要注意。

```js
const a1 = [{ foo: 1 }];
const a2 = [{ bar: 2 }];

const a3 = a1.concat(a2);
const a4 = [...a1, ...a2];

a3[0] === a1[0] // true
a4[0] === a1[0] // true
```

上面代码中，`a3`和`a4`是用两种不同方法合并而成的新数组，但是它们的成员都是对原数组成员的引用，这就是浅拷贝。如果修改了引用指向的值，会同步反映到新数组。

**(4）字符串转为数组**

扩展运算符还可以将字符串转为真正的数组。

```js
console.log(...'alex'); // a l e x
console.log('a', 'l', 'e', 'x'); // a l e x

console.log([...'alex']); // [ 'a', 'l', 'e', 'x' ]
// ES6 之前字符串转数组是通过：'alex'.split('');
```

**（5）类数组转为数组**

```js
// arguments
function func() {
    console.log(arguments); // [Arguments] { '0': 1, '1': 2 }
    console.log([...arguments]); // [1, 2]
}
func(1, 2);

// NodeList
console.log(document.querySelectorAll('p'));// NodeList [<p>, <p>, <p>, <p>]
console.log([...document.querySelectorAll('p')]); // [<p>, <p>, <p>, <p>]
```

### 5.2 方法

#### 1.Array.from()

在前端开发中经常会遇到类数组，但是我们不能直接使用数组的方法，需要先把类数组转化为数组。本节介绍 ES6 数组的新增方法 `Array.from()`，该方法用于将类数组对象（array-like）和可遍历的对象（iterable）转换为真正的数组进行使用。

`Array.from()` 方法会接收一个类数组对象然后返回一个真正的数组实例，返回的数组可以调用数组的所有方法。

**语法使用：**

```js
Array.from(arrayLike[, mapFn[, thisArg]])
```

| 参数      | 描述                                                 |
| --------- | ---------------------------------------------------- |
| arrayLike | 想要转换成数组的类数组对象或可迭代对象               |
| mapFn     | 如果指定了该参数，新数组中的每个元素会执行该回调函数 |
| thisArg   | 可选参数，执行回调函数 mapFn 时 this 对象            |

##### 1.1 类数组转化

所谓类数组对象，就是指可以通过索引属性访问元素，并且对象拥有 length 属性，类数组对象一般是以下这样的结构：

```js
var arrLike = {
  '0': 'apple',
  '1': 'banana',
  '2': 'orange',
  length: 3
};
```

在 ES5 中没有对应的方法将类数组转化为数组，但是可以借助 call 和 apply 来实现：

```js
var arr = [].slice.call(arrLike);
// 或
var arr = [].slice.apply(arrLike);
```

有了 ES6 的 `Array.from()` 就更简单了，对类数组对象直接操作，即可得到数组。

```js
var arr = Array.from(arrLike);
console.log(arr)  // ['apple', 'banana', 'orange']
```

##### 1.2 第二个参数 —— 回调函数

在 `Array.from` 中第二个参数是一个类似 map 函数的回调函数，该回调函数会依次接收数组中的每一项作为传入的参数，然后对传入值进行处理，最得到一个新的数组。 `Array.from(obj, mapFn, thisArg)` 也可以用 map 改写成这样 `Array.from(obj).map(mapFn, thisArg)`。

```js
var arr = Array.from([1, 2, 3], function (x) {
  return 2 * x;
});
var arr = Array.from([1, 2, 3]).map(function (x) {
  return 2 * x;
});
//arr: [2, 4, 6]
```

上面的例子展示了，`Array.from` 的参数可以使用 `map` 方法来进行替换，它们是等价的操作。

##### 1.3 第三个参数 ——this

`Array.from` 中第三个参数可以对回调函数中 this 的指向进行绑定，该参数是非常有用的，我们可以将被处理的数据和处理对象分离，将各种不同的处理数据的方法封装到不同的的对象中去，处理方法采用相同的名字。

在调用 `Array.from` 对数据对象进行转换时，可以将不同的处理对象按实际情况进行注入，以得到不同的结果，适合解耦。

```js
let obj = {
  handle: function(n){
    return n + 2
  }
}

Array.from([1, 2, 3, 4, 5], function (x){
  return this.handle(x)
}, obj)
// [3, 4, 5, 6, 7]
```

定义一个 `obj` 对象可以认作是，`Array.from` 回调函数中处理数据的方法集合，`handle` 是其中的一个方法，把 `obj` 作为第三个参数传给 `Array.from` 这样在回调函数中可以通过 `this` 来拿到 `obj` 对象。

##### 1.4 生成数组

**（1）从字符串里生成数组**

`Array.from()` 在传入字符串时，会把字符串的每一项都拆成单个的字符串作为数组中的一项。

```js
Array.from('imooc'); 
// [ "i", "m", "o", "o", "c" ]
```

**（2）从 Set 中生成数组**

用 `Set` 定义的数组对象，可以使用 `Array.from()` 得到一个正常的数组。

```js
const set = new Set(['a', 'b', 'c', 'd']);
Array.from(set);
// [ "a", "b", "c", "d" ]
```

上面的代码中创建了一个 Set 数据结构，把实例传入 `Array.from()` 可以得到一个真正的数组。

**（3）从 Map 中生成数组**

`Map` 对象保存的是一个个键值对，`Map` 中的参数是一个数组或是一个可迭代的对象。 `Array.from()` 可以把 Map 实例转换为一个二维数组。

```js
const map = new Map([[1, 2], [2, 4], [4, 8]]);

Array.from(map);  // [[1, 2], [2, 4], [4, 8]]
```

##### 1.5 使用案例

**（1）创建一个包含从 0 到 99 (n) 的连续整数的数组**

1. 一般情况下我们可以使用 for 循环来实现

   ```js
   var arr = [];
   for(var i = 0; i <= 99; i++) {
     arr.push(i);
   }
   ```

   这种方法的主要优点是最直观了，性能也最好的，但是很多时候我们不想使用 for 循环来进行操作。

2. 使用 Array 配合 map 来实现

   ```js
   var arr = Array(100).join(' ').split('').map(function(item,index){return index});
   ```

   Array (100) 创建了一个包含 100 个空位的数组，但是这样创建出来的数组是没法进行迭代的。所以要通过字符串转换，覆盖 undefined，最后调用 map 修改元素值。

3. 使用 es6 的 `Array.from` 实现

   ```js
   var arr = Array.from({length:100}).map(function(item,index){return index});
   ```

   `Array.from({length:100})` 可以定义一个可迭代的数组，数组的每一项都是 undefined，这样就非常方便的定义出所需要的数组了，但是这样定义的数组性能最差，具体可以参考 [constArray](https://jsperf.com/constarray/4) 的测试结果。

**（3）数组去重合并**

```js
function combine(){ 
  // let arr = [].concat(...arguments);
  let arr = [].concat.apply([], arguments); //没有去重复的新数组 
  return Array.from(new Set(arr));
} 

var m = [1, 2, 2], n = [2,3,3]; 
console.log(combine(m,n)); // [1, 2, 3]
```

首先定义一个去重数组函数，通过 `concat` 把传入的数组进行合并到一个新的数组中去，通过 `new Set()` 可以对 arr 进行去重操作，再使用 `Array.from()` 返回一个拷贝后的数组。

### 2.Array.of()

`Array.of()`方法用于将一组值，转换为数组。

```js
Array.of(3, 11, 8) // [3,11,8]
Array.of(3) // [3]
Array.of(3).length // 1
```

这个方法的主要目的，是弥补数组构造函数`Array()`的不足。因为参数个数的不同，会导致`Array()`的行为有差异。

```js
Array() // []
Array(3) // [, , ,]
Array(3, 11, 8) // [3, 11, 8]
```

上面代码中，`Array()`方法没有参数、一个参数、三个参数时，返回的结果都不一样。只有当参数个数不少于 2 个时，`Array()`才会返回由参数组成的新数组。参数只有一个正整数时，实际上是指定数组的长度。

`Array.of()`基本上可以用来替代`Array()`或`new Array()`，并且不存在由于参数不同而导致的重载。它的行为非常统一。

```js
Array.of() // []
Array.of(undefined) // [undefined]
Array.of(1) // [1]
Array.of(1, 2) // [1, 2]
```

`Array.of()`总是返回参数值组成的数组。如果没有参数，就返回一个空数组。

`Array.of()`方法可以用下面的代码模拟实现。

```js
function ArrayOf(){
  return [].slice.call(arguments);
}
```

## 4.find()，findIndex(),findLast()，findLastIndex()

数组实例的`find()`方法，用于找出第一个符合条件的数组成员。它的参数是一个回调函数，所有数组成员依次执行该回调函数，直到找出第一个返回值为`true`的成员，然后返回该成员。如果没有符合条件的成员，则返回`undefined`。

```
[1, 4, -5, 10].find((n) => n < 0)
// -5
```

上面代码找出数组中第一个小于 0 的成员。

```
[1, 5, 10, 15].find(function(value, index, arr) {
  return value > 9;
}) // 10
```

上面代码中，`find()`方法的回调函数可以接受三个参数，依次为当前的值、当前的位置和原数组。

数组实例的`findIndex()`方法的用法与`find()`方法非常类似，返回第一个符合条件的数组成员的位置，如果所有成员都不符合条件，则返回`-1`。

```
[1, 5, 10, 15].findIndex(function(value, index, arr) {
  return value > 9;
}) // 2
```

这两个方法都可以接受第二个参数，用来绑定回调函数的`this`对象。

```
function f(v){
  return v > this.age;
}
let person = {name: 'John', age: 20};
[10, 12, 26, 15].find(f, person);    // 26
```

上面的代码中，`find()`函数接收了第二个参数`person`对象，回调函数中的`this`对象指向`person`对象。

另外，这两个方法都可以发现`NaN`，弥补了数组的`indexOf()`方法的不足。

```
[NaN].indexOf(NaN)
// -1

[NaN].findIndex(y => Object.is(NaN, y))
// 0
```

上面代码中，`indexOf()`方法无法识别数组的`NaN`成员，但是`findIndex()`方法可以借助`Object.is()`方法做到。

`find()`和`findIndex()`都是从数组的0号位，依次向后检查。[ES2022](https://github.com/tc39/proposal-array-find-from-last) 新增了两个方法`findLast()`和`findLastIndex()`，从数组的最后一个成员开始，依次向前检查，其他都保持不变。

```
const array = [
  { value: 1 },
  { value: 2 },
  { value: 3 },
  { value: 4 }
];

array.findLast(n => n.value % 2 === 1); // { value: 3 }
array.findLastIndex(n => n.value % 2 === 1); // 2
```

上面示例中，`findLast()`和`findLastIndex()`从数组结尾开始，寻找第一个`value`属性为奇数的成员。结果，该成员是`{ value: 3 }`，位置是2号位。

## 5.filter()

`filter()`方法用于过滤数组成员，满足条件的成员组成一个新数组返回。

它的参数是一个函数，所有数组成员依次执行该函数，返回结果为`true`的成员组成一个新数组返回。该方法不会改变原数组。

```
[1, 2, 3, 4, 5].filter(function (elem) {
  return (elem > 3);
})
// [4, 5]
```

上面代码将大于`3`的数组成员，作为一个新数组返回。

```
var arr = [0, 1, 'a', false];

arr.filter(Boolean)
// [1, "a"]
```

上面代码中，`filter()`方法返回数组`arr`里面所有布尔值为`true`的成员。

`filter()`方法的参数函数可以接受三个参数：当前成员，当前位置和整个数组。

```
[1, 2, 3, 4, 5].filter(function (elem, index, arr) {
  return index % 2 === 0;
});
// [1, 3, 5]
```

上面代码返回偶数位置的成员组成的新数组。

`filter()`方法还可以接受第二个参数，用来绑定参数函数内部的`this`变量。

```
var obj = { MAX: 3 };
var myFilter = function (item) {
  if (item > this.MAX) return true;
};

var arr = [2, 8, 3, 4, 1, 3, 2, 9];
arr.filter(myFilter, obj) // [8, 4, 9]
```

上面代码中，过滤器`myFilter()`内部有`this`变量，它可以被`filter()`方法的第二个参数`obj`绑定，返回大于`3`的成员。

## 6.map()

`map()`方法将数组的所有成员依次传入参数函数，然后把每一次的执行结果组成一个新数组返回。

```
var numbers = [1, 2, 3];

numbers.map(function (n) {
  return n + 1;
});
// [2, 3, 4]

numbers
// [1, 2, 3]
```

上面代码中，`numbers`数组的所有成员依次执行参数函数，运行结果组成一个新数组返回，原数组没有变化。

`map()`方法接受一个函数作为参数。该函数调用时，`map()`方法向它传入三个参数：当前成员、当前位置和数组本身。

```
[1, 2, 3].map(function(elem, index, arr) {
  return elem * index;
});
// [0, 2, 6]
```

上面代码中，`map()`方法的回调函数有三个参数，`elem`为当前成员的值，`index`为当前成员的位置，`arr`为原数组（`[1, 2, 3]`）。

`map()`方法还可以接受第二个参数，用来绑定回调函数内部的`this`变量（详见《this 变量》一章）。

```
var arr = ['a', 'b', 'c'];

[1, 2].map(function (e) {
  return this[e];
}, arr)
// ['b', 'c']
```

上面代码通过`map()`方法的第二个参数，将回调函数内部的`this`对象，指向`arr`数组。

如果数组有空位，`map()`方法的回调函数在这个位置不会执行，会跳过数组的空位。

```
var f = function (n) { return 'a' };

[1, undefined, 2].map(f) // ["a", "a", "a"]
[1, null, 2].map(f) // ["a", "a", "a"]
[1, , 2].map(f) // ["a", , "a"]
```

上面代码中，`map()`方法不会跳过`undefined`和`null`，但是会跳过空位。

## 7.reduce()

`reduce()`方法依次处理数组的每个成员，最终累计为一个值。它们的差别是，`reduce()`是从左到右处理（从第一个成员到最后一个成员）。

语法:`arr.reduce(function(累计值, 当前元素){}, 起始值)`

```
[1, 2, 3, 4, 5].reduce(function (a, b) {
  console.log(a, b);
  return a + b;
})
// 1 2
// 3 3
// 6 4
// 10 5
//最后结果：15
```

上面代码中，`reduce()`方法用来求出数组所有成员的和。`reduce()`的参数是一个函数，数组每个成员都会依次执行这个函数。如果数组有 n 个成员，这个参数函数就会执行 n - 1 次。

- 第一次执行：`a`是数组的第一个成员`1`，`b`是数组的第二个成员`2`。
- 第二次执行：`a`为上一轮的返回值`3`，`b`为第三个成员`3`。
- 第三次执行：`a`为上一轮的返回值`6`，`b`为第四个成员`4`。
- 第四次执行：`a`为上一轮返回值`10`，`b`为第五个成员`5`。至此所有成员遍历完成，整个方法的返回值就是最后一轮的返回值`15`。

`reduce()`方法的第一个参数都是一个函数。该函数接受以下四个参数。

1. 累积变量。第一次执行时，默认为数组的第一个成员；以后每次执行时，都是上一轮的返回值。
2. 当前变量。第一次执行时，默认为数组的第二个成员；以后每次执行时，都是下一个成员。
3. 当前位置。一个整数，表示第二个参数（当前变量）的位置，默认为`1`。
4. 原数组。

这四个参数之中，只有前两个是必须的，后两个则是可选的。

```
[1, 2, 3, 4, 5].reduce(function (
  a,   // 累积变量，必须
  b,   // 当前变量，必须
  i,   // 当前位置，可选
  arr  // 原数组，可选
) {
  // ... ...
```

如果要对累积变量指定初值，可以把它放在`reduce()`方法的第二个参数。

```
[1, 2, 3, 4, 5].reduce(function (a, b) {
  return a + b;
}, 10);
// 25
```

上面代码指定参数`a`的初值为10，所以数组从10开始累加，最终结果为25。注意，这时`b`是从数组的第一个成员开始遍历，参数函数会执行5次。

建议总是加上第二个参数，这样比较符合直觉，每个数组成员都会依次执行`reduce()`方法的参数函数。另外，第二个参数可以防止空数组报错。

```
function add(prev, cur) {
  return prev + cur;
}

[].reduce(add)
// TypeError: Reduce of empty array with no initial value
[].reduce(add, 1)
// 1
```

上面代码中，由于空数组取不到累积变量的初始值，`reduce()`方法会报错。这时，加上第二个参数，就能保证总是会返回一个值。

**总结**

```
//reduce 返回函数累计处理的结果，经常用于求和等
/*
计值参数：
1. 如果有起始值，则以起始值为准开始累计， 累计值 = 起始值
2. 如果没有起始值， 则累计值以数组的第一个数组元素作为起始值开始累计
3. 后面每次遍历就会用后面的数组元素 累计到 累计值 里面（类似求和里面的 sum ）
*/
```

## 8.some()，every()

这两个方法类似“断言”（assert），返回一个布尔值，表示判断数组成员是否符合某种条件。

它们接受一个函数作为参数，所有数组成员依次执行该函数。该函数接受三个参数：当前成员、当前位置和整个数组，然后返回一个布尔值。

`some`方法是只要一个成员的返回值是`true`，则整个`some`方法的返回值就是`true`，否则返回`false`。

```
var arr = [1, 2, 3, 4, 5];
arr.some(function (elem, index, arr) {
  return elem >= 3;
});
// true
```

上面代码中，如果数组`arr`有一个成员大于等于3，`some`方法就返回`true`。

`every`方法是所有成员的返回值都是`true`，整个`every`方法才返回`true`，否则返回`false`。

```
var arr = [1, 2, 3, 4, 5];
arr.every(function (elem, index, arr) {
  return elem >= 3;
});
// false
```

上面代码中，数组`arr`并非所有成员大于等于`3`，所以返回`false`。

注意，对于空数组，`some`方法返回`false`，`every`方法返回`true`，回调函数都不会执行。

```
function isEven(x) { return x % 2 === 0 }

[].some(isEven) // false
[].every(isEven) // true
```

`some`和`every`方法还可以接受第二个参数，用来绑定参数函数内部的`this`变量。

## 9.fill()

`arr.fill(value[, start[, end]])`方法用一个固定值填充一个数组中从起始索引到终止索引内的全部元素。不包括终止索引。 起始索引，默认值为 0。 终止索引，默认值为 this.length。

```
['a', 'b', 'c'].fill(7)
// [7, 7, 7]

new Array(3).fill(7)
// [7, 7, 7]
```

上面代码表明，`fill`方法用于空数组的初始化非常方便。数组中已有的元素，会被全部抹去。

`fill`方法还可以接受第二个和第三个参数，用于指定填充的起始位置和结束位置。

```
['a', 'b', 'c'].fill(7, 1, 2)
// ['a', 7, 'c']
```

上面代码表示，`fill`方法从 1 号位开始，向原数组填充 7，到 2 号位之前结束。

注意，如果填充的类型为对象，那么被赋值的是同一个内存地址的对象，而不是深拷贝对象。

```
let arr = new Array(3).fill({name: "Mike"});
arr[0].name = "Ben";
arr
// [{name: "Ben"}, {name: "Ben"}, {name: "Ben"}]

let arr = new Array(3).fill([]);
arr[0].push(5);
arr
// [[5], [5], [5]]
```

## 10.at()

长久以来，JavaScript 不支持数组的负索引，如果要引用数组的最后一个成员，不能写成`arr[-1]`，只能使用`arr[arr.length - 1]`。

这是因为方括号运算符`[]`在 JavaScript 语言里面，不仅用于数组，还用于对象。对于对象来说，方括号里面就是键名，比如`obj[1]`引用的是键名为字符串`1`的键，同理`obj[-1]`引用的是键名为字符串`-1`的键。由于 JavaScript 的数组是特殊的对象，所以方括号里面的负数无法再有其他语义了，也就是说，不可能添加新语法来支持负索引。

为了解决这个问题，ES2022为数组实例增加了`at()`方法，接受一个整数作为参数，返回对应位置的成员，并支持负索引。这个方法不仅可用于数组，也可用于字符串和类型数组（TypedArray）。

```
const arr = [5, 12, 8, 130, 44];
arr.at(2) // 8
arr.at(-2) // 130
```

如果参数位置超出了数组范围，`at()`返回`undefined`。

```
const sentence = 'This is a sample sentence';

sentence.at(0); // 'T'
sentence.at(-1); // 'e'

sentence.at(-100) // undefined
sentence.at(100) // undefined
```

## 11.entries()，keys() 和 values()

ES6 提供三个新的方法——`entries()`，`keys()`和`values()`——用于遍历数组。它们都返回一个遍历器对象，可以用`for...of`循环进行遍历，唯一的区别是`keys()`是对键名的遍历、`values()`是对键值的遍历，`entries()`是对键值对的遍历。

```
for (let index of ['a', 'b'].keys()) {
  console.log(index);
}
// 0
// 1

for (let elem of ['a', 'b'].values()) {
  console.log(elem);
}
// 'a'
// 'b'

for (let [index, elem] of ['a', 'b'].entries()) {
  console.log(index, elem);
}
// 0 "a"
// 1 "b"
```

## 12.includes()

`Array.prototype.includes`方法返回一个布尔值，表示某个数组是否包含给定的值，与字符串的`includes`方法类似。ES2016 引入了该方法。

```
[1, 2, 3].includes(2)     // true
[1, 2, 3].includes(4)     // false
[1, 2, NaN].includes(NaN) // true
```

该方法的第二个参数表示搜索的起始位置，默认为`0`。如果第二个参数为负数，则表示倒数的位置，如果这时它大于数组长度（比如第二个参数为`-4`，但数组长度为`3`），则会重置为从`0`开始。

```
[1, 2, 3].includes(3, 3);  // false
[1, 2, 3].includes(3, -1); // true
```

没有该方法之前，我们通常使用数组的`indexOf`方法，检查是否包含某个值。

```
if (arr.indexOf(el) !== -1) {
  // ...
}
```

`indexOf`方法有两个缺点，一是不够语义化，它的含义是找到参数值的第一个出现位置，所以要去比较是否不等于`-1`，表达起来不够直观。二是，它内部使用严格相等运算符（`===`）进行判断，这会导致对`NaN`的误判。

```
[NaN].indexOf(NaN)
// -1
```

`includes`使用的是不一样的判断算法，就没有这个问题。

```
[NaN].includes(NaN)
// true
```

另外，Map 和 Set 数据结构有一个`has`方法，需要注意与`includes`区分。

- Map 结构的`has`方法，是用来查找键名的，比如`Map.prototype.has(key)`、`WeakMap.prototype.has(key)`、`Reflect.has(target, propertyKey)`。
- Set 结构的`has`方法，是用来查找值的，比如`Set.prototype.has(value)`、`WeakSet.prototype.has(value)`。

## 13.toReversed()，toSorted()，toSpliced()，with()

很多数组的传统方法会改变原数组，比如`push()`、`pop()`、`shift()`、`unshift()`等等。数组只要调用了这些方法，它的值就变了。现在有一个提案，允许对数组进行操作时，不改变原数组，而返回一个原数组的拷贝。

这样的方法一共有四个。

- `Array.prototype.toReversed() -> Array`
- `Array.prototype.toSorted(compareFn) -> Array`
- `Array.prototype.toSpliced(start, deleteCount, ...items) -> Array`
- `Array.prototype.with(index, value) -> Array`

它们分别对应数组的原有方法。

- `toReversed()`对应`reverse()`，用来颠倒数组成员的位置。
- `toSorted()`对应`sort()`，用来对数组成员排序。
- `toSpliced()`对应`splice()`，用来在指定位置，删除指定数量的成员，并插入新成员。
- `with(index, value)`对应`splice(index, 1, value)`，用来将指定位置的成员替换为新的值。

上面是这四个新方法对应的原有方法，含义和用法完全一样，唯一不同的是不会改变原数组，而是返回原数组操作后的拷贝。

下面是示例。

```
const sequence = [1, 2, 3];
sequence.toReversed() // [3, 2, 1]
sequence // [1, 2, 3]

const outOfOrder = [3, 1, 2];
outOfOrder.toSorted() // [1, 2, 3]
outOfOrder // [3, 1, 2]

const array = [1, 2, 3, 4];
array.toSpliced(1, 2, 5, 6, 7) // [1, 5, 6, 7, 4]
array // [1, 2, 3, 4]

const correctionNeeded = [1, 1, 3];
correctionNeeded.with(1, 2) // [1, 2, 3]
correctionNeeded // [1, 1, 3]
```

## 14.isArray()

### 14.1 前言

在程序中判断数组是很常见的应用，但在 ES5 中没有能严格判断 JS 对象是否为数组，都会存在一定的问题，比较受广大认可的是借助 toString 来进行判断，很显然这样不是很简洁。ES6 提供了 `Array.isArray()` 方法更加简洁地判断 JS 对象是否为数组。

### 14.2 方法详情

判断 JS 对象，如果值是 `Array`，则为 true; 否则为 false。

**语法使用：**

```
Array.isArray(obj)
```

**参数解释：**

| 参数 | 描述               |
| ---- | ------------------ |
| obj  | 需要检测的 JS 对象 |

### 14.3 ES5 中判断数组的方法

通常使用 `typeof` 来判断变量的数据类型，但是对数组得到不一样的结果

```
// 基本类型
typeof 123;  //number
typeof "123"; //string
typeof true; //boolean

// 引用类型
typeof [1,2,3]; //object
```

上面的代码中，对于基本类型的判断没有问题，但是判断数组时，返回了 object 显然不能使用 `typeof` 来作为判断数组的方法。

#### 14.3.1 通过 instanceof 判断

`instanceof` 运算符用于检测构造函数的 `prototype` 属性是否出现在某个实例对象的原型链。

`instanceof` 可以用来判断数组是否存在，判断方式如下：

```
var arr = ['a', 'b', 'c'];
console.log(arr instanceof Array);            // true 
console.log(arr.constructor === Array;); // true
```

在解释上面的代码时，先看下数组的原型链指向示意图：
