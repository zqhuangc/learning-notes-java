## 介绍

jQuery 对象就是通过jQuery包装DOM对象后产生的对象。
jQuery 对象是 jQuery 独有的. 如果一个对象是 jQuery 对象, 那么它就可以使用 jQuery 里的方法: $(“#test”).html();
​    比如： 
​     $("#test").html()   意思是指：获取ID为test的元素内的html代码。其中html()是jQuery里的方法 
​    这段代码等同于用DOM实现代码： 
​    document.getElementById(" test ").innerHTML; 
虽然jQuery对象是包装DOM对象后产生的，但是jQuery对象无法使用DOM对象的任何方法，同理DOM对象也不能使用jQuery里的方法.乱使用会报错
约定：如果获取的是 jQuery 对象, 那么要在变量前面加上 $. 	
var $div                = jQuery 对象
var divElement    = DOM 对象
var divText           = “文本值”，即是一个字符串

对于已经是一个DOM对象，只需要用$()把DOM对象包装起来，就可以获得一个jQuery对象了。$(DOM对象) 

(1) jQuery对象是一个数组对象，可以通过[index]的方法，来得到相应的DOM对象
(2) jQuery本身提供，通过.get(index)方法，得到相应的DOM对象
## jQuery选择器

选择器是 jQuery 的根基, 在 jQuery 中, 对事件处理, 遍历 DOM 和 Ajax 操作都依赖于选择器

$(“#id”)   等价于    document.getElementById("id")
$(“tagName”)     等价于   document.getElementsByTagName("tagName")
$(“.class”)

${}

### 基本

* ${"#id"} 

根据给定的ID匹配一个元素。使用任何的元字符作为名称的文本部分， 它必须被两个反斜杠转义：\\。用于搜索的，通过元素的 id 属性中给定的值

* ${"element"}

根据给定的元素标签名匹配所有元素，一个用于搜索的元素。指向 DOM 节点的标签名。

* ${".class"}

根据给定的css类名匹配元素。一个用以搜索的类。一个元素可以有多个类，只要有一个符合就能被匹配到。

* $("*")

匹配所有元素

多用于结合上下文来搜索。

* ${"selector1,selector2,selectorN"}

将每一个选择器匹配到的元素合并后一起返回。

你可以指定任意多个选择器，并将匹配到的元素合并到一个结果内。

### 层级

> ancestor descendant
>
> 在给定的祖先元素下匹配所有的后代元素
>
> parent > child
>
> 在给定的父元素下匹配所有的子元素
>
> prev + next
>
> 匹配所有紧接在 prev 元素后的 next 元素(同层级及以下)
>
> prev ~ siblings
>
> 匹配 prev 元素之后的所有 siblings 元素

### 基本筛选器

${"element:xxx"}

> :first  获取第一个元素
>
> :not(selector) 去除所有与给定选择器匹配的元素
>
> :eq(index) 匹配一个给定索引值的元素
>
> :even
>
> :odd

#### 内容

:contains(text)

:empty

:has(selector)

:parent

#### 可见性

:hidden

:visible

### 属性

${"element[attribute]"}

>  [attribute] 匹配包含给定属性的元素。
>
>  [attribute=value] 匹配给定的属性是某个特定值的元素
>
> [attribute!=value] 匹配给定的属性不是某个特定值的元素
>
> [attribute^=value] 匹配给定的属性是以某些值开始的元素
>
>  [attribute$=value]  匹配给定的属性是以某些值结尾的元素
>
> [attribute*=value] 匹配给定的属性是以包含某些值的元素
>
> [attrSel1]\[attrSel2][attrSelN]  复合属性选择器，需要同时满足多个条件时使用。



### 子元素

> :first-child 匹配所给选择器( :之前的选择器)的第一个子元素
>
> 类似的 :first 匹配第一个元素，但是:first-child选择器可以匹配多个：即为每个父级元素匹配第一个子元素。这相当于:nth-child(1)
>
>

### 表单

> ${":input"} 匹配所有 input, textarea, select 和 button 元素
>
>  :text  匹配所有的单行文本框
>
> ...

### 表单对象属性



### 混淆选择器

## jQuery 中的 Ajax操作

JQuery 对 Ajax 操作进行了封装, 在 jQuery 中最常用的方法是 load()，$.ajax()，$.get() 和 $.post()

load()方法是 jQuery 中最为简单和常用的 Ajax 方法, 能载入远程的 HTML 代码并插入到 DOM 中. 它的结构是:   load(url[, data][,callback])
传递方式: load() 方法的传递参数根据参数 data 来自动自定. 如果没有参数传递, 采用 GET 方式传递, 否则采用 POST 方式
对于必须在加载完才能继续的操作, load() 方法提供了回调函数, 该函数有三个参数: 代表请求返回内容的 data; 代表请求状态的 textStatus 对象和 XMLHttpRequest 对象


$.get() 方法使用 GET 方式来进行异步请求. 它的结构是: $.get(url[, data][, callback]);
$.get() 方法的回调函数只有两个参数: data 代表返回的内容, 可以是 XML 文档, JSON 字符串, HTML 片段等; textstatus 代表请求状态, 其值可能为: succuss, error, notmodify, timeout 4 种.

序列化元素
jQuery 为准备 “发送到服务器的 key/value 数据” 提供了一个简化的方法: serialize(). 该方法作用于一个 jQuery 对象, 能将 DOM 元素内容序列化为字符串, 用于 Ajax 请求.
var xhr=$.get("base01.jsp",{username:"aa",psw:“111"});
var xhr=$.get("base01.jsp",$("#form1").serialize());
使用 serialize() 方法可以自动完成对参数的 url 编码
因为该方法作用于 jQuery 对象, 所以不光只要表单能使用, 其它选择器选取的元素也能使用它. 


## 其他
内部插入节点（父子关系）
append(content) :向每个匹配的元素的内部的结尾处追加内容 
prepend(content):向每个匹配的元素的内部的开始处插入内容
外部插入节点（兄弟关系）
after(content) ：在每个匹配的元素之后插入内容，例如A.after(B)，即B在后 
before(content)：在每个匹配的元素之前插入内容 ，例如A.before(B)，即B在前 

查找节点及读取或设置属性操作
查找节点: 
查找属性节点: 通过 jQuery 九类选择器完成.
查找属性节点: 查找到所需要的元素之后, 可以调用 jQuery 对象的 attr() 方法来获取它的各种属性值

创建元素/文本/属性节点
创建节点: 使用 jQuery 的函数 $(): $(“html”); 会根据传入的 html 标记字符串创建一个 DOM 对象, 并把这个 DOM 对象包装成一个 jQuery 对象返回.
注意: 
动态创建的新元素节点不会被自动添加到文档中, 而是需要使用其他方法将其插入到文档中; 
当创建单个元素时, 需注意闭合标签格式. 例如创建一个<p>元素, 可以使用 $(“<p/>”) 或 $(“<p></p>”), 但不能使用 $(“<p>”) 或 $(“</P>”)
创建文本节点就是在创建元素节点时直接把文本内容写出来; 创建属性节点也是在创建元素节点时一起创建

删除所有/指定ID的元素节点
remove(): 从 DOM 中删除所有匹配的元素, 当某个节点用 remove() 方法删除后, 该节点所包含的所有后代节点将被同时删除. 这个方法的返回值是一个指向已被删除的节点的引用.

复制节点
clone(): 克隆匹配的 DOM 元素, 返回值为克隆后的副本. 但此时复制的新节点不具有任何行为。
clone(true): 复制元素的同时也复制元素中的行为。 

替换节点
replaceWith(): 将所有匹配的元素都替换为指定的 HTML元素
注意: 若在替换之前, 已经在元素上绑定了事件, 替换后原先绑定的事件会与原先的元素一起消失

属性操作
attr(): 获取属性和设置属性
当为该方法传递一个参数时, 即为某元素的获取指定属性
当为该方法传递两个参数时, 即为某元素设置指定属性的值
jQuery 中有很多方法都是一个函数实现获取和设置. 如: attr(), html(), val(),  css() 等.
removeAttr(): 删除指定元素的指定属性

样式操作
获取 class 和设置 class : class 是元素的一个属性, 所以获取 class 和设置 class 都可以使用 attr() 方法来完成.
追加样式: addClass() 
移除样式: removeClass() --- 从匹配的元素中删除全部或指定的 class
切换样式: toggleClass()  --- 控制样式上的重复切换.如果类名存在则删除它, 如果类名不存在则添加它.
判断是否含有某个样式: hasClass() --- 判断元素中是否含有某个 class, 如果有, 则返回 true; 否则返回 false

常用的遍历节点方法
取得匹配元素的所有子元素组成的集合: children(). 该方法只考虑子元素而不考虑任何后代元素.
取得匹配元素后面紧邻的同辈元素的集合:next(); 
取得匹配元素前面紧邻的同辈元素的集合:prev()
取得匹配元素前后所有的同辈元素的集合: siblings()

jQuery 中的事件 --  加载 DOM 
在页面加载完毕后, 浏览器会通过 JavaScript 为 DOM 元素添加事件. 在常规的 JavaScript 代码中, 通常使用 window.onload 方法, 在 jQuery 中使用$(document).ready() 方法.

