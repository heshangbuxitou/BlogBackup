---
title: 详解JavaScript中的this
date: 2017-10-19 00:12:23
tags:
- JavaScript
categories: 
- 前端
---
>JavaScript中的this是一直是比较抽象的概念，以前在学习JS的对this指代的对象一知半解，后面看到一篇关于this的文章《[详解this](http://www.cnblogs.com/Wayou/p/all-this.html)》，这对于我对this的理解帮助很大，很感激这篇文章，现在把它分享出来，希望可以有更多的人能够看到这篇好文章，可以灵活的使用this对象。

# 详解this

习惯了高级语言的你或许觉得JavaScript中的this跟Java这些面向对象语言相似，保存了实体属性的一些值。其实不然。将它视作幻影魔神比较恰当，手提一个装满未知符文的灵龛。

## 全局this

浏览器宿主的全局环境中，this指的是window对象。

```javascript
<script type="text/javascript">
    console.log(this === window); //true
</script>

```

浏览器中在全局环境下，使用var声明变量其实就是赋值给this或window。

```javascript
<script type="text/javascript">
    var foo = "bar";
    console.log(this.foo); //logs "bar"
    console.log(window.foo); //logs "bar"
</script>

```

任何情况下，创建变量时没有使用var或者let(ECMAScript 6)，也是在操作全局this。

```javascript
<script type="text/javascript">
    foo = "bar";

    function testThis() {
      foo = "foo";
    }

    console.log(this.foo); //logs "bar"
    testThis();
    console.log(this.foo); //logs "foo"
</script>

```

Node命令行（REPL）中，this是全局命名空间。可以通过global来访问。

```javascript
> this
{ ArrayBuffer: [Function: ArrayBuffer],
  Int8Array: { [Function: Int8Array] BYTES_PER_ELEMENT: 1 },
  Uint8Array: { [Function: Uint8Array] BYTES_PER_ELEMENT: 1 },
  ...
> global === this
true

```

在Node环境里执行的JS脚本中，this其实是个空对象，有别于global。

```javascript
console.log(this);
console.log(this === global);

###
$ node test.js
{}
false

```

但在命令行里进行求值却会赋值到this身上。

```javascript
> var foo = "bar";
> this.foo
bar
> global.foo
bar

```

在Node里执行的脚本中，创建变量时没带var或let关键字，会赋值给全局的global但不是this（译注：上面已经提到this和global不是同一个对象，所以这里就不奇怪了）。

```javascript
foo = "bar";
console.log(this.foo);
console.log(global.foo);

###
$ node test.js
undefined
bar

```

但在Node命令行里，就会赋值给两者了。

**简单来说，Node脚本中global和this是区别对待的，而Node命令行中，两者可等效为同一对象。**

## 函数或方法里的this

除了DOM的事件回调或者提供了执行上下文（后面会提到）的情况，函数正常被调用（不带new）时，里面的this指向的是全局作用域。

```javascript
<script type="text/javascript">
    foo = "bar";

    function testThis() {
      this.foo = "foo";
    }

    console.log(this.foo); //logs "bar"
    testThis();
    console.log(this.foo); //logs "foo"
</script>

```

测试结果：

```javascript
$ node test.js
bar
foo

```

还有个例外，就是使用了"use strict";。此时this是undefined。

```javascript
<script type="text/javascript">
    foo = "bar";

    function testThis() {
      "use strict";
      this.foo = "foo";
    }

    console.log(this.foo); //logs "bar"
    testThis();  //Uncaught TypeError: Cannot set property 'foo' of undefined 
</script>

```

当用调用函数时使用了new关键字，此刻this指代一个新的上下文，不再指向全局this。

```javascript
<script type="text/javascript">
    foo = "bar";

    function testThis() {
      this.foo = "foo";
    }

    console.log(this.foo); //logs "bar"
    new testThis();
    console.log(this.foo); //logs "bar"

    console.log(new testThis().foo); //logs "foo"
</script>

```

通常我将这个新的上下文称作实例。

## 原型中的this

函数创建后其实以一个函数对象的形式存在着。既然是对象，则自动获得了一个叫做prototype的属性，可以自由地对这个属性进行赋值。当配合new关键字来调用一个函数创建实例后，此刻便能直接访问到原型身上的值。

```javascript
function Thing() {
    console.log(this.foo);
}

Thing.prototype.foo = "bar";

var thing = new Thing(); //logs "bar"
console.log(thing.foo);  //logs "bar"

```

当通过new的方式创建了多个实例后，他们会共用一个原型。比如，每个实例的this.foo都返回相同的值，直到this.foo被重写。

```javascript
function Thing() {
}
Thing.prototype.foo = "bar";
Thing.prototype.logFoo = function () {
    console.log(this.foo);
}
Thing.prototype.setFoo = function (newFoo) {
    this.foo = newFoo;
}

var thing1 = new Thing();
var thing2 = new Thing();

thing1.logFoo(); //logs "bar"
thing2.logFoo(); //logs "bar"

thing1.setFoo("foo");
thing1.logFoo(); //logs "foo";
thing2.logFoo(); //logs "bar";

thing2.foo = "foobar";
thing1.logFoo(); //logs "foo";
thing2.logFoo(); //logs "foobar";

```

在实例中，this是个特殊的对象，而this自身其实只是个关键字。你可以把this想象成在实例中获取原型值的一种途径，同时对this赋值又会覆盖原型上的值。完全可以将新增的值从原型中删除从而将原型还原为初始状态。

```javascript
function Thing() {
}
Thing.prototype.foo = "bar";
Thing.prototype.logFoo = function () {
    console.log(this.foo);
}
Thing.prototype.setFoo = function (newFoo) {
    this.foo = newFoo;
}
Thing.prototype.deleteFoo = function () {
    delete this.foo;
}

var thing = new Thing();
thing.setFoo("foo");
thing.logFoo(); //logs "foo";
thing.deleteFoo();
thing.logFoo(); //logs "bar";
thing.foo = "foobar";
thing.logFoo(); //logs "foobar";
delete thing.foo;
thing.logFoo(); //logs "bar";

```

或者不通过实例，直接操作函数的原型。

```javascript
function Thing() {
}
Thing.prototype.foo = "bar";
Thing.prototype.logFoo = function () {
    console.log(this.foo, Thing.prototype.foo);
}

var thing = new Thing();
thing.foo = "foo";
thing.logFoo(); //logs "foo bar";

```

同一函数创建的所有实例均共享一个原型。如果你给原型赋值了一个数组，那么所有实例都能获取到这个数组。除非你在某个实例中对其进行了重写，实事上是进行了覆盖。

```javascript
function Thing() {
}
Thing.prototype.things = [];

var thing1 = new Thing();
var thing2 = new Thing();
thing1.things.push("foo");
console.log(thing2.things); //logs ["foo"]

```

通常上面的做法是不正确的（译注：改变thing1的同时也影响了thing2）。如果你想每个实例互不影响，那么请在函数里创建这些值，而不是在原型上。

```javascript
function Thing() {
    this.things = [];
}

var thing1 = new Thing();
var thing2 = new Thing();
thing1.things.push("foo");
console.log(thing1.things); //logs ["foo"]
console.log(thing2.things); //logs []

```

多个函数可以形成原型链，这样this便会在原型链上逐步往上找直到找到你想引用的值。

```javascript
function Thing1() {
}
Thing1.prototype.foo = "bar";

function Thing2() {
}
Thing2.prototype = new Thing1();

var thing = new Thing2();
console.log(thing.foo); //logs "bar"

```

很多人便是利用这个特性在JS中模拟经典的对象继承。**注意原型链底层函数中对this的操作会覆盖上层的值。**

```javascript
function Thing1() {
}
Thing1.prototype.foo = "bar";

function Thing2() {
    this.foo = "foo";
}
Thing2.prototype = new Thing1();

function Thing3() {
}
Thing3.prototype = new Thing2();

var thing = new Thing3();
console.log(thing.foo); //logs "foo"

```

我习惯将赋值到原型上的函数称作方法。上面某些地方便使用了方法这样的字眼，比如logFoo方法。这些方法中的this同样具有在原型链上查找引用的魔力。通常将最初用来创建实例的函数称作构造函数。

原型链方法中的this是从实例中的this开始住上查找整个原型链的。也就是说，如果原型链中某个地方直接对this进行赋值覆盖了某个变量，那么我们拿到 的是覆盖后的值。

```javascript
function Thing1() {
}
Thing1.prototype.foo = "bar";
Thing1.prototype.logFoo = function () {
    console.log(this.foo);
}

function Thing2() {
    this.foo = "foo";
}
Thing2.prototype = new Thing1();

var thing = new Thing2();
thing.logFoo(); //logs "foo";

```

在JavaScript中，函数可以嵌套函数，也就是你可以在函数里面继续定义函数。但内层函数是通过闭包获取外层函数里定义的变量值的，而不是直接继承this。

```javascript
function Thing() {
}
Thing.prototype.foo = "bar";
Thing.prototype.logFoo = function () {
    var info = "attempting to log this.foo:";
    function doIt() {
        console.log(info, this.foo);
    }
    doIt();
}

var thing = new Thing();
thing.logFoo();  //logs "attempting to log this.foo: undefined"

```

上面示例中，doIt 函数中的this指代是全局作用域或者是undefined如果使用了"use strict";声明的话。对于很多新手来说，理解这点是非常头疼的。

还有更奇葩的。把实例的方法作为参数传递时，实例是不会跟着过去的。也就是说，此时方法中的this在调用时指向的是全局this或者是undefined在声明了"use strict"时。

```javascript
function Thing() {
}
Thing.prototype.foo = "bar";
Thing.prototype.logFoo = function () {  
    console.log(this.foo);   
}

function doIt(method) {
    method();
}

var thing = new Thing();
thing.logFoo(); //logs "bar"
doIt(thing.logFoo); //logs undefined

```

所以很多人习惯将this缓存起来，用个叫self或者其他什么的变量来保存，以将外层与内层的this区分开来。

```javascript
function Thing() {
}
Thing.prototype.foo = "bar";
Thing.prototype.logFoo = function () {
    var self = this;
    var info = "attempting to log this.foo:";
    function doIt() {
        console.log(info, self.foo);
    }
    doIt();
}

var thing = new Thing();
thing.logFoo();  //logs "attempting to log this.foo: bar"

```

但上面的方式不是万能的，在将方法做为参数传递时，就不起作用了。

```javascript
function Thing() {
}
Thing.prototype.foo = "bar";
Thing.prototype.logFoo = function () { 
    var self = this;
    function doIt() {
        console.log(self.foo);
    }
    doIt();
}

function doItIndirectly(method) {
    method();
}

var thing = new Thing();
thing.logFoo(); //logs "bar"
doItIndirectly(thing.logFoo); //logs undefined

```

解决方法就是传递的时候使用bind方法显示指明上下文，bind方法是所有函数或方法都具有的。

```javascript
function Thing() {
}
Thing.prototype.foo = "bar";
Thing.prototype.logFoo = function () { 
    console.log(this.foo);
}

function doIt(method) {
    method();
}

var thing = new Thing();
doIt(thing.logFoo.bind(thing)); //logs bar

```

同时也可以使用apply或call 来调用该方法或函数，让它在一个新的上下文中执行。

```javascript
function Thing() {
}
Thing.prototype.foo = "bar";
Thing.prototype.logFoo = function () { 
    function doIt() {
        console.log(this.foo);
    }
    doIt.apply(this);
}

function doItIndirectly(method) {
    method();
}

var thing = new Thing();
doItIndirectly(thing.logFoo.bind(thing)); //logs bar

```

使用bind可以任意改变函数或方法的执行上下文，即使它没有被绑定到一个实例的原型上。

```javascript
function Thing() {
}
Thing.prototype.foo = "bar";

function logFoo(aStr) {
    console.log(aStr, this.foo);
}

var thing = new Thing();
logFoo.bind(thing)("using bind"); //logs "using bind bar"
logFoo.apply(thing, ["using apply"]); //logs "using apply bar"
logFoo.call(thing, "using call"); //logs "using call bar"
logFoo("using nothing"); //logs "using nothing undefined"

```

避免在构造函数中返回作何东西，因为返回的东西可能覆盖本来该返回的实例。

```javascript
function Thing() {
    return {};
}
Thing.prototype.foo = "bar";

Thing.prototype.logFoo = function () {
    console.log(this.foo);
}

var thing = new Thing();
thing.logFoo(); //Uncaught TypeError: undefined is not a function

```

但，如果你在构造函数里返回的是个原始值比如字符串或者数字什么的，上面的错误就不会发生了，返回语句将被忽略。所以最好别在一个将要通过new来调用的构造函数中返回作何东西，即使你是清醒的。如果你想实现工厂模式，那么请用一个函数来创建实例，并且不通过new来调用。当然这只是个人建议。

诚然，你也可以使用Object.create从而避免使用new。这样也能创建一个实例。

```javascript
function Thing() {
}
Thing.prototype.foo = "bar";

Thing.prototype.logFoo = function () {
    console.log(this.foo);
}

var thing =  Object.create(Thing.prototype);
thing.logFoo(); //logs "bar"

```

这种方式不会调用该构造函数。

```javascript
function Thing() {
    this.foo = "foo";
}
Thing.prototype.foo = "bar";

Thing.prototype.logFoo = function () {
    console.log(this.foo);
}

var thing =  Object.create(Thing.prototype);
thing.logFoo(); //logs "bar"

```

正因为Object.create没有调用构造函数，这在当你想实现一个继承时是非常有用的，随后你可能想要重写构造函数。

```javascript
function Thing1() {
    this.foo = "foo";
}
Thing1.prototype.foo = "bar";

function Thing2() {
    this.logFoo(); //logs "bar"
    Thing1.apply(this);
    this.logFoo(); //logs "foo"
}
Thing2.prototype = Object.create(Thing1.prototype);
Thing2.prototype.logFoo = function () {
    console.log(this.foo);
}

var thing = new Thing2();

```

## 对象中的this

可以在对象的任何方法中使用this来访问该对象的属性。这与用new得到的实例是不一样的。

```javascript
var obj = {
    foo: "bar",
    logFoo: function () {
        console.log(this.foo);
    }
};

obj.logFoo(); //logs "bar"

```

注意这里并没有使用new，也没有用Object.create，更没有函数的调用来创建对象。也可以将函数绑定到对象，就好像这个对象是一个实例一样。

```javascript
var obj = {
    foo: "bar"
};

function logFoo() {
    console.log(this.foo);
}

logFoo.apply(obj); //logs "bar"

```

此时使用this没有向上查找原型链的复杂工序。通过this所拿到的只是该对象身上的属性而以。

```javascript
var obj = {
    foo: "bar",
    deeper: {
        logFoo: function () {
            console.log(this.foo);
        }
    }
};

obj.deeper.logFoo(); //logs undefined

```

也可以不通过this，直接访问对象的属性。

```javascript
var obj = {
    foo: "bar",
    deeper: {
        logFoo: function () {
            console.log(obj.foo);
        }
    }
};

obj.deeper.logFoo(); //logs "bar"

```

## DOM 事件回调中的this

在DOM事件的处理函数中，this指代的是被绑定该事件的DOM元素。

```javascript
function Listener() {
    document.getElementById("foo").addEventListener("click",
       this.handleClick);
}
Listener.prototype.handleClick = function (event) {
    console.log(this); //logs "<div id="foo"></div>"
}

var listener = new Listener();
document.getElementById("foo").click();

```

除非你通过bind人为改变了事件处理器的执行上下文。

```javascript
function Listener() {
    document.getElementById("foo").addEventListener("click", 
        this.handleClick.bind(this));
}
Listener.prototype.handleClick = function (event) {
    console.log(this); //logs Listener {handleClick: function}
}

var listener = new Listener();
document.getElementById("foo").click();

```

## HTML中的this

HTML标签的属性中是可能写JS的，这种情况下this指代该HTML元素。

```javascript
<div id="foo" onclick="console.log(this);"></div>
<script type="text/javascript">
document.getElementById("foo").click(); //logs <div id="foo"...
</script>

```

## 重写this

无法重写this，因为它是一个关键字。

```javascript
function test () {
    var this = {};  // Uncaught SyntaxError: Unexpected token this 
}

```

## eval中的this

eval 中也可以正确获取当前的 this。

```javascript
function Thing () {
}
Thing.prototype.foo = "bar";
Thing.prototype.logFoo = function () {
    eval("console.log(this.foo)"); //logs "bar"
}

var thing = new Thing();
thing.logFoo();

```

这里存在安全隐患。最好的办法就是避免使用eval。

使用Function关键字创建的函数也可以获取this：

```javascript
function Thing () {
}
Thing.prototype.foo = "bar";
Thing.prototype.logFoo = new Function("console.log(this.foo);");

var thing = new Thing();
thing.logFoo(); //logs "bar"

```

## 使用with时的this

使用with可以将this人为添加到当前执行环境中而不需要显示地引用this。

```javascript
function Thing () {
}
Thing.prototype.foo = "bar";
Thing.prototype.logFoo = function () {
    with (this) {
        console.log(foo);
        foo = "foo";
    }
}

var thing = new Thing();
thing.logFoo(); // logs "bar"
console.log(thing.foo); // logs "foo"

```

正如很多人认为的那样，使用with是不好的，因为会产生歧义。

## jQuery中的this

一如HTML DOM元素的事件回调，jQuery库中大多地方的this也是指代的DOM元素。页面上的事件回调和一些便利的静态方法比如$.each 都是这样的。

```javascript
<div class="foo bar1"></div>
<div class="foo bar2"></div>
<script type="text/javascript">
$(".foo").each(function () {
    console.log(this); //logs <div class="foo...
});
$(".foo").on("click", function () {
    console.log(this); //logs <div class="foo...
});
$(".foo").each(function () {
    this.click();
});
</script>

```

## 传递 this

如果你用过underscore.js或者lo-dash你便知道，这两个库中很多方法你可以传递一个参数来显示指定执行的上下文。比如_.each。自ECMAScript 5 标准后，一些原生的JS方法也允许传递上下文，比如forEach。事实上，上文提到的bind，apply还有call 已经给我们手动指定函数执行上下文的能力了。

```javascript
function Thing(type) {
    this.type = type;
}
Thing.prototype.log = function (thing) {
    console.log(this.type, thing);
}
Thing.prototype.logThings = function (arr) {
   arr.forEach(this.log, this); // logs "fruit apples..."
   _.each(arr, this.log, this); //logs "fruit apples..."
}

var thing = new Thing("fruit");
thing.logThings(["apples", "oranges", "strawberries", "bananas"]);

```

这样可以使得代码简洁些，不用层层嵌套bind，也不用不断地缓存this。

一些编程语言上手很简单，比如Go语言手册可以被快速读完。然后你差不多就掌握这门语言了，只是在实战时会有些小的问题或陷阱在等着你。

而JavaScript不是这样的。手册难读。非常多缺陷在里面，以至于人们抽离出了它好的部分（The Good Parts）。最好的文档可能是MDN上的了。所以我建议你看看他上面关于this的介绍，并且始终在搜索JS相关问题时加上"mdn" 来获得最好的文档资料。静态代码检查也是个不错的工具，比如jshint。

## 参考资料

*   [all this](http://bjorn.tipling.com/all-this)
*   [详解this](http://www.cnblogs.com/Wayou/p/all-this.html)