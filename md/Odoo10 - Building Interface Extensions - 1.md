---
title: Odoo10 - Building Interface Extensions - 1
date: 2017-08-04 15:08:29
tags:
---

这一篇教程主要是为了创建 web client 的 module。
## A Simple Module
让我们从一个基本的包含web组件配置的odoo module开始，并测试web framework。
例子可以从下面的命令下载
```sh
$ git clone https://github.com/odoo/petstore
```
执行上面的命令后，将在当前文件目录下创建一个`petstore`文件夹，将这个目录加入到 odoo 的 addons-path中，然后创建一个新的db,并且安装这个模块。
文件的目录如下
```sh
oepetstore/
├── images
│   ├── alligator.jpg
│   ├── ball.jpg
│   ├── crazy_circle.jpg
│   ├── fish.jpg
│   └── mice.jpg
├── __init__.py
├── oepetstore.message_of_the_day.csv
├── __openerp__.py
├── petstore_data.xml
├── petstore.py
├── petstore.xml
└── static
    └── src
        ├── css
        │   └── petstore.css
        ├── js
        │   └── petstore.js
        └── xml
            └── petstore.xml

6 directories, 14 files

```
此模块已经定义了各种的服务配置。我们稍后再细说，先将目光集中在与web相关的内容上，也就是`static`这个文件夹的内容。
所有web要用到的文件必须放在`static`文件夹下面，这样才能被浏览器等所访问到，此文件夹之外的文件则不能被浏览器直接访问。`src/css`, `src/js`, `src/xml`子文件夹不是必须的，只是通常的文件放置习惯。
`oepetstore/static/css/petstore.css`， 目前为空，用来存放pet store的css
`oepetstore/static/xml/petstore.xml`，通常为空，放置Qweb模板
`oepetstore/static/js/petstore.js`，最重要的就是这个文件了，包含了此模块的各种业务逻辑，通常应该长成这样：
```javascript
odoo.oepetstore = function(instance, local) {
    var _t = instance.web._t,
        _lt = instance.web._lt;
    var QWeb = instance.web.qweb;

    local.HomePage = instance.Widget.extend({
        start: function() {
            console.log("pet store home page loaded");
        },
    });

    instance.web.client_actions.add(
        'petstore.homepage', 'instance.oepetstore.HomePage');
}
```
仅仅用来在浏览器的console上，输出一小段message。
`static`目录下的所有文件都必须在模块文件里面定义好，这样才能被自动的正确的导入。`src/xml`下的所有东西，都在`__manifest__.py`文件中设置了导入，其他的css, js则是定义在`petstore.xml`中。

> 提示
> 所有的js文件都会被自动的压缩，最小化。这导致前端调试起来非常的困难，但是，你可以通过开启odoo的调试模式来避免这个

## Odoo JavaScript Module
JavaScript没有模块这一概念，导致在各个文件中的javascript代码可能会互相冲突。Odoo为了解决这一问题，提供了自己定义方式，用来避免各种冲突。
`oepetstore/static/js/petstore.js` 包含了一个module定义
```js
odoo.oepetstore = function(instance, local) {
    local.xxx = ...;
}
```
在 Odoo web中，module的是像定义function一样被绑在odoo变量上。function的name必须和addon的名字一样（也就是`oepetstore`），这样 web framework才能找到它，并且自动的实例化它。
当web client导入你的module时，它会自动调用此方法，并且传入两个参数。

 - 第一个是当前 web client 的实例，通过它可以调用到odoo定义好的各种东西（如：翻译，network service）等，还有其他module或者 core定义的各种objects.
 - 第二个是自己模块的命名空间，对其添加各种属性，使得这个module的一些东西可以被外部访问。

## Classes
如同 module一样，javascript没有像其他面向对象的语言一样提供内置的Class，虽然提供了一个粗糙的基于原型链的东西。
为了方便开发，Odoo提供了一个Class，实现基本的继承等。
创建一个新的Class，需要调用`odoo.web.Class()`的`extend()`方法：
```js
var MyClass = instance.web.Class.extend({
	say_hello: function(){
		console.log("hello");
	}
});
```
`extend()`方法接受一个字典，包含一些方法以及属性。在上面的例子中，只有一个不接受参数的`say_hello`方法。
通过`new`关键字实例化一个类
```js
var my_object = new MyClass();
my_object.say_hello();
// print "hello" in the console
```
通过 `this`来访问属性
```js

var MyClass = instance.web.Class.extend({
    say_hello: function() {
        console.log("hello", this.name);
    },
});

var my_object = new MyClass();
my_object.name = "Bob";
my_object.say_hello();
// print "hello Bob" in the console
```
类也提供了一个类似的构造函数，用来执行一些初始化。需要定义`init()`方法，参数由 `new` 的时候传入。
```js
var MyClass = instance.web.Class.extend({
    init: function(name) {
        this.name = name;
    },
    say_hello: function() {
        console.log("hello", this.name);
    },
});

var my_object = new MyClass("Bob");
my_object.say_hello();
// print "hello Bob" in the console
```
也可以从已经定义好的Class中再extend一个Class作为其子类
```js
var MySpanishClass = MyClass.extend({
    say_hello: function() {
        console.log("hola", this.name);
    },
});

var my_object = new MySpanishClass("Bob");
my_object.say_hello();
// print "hola Bob" in the console
```
当覆写父类的方法时，可以通过`this._super()`调用父类的方法。
```js
var MySpanishClass = MyClass.extend({
    say_hello: function() {
        this._super();
        console.log("translation in Spanish: hola", this.name);
    },
});

var my_object = new MySpanishClass("Bob");
my_object.say_hello();
// print "hello Bob \n translation in Spanish: hola Bob" in the console
```
> 警告
> `_super`并不是一个标准方法，它是调用当前继承连上的下一个元素的方法，并且只能是在同步的方法中调用。异步的方式是不能直接使用的（包括网络调用，或者setTimeout回调）。
```
// broken, will generate an error
say_hello: function () {
    setTimeout(function () {
        this._super();
    }.bind(this), 0);
}

// correct
say_hello: function () {
    // don't forget .bind()
    var _super = this._super.bind(this);
    setTimeout(function () {
        _super();
    }.bind(this), 0);
}
```
