# 第2章_DOM和BOM

## 1.DOM基础

Document Object Model - 文档对象模型，通过 DOM 可以来任意来修改网页中各个内容。

- **文档**：指的是网页，一个网页就是一个文档

- **对象**：指将网页中的每一个节点都转换为对象，转换完对象以后，就可以以一种纯面向对象的形式来操作网页

- **模型**：模型用来表示节点和节点之间的关系，方便操作页面

- **节点（Node）**：构成网页的最基本的单元，网页中的每一个部分都可以称为是一个节点；虽然都是节点，但是节点的类型却是不同的

  常用的节点：

  - 文档节点 （Document）：代表整个网页
  - 元素节点（Element）：代表网页中的标签
  - 属性节点（Attribute）：代表标签中的属性
  - 文本节点（Text）：代表网页中的文本内容

在网页中浏览器已经为我们提供了`document`对象，它代表的是整个网页，是`window`对象的属性，可以在页面中直接使用。

### 1.1 事件简介

事件指的是用户和浏览器之间的交互行为。比如：点击按钮、关闭窗口、鼠标移动等，我们可以为事件来绑定回调函数来响应事件。

绑定事件的方式：

1. 可以在标签的事件属性中设置相应的 JS 代码（高耦合）

   ```html
   <button onclick="alert('hello world');">按钮</button>
   <!-- 双击触发事件 -->
   <button ondblclick="alert('hello world');">按钮</button>
   ```

2. 可以通过为对象的指定事件属性设置回调函数的形式来处理事件

   ```html
   <button id="btn">btn</button>
   <script>
       var btn = document.getElementById("btn");
       btn.onclick = function () {
           alert("hello world");
       }; 
   </script>
   ```
   
   > **注意**
   >
   > 绑定的回调函数只有在点击时才会触发，因此如果使用的是`for`循环注意变量`i`的使用问题：
   >
   > ```js
   > var btns = document.getElementsByTagName("button");
   > for (var i = 0; i < btns.length; i++) {
   >     btns[i].onclick = function () {
   >         alert("i"); // 当调用时 i 的值恒为 btns.length，需要使用相应对象时应该使用 this
   >         alert(this);
   >     }
   > }
   > ```

### 1.2 文档的加载

浏览器在加载一个页面时，是按照自上向下的顺序加载的，加载一行执行一行。如果将 JS 代码编写到页面的上边，当代码执行时，页面中的 DOM 对象还没有加载，此时将会无法正常获取到 DOM 对象，导致 DOM 操作失败。

解决方式一：

可以将 JS 代码编写到相应标签的下面。

```html
<body>  
    <button id="btn">按钮</button>  
    <script>  
        var btn = document.getElementById("btn");  
        btn.onclick = function(){  
        };  
	</script>  
</body>  
```

解决方式二：

将 JS 代码编写到`window.onload = function(){}`中，`window.onload`对应的回调函数会在整个页面加载完毕以后才执行，所以可以确保代码执行时，DOM 对象已经加载完毕了。

```html
<script>  
    window.onload = function(){  
        var btn = document.getElementById("btn");  
        btn.onclick = function(){  
        };  
    };  
</script>	    
```

### 1.3 常用属性和方法

#### 1.通过`document`调用

- 根据元素的`id`属性获取一个元素节点对象

  `document.getElementById("id属性值")`

- 根据元素的`name`属性值获取一组元素节点对象（类数组）

  `document.getElementsByName("name属性值")`

- 根据标签名来获取一组元素节点对象（类数组）

  `document.getElementsByTagName("标签名")`

> **注意**
>
> 即使查询到 1 个对象也会放到类数组中返回。

#### 2.通过元素调用

- **`元素.innerHTML`和`元素.innerText`**

  读取标签内部的文本内容。这两个属性并没有在 DOM 标准定义，但是大部分浏览器都支持这两个属性。两个属性作用类似，都可以获取到标签内部的内容，不同是`innerHTML`会获取到 html 标签，而`innerText`会自动去除标签。使用这两个属性来设置标签内部的内容时，没有任何区别。

- **`文本节点.nodeValue`**

  获取文本节点的文本值。

  ```js
  <ul id = "city">
      <li id = "beijing">beijing</li>
      <li>shanghai</li>
      <li>dongjing</li>
      <li>shouer</li>
  </ul>
  <input type="button" value="button" name="btn">
  <script>
      var btn = document.getElementsByName("btn")[0];
      btn.onclick = function() {
          var beijing = document.getElementById("beijing");
          alert(beijing.firstChild.nodeValue); // beijing
      }
  </script>
  ```

- **`元素.属性名`**

  读取元素属性。例如`element.name`、`element.value`等。但是读取`class`属性时需要使用`element.className`（因为`class`是保留字）。

- **`元素.getElementsByTagName()`**

  获取指定标签名的所有后代元素（不只是子元素）。

- **`元素.childNodes`**

  获取当前节点的所有子节点。`childNodes`属性会获取包括文本节点在内的所有节点，==标签间空白==也会当成文本节点。

  ```html
  <ul id = "city">
      <li>beijing</li>
      <li>shanghai</li>
      <li>dongjing</li>
      <li>shouer</li>
  </ul>
  <input type="button" value="button" name="btn">
  <script>
      var btn = document.getElementsByName("btn")[0];
      btn.onclick = function() {
          var ul = document.getElementById("city");
          var lis = ul.childNodes;
          alert(lis.length); // 9
      }
  </script>
  ```

  > **注意**
  >
  > 在 IE8 及以下的浏览器中，不会将空白文本当成子节点，所以该属性在 IE8 中会返回 4 个子元素而其他浏览器是 9 个。

- **`元素.children`**

  获取当前元素的所有子元素，不会获取到空白的文本子节点。

  > **注意**
  >
  > 子节点包括便签元素中的文本，子元素只包含标签元素。

- **`元素.firstChild`**

  获取当前元素的第一个子节点，会获取到空白的文本子节点。

- **`元素.firstElementChild`**

  获取当前元素的第一个子元素，但是不支持 IE8 及以下的浏览器。

- **`元素.lastChild`**

  获取当前元素的最后一个子节点，会获取到空白的文本子节点。

- **`元素.parentNode`**

  获取当前元素的父节点，肯定是元素。

- **`元素.previousSibling`**

  获取当前元素的前一个兄弟节点，会获取到空白的文本节点。

- **`元素.previousElementSibling`**

  获取前一个兄弟元素，IE8 及以下不支持。

- **`元素.nextSibling`**

  获取当前元素的后一个兄弟节点，会获取到空白的文本节点。

### 1.4 其他属性和方法

- **`document.all`**

  获取页面中的所有元素，相当于`document.getElementsByTagName("*")`。

- **`document.documentElement`**

  获取页面中 html 根元素。

- **`document.body`**

  获取页面中的 body 元素。

- **`document.getElementsByClassName()`**

  根据元素的 class 属性值查询一组元素节点对象，这个方法不支持 IE8 及以下的浏览器。

- **`document.querySelector()`**

  根据 CSS 选择器去页面中查询一个元素，如果匹配到的元素有多个，则它会返回查询到的第一个元素。

- **`document.querySelectorAll()`**

  根据 CSS 选择器去页面中查询一组元素，会将匹配到所有元素封装到一个数组中返回，即使只匹配到一个。

> **扩展：取消超链接的自动跳转**
>
> 当给超链接绑定回调函数时，如果设置了`href`属性但是想取消页面跳转，则可以返回`false`；或者将`a`标签的`href`属性设置为`javascript:;`。
>
> ```js
> var a = document.getElementsByTagName("a")[0];
> a.onclick = function() {
>     alert("hello");
> }
> ```

### 1.5 增删改相关属性和方法

- **`document.createElement("TagName")`**

  可以用于创建一个元素节点对象，它需要一个标签名作为参数，将会根据该标签名创建元素节点对象，并将创建好的对象作为返回值返回。

- **`document.createTextNode("textContent")`**

  可以根据文本内容创建一个文本节点对象。

- **`父节点.appendChild(子节点)`**

  向父节点中添加指定的子节点。

  例：

  ```js
  window.onload = function () {
      var li = document.createElement("li");
      var text = document.createTextNode("beijing");
      li.appendChild(text);
      var city = document.getElementById("city");
      city.appendChild(li);
  }
  ```

- **`父节点.insertBefore(新节点, 旧节点)`**

  将一个新的节点插入到旧节点的前边。

- **`父节点.replaceChild(新节点, 旧节点)`**

  使用一个新的节点去替换旧节点。

- **`父节点.removeChild(子节点)`**

  删除指定的子节点。

  常用方式：`子节点.parentNode.removeChild(子节点)`

以上方法，实际就是改变了相应元素（标签）的`innerHTML`的值，因此更常用方式如下：

```js
// 向 city 中添加广州  
var city = document.getElementById("city");  
/*  
*  使用 innerHTML 也可以完成 DOM 的增删改的相关操作  
*  city.innerHTML += "<li>广州</li>";
*  一般我们会两种方式结合使用
*/
// 创建一个 li  
var li = document.createElement("li");  
// 向 li 中设置文本  
li.innerHTML = "广州";  
// 将 li 添加到 city 中  
city.appendChild(li);  
```

### 1.6 CSS相关属性和方法

#### 1.内联样式

使用`style`属性来操作元素的内联样式，即在标签中声明的样式，如果写在头部的 CSS 中则无法操作。

- **读取内联样式**

  `元素.style.样式名`，例：`元素.style.width`、`元素.style.height`。

- **修改内联样式**

  `元素.style.样式名 = 样式值（字符串）`，通过`style`修改和读取的样式都是内联样式，由于内联样式的优先级比较高，所以我们通过 JS 来修改的样式往往会立即生效，但是如果样式中设置了`!important`，则内联样式将不会生效。

> **注意**
>
> 如果样式名中带有`-`，则需要将样式名修改为驼峰命名法将`-`去掉，然后`-`后首字母改大写，比如背景颜色：`background-color` -> `backgroundColor`。
>
> ```js
> box1.style.backgroundColor = "yellow"；
> box1.style.borderTopWidth = "20px";
> ```

#### 2.当前样式

- 正常浏览器（IE9 及以上）

  使用`getComputedStyle()`方法，这个方法是`window`对象的方法，可以返回一个对象，这个对象中封装了当前元素生效样式。

  参数：

  1. 要获取样式的元素
  2. 可以传递一个伪元素，一般传`null`

  例子：获取元素的宽度

  ```js
  getComputedStyle(box, null).width;
  ```

  可以获取到实际的样式而不是默认值（例如没有设置 width 时不会获取到 auto 而是获取到实际的值）。

- IE8

  使用`元素.currentStyle.样式名`，例如`box.currentStyle.width`。

> **注意**
>
> 通过以上两个方法读取到样式都是只读的不能修改。

**实现兼容性**

调用`对象.属性`时如果属性不存在也不会报错，如果直接使用该属性则找不到时会报错。

```js
/*  
* 定义一个函数，用来获取指定元素的当前的样式  
* 参数：  
* 	obj 要获取样式的元素  
* 	name 要获取的样式名  
*/  
function getStyle(obj , name){  
    if(window.getComputedStyle){  
        return getComputedStyle(obj , null)[name];  
    }else{  
        return obj.currentStyle[name];  
    }
    // return window.getComputedStyle?getComputedStyle(obj , null[name]:obj.currentStyle[name];	
    // return window.getComputedStyle || obj.currentStyle[name];
}  
```

### 1.7 其他样式相关的属性

> **注意**
>
> - 以下样式都是只读的并且返回的是数值可以直接计算
> - 使用`元素.属性`的方式读取
>
> - 未指明偏移量都是相对于当前窗口左上角

- `clientHeight`

  元素的可见高度，包括元素的内容区和内边距的高度，不包括边框和被滚动条隐藏的部分。

- `clientWidth`

  元素的可见宽度，包括元素的内容区和内边距的宽度，不包括边框和被滚动条隐藏的部分。

- `scrollHeight`、`scrollWidth`

  获取元素滚动区域的整体高度和宽度。

- `scrollTop`、`scrollLeft`

  获取元素垂直和水平滚动条滚动移动的距离。

  > **扩展：判断滚动条是否滚动到底**
  >
  > 垂直滚动：`scrollHeight - scrollTop <= clientHeight`
  >
  > 水平滚动：`scrollWidth - scrollLeft <= clientWidth`

- `offsetHeight`

  整个元素的高度，包括内容区、内边距、边框。

- `offfsetWidth`

  整个元素的宽度，包括内容区、内边距、边框。

- `offsetParent`

  获取当前元素的定位祖先元素（离他最近的开启了定位的祖先元素，`position`不是`static`），如果所有的元素都没有开启定位，则返回`body`。

- `offsetLeft`、`offsetTop`

  当前元素和定位祖先元素之间的水平和垂直偏移量（一般是定位祖先元素内边距）。

- `checkboxObject.disabled`

  true 或者 false，表示是否禁用 checkbox。

## 2.事件

### 2.1 事件对象

当响应函数被调用时，浏览器每次都会将一个事件对象作为实参传递进响应函数中，这个事件对象中封装了当前事件的相关信息，比如：鼠标的坐标，键盘的按键，鼠标的按键，滚轮的方向。

可以在响应函数中定义一个形参，来使用事件对象，但是在 IE8 以下浏览器中事件对象没有做完实参传递，而是作为`window`对象的属性保存。

兼容的使用方法：

```js
元素.事件 = function(event){  
    event = event || window.event;  
};  

元素.事件 = function(e){  
	e = e || event;  
}; 
```

**获取到指定区域内鼠标的坐标**

```js
var areaDiv = document.getElementById("areaDiv");
var showMsg = document.getElementById("showMsg");
areaDiv.onmousemove = function(event) {
    var x = event.clientX;
    var y = event.clientY;
    showMsg.innerHTML = "x = " + x + ", y = " + y;
}
```

- `clientX`和`clientY`

  用于获取当事件触发时鼠标在当前可见区域内的水平和垂直坐标。

- `pageX`和`pageY`

  可以获取鼠标相对于当前页面的坐标。

**例：元素跟随鼠标移动**

```js
window.onload = function () {
    document.onmousemove = function (event) {
        var left = event.clientX;
        var top = event.clientY;
        var box1 = document.getElementsByTagName("div")[0];
        box1.style.left = left + document.documentElement.scrollLeft + "px";
        box1.style.top = top + document.documentElement.scrollTop + "px";

        // var left = event.pageX;
        // var top = event.pageY;
        // var box1 = document.getElementsByTagName("div")[0];
        // box1.style.left = left + "px";
        // box1.style.top = top + "px";
    }
}
```

### 2.2 事件的冒泡

事件的冒泡指的是事件向上传导。当后代元素上的事件被触发时，将会导致其祖先元素上的同类事件也会触发。

例：

```js
// span 是 box1 的子元素，点击 span 后会先弹出"span"，再弹出"box1"
window.onload = function () {
    var box1 = document.getElementsByTagName("div")[0];
    box1.onclick = function() {
        alert("box1");
    }
    var span = document.getElementsByTagName("span")[0];
    span.onclick = function () {
        alert("span");
    }
}
```

事件的冒泡大部分情况下都是有益的，如果需要取消冒泡，则需要使用事件对象来取消，将事件对象的`cancelBubble`设置为`true`即可取消冒泡。

例：

```js
window.onload = function () {
    var box1 = document.getElementsByTagName("div")[0];
    box1.onclick = function() {
        alert("box1");
    }
    var span = document.getElementsByTagName("span")[0];
    span.onclick = function (event) {
        // 取消子元素的事件冒泡
        event.cancelBubble = true;
        alert("span");
    }
}
```

### 2.3 事件的委派

指将事件统一绑定给元素的共同的祖先元素，这样当后代元素上的事件触发时，会一直冒泡到祖先元素，从而通过祖先元素的响应函数来处理事件。事件委派是利用了冒泡，通过委派可以减少事件绑定的次数，提高程序的性能。

例：

```js
<!-- 点击任意连接都会触发窗口弹出函数 -->
<head>
    <script>
        window.onload = function () {
            var ul = document.getElementsByTagName("ul")[0];
            ul.onclick = function() {
                alert("link");
            }
        }
    </script>
</head>

<body>
    <ul>
        <li><a href="javascript:;">link</a></li>
        <li><a href="javascript:;">link</a></li>
        <li><a href="javascript:;">link</a></li>
    </ul>
</body>
```

上面的方法存在问题：由于`li`是块元素，因此点击范围内任意位置都会触发弹窗，期望的是只点击链接才触发。

这里引入一个事件的属性——**target**，表示触发事件的元素。

```js
<script>
    window.onload = function () {
        var ul = document.getElementsByTagName("ul")[0];
        ul.onclick = function(event) {
            // 如果点击的是 a 标签则触发弹窗，否则不触发
            if (event.target.innerHTML == "link") {
                alert("link");
            }
        }
    }
</script>
```

### 2.4 事件的绑定

通过以下方式只能同时为某个事件绑定一个响应函数，最后绑定的函数会覆盖之前绑定的函数。

```js
btn.onclick = function() {
    alert(1);
}
btn.onclick = function() {
    alert(2);
}
```

通过`addEventListener()`方法也可以为元素绑定响应函数。使用该函数可以同时为一个元素的相同事件同时绑定多个响应函数，这样当事件被触发时，响应函数将会按照函数的绑定顺序执行，不适用于 IE8 及以下。

参数：

1. 事件的字符串，不要`on`
2. 回调函数，当事件触发时该函数会被调用
3. 是否在捕获阶段触发事件，需要一个布尔值，一般都传`false`

```js
btn.addEventListener("click", function(){  
	alert(1);  
}, false);  
  
btn.addEventListener("click", function(){  
	alert(2);  
}, false);					  
```

在 IE8 中可以使用`attachEvent()`来绑定事件。这个方法也可以同时为一个事件绑定多个处理函数，不同的是它是后绑定先执行，执行顺序和`addEventListener()`相反。

参数：

1. 事件的字符串，要`on`
2. 回调函数

定义一个函数，用来为指定元素绑定响应函数：

```js
/* 
 * 注意：
 *  addEventListener() 中的 this 是绑定事件的对象  
 *  attachEvent() 中的 this 是 window  
 *  需要统一两个方法的 this  
 * 
 *  
 * 参数：  
 * 	obj 要绑定事件的对象  
 * 	eventStr 事件的字符串(不要 on)  
 *  callback 回调函数  
 */  
function bind(obj, eventStr, callback) {  
    if (obj.addEventListener){  
        // 大部分浏览器兼容的方式  
        obj.addEventListener(eventStr, callback, false);  
    } else {   
        // IE8 及以下  
        obj.attachEvent("on" + eventStr, function(){  
            // 在匿名函数中调用回调函数  
            callback.call(obj);  
        });  
    }  
}  

bind(btn, "click", function(){alert(this);});
```

### 2.5 事件的传播

关于事件的传播网景公司和微软公司有不同的理解：

微软公司认为事件应该是由内向外传播，也就是当事件触发时，应该先触发当前元素上的事件，然后再向当前元素的祖先元素上传播，也就说事件应该在冒泡阶段执行；网景公司认为事件应该是由外向内传播的，也就是当前事件触发时，应该先触发当前元素的最外层的祖先元素的事件，然后在向内传播给后代元素。

W3C 综合了两个公司的方案，将事件传播分成了三个阶段：

1. 捕获阶段

   在捕获阶段时从最外层的祖先元素，向目标元素进行事件的捕获，但是默认此时不会触发事件。

2. 目标阶段

   事件捕获到目标元素，捕获结束开始在目标元素上触发事件。

3. 冒泡阶段

   事件从目标元素向他的祖先元素传递，依次触发祖先元素上的事件。

如果希望在捕获阶段就触发事件，可以将`addEventListener()`的第三个参数设置为`true`。一般情况下我们不会希望在捕获阶段触发事件，所以这个参数一般都是`false`。

```js
obj.addEventListener(eventStr, callback, true);
```

IE8 及以下的浏览器中没有捕获阶段。

### 2.6 常用事件

#### 1.鼠标事件

##### 1.1 拖拽事件

拖拽的流程：

1. 当鼠标在被拖拽元素上按下时，开始拖拽——`onmousedown`
2. 当鼠标移动时被拖拽元素跟随鼠标移动——`onmousemove`
3. 当鼠标松开时，被拖拽元素固定在当前位置——`onmouseup`

```html
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <style>
        * {
            margin: 0;
            padding: 0;
        }

        body {
            height: 1000px;
        }

        .box1 {
            height: 200px;
            width: 200px;
            background-color: #bfa;
            position: absolute;
        }

        .box2 {
            height: 200px;
            width: 200px;
            background-color: red;
            position: absolute;
            left: 500px;
            top: 500px;
        }
    </style>
    <script>
        window.onload = function () {
            var box1 = document.getElementsByClassName("box1")[0];
            box1.onmousedown = function (event) {
                // 求 div 相对于鼠标的偏移
                var offsetL = event.pageX - box1.offsetLeft;
                var offsetT = event.pageY - box1.offsetTop;
                document.onmousemove = function (event) {
                    var left = event.pageX;
                    var top = event.pageY;
                    box1.style.left = left - offsetL + "px";
                    box1.style.top = top - offsetT + "px";
                }
                /*
                 * 注意：
                 * - 这里要给 document 绑定鼠标松开事件
                 * - 给 box1 绑定的话，如果鼠标是在 box2 上松开则不会触发该事件
                 */
                document.onmouseup = function () {
                    document.onmousemove = null;
                    /*
                     * 注意：
                     * - 要取消文档的 onmouseup 事件，否则每次点击松开都会触发
                     */
                    document.onmouseup = null;
                }
            }

        }
    </script>
</head>

<body>
    <ul>
        <div class="box1"></div>
        <div class="box2"></div>
    </ul>
</body>
```

当拖拽网页中的内容时，浏览器会默认去搜索引擎中搜索内容，此时会导致拖拽功能异常。可以在移动函数中返回`false`来取消该行为。

```js
box1.onmousedown = function (event) {
    ...
    return false;
}
```

以上方法对 IE8 不起作用。

##### 1.2 滚轮事件

- `onwheel`

  事件在鼠标滚轮在元素上下滚动时触发。同样可以在触摸板上滚动或放大缩小区域时触发（如笔记本上的触摸板）。

例：实现当滚轮向下滚时变长，向上滚动时变短。

```js
window.onload = function () {
    var box1 = document.getElementsByClassName("box1")[0];
    // 绑定滚动响应事件
    box1.onwheel = function(event) {
        // 当 wheelDelta 大于 0 表示向上滚
        if (event.wheelDelta > 0) {
            box1.style.height = box1.clientHeight - 100 + "px";
        } else {
            box1.style.height = box1.clientHeight + 100 + "px";
        }
        // 取消浏览器默认的滚动条行为
        // 也可以返回 false
        event.preventDefault();
    }
}
```

- `onscroll`

  事件在元素滚动条在滚动时触发。滚动条必须存在，否则不会触发。无论以那种方式，只要滚动条滚动，事件都会触发。触发方式：鼠标滚轮，鼠标拖动，键盘上下键，或者设置的滚动函数，如`scrollTo`，`scrollBy`，`scrollByLines`，`scrollByPages`。

当鼠标滚轮滚动时，`onwheel`事件先被触发，若滚动条滚动，则`onscroll`事件会相继被触发。

#### 2.键盘事件

- `onkeydown`：按键被按下

  如果一直按着某个按键不松手，则事件会一直触发。当`onkeydown`连续触发时，第一次和第二次之间会间隔稍微长一点，其他的会非常的快，这种设计是为了防止误操作的发生。

- `onkeyup`：按键被松开

  键盘事件一般都会绑定给一些可以获取到焦点的对象（例如输入框）或者是`document`。

相关事件属性：

- `keyCode`：获取按键的编码

  通过它可以判断哪个按键被按下。除此之外，`altKey`、`ctrlKey`、`shiftKey`这三个用来判断`alt`、`ctrl`和`shift`是否被按下，如果按下则返回`true`，否则返回`false`。

```js
// 判断一个 y 是否被按下  
// 判断 y 和 ctrl 是否同时被按下  
if(event.keyCode === 89 && event.ctrlKey){  
	console.log("ctrl 和 y 都被按下了");  
}  
input.onkeydown = function(event) {  
    event = event || window.event;  
    // 数字 48 - 57  
    // 使文本框中不能输入数字  
    if(event.keyCode >= 48 && event.keyCode <= 57) {  
        //在文本框中输入内容，属于 onkeydown 的默认行为  
        //如果在 onkeydown 中取消了默认行为，则输入的内容，不会出现在文本框中  
        return false;  
    }  
}; 
```

## 3.BOM基础

Browser Object Model - 浏览器对象模型，BOM 可以使我们通过 JS 来操作浏览器。

在 BOM 中为我们提供了一组对象，用来完成对浏览器的操作：

- `Window`

  代表的是整个浏览器的窗口，同时`Window`也是网页中的全局对象。

- `Navigator`

  代表的当前浏览器的信息，通过该对象可以来识别不同的浏览器。

- `Location`

  代表当前浏览器的地址栏信息，通过`Location`可以获取地址栏信息，或者操作浏览器跳转页面。

- `History`

  代表浏览器的历史记录，可以通过该对象来操作浏览器的历史记录。

  由于隐私原因，该对象不能获取到具体的历史记录，只能操作浏览器向前或向后翻页，而且该操作只在当次浏览器打开时有效。

- `Screen`

  代表用户的屏幕的信息，通过该对象可以获取到用户的显示器的相关的信息。

这些 BOM 对象在浏览器中都是作为`Window`对象的属性保存的，可以调用`Window`对象属性来使用，也可以直接使用。

### 3.1 Navigator

由于历史原因，`Navigator`对象中的大部分属性都已经不能帮助我们识别浏览器了，一般我们只会使用`userAgent`来判断浏览器的信息，`userAgent`是一个字符串，这个字符串中包含有用来描述浏览器信息的内容，
不同的浏览器会有不同的`userAgent`。

```js
var ua = navigator.userAgent;
if (/firefox/i.test(ua)) {
    alert("你是火狐！！！");
} else if (/chrome/i.test(ua)) {
    alert("你是Chrome");
} else if (/msie/i.test(ua)) {
    alert("你是IE浏览器~~~");
}
// 通过 IE 的独有属性来判断是否是 IE11
else if ("ActiveXObject" in window) {
    alert("你是IE11，枪毙了你~~~");
}
```

### 3.2 History

**属性**

- `length`

  获取当前访问的链接数量。

**方法**

- `back()`

  可以用来回退到上一个页面，作用和浏览器的回退按钮一样。

- `forward()`

  可以跳转下一个页面，作用和浏览器的前进按钮一样。

- `go()`

  可以用来跳转到指定的页面，它需要一个整数作为参数：

  - `1`：表示向前跳转一个页面，相当于`forward()`
  - `2`：表示向前跳转两个页面
  - `-1`：表示向后跳转一个页面
  - `-2`：表示向后跳转两个页面

### 3.3 Location

如果直接打印`location`则可以获取到地址栏的信息（当前页面的完整路径）。

如果直接将`location`属性修改为一个完整的路径，或相对路径，则我们页面会自动跳转到该路径，并且会生成相应的历史记录。

**属性**

- `href`：返回完整 URL

- `hostname`

- `port`
- `protocol`

**方法**

- `assign(url)`

  用来跳转到其他的页面，作用和直接修改`location`一样。

- `reload()`

  用于重新加载当前页面，作用和刷新按钮一样。

- `replace(url)`

  可以使用一个新的页面替换当前页面，调用完毕也会跳转页面，但是不会生成历史记录，不能使用回退按钮回退。

### 3.4 Window

- `confirm(text)`

  弹出一个带有确认和取消的提示框并附有提示信息，返回`true`或者`false`。

- `alert(msg)`

  弹出一个显示文本的提示框。

- `setInterval(function, milliseconds)`

  定时调用，可以将一个函数，每隔一段时间执行一次。返回一个`Number`类型的数据，这个数字用来作为定时器的唯一标识。

  参数：

  1. 回调函数，该函数会每隔一段时间被调用一次
  2. 每次调用间隔的时间，单位是毫秒

- `clearInterval(id)`

  可以用来关闭一个定时器，方法中需要一个定时器的标识作为参数，这样将关闭标识对应的定时器。

  `clearInterval()`可以接收任意参数，如果参数是一个有效的定时器的标识，则停止对应的定时器；如果参数不是一个有效的标识，则什么也不做。

  ```js
  var num = 1;  
  var timer = setInterval(function() {  
  	count.innerHTML = num++;  
  	if(num == 11) {  
  		//关闭定时器  
  		clearInterval(timer);  
  	}  
  }, 1000);
  ```

  **案例：移动元素**

  ```js
  window.onload = function () {
      var box = document.getElementById("box");
      var direction = 0;
      var speed = 10;
  
      document.onkeydown = function (event) {
          direction = event.keyCode;
      }
      document.onkeyup = function () {
          direction = 0;
      }
  
      setInterval(function () {
          switch (direction) {
              case 37:
                  box.style.left = box.offsetLeft - speed + "px";
                  break;
              case 39:
                  box.style.left = box.offsetLeft + speed + "px";
                  break;
              case 38:
                  box.style.top = box.offsetTop - speed + "px";
                  break;
              case 40:
                  box.style.top = box.offsetTop + speed + "px";
                  break;
          }
      }, 30);
  }
  ```

- `setTimeout(function, milliseconds)`

  延时调用一个函数不马上执行，而是隔一段时间以后在执行，而且只会执行一次。

  延时调用和定时调用的区别，定时调用会执行多次，而延时调用只会执行一次，延时调用和定时调用实际上是可以互相代替的，在开发中可以根据自己需要去选择。

- `clearTimeout(id)`

  关闭一个延时调用。

### 3.5 应用：轮播图

```js
<head>
    <meta charset="UTF-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
    <style>
        * {
            padding: 0;
            margin: 0;
        }

        #outer {
            height: 300px;
            width: 620px;
            margin: 50px auto;
            background-color: black;
            padding: 10px 0;
            position: relative;
            overflow: hidden;
        }

        #imgList {
            list-style: none;
            position: absolute;
        }

        ul {
            display: flex;
        }

        li {
            margin: auto 10px;
        }

        .nav {
            position: absolute;
            bottom: 0px;
        }

        .nav a {
            display: inline-block;
            width: 15px;
            height: 15px;
            background-color: silver;
            margin: 0 5px;
            opacity: 0.5;
        }

        .nav a:hover {
            background-color: black;
        }

        /* 模拟图片 */
        .img1 {
            height: 300px;
            width: 600px;
            background-color: red;
        }

        .img2 {
            height: 300px;
            width: 600px;
            background-color: #bfa;
        }

        .img3 {
            height: 300px;
            width: 600px;
            background-color: yellow;
        }

        .img4 {
            height: 300px;
            width: 600px;
            background-color: blue;
        }

        .img5 {
            height: 300px;
            width: 600px;
            background-color: green;
        }
    </style>
    <script>
        window.onload = function () {
            var imgList = document.getElementById("imgList");
            var liList = document.getElementsByTagName("li");
            // 设置 nav 居中
            var outer = document.getElementById("outer");
            var nav = document.getElementsByClassName("nav")[0];
            nav.style.left = (outer.offsetWidth - nav.offsetWidth) / 2 + "px";
            // 设置导航块高亮
            var index = 0;
            var allA = document.getElementsByTagName("a");
            allA[index].style.backgroundColor = "black";
            // 设置点击导航切换图片
            for (var i = 0; i < allA.length; i++) {
                // 为每个 a 标签设置编号
                allA[i].num = i;
                allA[i].onclick = function () {
                    // 取消自动切换，当移动完成时再通过回调函数开启自动切换
                    clearInterval(autoTimer);
                    // 设置步长
                    // 防止当 index 被自动切换设置为下一个的同时点击下一个导航块会导致 step 为0的问题
                    var step = Math.abs(this.num - index) || 50;
                    index = this.num;
                    move(imgList, step * 50, -index * outer.offsetWidth, autoChange);
                    setHighLight();
                }
            }
            // 自动切换图片
            var autoTimer;
            function autoChange() {
                autoTimer = setInterval(
                    function () {
                        index++;
                        move(imgList, 10, -index * outer.offsetWidth, setHighLight)
                    }, 3000);
            }
            autoChange();

            // 设置标签高亮
            function setHighLight() {
                if (index >= liList.length - 1) {
                    index %= liList.length - 1;
                    imgList.style.left = 0;
                }
                for (var i = 0; i < allA.length; i++) {
                    // 取消高亮标签的内链样式使其恢复原样
                    // 如果在这里设置颜色为 silver，则会失去 hover 属性（被内链样式覆盖了）
                    allA[i].style.backgroundColor = "";
                }
                allA[index].style.backgroundColor = "black";
            }
        }

        /*
            移动函数，三个参数：
            1.移动对象
            2.每次移动的步长
            3.终止位置
            4.回调函数
        */
        function move(obj, step, endPosition, callback) {
            clearInterval(obj.timer);
            obj.timer = setInterval(function () {
                if (obj.offsetLeft === endPosition) {
                    clearInterval(obj.timer);
                    callback && callback();
                }
                // 如果当前偏移大于指定偏移，则向左移动
                else if (obj.offsetLeft > endPosition) {
                    obj.style.left = obj.offsetLeft - step + "px";
                    if (obj.offsetLeft <= endPosition) {
                        obj.style.left = endPosition + "px";
                    }
                } else {
                    obj.style.left = obj.offsetLeft + step + "px";
                    if (obj.offsetLeft >= endPosition) {
                        obj.style.left = endPosition + "px";
                    }
                }
            }, 30);
        }
    </script>
</head>

<body>
    <div id="outer">
        <ul id="imgList">
            <li>
                <div class="img1"></div>
            </li>
            <li>
                <div class="img2"></div>
            </li>
            <li>
                <div class="img3"></div>
            </li>
            <li>
                <div class="img4"></div>
            </li>
            <li>
                <div class="img5"></div>
            </li>
            <li>
                <div class="img1"></div>
            </li>
        </ul>
        <div class="nav">
            <a href="javascript:;"></a>
            <a href="javascript:;"></a>
            <a href="javascript:;"></a>
            <a href="javascript:;"></a>
            <a href="javascript:;"></a>
        </div>
    </div>
</body>
```

### 3.6 通过类修改样式

通过`style`属性来修改元素的样式，每修改一个样式，浏览器就需要重新渲染一次页面，这样的执行的性能是比较差的，而且这种形式当我们要修改多个样式时，也不太方便。

我们可以通过修改元素的`class`属性来间接的修改样式。这样一来我们只需要修改一次，即可同时修改多个样式，浏览器只需要重新渲染页面一次，性能比较好，并且这种方式可以使表现和行为进一步的分离。

```js
// 添加新的 class 属性，将新的样式定义在 b2 中
box.className += " b2";
```

```js
// 定义一个函数，用来向一个元素中添加指定的 class 属性值  
/*  
 * 参数:  
 * - obj：要添加 class 属性的元素  
 * - cn：要添加的 class 值  
 * 	  
 */  
function addClass(obj, cn) {  
	if (!hasClass(obj, cn)) {  
		obj.className += " " + cn;  
	}  
}  
/*  
 * 判断一个元素中是否含有指定的 class 属性值  
 * 如果有该 class 则返回 true，没有则返回 false  
 * 	  
 */  
function hasClass(obj, cn) {  
	var reg = new RegExp("\\b" + cn + "\\b");  
	return reg.test(obj.className);  
}  
/*  
 * 删除一个元素中的指定的 class 属性  
 */  
function removeClass(obj, cn) {  
	// 创建一个正则表达式  
	var reg = new RegExp("\\b" + cn + "\\b");  
	// 删除 class  
	obj.className = obj.className.replace(reg, "");  
}  
/*  
 * toggleClass 可以用来切换一个类  
 * - 如果元素中具有该类，则删除  
 * - 如果元素中没有该类，则添加  
 */  
function toggleClass(obj , cn){	  
	// 判断 obj 中是否含有 cn  
	if(hasClass(obj , cn)){  
		// 有，则删除  
		removeClass(obj , cn);  
	}else{  
		// 没有，则添加  
		addClass(obj , cn);  
	}  
}  
```

## 4.JSON基础

JavaScript Object Notation - JS 对象表示法，JSON 就是一个特殊格式的字符串，这个字符串可以被任意的语言所识别，并且可以转换为任意语言中的对象，JSON 在开发中主要用来数据的交互。

JSON 和 JS 对象的格式一样，只不过 JSON 字符串中的属性名必须加双引号，其他的和 JS 语法一致。

**JSON 分类**

1. 对象 {}
2. 数组 []

**JSON 格式**

1. 复合类型的值只能是数组或对象，不能是函数、正则表达式对象、日期对象
2. 原始类型的值只有四种：字符串、数值（必须以十进制表示）、布尔值和`null`（不能使用`NaN`、`Infinity`、`-Infinity`和`undefined`）
3. 字符串**必须使用双引号表示**，不能使用单引号
4. 对象的键名必须放在双引号里面
5. 数组或对象最后一个成员的后面，不能加逗号

**允许的值**

字符串、数值、布尔值、null、对象、数组

例：

```js
var arr = '[1,2,3,"hello",true]';  
			  
var obj2 = '{"arr":[1,2,3]}';  
  
var arr2 ='[{"name":"孙悟空","age":18,"gender":"男"},{"name":"孙悟空","age":18,"gender":"男"}]';  
```

**JSON 工具类**

JSON > JS 对象：`JSON.parse(json)`

```js
var o = JSON.parse(json);
var o2 = JSON.parse(arr);
```

JS 对象 > JSON：`JSON.stringify(object)`

```js
var obj3 = {name:"猪八戒", age:28, gender:"男"};
var str = JSON.stringify(obj3);
```

JSON 这个对象在 IE7 及以下的浏览器中不支持，所以在这些浏览器中调用时会报错。

还有一个函数`eval()`，这个函数可以用来执行一段字符串形式的 JS 代码，并将执行结果返回，可以用来转换 JSON 字符串。如果使用`eval()`执行的字符串中含有`{}`，它会将`{}`当成是代码块，如果不希望将其当成代码块解析，则需要在字符串前后各加一个`()`。

但是在开发中尽量不要使用，首先它的执行性能比较差，然后它还具有安全隐患。

```js
var str = '{"name":"孙悟空","age":18,"gender":"男"}';  
var obj = eval("(" + str + ")");
```
