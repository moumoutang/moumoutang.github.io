---
layout: post
title:  "关于JS继承 原型链 等等"
categories: js
---

学习JS的人都会被JS 各种prototype 或者 __proto__ 各种乱七八糟的概念弄的混乱，
对于继承啊什么的..也是迷糊..    

本文想来捋一捋，现在开始： 

####  __proto__    

设想我们有一个对象：    
 
``` javascript
// me是一个字面量对象
var me = {
  "name": "ctyuna"
}

me.__proto__ === Object.prototype //true

me instanceof Object  //true

me.toString() // "[object Object]"

```
虽然它是一个字面量对象,但内部的 __proto__还是指向Object，所以它可以拿到基本的Object的方法          
__proto__存在的意义其实就是 一个指向对象真实的原型的指针,它理论上应该是每个对象的私有属性，不论是任何类型的变量都有这个值,除了值为undefined的变量    

#### 那么prototype呢？   

遵循ECMAScript标准，someObject.[[Prototype]] 符号是用于指派 someObject 的原型。这个等同于 JavaScript 的 __proto__ 属性。从 ECMAScript 6 开始, [[Prototype]]        可以用Object.getPrototypeOf()和Object.setPrototypeOf()访问器来访问。           
prototype是对Function派生的对象来说的；js的设计初衷是想学习其他语言的构造器概念，每一个Function都相当于一个构造器，可以通过new来派生新的实体对象         

``` javascript
   function animal(name) { this.name = name}

    var test1 = new animal("hope")

    test1.__proto__ === animal.prototype

    animal.prototype.value2 = "hope2"

    test1.value2  //"hope2"

```


那么JS的继承跟原型链又有什么关系呢？    
我们平时所说的继承两个意思：                    
- 儿子继承父亲的各种方法和属性                       	
- 一旦父亲的一些属性变了，儿子若没有能覆盖它的新值，则也会跟着父亲变化；	
	                 	
那么原型链不就可以办到这些？只要我们显视的去指定原型是什么， 假设我们有一个基类：		          	

```
    function Base(){
      this.name  = 'Base'
    }

    Base.prototype.sayName = function(){
      console.log(this.name)
    }

    var  test1 = new Base()

    假设我们有一个儿子要继承Base:
    如果把原型链都指向一个地方 把constructor再执行一遍  
    这样是否就可以了？

    function Son(){
      Base.apply(this)
    }

    Son.prototype =  Base.prototype

    var test2 = new  Son() 

    test2.sayName() //'Base'

    看起来好像是可以，but：
    如果我们要定一个Son自己的sayName ：


    Son.prototype.sayName = function(){ 
      console.log("I am son")
    }

    却发现：

    test1.sayName()  // I am son

    对，因为子和父的原型链都指向了一个地方， 那么子改变了原型，则会影响到父亲
    ，这就不符合我们最初对于继承的设想

    那么怎么样才能隔绝这种指针的联系，而又能让子保持对父亲的原型指向，
    但又不影响父亲呢？
    所以我们看到了js中经典的继承：

    function Son(){}

    Son.prototype = new Base()

    现在我们会发现，给Son原型中添加或者改变东西都不会影响到Base了；

    但是这里还有一个问题！！！！

    var test1 = new Son()
    function Base(){
      this.name  = 'Base'
    }

    test1的constructor指向了Base，
    因为constructor实际上是prototype的一个属性，
    如果完全重置了prototype，则constructor的指向也会改变，
    我们需要在这类操作之后把constructor指回来

    Son.prototype = new Base()
    Son.constructor = Son
```

我们通常用new来创建新的对象，在ES5中有了 Object.create()方法
接受的参数为产生对象的原型   

``` javascript
    Object.create(null)

    这个对象非常特殊，甚至没有指向Object的原型，是一个没有原型的非常干净的对象

    Object.create 的执行类似于

    function create(o){
      var  F = function(){}
      F.prototype = 0
      return  new F()
    }
```