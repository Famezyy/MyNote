# CSS

## 1.简介

CSS：

- 层叠样式表

- 网页实际上是一个多层结构，通过 CSS 可以分别为网页的每一层设置样式

- CSS用来设置网页中元素的样式

**内联样式**（行内样式）

- 在标签内部通过 style 属性设置元素的样式
- 只能对一个标签生效，不便于维护

```html
<p style="color: red; font-size: 20px;">测试CSS</p>
```

**内部样式表**

- 将样式编写到 head 中的 style 标签中
- 通过 CSS 的选择器来选中元素并为其设置各种样式
- 便于样式的重复使用
- 只能作用于单个网页

```html
<head>
    <meta charset="UTF-8">
    <title>Document</title>
    <style>
        p {
            color: red;
            font-size: 30px;
        }
    </style>
</head>

<body>
    <p>测试CSS</p>
    <p>测试CSS</p>
</body>
```

**外部样式表（最佳实践）**

- 可以将 CSS 样式编写到外部 CSS 文件中
- 通过 ==link== 标签引入外部 CSS 文件
- 可以在不同页面间复用
- 可以使用到浏览器的缓存机制，加快网页的加载速度，提高用户体验

```html
<head>
    <meta charset="UTF-8">
    <title>Document</title>
    <link rel="stylesheet" href="./style.css">
</head>
```

style.css

```css

p {
    color: red;
    font-size: 30px;
}
```

## 2.基本语法

- style 标签及 CSS 文件内部不能使用 html 的语法
- 注释：`/* */`
- 选择器
  - 通过选择器可以选中页面中的指定元素，比如 p 的作用就是选中页面中所有的 p 元素
- 声明块
  - 通过声明块指定要为元素设置的样式
  - 声明块由一个个的声明组成
  - 声明是一个名值对结构：`样式名：样式值；`

### 2.1 选择器

#### 1.常用选择器

**元素选择器**：根据标签名选中指定元素

- 语法：`标签名{}`
- 例：p{}，h1{}，div{}

**id 选择器**：根据元素的 id 属性值选中一个元素

- 语法：`#id属性值{}`
- 例：#id{}

**class 选择器**：根据元素的 class 属性值选中一组元素

- class 是一个标签的属性，和 id 类似，不同的是 class 可以重复使用，用来给元素分组

- 可以同时为一个元素指定多个 class，多个 class 间使用空格隔开

  ```html
  <p class="blue bold"></p>
  ```

- 语法：`.class属性值`

- 例：.class{}

**通配选择器**：选中页面中的所有元素

- 语法：`*{}`

#### 2.复合选择器

**交集选择器**：选中同时符合多个条件的元素

- 语法：`选择器1选择器2选择器3`

- 如果有元素选择器，必须使用元素选择器开头

```html
<head>
    <style>
        div.red{
            font-size: 30px;
        }
        .a.a.c{
            color: blue;
        }
    </style>
</head>
<body>
    <div class="red">it is a div</div>
    <div class="a b c">it is a div</div>
</body>
```

**并集选择器**：同时选中多个选择器对应的元素

- 语法：`选择器1,选择器2`

- 例：#b1,.p1,h1,span,div.red{}

```html
<head>
    <style>
    div,p{
        font-size: 30px;
    }
</style>
</head>
<body>
    <div>it is a div</div>
    <p>it is a div</p>
    <p id="b1"></p>
</body>
```

#### 3.关系选择器

**元素关系种类**

1. 父元素：直接包含子元素的元素叫父元素
2. 子元素：直接被父元素包含的元素是子元素
3. 祖先元素：直接或间接包含后代的元素，父元素也是祖先元素
4. 后代元素：直接或间接被祖先元素包含的元素，子元素也是后代元素
5. 兄弟元素：拥有相同父元素的元素是兄弟元素

**选择器**

1. 子元素选择器：选中指定父元素的子元素
   - 语法：`父元素>子元素`
   - 例：div>span{}，div.box>span>p{}
2. 后代元素选择器：选中指定元素的后代元素
   - 语法：`祖先 后代`
   - 例：div span{}，div.box span p{}
3. 兄弟元素选择器：选中兄弟元素
   - 语法：`前一个+下一个`（选中紧挨着的下一个元素） or `兄~弟`（选中后面所有的元素）
   - 例：p+span{} or p~span{}

#### 4.属性选择器

1. 选择含有指定属性的所有类型的元素
   - 语法：`元素[属性名]`
   - 例：p[title]{}
2. 选择含有指定属性的指定类型的元素
   - 语法：`[属性名]`
   - 例：[title]{}
3. 选择含有指定属性和属性值的元素
   - 语法：`元素[属性名=属性值]`
   - 例：p[title=abc]{}
4. 选择属性值以指定值开头的元素
   - 语法：`元素[属性名^=属性值]`
   - 例：p[title^=abc]{}
5. 选择属性值以指定值结尾的元素
   - 语法：`元素[属性名$=属性值]`
   - 例：p[title$=abc]{}
6. 选择属性值中含有某值的元素
   - 语法：`元素[属性名*=属性值]`
   - 例：p[title*=abc]{}

#### 5.伪类选择器

不存在的类，用来描述一个元素的特殊状态，比如：第一个子元素，被点击的元素，鼠标移入的元素等，一般情况下使用 ":" 开头。

1. `:first-child`：第一个子元素

   ```css
   ul > li:first-child
   ```

2. `:last-child`：最后一个子元素

3. `:nth-child(n)`：选中第 n 个子元素

   特殊值：

   - `n`：选中所有子元素（0 ~ 无穷）
   - `2n` or `even`：选中偶数位的元素
   - `2n+1` or `odd`：选中奇数位的元素

以上这些伪类都是根据**所有子元素**进行排序，若 li 不是 ul 的第一个元素，则不会使用样式。

4. `first-of-type`

5. `last-of-type`

6. `nth-of-type()`

这三个伪类与上述的类似，但是他们是在同类型元素中进行排序。

7. `not()`：否定伪类，将符合条件的元素从选择器中去除

   ```css
   /* 除了第三个 li 元素 */
   ul>li:not(:nth-of-type(3))
   ```

8. 超链接的伪类

   - 没有访问过的链接——`link`（正常的链接）**（若定义在 hover or active 后，会覆盖 hover 和 active 的相同的样式）**

     ```css
     a:link{
         color:red;
     }
     ```

   - 访问过的链接——`visited`**（若定义在 hover or active 后，会覆盖 hover 和 active 的相同的样式）**

     ```css
     a:visited{
         color:orange;
     }
     ```

     由于隐私的原因， visited 只能修改链接的颜色。

   - 鼠标移入的状态——`hover`**（可以绑定给其他标签）**

     ```css
     a:hover{
         color:red;
         font-size:50px;
     }
     ```

     > **注意**
     >
     > 在 CSS 中设置 hover 时，可通过`>`和`+`来实现对子集元素、下一元素、下一元素的子集元素的 CSS 样式控制，对父元素或者上一元素无法控制

   - 鼠标点击的状态——`active`**（可以绑定给其他标签）**

     ```css
     a:active{
         color:red;
     }
     ```

   > **扩展**
   >
   > **1.鼠标移入显示菜单**
   >
   > ```html
   > <head>
   >  <style>
   >      .location {
   >          background-color: yellow;
   >          float: left;
   >      }
   >      .location .city-list {
   >          /* 正常状态下隐藏菜单 */
   >          display: none;
   >          width: 320px;
   >          height: 430px;
   >          background-color: pink;
   >          /* 设置绝对定位，不占页面位置 */
   >          position: absolute;
   >      }
   >      /* 鼠标移入时显示 city-list */
   >      .location:hover .city-list {
   >          display: block;
   >      }
   >  </style>
   > </head>
   > <body>
   >  <div class="location">
   >      <div class="current-city">
   >          <a href="javascript:;">北京</a>
   >      </div>
   >      <div class="city-list">
   >          ...
   >      </div>
   >  </div>
   > </body>
   > ```
   >
   > **2.弹出框显示倒三角**
   >
   > ```html
   > <head>
   >     <meta charset="UTF-8">
   >     <meta name="viewport" content="width=device-width, initial-scale=1.0">
   >     <title>Document</title>
   >     <link rel="stylesheet" href="CSS/reset.css">
   >     <link rel="stylesheet" href="fa/css/all.css">
   >     <style>
   >         body {
   >             min-width: 1266px;
   >             font: 14px/1.5 Helvetica Neue,Helvetica,Arial,Microsoft Yahei,Hiragino Sans GB,Heiti SC,WenQuanYi Micro Hei,sans-serif;
   >         }
   >         .header {
   >             width: 100%;
   >             line-height: 40px;
   >             background-color: rgb(51, 51, 51);
   >         }
   > 
   >         .clearfix::before,
   >         .clearfix::after {
   >             content: "";
   >             display: table;
   >             clear:both
   >         }
   > 
   >         .w {
   >             width: 1240px;
   >             margin: 0 auto;
   >         }
   > 
   >         .header-content .menu, .menu li, .customer-bar ul, .customer-bar ul li {
   >             float: left;
   >         }
   > 
   >         .header-content .customer-bar {
   >             float: right;
   >         }
   > 
   >         a {
   >             text-decoration: none;
   >             color: rgb(176,176,176);
   >             font-size: 12px;
   >             display: block
   >         }
   >         li a:hover {
   >             color: rgb(255, 255, 255)
   >         }
   > 
   >         .line {
   >             display: block;
   >             color: rgb(66, 60, 55);
   >             margin: 0 7px;
   >             font-size: 12px;
   >             margin-top: -1px;
   >         }
   > 
   >         .info {
   >             margin-right: 27px;
   >         }
   > 
   >         .shop-cart {
   >             text-align: center;
   >             width: 120px;
   >             margin-right: 15px;
   >             background-color: rgb(66, 66, 66);
   >             position: relative;
   >         }
   >         
   >         .shop-cart:hover {
   >             background-color: rgb(255, 255, 255);
   >         }
   > 
   >         .shop-cart a {
   >             display: block;
   >         }
   > 
   >         .shop-cart:hover a {
   >             color: rgb(255, 103, 0);
   >         }
   > 
   >         .shop-cart:hover .cart-content {
   >             display: block;
   >             height: 100px;
   >             /* 设置渐隐 */
   >             transition: height 0.5s;
   >         }
   > 
   >         .shop-cart:hover::after {
   >             display: block;
   >         }
   > 
   >         /* 设置下拉菜单 */
   >         .cart-content {
   >             left: -197px;
   >             width: 317px;
   >             height: 0px;
   >             overflow: hidden;
   >             background-color: white;
   >             box-shadow: 1px 1px 1px rgba(0, 0, 0, .3);
   >             position: absolute;
   >         }
   > 
   >         /* 设置倒三角 */
   >         .shop-cart::after {
   >             display: none;
   >             content: "";
   >             position: absolute;
   >             border:10px solid transparent;
   >             border-top: none;
   >             border-bottom-color: black;
   >             width: 0;
   >             height: 0;
   >             left: 0;
   >             right: 0;
   >             bottom: 0;
   >             margin: auto;
   >         }
   >     </style>
   > </head>
   > <body>
   >     <div class="header clearfix">
   >         <div class="header-content w">
   >             <ul class="menu">
   >                 <li>
   >                     <a href="javascript:;">小米官网</a>
   >                 </li>
   >                 <li><i class="line">|</i></li>
   >                 <li>
   >                     <a href="javascript:;">小米商城</a>
   >                 </li>
   >                 <li><i class="line">|</i></li>
   >                 <li>
   >                     <a href="javascript:;">MIUI</a>
   >                 </li>
   >                 <li><i class="line">|</i></li>
   >                 <li>
   >                     <a href="javascript:;">IoT</a>
   >                 </li>
   >                 <li><i class="line">|</i></li>
   >                 <li>
   >                     <a href="javascript:;">云服务</a>
   >                 </li>
   >                 <li><i class="line">|</i></li>
   >                 <li>
   >                     <a href="javascript:;">天星数科</a>
   >                 </li>
   >                 <li><i class="line">|</i></li>
   >                 <li>
   >                     <a href="javascript:;">有品</a>
   >                 </li>
   >                 <li><i class="line">|</i></li>
   >                 <li>
   >                     <a href="javascript:;">小爱开放平台</a>
   >                 </li>
   >                 <li><i class="line">|</i></li>
   >                 <li>
   >                     <a href="javascript:;">企业团购</a>
   >                 </li>
   >                 <li><i class="line">|</i></li>
   >                 <li>
   >                     <a href="javascript:;">资质证照</a>
   >                 </li>
   >                 <li><i class="line">|</i></li>
   >                 <li>
   >                     <a href="javascript:;">协议规则</a>
   >                 </li>
   >                 <li><i class="line">|</i></li>
   >                 <li>
   >                     <a href="javascript:;">下载app</a>
   >                 </li>
   >                 <li><i class="line">|</i></li>
   >                 <li>
   >                     <a href="javascript:;">Select Location</a>
   >                 </li>
   >             </ul>
   >             <ul class="customer-bar">
   >                 <ul class="info">
   >                     <li>
   >                         <a href="javascript:;">登录</a>
   >                     </li>
   >                     <li><i class="line">|</i></li>
   >                     <li>
   >                         <a href="javascript:;">注册</a>
   >                     </li>
   >                     <li><i class="line">|</i></li>
   >                     <li>
   >                         <a href="javascript:;">消息通知</a>
   >                     </li>
   >                 </ul>
   >                 <ul class="shop-cart">
   >                     <a href="javascript:;">购物车</a>
   >                     <div class="cart-content"></div>
   >                 </ul>
   >             </ul>
   >         </div>
   >         
   >     </div>
   > </body>
   > ```

9. 输入框的伪类

   - 获取焦点-`focus`

     ```css
     /* 获取焦点后改变边框的颜色 */
     .inpuit {
         outline: none;
     }
     .input:focus {
         border-color: red;
     }
     ```

#### 6.伪元素选择器

表示页面中一些特殊的并不真实存在的元素（特殊的位置），使用`::`开头。

1. `::first-letter`：表示第一个字母

   ```css
   p::first-letter{
       font-size:50px;
   }
   ```

2. `::first-line`：表示第一行

   ```css
   p::first-line{
       background-color:yellow;
   }
   ```

3. `::selection`：表示选中的内容

4. `::before`：元素的开始位置

5. `::after`：元素的最后位置，默认行内元素

`before`和`after`必须结合`content`属性使用（该属性可以设置显示内容，无法被选中）

```css
div::before{
    content:'abc';
    color:red;
}
```

### 2.2 样式的继承

- 我们为一个元素设置的样式，同时也会应用到它的后代元素上，利用继承可以设置一些通用的样式
- 并不是所有的样式都会被继承，比如背景`background-color`，布局相关的样式不会被继承
  - 背景颜色会扩展至边框下面，当边框为透明时可在边框位置看到背景颜色
- 可以查看文档中对应标签的`inherited`属性是否支持继承

### 2.3 ==选择器的权重==

样式的冲突：当我们通过不同的选择器选中相同的元素，并且为相同的样式设置不同的值，此时就发生了样式的冲突。发生样式冲突时，由选择器的权重（优先级）决定。

选择器的权重：

1. 内联样式（直接声明在标签的 style 属性中）：1000

2. id 选择器：100

3. 类和伪类选择器：10

4. 元素选择器：1

5. 通配选择器`*{}`：0

6. 继承的样式：没有优先级

```html
<div>
    father
    <span>child</span>
</div>
```

```css
*{} 大于 div{}
```

- 比较优先级时，需要将所有的选择器的优先级进行相加计算，优先级越高，越优先显示

  ```html
  <div id="box1">div</div>
  ```

  ```css
  div#box1{} 大于 #box1{}
  ```

- 分组选择器单独计算

  ```css
  div,p,span{}
  ```

- 选择器的累加不会超过其最大的数量级，类选择不会超过 id 选择器

- 如果优先级计算后相等，则优先使用后定义的样式

- 可以在某一个样式后边添加 `!important`，则此时该样式会获得最高的优先级，超过内联样式，慎用！！

### 2.4 像素和百分比

**长度单位**

- 像素：不同屏幕像素大小不同，像素越小的屏幕显示效果越清晰，同一样的 200px 在不同设备下显示效果不同

- 百分比：设置属性**相对于父元素**属性的百分比

  ```html
  <div class="box1">
      <div class="box2"></div>
  </div>
  ```

  ```css
  .box1{
      width:300px;
      height:100px;
      background-color:orange;
  }
  .box2{
      width:50%;
      height:50%;
      background-color:aqua;
  }
  body {
      /* 设置 body 伸缩时的最小宽度 */
      min-width: 200px;
  }
  ```

- `em`：相对于元素自身的字体大小，`1em = 1font-size`，会根据字体大小改变而变，默认字体大小 16 像素

- `rem`：相对于根元素（html）的字体大小

  ```css
  html{
      font-size:10px;
  }
  div{
      /* 相当于 100px */
      width:10em;
  }
  ```

**颜色单位**

1. CSS 中可以直接使用**颜色名**来设置各种颜色
2. 还可使用**RGB**表示颜色，`RGB(红色，绿色，蓝色)`，每种颜色范围在 0 ~ 255（0% ~ 100%）
3. 也可使用**RGBA**，`RGBA(红色，绿色，蓝色，不透明度)`，不透明度范围在 0 ~ 1，1 表示完全不透明，`.5`表示半透明
4. **十六进制 RGB 值**：`#红色绿色蓝色`，范围在 00 ~ FF，如果颜色两位两位重复可以简写：#aabbcc -> #abc
5. **HSL** & **HSLA**：`hsl(H,S,L)`，H 指色相（0 ~ 360），S 指饱和度（0% ~ 100%），L 指亮度（0% ~ 100%）

## 3.布局

### 3.1 文档流（normal flow）

网页是一个多层的结构，通过 CSS 可以给每一层来设置样式，用户只能看到最顶上一层，最下的一层称为 **文档流**，文档流是网页的基础，我们所创建的元素默认都是在文档流中进行排列。

元素主要有两个状态：

- 在文档流中
  - **块元素**
    - 块元素会在页面中**独占一行**，自上向下垂直排列
    - 默认宽度是父元素的全部（会把父元素撑满）
    - 默认高度是被内容（子元素）撑开
  - **行内元素**
    - 行内元素不会独占一行，只占自身的大小
    - 行内元素在页面中自左向右水平排列
    - 会自动换行
    - 默认高度和宽度总是被内容撑开
- 不在文档流中（脱离文档流）

### ==3.2 盒子模型==

CSS 将页面中的所有元素都设置为了一个矩形的盒子，只需将不同的盒子摆放到不同的位置。

每一个盒子都由以下几个部分组成：

- 内容区（content）
- 内边距（padding）
- 边框（border）
- 外边距（margin）

一个盒子的可见框的大小，由**内容区**，**内边距**和**边框**共同决定，在计算盒子大小时，需要将三个区域加到一起计算。

相关属性：

```css
.box1{
    /**
    1. 内容区，元素中的所有子元素和文本内容都在内容区排列
    	width：内容区的宽度，默认 auto，被内容撑开
    	height：内容区的高度 	
    **/
    width:200px;
    height:200px;
    background-color:#bfa;
    /**
    2. 边框，属于盒子边缘，边框大小会影响整个盒子大小
    	border-width：边框的宽度
    	border-color：边框的颜色
    	border-style：边框的样式
    **/
    border-width:10px;
    border-color:red;
    border-style:solid;
}
```

#### 1）边框

- `border-width`：用来指定四个方向的边框的宽度，默认 3 个像素
  - 四个值：上 右 下 左
  - 三个值：上 左右 下
  - 两个值：上下 左右
  - 一个值：上下左右
  - 除了 border-width 还有一组 `border-xxx-width`（top、right、bottom、left），用来单独指定某一边的宽度，border-color 和 border-style 同样拥有该组属性
- `border-color`：指定边框颜色，同样可以分别指定四个边的边框，规则同 border-width；如果省略，则自动使用**color**标签的颜色值
  - **transparent**：边框透明
- `border-style`：指定边框的样式，同样可以指定四个边的样式，默认值是**none**，表示没有边框
  - solid：实线
  - dotted：点状虚线
  - dashed：虚线
  - double：双线

- ==border==：同时设置边框所有的相关样式，没有顺序要求，用空格隔开

  ```css
  div{
      border:solid 10px orange;
  }
  ```

  - `border-top`，`border-right`，`border-bottom`，`border-left`

#### 2）内边距

内边距指内容区和边框之间的距离，共有四个方向的内边距：`padding-top`，`padding-right`，`padding-bottom`，`padding-left`。

- 内边距的设置会影响到盒子的大小

- 背景颜色会延伸到内边距上
- ==padding==：同时指定四个方向的内边距，规则同`border-width`

#### 3）外边距

外边距不会影响盒子可见框的大小，但会影响盒子的位置。

一共四个方向的外边距（默认为 0）：

- `margin-top`：正值元素向下移动
- `margin-right`：**默认情况下不会生效**
- `margin-bottom`：正值其下方元素向下移动
- `margin-left`：正值元素向右移动
- ==mergin==：同时指定四个方向的外边距，规则同 `border-width`

元素在页面中按照自左向右顺序排列，所以默认情况下如果设置**左**和**上**外边距则会移动元素自身，而设置**下**和**右**外边距会移动其他元素。负值则元素会向相反方向移动。

#### 4）<span id="水平方向布局">水平方向布局</span>

元素在其父元素中水平方向的位置有以下几个属性共同决定：

- margin-left
- border-left
- padding-left
- width
- padding-right
- border-right
- margin-right

元素在其父元素中，水平布局必须满足以下等式：

`margin-left + border-left + padding-left + width + padding-right + border-right + margin-right = 父元素内容区宽度`

- 如果相加结果不成立，则称为过度约束，则等式会自动调整

- 如果所有的值中没有 auto，则浏览器会自动调整**margin-right**值使等式成立

- 7 个值中 3 个可以设置成 auto：**width**，**margin-left**，**margin-right**

  - 如果某个值为 auto，则会自动调整 auto 的值使等式成立

  - 如果将 width 和一个外边距设置为 auto，则会将宽度会调整最大，另一个外边距为 0

  - 如果三个值都设置为 auto，则外边距为0，宽度调整至最大

  - 如果两个外边距设置为 auto，宽度值固定，则会将外边距设置为相同的值，效果为**居中**

    ```css
    width:200px;
    margin: 0 auto;
    ```

#### 5）垂直方向布局

默认情况下父元素的高度被内容撑开，如果子元素的大小超过父元素，则子元素会从父元素中溢出。

使用<span id="overflow">`overflow`</span>属性设置父元素如何处理溢出的子元素：

`overflow-x`：处理水平方向

`overflow-y`：处理垂直方向

- `visible`：默认值，子元素会从父元素中溢出，在父元素外部显示
- `hidden`：溢出的内容将会被裁剪，不会显示
- `scroll`：生成两个滚动条，通过滚动条查看完整内容
- `auto`：根据需要生成滚动条

#### 6）<span id="垂直外边距的折叠">垂直外边距的折叠</span>

==相邻==的==垂直方向==的外边距会发生折叠现象：

**兄弟元素**

- 兄弟元素间的相邻垂直外边距会取两者之间的较大值（两者都是正值）
- 特殊情况：
  - 如果相邻外边距一正一负，则取两者的和
  - 如果相邻外边距都是负值，则取两者绝对值较大的

**父子元素**

- 父子元素间的相邻外边距，子元素会传递给父元素（上外边距）
  - 可通过设置父元素外边框或父元素上内边距使子元素与父元素上边界分开
  - 或开启 BFC

#### 7）行内元素的盒模型

行内元素不支持设置高度和宽度，默认被内容撑开，行内元素可以设置`padding`，`border`，`margin`，但是垂直方向不会挤开下面的元素，自身文字位置也不会变，不会影响页面的布局。

`display`：用来设置元素显示的类型

- `inline`：将元素设置为行内元素
- `block`：将行内元素设置为块元素
- `inline-block`：将元素设置为行内块元素，既可以设置高度和宽度，又不会独占一行
- `table`：将元素设置为一个表格
- `none`：元素不在页面显示

`visibility`：用来设置元素的显示状态

- `bisible`：默认值，元素在页面正常显示
- `hidden`：元素在页面中隐藏，不显示，但是依然占据页面位置

### 3.3 浏览器的默认样式

通常情况下，浏览器都会为元素设置默认样式，会影响到页面的布局，编写时要去除浏览器的默认样式（PC端）。

```css
body{
    margin:0;
}
p{
    margin:0;
}
ul{
    margin:0;
    padding:0;
    /* 去除项目符号 */
    list-style:none;
}
```

```css
/* 可以直接声明所有的 margin 和 padding */
*{
    margin:0;
    padding:0;
}
```

外部 CSS 重置样式表：重置浏览器的样式：

- **reset.css**：直接去除了浏览器的默认样式
- **normalize.css**：对默认样式进行统一，保证在所有浏览器中都是同一样式

### 3.4 练习

```html
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <link rel="stylesheet" href="./CSS/reset.css">
    <style>
        body {
            background-color: rgb(244, 244, 244);
        }

        .left_menu {
            margin: 50px auto;
            width: 190px;
            height: 450px;
            background-color: white;
            padding: 10px 0;
        }

        .left_menu li {
            height: 25px;
            /* 让文字垂直居中，只需将父元素的 line-height 设置与父元素的 height 相同 */
            line-height: 25px;
            /* 将文字向右移动 */
            padding-left: 18px;
            font-size: 14px;
        }

        /* 设置鼠标移入状态 */
        .left_menu li:hover {
            background-color: #d9d9d9;
        }

        .left_menu a {
            color: #333;
            /* 
              去除超链接下划线 
            */
            text-decoration: none;
        }

        .left_menu a:hover {
            color: rgb(200, 22, 35);
        }
        
        /* 通过伪元素为每一个li项目添加符号 */
        .left_menu>li::before{
            content: "■";
            color: rgb(218, 218, 218);
            margin-right:8px;
            font-size: 1px;
        }
    </style>
</head>

<body>
    <ul class="left_menu">
        <li><a href="#">家用电器</a></li>
        <li>
            <a href="#">手机</a>/<a href="#">运营商</a>/<a href="#">数码</a>
        </li>
        <li>电脑/办公</li>
        <li>家居/家具/家装/厨具</li>
        <li>男装/女装/童装/内衣</li>
        <li>美妆/个护清洁/宠物</li>
        <li>女鞋/箱包/钟表/珠宝</li>
        <li>男鞋/运动/户外</li>
        <li>房产/汽车/汽车用品</li>
        <li>母婴/玩具乐器</li>
        <li>食品/酒类/生鲜/特产</li>
        <li>艺术/礼品鲜花/农资绿植</li>
        <li>医药保健/计生情趣</li>
        <li>图书/文娱/教育/电子书</li>
        <li>机票/酒店/旅游/生活</li>
        <li>理财/众筹/白条/保险</li>
        <li>安装/维修/清洗/二手</li>
        <li>工业品</li>
    </ul>
</body>
```

### 3.5 盒子的大小

默认情况下，盒子可见框的大小由**内容区**，**内边距**和**边框**共同决定。

`box-sizing` 用来设置盒子尺寸的计算方式（设置 width 和 height 的作用） ：

- `content-box`：默认值，宽度和高度用来设置内容区的大小
- `border-box`：宽度和高度设置整个盒子可见框的大小，即此时 width 和 height 指的是内容区，边框，内边距的总大小

### 3.6 轮廓、阴影和圆角

**轮廓阴影**

`outline`：用来设置元素的轮廓线，用法和 border 相同，不同的是轮廓不会影响可见框的大小，不会挤开下方元素，相当于描边。

**阴影**

`box-shadow`：设置元素的阴影效果，同样不会影响页面布局

- 第一个值：水平偏移量，设置阴影的水平位置，正值向右

- 第二个值：垂直偏移量，设置阴影的垂直位置，正值向下

- 第三个值：阴影的模糊半径（像素），可以不指定

- 第四个值：阴影的颜色

```css
box-shadow: 10px 10px 50px rgba(0, 0, 0, .3);
```

**圆角**

`border-radius`：设置圆角的半径大小（像素），或者椭圆的长短半轴。

`border-top-left-radius`，`border-top-right-radius`，`border-bottom-left-radius`，`border-bottom-right-radius`。

```css
/* 
 四个值：左上，右上，右下，左下
 三个值：左上，右上/左下，右下
 两个值：左上/右下，右上/左下
*/
border-radius: 10px 20px 30px 40px;
/* 指定四个角的椭圆半轴 */
border-radius: 10px / 20px; 
/* 圆形 */
border-radius: 50%; 
```

## 4.浮动

### 4.1 简介

通过浮动可以使一个元素向父元素的左侧或右侧移动，可以制作**水平方向**的布局。

使用 `float` 属性设置：

- `null`：默认值，不浮动
- `left`：向左浮动
- `right`：向右浮动

> **注意**
>
> - 设置浮动以后，水平布局的等式不再强制成立，该元素不会再占据一行
> - 完全从文档流脱离，不占用文档流位置，所以元素下面还在文档流的元素会自动向上移动

**特点**

1. 浮动元素会完全脱离文档流，不再占据文档流的位置
   1. 块元素不再独占一行，宽度和高度被内容撑开
   2. 行内元素和浮动块元素一样，此时不再区分行内元素和块元素
2. 设置浮动以后，元素会向左侧或右侧移动
3. 浮动元素默认不会从父元素移出
4. 浮动元素向左或向右移动时，不会超过前面定义的其他浮动元素（水平方向和垂直方向）
5. 如果浮动元素上边是一个没有浮动的块元素，则浮动元素无法上移
6. **浮动元素不会盖住文字，文字自动环绕在浮动元素周围**，可以用来设置文字环绕效果

### ==4.2 高度塌陷和BFC==

**高度塌陷问题**：浮动布局中常见问题

- 在浮动布局中，父元素的高度默认被子元素撑开，当子元素浮动后，会完全脱离文档流，无法再撑起父元素高度，导致父元素高度丢失
- 父元素高度丢失后，其下方元素会自动上移，导致页面布局混乱

**BFC**: Block Formatting Context，块级格式化环境

- CSS 中隐含属性，可以为一个元素开启 BFC
- 开启 BFC 后，元素会变成一个独立的布局区域
- 开启后特点：
  1. 不会被浮动元素覆盖
  2. 子元素和父元素外边距不会重叠
  3. 可以包含浮动的子元素
- 通过特殊方式开启
  1. 设置父元素浮动
     - 宽度塌陷，父元素也脱离文档
  2. 将父元素设置为行内块元素
     - 宽度塌陷
  3. 将元素<a href="#overflow">overflow</a>设置为非 visible
     - 常设置为 hidden 开启 BFC

#### 1.clear属性

作用：用于清除浮动元素对当前元素产生的影响

- 用于解决当 box1 浮动后，box2 受到影响位置上移

可选值：

- left：清楚左侧浮动元素对当前元素的影响

- right：清除右侧浮动元素对当前元素的影响
- both：清除两侧中影响最大的

原理：设置后，浏览器会自动为元素添加一个上边距，使其位置不受影响

```css
.box1{
    float:left;
}
.box2{
    clear:left;
}
```

#### 2.使用after伪类解决高度塌陷

解决想法

```html
<style>
    .box2{
        float:left;
    }
    .box3{
        clear:left;
    }
</style>
<div class="box1">
    <div class="box2"></div>
    <!-- 添加一个 box3，清除浮动元素对它的影响，则box1 会被 box3 的位置撑开 -->
    <div class="box3"></div>
</div>
```

高级实现

```html
<style>
    .box2{
        float:left;
    }
    .box1::after{
        content:"";
        /** 默认行内元素，需要转换为块元素，或者 table **/
        display:block;
        clear:both;
    }
</style>
<div class="box1">
    <div class="box2"></div>
</div>
```

#### 3.扩展：利用 before 解决[父子元素外边距折叠](#垂直外边距的折叠)

```html
<style>
    .box2{
        margin-top:100px;
    }
    .box1::before{
        content:"";
        /** table 不会占一行 **/
        display:table;
    }
</style>
<div class="box1">
    <div class="box2"></div>
</div>
```

#### 4.终极版本

```html
<style>
    .box2{
        margin-top:100px;
    }
    /** 用来同时解决高度塌陷和外边距重叠问题 **/
	.clearfix::before,
    .clearfix::after{
        content:"";
        display:table;
        clear:both;
    }
</style>
<div class="box1 clearfix">
    <div class="box2"></div>
</div>
```

## 5.定位

定位是一种更加高级的布局手段，通过定位可以将元素摆放到任意位置。

使用**position**属性设置定位：

- `static`：默认值，元素是静止的没有开启定位
- `relative`：相对定位
- `absolute`：绝对定位
- `fixed`：固定定位
- `sticky`：粘滞定位

### 5.1 相对定位-relative

```css
postion:relative;
```

相对于元素在文档流中**本来的位置**进行定位，会提升元素的层级，盖住其他元素。但不会使元素脱离文档流，不会影响其他元素，并且不会改变元素的性质：块->块，行内->行内。

可以通过偏移量（offset）改变元素位置：

- `top`：定位元素和定位位置上边的距离，越大则向下移动

- `bottom`：定位元素和定位位置下边的距离，越大则向上移动

- `left`：定位元素和定位位置左边的距离

- `right`：定位元素和定位位置右边的距离

### 5.2 绝对定位-absolute

相对于其**包含块**进行定位。正常情况下，包含块指离当前元素最近的祖先块元素。

> 绝对定位的包含块：**最近的开启了定位（position 的值不是 static）的祖先元素**，如果所有的祖先元素都没有开启定位，则相对于根元素进行定位——html（初始包含块），一般配合相对定位使用。

会提升元素的层级，盖住其他元素，元素会从文档流脱离，但不会改变位置，会影响下面元素的位置。

会改变元素性质，行内->块，**块->大小会被内容撑开**，但不是行内元素，宽度和高度依然可以设置。

可以通过偏移量（offset）改变元素位置：

- `top`：定位元素和定位位置上边的距离，越大则向下移动
- `bottom`：定位元素和定位位置下边的距离，越大则向上移动
- `left`：定位元素和定位位置左边的距离
- `right`：定位元素和定位位置右边的距离

**元素的布局**

1. 水平布局：开启绝对定位后，水平方向的[布局等式](#水平方向布局)就需要添加 left 和 right

   left(auto) + margin-left(0) + width + margin-right(0) + right(auto) = 视口宽度

   - 发生过度约束时，如果 9 个值中没有 auto，则自动调整 right 值以使等式满足

   - 如果有 auto，则自动调整 auto 以使等式满足，可以调整的属性为：margin、width、left、right

   - left 和 right 默认 auto，默认 left 为 0，right 最大，如果不设置 left 和 right，当等式不满足时，会自动优先调整 left 和 right

     ```css
     /* 水平居中 */
     margin-left: auto;
     margin-right: auto;
     left: 0px;
     right: 0px;
     ```

   > **扩展：保持元素随窗口移动**
   >
   > 设置等式值分别为：auto + 0 + width + -500px + 50%

2. 垂直布局

   - `top + margin-top / bottom + padding-top / bottom + height + border-top / bottom + bottom = 包含块高度`

   - 默认自动调整 bottom

     ```css
     /* 垂直居中 */
     top: 0px;
     bottom: 0px;
     margin-top: auto;
     margin-bottom: auto;
     ```

### 5.3 固定定位-fixed

- 也是一种绝对定位，大部分特点和绝对定位相同
- 不同的是固定定位永远参照浏览器的**视口**进行定位
- 不会随滚动条滚动

### 5.4 粘滞定位-sticky

- 和相对定位特点基本一致，但是相对于 body 的位置
- 当元素通过滚动条滚动到达某一位置后固定不动
- 通过偏移量指定到达的位置
- 兼容性不好

```css
postion:sticky;
top:10px;
```

### 5.5 元素的层级

- 对于开启了定位的元素，可以使用`z-index`属性指定元素的层级，传入一个整数，值越大层级越高越优先显示

- 如果元素的层级一样，则优先显示靠下的元素
- 祖先元素的层级再高，也不会盖住后代元素

## 6.字体与文本

### 6.1 基本设置

- `color`：用来设置前景色：主要是字体颜色

- `font-size`：字体大小，单位有 px、em、rem

- `font-family`：指定显示的字体格式，以下值指字体的分类，具体字体如'雅黑'等，由浏览器决定

  - serif：衬线字体

  - sans-serif：非衬线字体

  - monospace：等宽字体

  - cursive：草书

  - fantasy：虚幻

  可以指定多个字体，使用逗号隔开，优先使用第一个，如果没有相应字体则依次尝试使用后面的。

```css
p {
    color: red;
    font-size: 40px;
    /* 字体类型中有空格时，使用''括起来 */
    font-family: 'Courier New', Courier, monospace;   
}
```

- `@font-face`：将服务器存储的字体直接提供给用户使用，但受网络加载速度影响，同时要注意版权问题

  - `font-family`：指定字体的名字
  - `url`：服务器中字体的路径
  - `format`：指定文件格式，一般不需要，浏览器会自动解析

  ```css
  @font-face：{
      font-family: 'myFont';
      src: url('./font/external-font.ttf') format("truetype"),
          url('./font/external-font2.ttf');
  }
  
  p {
      font-family: myFont;   
  }
  ```

### 6.2 图标字体

使图标可以按照字体来设置格式。

- font awesome

  1. 下载包

  2. 将 css 和 webfonts 移动到项目中

  3. 引入 all.min.css 到网页中

  4. 使用

     - 通过类名来使用

       ```html
       <head>
       	...
           <link rel="stylesheet" href="fa/css/all.min.css">
       </head>
       <body>
           <!-- 使用 fas 或者 fab 类 -->
           <i class="fas fa-bed" style="font-size: 20px; color: red;"></i>
           <i class="fab fa-accessible-icon"></i>
       </body>
       ```

     - 通过伪元素使用

       ```html
       <head>
       	<style>
               li {
                   list-style: none;
               }
               li::before {
                   /*
                   	content 中设置对应图标的编码
                       设置字体的样式：
                           fab
                           'Font Awesome 6 Brands';
                           fas
                           font-family: 'Font Awesome 6 Free';
                   		font-weight: 900;
                   */
                   content: '\f1b0';
                   font-family: 'Font Awesome 6 Free';
                   font-weight: 900;
               }
           </style>
       </head>
       <body>
           <ul>
               <li>123</li>
               <li>234</li>
           </ul>
       </body>
       ```

     - 使用实体方式：`$#x图标编码;`

       ```html
       <span class="fas">&#xf0f3;</span>
       ```

- [iconfont-阿里巴巴矢量图标库](https://www.iconfont.cn/)

  1. 选择下载的图标添加到项目
  2. 下载项目
  3. 引入`iconfont.css`

### 6.3 行高

行高（line-height）指的是内容占有的实际高度，不指定则默认被内容撑开。可以设置 px，em；如果为整数，则表示字体大小的倍数。其中的单行文字会**默认上下居中**。

`font-size`设置的是字体框的大小，每个字体都是存在于一个字体框中，默认比实际设置的字体框大小小一点。

```html
<style>
    div {
        font-size: 50px;
        line-height: 100px;
    }
</style>
```

通常可以将`line-height`设置为与`height`一样，就可以让单行文字在元素中垂直居中（height也可以不写，默认被内容撑开）。对于多行文字，`line-height`减去`font-size`等于行间距。

### 6.4 其他属性

- `font-weight`

  - `normal`：正常
  - `bold`：加粗

  - `100~900`：九个级别，一般不用

- `font-style`

  - `normal`：正常
  - `italic`：斜体

- `text-decoration`

  - `none`
  - `underline`
  - `line-through`：删除线
  - `overline`

  可同时指定线条的样式：`text-decoration: line-through red dotted;`

- `white-space`：如何处理空白

  - `normal`（默认）：会进行换行和整合空白
  - `nowrap`：不换行
  - `pre`：不整合空白

  > **实用：实现长文本的省略号显示**
  >
  > ```html
  > div {
  >     /* 指定元素宽度 */
  >     width:200px;
  >     /* 空白不换行 */
  >     white-space: nowrap;
  >     /* 超出范围不显示 */
  >     overflow: hidden;
  >     /* 文本超出范围显示省略号 */
  >     text-overflow: ellipsis;
  > }
  > ```

### 6.5 简写属性

`font`可用来设置字体相关的所有属性。格式：

`font: [bold italic] 字体大小[/行高] 字体族`

如果不指定加粗、斜体、行高则使用默认值，会覆盖前面定义的加粗，斜体和行高（如果有）。

### 6.6 文本对齐

- `text-align`

  可选值：`center`、`left`（默认）、`right`、`justify`（两端对齐）；只适用于块元素

- `vertical-align`

  可选值：`baseline`（默认）、`top`、`bottom`、`middle`（和字母x中线对齐）

  > **注意**
  >
  > 在元素中引入图片时由于默认基线对齐，会导致图片底部与元素边框间有一条缝隙，可指定垂直对齐为`top`或者`bottom`。

## 7.背景

### 7.1 常用属性

- `background-color`：设置背景颜色

- `background-image: url("img/1.png")`：设置背景图片，如果背景图片小于元素大小，则背景图片自动平铺；如果大于元素大小，则只会显示部分图片。可以同时设置背景颜色和图片

- `background-repeat`：设置背景的重复方式
  - `repeat`：默认，沿着横纵坐标重复平铺
  - `repeat-x`：沿横坐标重复
  - `repeat-y`：沿纵坐标重复
  - `no-repeat`：背景图片不重复
- `background-position`：设置背景图片位置
  - 通过 top、left、right、bottom、center 来设置各个方位的位置，必须同时指定两个值，只写一个则第二个默认 center，`background-position: top right`
  - 通过像素指定偏移位置，`background-position: 10px(水平) 10px(垂直)`

- `background-clip`：设置背景颜色范围
  - `border-box`（默认）：背景会出现在边框下边
  - `padding-box`：背景不会出现在边框下，只出现在内边距和内容区
  - `content-box`：背景只出现在内容区
- `background-origin`：设置**背景图片**偏移计算的原点
  - `padding-box`（默认）：背景图片从内边距处开始
  - `content-box`：背景图片从内容区处开始
  - `border-box`：背景图片从边框处开始
- `background-size`：设置背景图片大小
  - 通过像素指定，第一个值表示宽度，第二个值表示高度，只写一个则第二个值默认 auto；可设置 100% 表示撑满
  - `cover`：图片比例不变，将元素铺满
  - `contain`：图片比例不变，将图片在元素中完整显示
- `background-attachment`：设置背景图片是否跟随元素移动
  - `scroll`：背景图片会跟随移动
  - `fixed`：背景图片不会跟随移动

### 7.2 简写属性

`background`可用来设置所有背景相关的属性，无顺序要求。例如：`background: #bfa url('img/1.png') center center/contain border-box content-box no-repeat;`

> **注意**
>
> - `background-size`必须写在`background-position`后，使用`/`隔开
> - `background-origin`必须写在`background-clip`前面

### 7.3 雪碧图

当为某个按钮设置 3 种背景分别在默认情况、移入、点击时触发的场景下，由于网络加载的原因，会导致在第一次进行图片切换时出现闪烁。为了解决这种情况可以将三种状态放在同一个大图片中，每次状态改变时仅改变显示图片的偏移。

```css
a:link {
	display: block;
    width: 100px;
    height: 10px;
	background-image: url('img/01.png')    
}
a:hover {
    background-position: -50px 0;
}
a:active {
    background-position: -100px 0;
}
```

### 7.4 渐变

渐变是图片，需要通过`background-image`来设置。

- `linear-gradient(to right, red, yellow[, green...])`

  颜色沿着一个方向渐变，`to left`，`to right`，`to top`，`to bottom`-默认，`to top left`，`xxxdeg`-度数，`xxxturn`-圈，也可以在颜色后指定像素，表示偏移：`linear-gradient(red 60px, yellow)`。

- `repeating-linear-gradient()`

  重复渐变，不受`background-repeat: no-repeat`影响。

- `radial-gradient(100px 100px at top left, red, yellow)`

  放射性渐变，第一个值指定渐变的大小和位置，也可使用`circle`指定正圆，`ellipse`指定椭圆，`closet-side`指定与近边相切，`farthest-side`指定与远边相切。

## 8.图标

```html
<head>
    <link rel="icon" href="favicon.ico">
</head>
```

## 9.VSCODE插件

- JS & CSS Minifier

  用来压缩 CSS 和 JS，使用时打开需要压缩的文件，按 F1，然后选择 Minify:Document 就会在同文件夹下生成 min 压缩文件。

## 10.效果

### 10.1 过渡-transition

过渡时必须是从一个有效值向另一个有效值转换，如果是`auto`则没有过渡效果。

- `transition-property`

  指定要执行过渡的属性，多个属性间使用逗号隔开；`all`表示所有属性。例：`transition-property:width, height;`

- `transition-duration`：指定过渡效果持续时间，单位 s 或者 ms，多个时间用逗号隔开。例：`transition-duration:2s, 3s;`

- `transition-timing-function`：过渡的时序函数

  - `ease`（默认）：先加速再减速
  - `linear`：匀速运动
  - `ease-in`：加速运动
  - `ease-out`：减速运动
  - `ease-in-out`：先加速后减速，比 ease 更强烈
  - `cubic-bezier()`：手动指定贝塞尔曲线参数，参考：https://cubic-bezier.com/

  - `steps(n, start/end)`：分步执行，n 为数字，第二个参数指定读秒前或者读秒后执行，默认 end

- `transition-deplay`：过渡效果的延迟时间，单位 s 或者 ms

- `transition`：简写属性，可同时指定所有过渡相关的属性；如果同时声明延迟时间，则第一个时间为持续时间，第二个时间为延迟时间，其他属性无顺序要求

```html
<style>
    .box1 {
        width: 800px;
        height: 800px;
        background-color: silver;
        overflow: hidden;
    }

    .box1 div {
        margin-top: 20px;
        background-color: red;
        width: 100px;
        height: 100px;
    }

    .box2 {
        /** 过渡，指定一个属性发生变化时的切换方式 */
        transition-duration: 2s;
        transition-timing-function: cubic-bezier(0,1.89,.88,-0.83);
    }

    .box1:hover .box2 {
        margin-left: 700px;
    }
</style>
<body>
    <div class="box1">
        <div class="box2"></div>
    </div>
</body>
```

### 10.2 动画-animation

过渡需要在某个属性发生变化时才触发，动画可以自动触发。需要先设置一个关键帧`@keyframes`，指定动画执行的每一个步骤，然后在元素中指定动画名称和时间。

```css
.box2 {
    /* 动画名称 */
    animation-name: test;
    /* 动画执行时间 */
    animation-duration: 2s;
    /* 动画延迟 */
    animation-delay: 2s;
    /* 动画执行次数，默认 1 次 */
    animation-iteration-count: infinite;
    /* 
    动画运行方向
    - normal（默认）：表示从 from 到 to
    - reverse：反过来
    - alternate：from 到 to 间交替执行
    */
    animation-direction: alternate;
    /* 动画执行步次 */
    animation-timing-function: steps(3);
    /* 
    动画执行状态
    - running：动画执行
    - paused：停止
     */
    animation-play-state: running;
    /* 
    动画填充模式
    - none（默认）：动画执行完毕后元素回到原来位置
    - forwards：动画执行完毕后留在最后的位置
    - backwards：设置了延迟后，等待时元素默认保持自己原来的属性，设置后等待时元素就已经按照关键帧相关的属性设置走
    - both：结合了 forwards 和 backwards
     */
    animation-fill-mode: forwards;
}

@keyframes test {

    /* 动画开始 */
    from {
        margin-left: 0px;
    }

    /* 动画结束 */
    to {
        margin-left: 700px;
    }
}
```

还有一个`animation`简写属性。

**案例：小球下落**

```html
<head>
    <style>
        .outer {
            height: 500px;
            background-color: silver;
            border-bottom: 10px solid black;
            /* 开启 BFC 解决外边距折叠 */
            overflow: hidden;
        }

        .ball {
            border-radius: 50%;
            height: 50px;
            width: 50px;
            background-color: red;
            margin: 0 auto;
            margin-top: 50px;
            animation: fall 2s ease-in forwards;
        }

        @keyframes fall {
            from {
                margin-top: 50px;
            }

            /* 指定当进程达到 33% 时的状态 */
            33% {
                margin-top: 450px;
                animation-timing-function: ease-out;
            }

            66% {
                margin-top: 250px;
                animation-timing-function: ease-in;
            }

            to {
                margin-top: 450px;
                background-color: orange;
            }
        }
    </style>
</head>

<body>
    <div class="outer">
        <div class="ball"></div>
    </div>
</body>
```

### 10.3 变形-transform

变形不会影响页面的布局，可配合`transition`进行设置，例如设置其生效时间。

- `translateX(n)`：元素沿着 X 轴方向平移，n 可以为像素或者百分比 %，百分比是相对于自身计算

- `translateY(n)`：元素沿着 Y 轴方向平移

- `translateZ(n)`：元素沿着 Z 轴方向平移，默认情况下网页不支持透视（进大远小），需要设置网页的视距

  ```html
  <style>
      body {
          /** 人眼距离网页 800 像素 */
          perspective: 800px;
      }
      body:hover .box1 {
          transform: translateZ(-200px);
      }
  </style>
  
  <body>
      <div class="box1"></div>
  </body>
  ```

- `rotateX()`：元素沿着 X 轴方向旋转，单位 deg 或者 turn

- `rotateY()`：元素沿着 Y 轴方向旋转

- `rotateZ()`：元素沿着 Z 轴方向旋转

- `scaleX()`：元素沿着 X 轴缩放，单位为倍数
- `scaleY()`：元素沿着 Y 轴缩放
- `scale()`：元素沿着 X 和 Y 轴缩放

`transform-origin`表示缩放的原点，默认 center，可指定两个值，例：`transform-origin: 20px 20px`。

### 10.4 实战

#### 1.保持可变元素居中

由于 box3 没有设置 width 和 height，大小被内容撑开，因此不能通过设置`left:0 right:0 top:0 bottom:0 margin:auto`使其居中，因为此时会作用于 width 和 height 的大小。

```html
<style>
    .box3 {
        position: absolute;
        left: 50%;
        top: 50%;
        transform: translateX(-50%) translateY(-50%);
    }
</style>
<body>
    <div class="box3">
        aaa
    </div>
</body>
```

#### 2.表

```html
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <link rel="stylesheet" href="CSS/reset.css">
    <link rel="stylesheet" href="fa/css/all.css">
    <style>

        .clock {
            height: 300px;
            width: 300px;
            border-radius: 50%;
            background-color: #bfa;
            margin: 50px auto;
            position: relative;
        }

        /* 设置子元素居中 */
        .clock > div {
            position: absolute;
            top: 0;
            bottom: 0;
            left: 0;
            right: 0;
            margin: auto;
        }

        /* 时针 */
        .hour-wrapper {
            height: 50%;
            width: 50%;
            animation: run 43200s infinite;
        }

        .hour {
            width: 4px;
            height: 50%;
            background-color: black;
            margin: 0 auto;
        }

         /* 分针 */
        .min-wrapper {
            height: 70%;
            width: 70%;
            animation: run 3600s infinite;
        }

        .min {
            width: 2px;
            height: 50%;
            background-color: black;
            margin: 0 auto;
        }

         /* 秒针 */
        .second-wrapper {
            height: 80%;
            width: 80%;
            animation: run 60s steps(60) infinite;
        }

        .second {
            width: 1px;
            height: 50%;
            background-color: black;
            margin: 0 auto;
        }

        /* 旋转 */
        @keyframes run {
            from {
                transform: rotateZ(0);
            }

            to {
                transform: rotateZ(360deg);
            }
        }

    </style>
</head>

<body>
    <div class="clock">
        <div class="hour-wrapper">
            <div class="hour"></div>
        </div>

        <div class="min-wrapper">
            <div class="min"></div>
        </div>

        <div class="second-wrapper">
            <div class="second"></div>
        </div>
    </div>
</body>
```

#### 3.立方体

```html
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <link rel="stylesheet" href="CSS/reset.css">
    <link rel="stylesheet" href="fa/css/all.css">
    <style>
        body {
            perspective: 800px;
        }

        .cube {
            width: 200px;
            height: 200px;
            margin: 100px auto;
            /* 显示 3D 效果，不开则旋转时只显示 2D 的效果 */
            transform-style: preserve-3d;
            animation: rotate 30s infinite linear;
        }

        .cube > div {
            width: 200px;
            height: 200px;
            /* 透明效果 */
            opacity: 0.8;
            position: absolute;
        }

        .box1 {
            width: 200px;
            height: 200px;
            background-color: red;
            transform: rotateY(90deg) translateZ(100px);
        }

        .box2 {
            width: 200px;
            height: 200px;
            background-color: green;
            transform: rotateY(-90deg) translateZ(100px);
        }

        .box3 {
            width: 200px;
            height: 200px;
            background-color: blue;
            transform: rotateX(90deg) translateZ(100px);
        }

        .box4 {
            width: 200px;
            height: 200px;
            background-color: yellow;
            transform: rotateX(-90deg) translateZ(100px);
        }

        .box5 {
            width: 200px;
            height: 200px;
            background-color: orange;
            transform: rotateY(180deg) translateZ(100px)
        }

        .box6 {
            width: 200px;
            height: 200px;
            background-color: turquoise;
            transform: translateZ(100px);
        }

        @keyframes rotate {
            from {
                transform: rotateX(0) rotateZ(0);
            }

            to {
                transform: rotateX(360deg) rotateZ(360deg);
            }
        }
    </style>
</head>

<body>
    <div class="cube">
        <div class="box1"></div>
        <div class="box2"></div>
        <div class="box3"></div>
        <div class="box4"></div>
        <div class="box5"></div>
        <div class="box6"></div>
    </div>
</body>
```

## 11.less

一些兼容性较低的特性：

```html
html {
    /* 声明变量 */
    --color: #bfa;
    --height: 200px;
}

div {
    /* calc()：计算函数 */
    width: calc(100px*2);
    height: var(--height);
}

.box1 {
    background-color: var(--color);
}
```

less 是 CSS 的预处理语言，是 CSS 的增强版，以更少的代码实现更多的功能。其语法与 CSS 基本一致，但是增添了许多扩展。所以浏览器无法直接执行 less 代码，需要先转换为 CSS。

安装 VSCODE 插件：easy LESS，创建 .less 文件：

```less
.box1 {
    background-color: #bfa;

    .box2 {
        background-color: #ff0;

        .box3 {
            background-color: red;
        }
    }
}
```

当保存时，它会自动转换为相应的 CSS：

```css
.box1 {
  background-color: #bfa;
}
.box1 .box2 {
  background-color: #ff0;
}
.box1 .box2 .box3 {
  background-color: red;
}
```

**常用场景**

```less
// 这是个注释，不会被解析
.box1 {
    /*
        这是个 CSS 注释，会被解析到 CSS 文件中
    */
    background-color: #bfa;

    .box2 {
        background-color: #ff0;

        .box3 {
            background-color: red;
        }
    }
}

// 变量
@a: 100px;
@b: #bfa;
@c: box5;

.box4 {
    // 使用变量时使用 @变量名
    width: @a;
    color: @b;
}

// 作为类名或一部分值使用时必须使用 @{变量名} 的形式
.@{c} {
    width: @a;
    background-image: url("@{c}/1.png");
}

// 后面的变量会覆盖前面的
@d: 200px;

.box6 {
    width: @d;
}

@d: 300px;

// 引用其他属性的值
div {
    // 可以直接使用运算 + - * /
    width: 300px + 100px;
    height: $width;
}

.box1 {

    // 子类选择器
    >.box2 {
        color: red;
    }

    // & 表示外层的父元素，这里是 .box1，不会使用嵌套
    &:hover {
        color: red;
    }
}

// p2 复用 box1 的子类 box2 的样式
.p2:extend(.box1 > .box2) {
    color: red;
}
// 另一种写法，但不常用于此场景，见下一例
.p3 {
    .box1();
}

// () 表示创建一个 mixins（混合函数），该样式不会被解析，但是当其他样式引用时就会被复用
.p4() {
    width: 100px;
    height: 100px;
}

.p5 {
    .p4;
}

// mixins 可以传递参数
// 指定默认值后使用时可以不传递参数
.test(@w:100px, @h:100px) {
    width: @w;
    height: @h;
    border: 1px solid red;
}

div {
    .test(300px, 300px);
    // 也可以直接指定参数名乱序传递：.test(@h:300px, @w:200px);
}

// less 提供了一些内置函数
div {
    width: 100px;
    height: 100px;
    background-color: #bfa;
    &:hover {
        // 加深颜色
        background-color: darken(#bfa, 50%);
    }
}

// 引入其他的 less
// @import "syntax2.less";
```

**开启 css 文件和 less 文件的映射关系**

设置 - 扩展 - 在 settings.json 中编辑

## 12.弹性盒-flex

### 12.1 基本使用

flex 是 CSS 中的一种布局手段，用来代替浮动完成页面布局。浮动在一些老项目中仍然需要掌握。flex 可以让元素具有弹性，让元素随页面大小的改变而改变。

**弹性容器**

要使用弹性盒，必须先将一个元素设置为弹性容器，使用`display:flex`（块状）或者`display:inline-flex`（行内）设置。

```css
ul {
    width: 800px;
    border: 1px solid red;
    /* 开启弹性容器 */
    display:flex;
    flex-direction: row;
}
```

- `flex-direction`：指定容器中弹性元素的排列方式
  - `row`（默认值）：主轴从左到右水平排列
  - `row-reverse`：主轴从右到左水平排列
  - `column`：主轴从上到下纵向排列（**改变后主轴方向为纵向**）
  - `column-reverse`：主轴从下到上纵向排列
- 两个概念
  - 主轴：弹性元素排列方向，如果是 row 则表示自左向右
  - 侧轴：主轴的垂直方向

**弹性元素**

弹性容器的直接子元素（不是后代元素）是弹性元素。

```html
<!-- ul 是弹性容器，因此 li 是弹性元素 -->
<ul>
    <li>1</li>
    <li>2</li>
    <li>3</li>
</ul>
```

- `flex-grow`：指定弹性元素的伸展系数
  - 当父元素有多余空间时，子元素如何伸展，0 表示不伸展
  - 父元素的剩余空间会按照比例进行分配
- `flex-shrink`：指定弹性元素的收缩系数
  - 当父元素中空间不足以容纳所有元素时，如何对子元素进行收缩，0 表示不会收缩
  - 子元素会按照比例进行缩小

```css
* {
    margin: 0;
    padding: 0;
    list-style: none;
}

ul {
    width: 800px;
    border: 1px solid red;
    display:flex;
}

li {
    width: 100px;
    height: 100px;
    text-align: center;
    line-height: 100px;
}

li:nth-child(1) {
    background-color: #bfa;
    /* flex-grow: 1; */
    flex-shrink: 1;
}

li:nth-child(2) {
    background-color: silver;
    /* flex-grow: 2; */
    flex-shrink: 2;
}

li:nth-child(3) {
    background-color: gold;
    /* flex-grow: 3; */
    flex-shrink: 3;
}
```

### 12.2 常用样式

#### 1.弹性容器

- `flex-wrap`：弹性元素是否换行
  - `nowrap`（默认）：不会自动换行
  - `wrap`：沿侧轴方向自动换行
  - `wrap-reverse`：沿着侧轴反方向换行

- `flex-flow`：`flex-wrap`和`flex-direction`的简写属性，例如：`flex-flow: row wrap`
- `justify-content`：如何分配主轴上的空白空间
  - `flex-start`（默认）：元素沿着主轴的起边排列
  - `flex-end`：元素沿着主轴的终边排列
  - `center`（常用）：元素居中排列
  - `space-around`：空白分配在每个元素的两侧，元素间的空白长度会叠加
  - `space-evenly`：空白分配在元素的单侧，每个空白长度一致
  - `space-between`：空白分配在元素间
- `align-items`：控制单个 flex 容器内的 flex 项目如何在侧轴上对齐。如果 flex 容器只有一行，则该属性会将所有项目对齐到该行的基线上，如果有多行，则每一行的项目都会根据其自身的`align-self`属性对齐
  - `stretch`（默认）：将每行的各个元素长度设置为相同
  - `flex-start`：元素不会拉伸，沿着侧轴起边对齐
  - `flex-end`：沿着侧轴终边对齐
  - `center`：居中对齐
  - `baseline`：基线对齐
- `align-content`：控制整个 flex 容器内的所有行如何对齐
  - `center`：让最上和最下的空白相等，元素居中显示
  - `flex-start`：元素沿起边显示
  - `flex-end`：元素沿终边显示
  - `space-aroung`：空白分配在元素中间
  - `justify-content`有的它也有

#### 2.弹性元素

- `flex-basis`：元素基础长度，指定的是元素在主轴上的基础长度
  - 如果主轴横向，则指定 width；如果主轴纵向，则指定 height
  - 默认 auto，表示参考元素自身的 width 和 height
  - 如果传递了值则以该值为准
- `flex`：设置`flex-grow`、`flex-shrink`、`flex-basis`的简写属性，例如`flex: 1 1 auto`
  - `initial`：相当于`0 1 auto`
  - `auto`：相当于`1 1 auto`
  - `none`：相当于`0 0 auto`
- `align-self`：弹性元素上设置的用来覆盖`align-items`的属性
- `order`：指定弹性元素的排列顺序

## 13.像素与视口

浏览器解析时，会将 CSS 像素转换为物理像素再呈现。一个 CSS 像素最终呈现多少物理像素由浏览器决定，默认情况下在 PC 端是 1：1 的关系。

默认情况下移动端的网页会把视口设置为 980px（CSS 像素），以确保 PC 端网页可以正常显示在移动端；如果网页大于 980px，则浏览器会自动进行缩放以显示整个网页。最终显示效果是 980px:手机物理像素。因此在编写移动端界面时，必须确保一个比较合理的像素比。

视口是屏幕中用来显示网页的区域，可以通过改变视口的大小改变 CSS 像素与物理像素的比值。

```html
<!-- 改变视口大小 -->
<meta name="vieport" content="width=100px">
```

移动端上最佳显示往往为 1px CSS:2px 物理，因此视口可以设置为：实际物理像素 / 2，CSS 中提供了 device-width 变量来设置完美视口。

```html
<meta name="vieport" content="width=device-width">
```

同时还会追加一个属性设置：

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0">
```

这个设置就是移动端开发需要的设置。

同时由于不同设备的完美视口不同，因此不能再使用 px 布局了，可以设置单位为 vw，表示视口宽度，`100vm=1视口宽度`

```css
.box1 {
    /* 表示占满一行 */
    width: 100vw;
}
```

**VW 适配**

可以使用`rem`来自动适配（自动适用 html 的`font-size`），假设设计图的像素为 750px，则 1px CSS 的 vw 应该为`1 / 750 * 100`约等于 0.133333333vw。

例如一个 48px * 35px 的图标在设计时应该设计为：

```css
html {
    font-size: 0.133333333vw;
}

.box1 {
    width: 48rem;
    height: 35rem;
}
```

但是由于网页中最小的字体为 12px，小于 12px 时自动调整为 12px，因此可以扩大 100 倍：

```html
html {
    font-size: 13.3333333vw;
}

.box1 {
    width: 0.48rem;
    height: 0.35rem;
}
```

在 less 中可以直接使用运算：

```less
window-px: 750;

html {
    /* 设计图的像素需要除以 100 得到 css 中的 rem */
    font-size: (100vw / @window-px * 100);
}

.w {
    width: 7rem;
    margin: 0 auto;
}

header:extend(.w) {
    margin-top: 1rem;
    height: 1rem;
    background-color: #bfa;
    display: flex;
    justify-content: space-between;
    align-items: center;

    .search {
        width: 0.5rem;
        height: 0.5rem;
        background-color: red;
    }

    .logo {
        width: 0.8rem;
        height: 0.8rem;
        background-color: yellow;
    }

    .menu {
        width: 0.3rem;
        height: 0.3rem;
        background-color: blue;
    }
}

.menu-list:extend(.w) {
    margin-top: 1rem;
    height: 3rem;
    background-color: silver;
    display: flex;
    flex-wrap: wrap;
    justify-content: space-around;
    align-content: space-around;

    .menu1 {
        width: 3rem;
        height: 0.7rem;
        background-color: red;
    }

    .menu2 {
        width: 3rem;
        height: 0.7rem;
        background-color: yellow;
    }

    .menu3 {
        width: 3rem;
        height: 0.7rem;
        background-color: blue;
    }

    .menu4 {
        width: 3rem;
        height: 0.7rem;
        background-color: orange;
    }
}
```

## 14.响应式布局

网页可以根据不同的设备或窗口大小呈现不同的效果，使一个网页适用于所有设备。

响应式布局的关键在于媒体查询，通过媒体查询获取设备信息和状态。

### 14.1 媒体查询

#### 1.基本语法

- `@media` 查询规则1[,查询规则2...] {}

  - `all`：适用于所有设备

    ```css
    @media all {
        body {
            background-color: #bfa;
        }
    }
    ```

  - `print`：适用于打印设备

  - `screen`：带屏幕的设备

  - `speech`：屏幕阅读器

#### 2.媒体特性

- `width`：视口宽度（一般都是设置 width）

- `height`：视口高度

- `min-width`：视口最小宽度，视口大于该宽度时生效

- `max-width`：视口最大宽度

  ```css
  <style>
  	/* 当宽度大于等于 500px 或者小于等于 300px 时调整 */
      @media (min-width: 500px), (max-width: 300px) {
          body {
              background-color: #bfa;
          }
      }
  	/* 当宽度大于等于 500px 并且小于等于 700px 时调整 */
      @media (min-width: 500px) and (max-width: 700px) {
          body {
              background-color: #bfa;
          }
      }
  </style>
  ```

常用的断点：

- 小于 768，超小屏幕：`@media (max-width: 768px)`
- 大于 768，小屏幕：`@media (min-width: 768px)`
- 大于 992，中性屏幕：`@media (max-width: 992px)`
- 大于 1200，大屏幕：`@media (max-width: 1200px)`

完整写法

```css
@media only screen and (min-width: 500px) and (max-width: 700px) {
    body {
        background-color: #bfa;
    }
}
```

### 14.2 练习

设计响应式布局时移动端优先，渐进式添加功能。

```html
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <link rel="stylesheet" href="CSS/reset.css">
    <link rel="stylesheet" href="CSS/new.css">
    <link rel="stylesheet" href="fa/css/all.min.css">
    <style>
    </style>
</head>

<body>
    <!-- 导航条 -->
    <div class="top-bar-wrapper">
        <div class="top-bar">
            <!-- 左侧菜单 -->
            <div class="left-menu">
                <!-- 菜单图标 -->
                <ul class="menu-icon">
                    <li></li>
                    <li></li>
                    <li></li>
                </ul>

                <!-- 菜单内容 -->
                <ul class="nav">
                    <li><a href="#">手机</a></li>
                    <li><a href="#">仪器</a></li>
                    <li><a href="#">配件</a></li>
                    <li><a href="#">支持</a></li>
                    <li><a href="#">企业</a></li>
                    <li>
                        <a href="#"><i class="fas fa-search"></i></a>
                        <span>搜索</span>
                    </li>
                </ul>
            </div>
            <!-- 中间的 logo -->
            <h1 class="logo">
                <a href="/">美图手机</a>
            </h1>
            <!-- 右侧用户信息 -->
            <div class="user-info">
                <a href="#"><i class="fas fa-user"></i></a>
            </div>
        </div>
    </div>
</body>

</html>
```

**less 样式**

```less
a {
    text-decoration: none;
    color: #fff;

    &:hover {
        color: darken(#fff, 50%);
    }
}

.top-bar-wrapper {
    background-color: #000;
}

// 导航条
.top-bar {
    // 最大宽度
    max-width: 1200px;
    height: 48px;
    padding: 0 14px;
    display: flex;
    align-items: center;
    justify-content: space-between;
}

// 左侧菜单
.left-menu {
    transition-duration: 0.5s;

    // 图标
    .menu-icon {
        width: 18px;
        height: 48px;
        position: relative;

        // 图标的线 
        li {
            width: 18px;
            height: 1px;
            background-color: #fff;
            position: absolute;

            // 修改变形的时间
            transition-duration: 0.5s;
            // 修改变形的原点
            transform-origin: left center;

            &:nth-child(1) {
                top: 18px;
            }

            &:nth-child(2) {
                top: 24px;
            }

            &:nth-child(3) {
                top: 30px;
            }
        }

        // 鼠标移入效果
        &:hover {
            // 显示隐藏菜单
            & + .nav {
                display: block;
            }
    
            // 图标动画
            li:nth-child(1) {
                transform: rotateZ(41deg);
            }
    
            li:nth-child(2) {
                opacity: 0;
            }
    
            li:nth-child(3) {
                transform: rotateZ(-41deg);
            }
    
        }
    }

    // 弹出菜单样式
    .nav {
        display: none;
        position: absolute;
        top: 48px;
        background-color: #000;
        // 设置 left、right、bottom 为 0，则会自动调整 widht 和 height 达到撑满父元素的效果，此处父元素是 html，因为开启了绝对定位
        left: 0;
        right: 0;
        bottom: 0;
        padding-top: 40px;

        li {
            width: 80%;
            margin: 0 auto;
            border-bottom: 1px solid darken(#fff, 80%);

            a {
                display: block;
                line-height: 44px;
                font-size: 14px;
            }

            // 最后一个 a 元素设置为行内块，防止将 span 挤到下一行
            &:last-child a {
                display: inline-block;
                margin-right: 6px;
            }

            span {
                color: #fff;
                font-size: 14px;
            }
        }
    }
}

// 中间 logo
.logo {
    a {
        text-indent: -9999px;
        display: block;
        width: 122px;
        height: 32px;
        background-color: red;
    }
}

// 设置媒体查询
@media only screen {

    // 断点 768px
    @media (min-width: 768px) {
        body {
            .left-menu {
                order: 2;
                flex: auto;

                // 隐藏菜单图标
                .menu-icon {
                    display: none;
                }

                // 显示菜单
                .nav {
                    display: flex;
                    // 取消 top 等位置的设定
                    position: static;
                    padding: 0px;
                    justify-content: space-around;
                    li {
                        width: auto;
                        border-bottom: none;
                        margin: 0;
                        a {
                            line-height: 48px;
                        }
    
                        span {
                            display: none;
                        }
                    }
                }
            }

            .logo {
                order: 1;
            }

            .user-info {
                order: 3;
            }
        }
    }
}
```

