---
title: javascript对象详解：__proto__和prototype的区别和联系
date: 2017-06-06 16:37:57
tags: javascript 对象 原型 前端
---

# 前言

本篇文章用来记录下最近研究对象的一些心得，做一个记录与总结，以加深自己的印象，同时，希望也能给正在学习中的你一点启发。本文适合有一定JavaScript基础的童鞋阅读。

# 引言

在JavaScript中，万物皆对象。咱们写一个JavaScript对象，大多数时候是用构造函数创建一个对象或者用对象字面量创建一个对象。比如：

```javascript
//通过构造函数来创建对象
function Person() {
    //...
}

var person1 = new Person();

//通过对象字面量创建对象
var person2 = {
    name: 'jessica',
    age: 27,
    job: 'teacher'
}
```

当然还有其他方式创建对象，这里就不列举出来了。那么问题来了，通过不同的方式创建的对象有什么区别呢？

我们知道，每个JS对象一定对应一个原型对象，并从原型对象继承属性和方法。那么对象是怎么和这个原型对象对应的呢？带着问题慢慢看下面的内容吧~

# `__proto__`和`prototype`概念区分

其实说`__proto__`并不准确，确切的说是对象的`[[prototype]]`属性，只不过在主流的浏览器中，都用`__proto__`来代表`[[prototype]]`属性，因为`[[prototype]]`只是一个标准，而针对这个标准，不同的浏览器有不同的实现方式。在ES5中用`Object.getPrototypeOf`函数获得一个对象的`[[prototype]]`。ES6中，使用`Object.setPrototypeOf`可以直接修改一个对象的`[[prototype]]`。为了方便，我下面的文章用`__proto__`来代表对象的`[[prototype]]`。

而`prototype`属性是只有函数才特有的属性，当你创建一个函数时，`js`会自动为这个函数加上`prototype`属性，值是一个空对象。所以，函数在`js`中是非常特殊的，是所谓的`一等公民`。
那么`__proto__`和`prototype`是怎么联系起来的呢？让我们来看下下面的代码：

```javascript
function Person(name, age) {
    this.name = name;
    this.age = age;
}

var person1 = new Person('jessica', 27);
```

## 当我们`new Person()`的时候到底发生了什么？

`new`一个构造函数，相当于实例化一个对象，这期间其实进行了这三个步骤：

1. 创建对象，设为o，即： `var o = {}`;
2. 上文提到了，每个对象都有`__proto__`属性，该属性指向一个对象，这里，将`o`对象的`__Proto__`指向构造函数`Person`的原型对象（`Person.prototype`）;
3. 将`o`作为`this`去调用构造函数`Person`，从而设置`o`的属性和方法并初始化。

当这3步完成，这个`o`对象就与构造函数`Person`再无联系，这个时候即使构造函数`Person`再加任何成员，都不再影响已经实例化的`o`对象了。
此时，`o`对象具有了`name`和`age`属性，同时具有了构造函数`Person`的原型对象的所有成员，当然，此时该原型对象是没有成员的。

现在大家都明白了吧，简单的总结下就是：

**js在创建对象的时候，都有一个叫做`__proto__`的内置属性，用于指向创建它的函数对象的原型对象`prototype`**

那么一个对象的`__proto__`属性究竟怎么决定呢？答案显而易见了：是由构造该对象的方法决定的。

# 创建对象的不同方法解析

下面讲解三种常见的创建对象方法。

## 对象字面量

比如：
```javascript
var Person = {
    name: 'jessica',
    age: 27
}
```

这种形式就是对象字面量，通过对象字面量构造出的对象，其`__proto__`指向`Object.prototype`。

所以，其实`Object`是一个函数也不难理解了。**Object、Function都是是js自带的函数对象。**

可以跑下面的代码看看：

```javascript
console.log(typeof Object); 
console.log(typeof Function); 
```

## 构造函数

就如我前面讲的,形如：
```javascript
function Person(){}
var person1 = new Person();
```
这种形式创建对象的方式就是通过构造函数创建对象，这里的构造函数是`Person`函数。上面也讲过了，通过构造函数创建的对象，其`__proto`指向的是构造函数的`prototype`属性指向的对象。

## Object.create

```javascript
var person1 = {
    name: 'jessica',
    age: 27
}
var person2 = Object.create(person1);
```

这种情况下，`person2`的`__proto__`指向`person1`。在没有`Object.create`函数的时候，人们大多是这样做的：

```javascript
Object.create = function(p) {
    function f(){};
    f.prototype = p;
    return new f();
}
```

一看大家就会明白了。

## 总结

其实仔细思考下上面提到的三种创建对象的方法，追究其本质，不难发现，最根本的还是利用构造函数再通过`new`来创建对象。所谓的对象字面量也只不过是语法糖而已，本质上是`var o = new Object(); o.xx = xx;o.yy=yy;`。 所以，函数真不愧是js中的**一等公民**呀~

# 原型链

既然已经提到了原型，就不得不提一下原型链了，毕竟这是实现继承最关键所在，也是js对象精妙所在。

还记得上文提到的一个总结吗？不记得？没关系，我贴出来让大家温故而知新，哈哈~

**js在创建对象的时候，都有一个叫做`__proto__`的内置属性，用于指向创建它的函数对象的原型对象`prototype`**

而原型链的基本思想就是利用原型让一个引用类型继承另一个引用类型的属性和方法。

让我们再简单回顾下构造函数、原型和实例的关系：

每个构造函数都有一个原型对象，原型对象包含一个指向构造函数的指针（`constructor`），而实例则包含一个指向原型对象的内部指针（`__proto__`）。

我们拿一个例子来讲解：

```javascript
function Person(name, age) {
    this.name = name;
    this.age = age;
}

var person1 = new Person('jessica', 27);
```
一图胜前言，我们用画图的形式来讲解下上面的例子：

![alt 原型链示例1](/images/object_proto.png)

从上图可以看到，其实原型链的顶端是`Object.prototype.__proto__`,也即为`null`。


# 总结

函数是`js`中的一等公民，`js`在创建对象的时候，都有一个叫做`__proto__`的内置属性，用于指向创建它的函数对象的原型对象`prototype`。只有函数有`prototype`, 当你创建一个函数时，`js`会自动为这个函数加上`prototype`属性，值是一个空对象。

# 参考文献

[js 对象、原型、继承详解](http://007sair.github.io/javascript/2015/07/22/js-prototype.html)
[js中__proto__和prototype的区别和关系？](https://www.zhihu.com/question/34183746)
[理解JavaScript原型](http://blog.jobbole.com/9648/)