---
layout: post
title:  "看源码：vue是怎么监听数组变化"
categories: vue
color: 00CD00
---


对数组的操作是不能通过get set来响应的，所以vue对于数组的操作，

如下图数组内的这些操作，都重写了Array的原生方法，

对于push unshift 和 splice的添加操作，对新的数组元素进行监听
在重写的办法中，手动触发dep.notify()方法进行响应

``` javascript
var arrayProto = Array.prototype;
  var arrayMethods = Object.create(arrayProto);[
    'push',
    'pop',
    'shift',
    'unshift',
    'splice',
    'sort',
    'reverse'
  ]
  .forEach(function (method) {
    // cache original method
    var original = arrayProto[method];
    def(arrayMethods, method, function mutator () {
      var arguments$1 = arguments;

      // avoid leaking arguments:
      // http://jsperf.com/closure-with-arguments
      var i = arguments.length;
      var args = new Array(i);
      while (i--) {
        args[i] = arguments$1[i];
      }
      var result = original.apply(this, args);
      var ob = this.__ob__;
      var inserted;
      switch (method) {
        case 'push':
          inserted = args;
          break
        case 'unshift':
          inserted = args;
          break
        case 'splice':
          inserted = args.slice(2);
          break
      }
      if (inserted) { ob.observeArray(inserted); }
      // notify change
      ob.dep.notify();
      return result
    });
  });
  ```