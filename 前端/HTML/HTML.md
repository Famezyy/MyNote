# HTML

## 1.前言

==W3C==：制定网页开发的标准，为了使同一个网页在不同的浏览器中有相同的效果，伯纳斯李1994年建立了万维网联盟（W3C）

- 根据 W3C 标准，一个网页主要由三部分组成：结构，表现和行为

  ```mermaid
  graph TD
  W3C标准 --> 结构
  结构 --> HTML
  W3C标准 --> 表现
  表现 --> CSS
  W3C标准 --> 行为
  行为 --> JavaScript
  ```

  - **结构**：HTML 用于描述页面的结构
  - **表现**：CSS 用于控制页面中的元素的样式
  - **行为**：JavaScript 用于响应用户操作

- [W3shool：使用文档](https://www.w3school.com.cn/h.asp)

## 2.HTML简介

HTML：超文本标记语言（Hypertext Markup Language），负责网页三要素中的**结构**。

HTML 使用**标签**的形式来标识网页中的不同组成部分。

- 一般成对出现：`<head> </head>`
- 自结束标签：没有终止标签，例如`<img>, <input>`，也可以写成 `<img />`，`<input />`
- 注释：`<!-- -->`，注释的内容不会在网页中显示，可以在源码中查看，**不能嵌套**

所谓超文本指的是超链接，使用超链接可以让我们从一个页面跳转到另一个页面。

简单示例：

```html
<html>
    <!-- head 中的内容不会出现在网页中，用于浏览器解析 -->
    <head>
        <!-- title 中的内容会显示在标题行 -->
        <title>title</title>
    </head>
    <!-- body 中的内容才会出现在网页中 -->
    <body>
        <h1>主题</h1>
        <p>一句话</p>
    </body>
</html>
```

### 2.1 标签中的标签

```html
<html>
	<head>
		<title>标签的属性</title>
	</head>
	<body>
		<!--
			1. 在标签中可以设置属性，属性是一个键值对
			2. 属性用来设置标签中的内容如何显示
			3. 属性和标签名或其他属性应该是用空格隔开
			4. 属性名要根据文档中的规定来编写
			5. 有些属性没有值，有值时，应该使用引号引起来
		-->
		<h1>
			第<font color="blue" size="3">3</font>天
		</h1>
	</body>
</html>
```

### 2.2 文档声明

```html
<!--
	文档声明（doctype）：用来告诉浏览器当前网页的版本
-->
<!doctype html>
<html>
	<head>
		<title>网页的基本结构</title>
	</head>
	<body>
	</body>
</html>
```

### 2.3 进制

最小可操作单位为 1 字节，1 byte（字节） = 8 bit，1 kb = 1024 byte。

### 2.4 字符编码

所有数据在计算机中存储时都是以二进制形式存储的，所以一段文字在存储到内存中时，都需要转换为二进制编码，当读取这段文字时，计算机会转换为字符

- **编码**：将字符转换为二进制码的过程
- **解码**：将二进制码转换为字符的过程
- 字符集（charset）
- 编码和解码所采用的规则称为字符集
- 若编码和解码所采用的字符集不同就会出现乱码问题
- 常见字符集：ASCII，ISO88591，GB2312，GBK，UTF-8（万国码）

```html
<!doctype html>
<html>
    <head>
        <!-- 通过 meta 标签来设置网页的字符集 -->
        <meta charset="utf-8">
        <title>网页的基本结构</title>
    </head>
    <body>
    </body>
</html>
```

### 2.5 小结

```html
<!-- 文档声明，声明当前网页的版本 -->
<!doctype html>
<!-- html 的根标签（根元素），网页中的所有内容都要写在根元素中 -->
<html>
	<!-- 网页的头部，head中的内容不会在网页中直接出现，主要用来帮助浏览器或搜索引擎来解析网页 -->
	<head>
		<!-- meta 标签用来设置网页的元数据，这里 meta 用来设置字符集 -->
		<meta charset="utf-8">
		<!-- title 中的内容会显示在浏览器的标题栏，搜索引擎会根据 title 中的内容来判断网页的主要内容 -->
		<title>网页的标题</title>
	</head>
	<!-- body 是 html 的子元素，表示网页的主题，网页中所有可见内容都应该写在 body 中 -->
	<body>
	<!-- h1 是网页的一级标题 -->
	<h1>网页大标题</h1>
	</body>
	
</html>
```

### 2.6 VS Code

- ctrl + /：注释
- ! + tab：快速生成基本标签
- live server 插件：动态刷新页面（html 文件需要放在文件夹中）
- 设置 -> Auto Save -> afterDelay
- ctrl + 回车：光标移动到下一行
- alt + shift + 方向键：当前行向上或下复制
- alt + 回车 + 方向键：当前行向上或向下移动
- `父元素>子元素`：自动生成父子元素

## 3.语法

### 3.1 实体

[文档](https://www.w3school.com.cn/html/html_entities.asp)

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>实体</title>
</head>
<body>
    <!-- 
        在 html 中有些时候，不能直接书写一些特殊符号
            1. 多个连续空格
            2. 大于小于号
        如果需要在网页中书写特殊符号，需要使用 html 中的实体（转义字符）
        语法：&实体的名字
            1. &nbsp：空格
            2. &gt：大于号
            3. &lt：小于号
            4. &copy：版权符号
    -->
    <p>今天&nbsp;&nbsp;&nbsp;&nbsp;天气真不错</p>
    <p>a&lt;b&gt;c</p>
</body>
</html>
```

### 3.2 meta

[文档](https://www.w3school.com.cn/tags/tag_meta.asp)

主要用于设置网页中的一些元数据

- name：指定的数据名称
- content：指定的数据内容
- charset：指定字符集
- http-equiv：指定 HTTP 头部

```html
<!DOCTYPE html>
<html lang="en">

    <head>
        <meta charset="UTF-8">

        <!-- keywords 表示网站的关键字，可被搜索引擎搜索到，可同时指定多个关键字，用逗号隔开 -->
        <meta name="keywords" content="HTML5,前端,CSS3">

        <!-- description 用于指定网站的描述，该描述会显示在搜索引擎的搜索结果页面中-->
        <meta name="description" content="这是一个描述">

        <!-- 将页面重定向到另一个网站（3秒后跳转到百度） -->
        <meta http-equiv="refresh" content="3;url=https://www.baidu.com">

        <!-- title 标签的内容会作为搜索结果的超链接上的文字显示 -->
        <title>Document</title>
    </head>
    <body></body>
</html>
```

### 3.3 语义化标签

在使用 html 标签时，应该关注的是==标签的语义==，而不是他的样式

#### 1.块元素

在页面中独占一行的元素称为块元素（block element），在网页中一般通过块元素对页面进行布局，默认 width 拉满

1. 标题标签：`h1` ~ `h6`，重要性依次递减

   - h1 重要性仅次于 title，一般情况下一个页面中只有一个 h1 标签

   - 一般情况下，只会使用到 h1 ~ h3

   - `hgroup` 标签用来为标题分组，可以将一组相关的标题放入 `hgroup`

     ```html
     <hgroup>
         <h1></h1>
         <h2></h2>
     </hgroup>
     ```

2. `P`：表示页面中的一个段落

   - `title`属性：鼠标移动到上方后会显示提示文字

3. `blockquote`：表示一个长引用

4. `br`：表示页面中的换行

5. ==div==：没有语义，就表示一个区块，目前还是主要的布局元素

#### 2.行内元素

在页面中不会独占一行的元素（inline element），主要用来包裹文字

- `em`：用于表示语音语调的加重

- `strong`：表示强调作用，指重要内容

- `q`：表示一个短引用

- ==span==：没有语义，一般用于在网页中选中文字

- ==a==：超链接，在 a 标签中可以嵌套除它自身外的任何元素

  - `href`属性：指定跳转的目标路径

    1. 可以是一个**外部网站**的地址

       ```html
       <a href="https:wwww.baidu.com">超链接</a>
       ```

    2. 也可以是一个**项目内部页面**的地址（指定相对地址）

       ```html
       <a href="new 2.html">超链接</a>
       ```

       - `./`表示当前文件所在目录，可以省略
        - `../`表示上一级目录

    3. 还可以是**当前页面**的某个位置

       - 将 href 属性设置为`#`， 则会跳转到当前页面的顶部
        - 将 href 属性设置为`#id`属性（元素唯一标识，同一页面 id 唯一），跳转到指定元素位置 

  - 路径占位符：==javascript:;==，使超链接无效

- `target`属性：指定超链接打开的位置

  - `_self`：默认值，在当前页面中打开超链接
    - `_blank`：在一个新的页面中打开超链接

> **注意**
>
> - 浏览器解析网页时，会自动修正不符合规范的内容
> - 一般情况下可以在块元素中放行内元素，而不会在行内元素中放块元素
> - 块元素中基本上什么都能发
> - p 元素中不能放任何的块元素

#### 3.结构化语义标签

- `header`：表示网页的头部
- `main`：表示网页的主题部分，一个页面只会有一个 main
- `footer`：表示网页的底部
- `nav`：表示网页中的导航
- `aside`：表示和主体相关的其他内容（侧边栏）
- `article`：表示一个独立的文章
- `section`：表示一个独立的区块，上边的标签都不能表示时使用 section

#### 4.列表

1. 有序列表
   - 使用 `ol` 标签来创建有序列表
   - 使用 `li` 表示列表项

2. 无序列表（使用较多）
   - 使用 `ul` 标签来创建无序列表
   - 使用 `li` 表示列表项

3. 定义列表
   - 使用 `dl` 标签创建定义列表
   - 使用 `dt` 来表示定义的内容
   - 使用 `dd` 来对内容进行解释说明

4. 注意：列表可以互相嵌套


#### 5.图片

==img== 标签：引入外部图片，是一个自结束标签，属于替换元素（介于块和行内元素之间，具有两种元素特点）

- `src`属性指定外部图片路径
- `alt`属性用来描述图片，默认不会显示，有些浏览器会在图片无法加载时显示，但搜索引擎会根据 alt 中的内容来识别图片，如果不写 alt 属性则不会被搜索引擎识别
- `width`属性指定图片宽度，单位像素
- `height`属性指定图片高度，单位像素
  - 宽度和高度如果只修改一个，则另一个会等比例缩放
  - 注意：一般情况下 PC 端不建议修改图片大小，需要多大的图片就裁多大，但是在移动端，经常需要对图片缩放（大图缩小）

图片的格式：

- jpeg（jpg）
  - 支持的颜色比较丰富，不支持透明效果和动图
  - 一边拿用来显示照片
- gif
  - 支持的颜色比较少，支持简单透明，支持动图
  - 适合表示颜色单一的图片和动图
- png
  - 支持的颜色丰富，支持复杂透明，不支持动图
  - 网页中使用较多
- webp
  - 谷歌新推出的专门用来表示网页中的图片的一种格式，具备了其他格式的所有优点，而且文件较小
  - 但兼容性较差，不兼容老版本的浏览器
- base64
  - 将图片使用 base64 编码，这样可以将图片转换为字符，通过字符的形式来引用图片
  - 一般都是需要和网页一起加载的图片才会使用 base64
- 总结：效果一样用小的，效果不一样，用效果好的

#### 6.内联框架

`iframe`标签：用于向当前页面中引入一个其他页面

- `src`属性：指定要引入的网页的路径
- `frameborder`属性：指定内联框架的边框，0 or 1

#### 7.引入音频

> 默认不允许用户自己控制播放停止

`audio`标签：向页面中引入一个外部的音频文件

  1. `src`属性：文件路径

     ```html
     <audio src=""></audio>
     ```

  1. `source`标签：同样可以指定文件路径，可以添加提示文字，当浏览器不支持时会显示文字信息，添加多个 source 时，浏览器会依次识别文件信息直到找到能支持的音频文件

     ```html
     <audio>
         对不起，您的浏览器不支持播放音频，请升级浏览器！
         <source src="">
         <source src="">
     </audio>
     ```

  1. `controls`属性：是否允许用户控制播放，不用给值

  1. `autoplay`属性：音频文件是否自动播放

     - 目前来讲大部分浏览器都不会对音乐自动播放

  1. `loop`属性：是否循环播放

1. `embed`：同样可以引用音频文件（不推荐），支持 ie8 之前的版本，可配合 source 使用

   ```html
   <audio>
       <source src="">
   	<embed src="" type="audio/mp3" width="300" height="100">
   </audio> 
   ```

#### 8.引入视频

`video`标签：引入一个视频文件，使用方式和 audio 相同

```html
<video controls>
    <source src=""> 
    <embed src="" type="video/mp4" width="300" height="100">
</video>
```

#### 9.表格

##### 9.1 基本使用

```html
<table border="1">
    <!-- tr 表示一行 -->
    <tr>
        <!-- th 表示一个标题单元格 -->
        <th>1</th>
        <th>2</th>
    </tr>
    <tr>
        <!-- tb 表示一个内容单元格 -->
        <td>A1</td>
        <td>A2</td>
    </tr>
    <tr>
        <td colspan="2" style="text-align:center">B1</td>
    </tr>
</table>
```

##### 9.2 长表格

可以将表格分成`thead`、`tbody`、`tfoot`三部分

##### 9.3 样式

```css
table {
    width: 50%;
    border: 1px solid black;
    /* 
        指定边框距离
        border-spacing: 0px;
    */
    /* 指定合并边框 */
    border-collapse: collapse;
}

td {
    border: 1px solid black;
    /* 默认td中的内容垂直居中 */
    vertical-align: center;
    text-align: center;
}

/* 指定奇数行颜色 */
tr:nth-child(odd) {
    background-color: red;
}
```

> **注意**
>
> 如果表格中没有使用`tbody`而是直接使用`tr`，则浏览器会自动创建一个`tbody`将`tr`放入，所以`tr`不是`table`的子元素，而是`tbody`的子元素，因此要通过`tbody > tr`选择。

> **扩展**
>
> 可通过将标签指定为`display: table-cell`来使用单元格的一些属性，例如让 box1 中的 box2 位于 box1 的正中心：
>
> ```html
> <style>
>     .box1 {
>         display: table-cell;
>         /* 此时就可以使用 vertical-align 来指定内容的垂直居中，尽管他是用来指定文本的垂直居中 */
>         vertical-align: middle;
>         background-color: blue;
>     }
>     .box2 {
>         background-color: red;
>         margin: 0 auto;
>     }
> </style>
> <body>
>     <div class="box1">
>         <div class="box2">
>         </div>
>     </div>
> </body>
> ```

#### 10.表单

网页中的表单用于将数据提交给远端的服务器。

```html
<!-- action: 服务器地址 -->
<form action="#">
    <!-- 
        文本框，必须指定 name 用于发送数据
        autocomplete 用于指定是否开启历史记录，默认开启，也可在 form 标签中声明
        readonly 将内容设置为只读，可以被提交；disable 将内容设置为禁用，不会被提交
     -->
    文本框<input type="text" name="param"  value="text" autocomplete="off" readonly>
    <br><br>
    <!-- 
        密码框
        autofocus 指定自动获取焦点
     -->
    密码框<input type="password" name="password" id="password" autofocus>
    <br><br>
    <!-- 单选按钮 -->
    <input type="radio" name="gender" id="gender" value="0" checked>男
    <input type="radio" name="gender" id="gender" value="1">女
    <br><br>
    <!-- 多选框 -->
    兴趣
    <input type="checkbox" name="hoby" id="hoby" value="0">
    <input type="checkbox" name="hoby" id="hoby" value="1">
    <br><br>
    <!-- 下拉列表 -->
    <select name="address" id="address">
        <option value="beijing">beijing</option>
        <option value="nanjing" selected>nanjing</option>
    </select>
    <br><br>
    <!-- 重置按钮 -->
    <input type="reset">
    <br><br>
    <!-- 普通的按钮，和 js 配合 -->
    <input type="button" value="button">
    <br><br>
    <!-- 提交按钮 -->
    <input type="submit" value="submit">   

    <!-- button标签可以实现同样的效果，且灵活度更高 -->
    <button type="reset">提交</button>
    <button type="button">提交</button>
    <button type="submit">提交</button>
</form>
```
