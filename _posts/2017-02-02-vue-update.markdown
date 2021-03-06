---
layout: post
title:  "看源码：简单例子来窥探一下vue update"
categories: vue
color: 00CD00
---

之前有一篇大概讲了讲mount的过程：[简单例子来窥探一下vue mount的过程](https://moumoutang.github.io/vue-mount/)

还接着那个例子来说，为了看到update过程，将click的响应稍微改动一下

``` javascript
haha: function(){
	this.currentBranch = "develop"
}

``` 

- 因为是设值，所以自然跳到了对应的set方法;
- 没有相对应的setter，需要observe，但是因为只是一个字符串值，不需要进行监听;
- 如果是object或者数组等引用类型的值就需要加上新的监听

到了关键的dep.notify()

``` javascript
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      ...
    },
    set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val;
      /* eslint-disable no-self-compare */
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      /* eslint-enable no-self-compare */
      if ("development" !== 'production' && customSetter) {
        customSetter();
      }
      if (setter) {
        setter.call(obj, newVal);
      } else {
        val = newVal;
      }
      childOb = observe(newVal);
      dep.notify();
    }
  });
``` 

那么此时dep的值是什么呢？

我们回顾上一次讲mount的时候，在执行eval函数的时候会调用到get句柄，在此时给每个被监听的值上加了watcher

![Alt text](https://moumoutang.github.io/public/images/vue/2017-04-20_14-52-14.png)

我们会发现对于currentBranch有两个watcher    

怎么会有两个咧？

原来前面还有个watch。。。

``` javascript
 watch: {
    currentBranch: 'fetchData'
  }
``` 

用户自定义的watcher是先init于render形成的watcher的

接下来就是调用每一个watcher的update方法：


``` javascript
  Dep.prototype.notify = function notify () {
    // stabilize the subscriber list first
    var subs = this.subs.slice();
    for (var i = 0, l = subs.length; i < l; i++) {
      subs[i].update();
    }
  };
``` 

如果不是lazy 或者 sync同步的,则将现在的watcher 塞入一个队列中,然后执行nextTick函数。我们先不看nextTick，先看看

``` javascript
Watcher.prototype.update = function update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true;
    } else if (this.sync) {
      this.run();
    } else {
      queueWatcher(this);
    }
  };

  function queueWatcher (watcher) {
  var id = watcher.id;
  if (has[id] == null) {
    has[id] = true;
    if (!flushing) {
      queue.push(watcher);
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      var i = queue.length - 1;
      while (i >= 0 && queue[i].id > watcher.id) {
        i--;
      }
      queue.splice(Math.max(i, index) + 1, 0, watcher);
    }
    // queue the flush
    if (!waiting) {
      waiting = true;
      nextTick(flushSchedulerQueue);
    }
  }
}
``` 

flushSchedulerQueue的注释很有意思，给了很多信息:

- 组件更新是有顺序的，先更新祖先节点然后再更新孩子节点，它们的id watcher在初始化的时候也一定时有顺序的

- 用户定义的watcher的运行顺序一定是高于render的watcher，这是在初始化的时候自定义的wather是先与render的watcher

- 如果一个组件在父组件的watcher的运行期间被销毁，则它的watcher也会跳过

- 不要缓存队列的length，因为随时可能有新的wather入队

- 在dev环境中会检查回环

``` javascript
function flushSchedulerQueue () {
  flushing = true;
  var watcher, id, vm;

  // Sort queue before flush.
  // This ensures that:
  // 1. Components are updated from parent to child. (because parent is always
  //    created before the child)
  //    组件更新是有顺序的，先更新祖先节点然后再更新孩子节点，它们的id在初始化的时候也一定时有顺序的
  // 2. A component's user watchers are run before its render watcher (because
  //    user watchers are created before the render watcher)
  // 3. If a component is destroyed during a parent component's watcher run,
  //    its watchers can be skipped.
  queue.sort(function (a, b) { return a.id - b.id; });

  // do not cache length because more watchers might be pushed
  // as we run existing watchers
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index];
    id = watcher.id;
    has[id] = null;
    watcher.run();
    // in dev build, check and stop circular updates.
    if ("development" !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1;
      if (circular[id] > config._maxUpdateCount) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? ("in watcher with expression \"" + (watcher.expression) + "\"")
              : "in a component render function."
          ),
          watcher.vm
        );
        break
      }
    }
  }
``` 


重磅的nextTick，可以看到nextTick其实是个闭包函数的运行结果，返回一个function
我们调用nextTick 其实就是调用返回的那个function


``` javascript
var nextTick = (function(){

  ......
   
  return function queueNextTick (cb, ctx) {
    var _resolve;

    //将flushSchedulerQueue的调用加入到callbacks中。。

    callbacks.push(function () {
      if (cb) { cb.call(ctx); }
      if (_resolve) { _resolve(ctx); }
    });

    //执行timerFunc()
    if (!pending) {
      pending = true;
      timerFunc();
    }
    if (!cb && typeof Promise !== 'undefined') {
      return new Promise(function (resolve) {
        _resolve = resolve;
      })
    }
  }

})()
``` 

timerFunc的定义：
可以看到有三种可能的方案，它的本质就是一个异步执行
- 如果有原生的Promise的支持则使用promise
- 如果没有则检测有没有MutationObserver，如果有的话，因为MutationObserver是检测dom变化的，所以create了一个 textnode，并改变其中的值，来引发MutationObserver的执行
-如果都没有就是最普通的setTimeOut

可以看到不管是哪种方法都是异步执行的


``` javascript
if (typeof Promise !== 'undefined' && isNative(Promise)) {
    var p = Promise.resolve();
    var logError = function (err) { console.error(err); };
    timerFunc = function () {
      p.then(nextTickHandler).catch(logError);
      // in problematic UIWebViews, Promise.then doesn't completely break, but
      // it can get stuck in a weird state where callbacks are pushed into the
      // microtask queue but the queue isn't being flushed, until the browser
      // needs to do some other work, e.g. handle a timer. Therefore we can
      // "force" the microtask queue to be flushed by adding an empty timer.
      if (isIOS) { setTimeout(noop); }
    };
  } else if (typeof MutationObserver !== 'undefined' && (
    isNative(MutationObserver) ||
    // PhantomJS and iOS 7.x
    MutationObserver.toString() === '[object MutationObserverConstructor]'
  )) {
    // use MutationObserver where native Promise is not available,
    // e.g. PhantomJS IE11, iOS7, Android 4.4
    var counter = 1;
    var observer = new MutationObserver(nextTickHandler);
    var textNode = document.createTextNode(String(counter));
    observer.observe(textNode, {
      characterData: true
    });
    timerFunc = function () {
      counter = (counter + 1) % 2;
      textNode.data = String(counter);
    };
  } else {
    // fallback to setTimeout
    /* istanbul ignore next */
    timerFunc = function () {
      setTimeout(nextTickHandler, 0);
    };
  }
``` 
我们可以看到nextTickHandler 就是执行 callbacks

``` javascript
  function nextTickHandler () {
    pending = false;
    var copies = callbacks.slice(0);
    callbacks.length = 0;
    for (var i = 0; i < copies.length; i++) {
      copies[i]();
    }
  }
``` 

比较完整的nextTick函数：  

这里还有个兼容性问题，在一写ios里，promise不会完全的break，它会之一直待在一个队列里，直到某些动作才能触发flush这个队列，所以这里强制手动的执行了一个什么都没有的setTimeout来避免这种环境

``` javascript
var nextTick = (function () {
  var callbacks = [];
  var pending = false;
  var timerFunc;

  function nextTickHandler () {
    pending = false;
    var copies = callbacks.slice(0);
    callbacks.length = 0;
    for (var i = 0; i < copies.length; i++) {
      copies[i]();
    }
  }

  // the nextTick behavior leverages the microtask queue, which can be accessed
  // via either native Promise.then or MutationObserver.
  // MutationObserver has wider support, however it is seriously bugged in
  // UIWebView in iOS >= 9.3.3 when triggered in touch event handlers. It
  // completely stops working after triggering a few times... so, if native
  // Promise is available, we will use it:
  /* istanbul ignore if */
  if (typeof Promise !== 'undefined' && isNative(Promise)) {
    var p = Promise.resolve();
    var logError = function (err) { console.error(err); };
    timerFunc = function () {
      p.then(nextTickHandler).catch(logError);
      // in problematic UIWebViews, Promise.then doesn't completely break, but
      // it can get stuck in a weird state where callbacks are pushed into the
      // microtask queue but the queue isn't being flushed, until the browser
      // needs to do some other work, e.g. handle a timer. Therefore we can
      // "force" the microtask queue to be flushed by adding an empty timer.
      if (isIOS) { setTimeout(noop); }
    };
  } else if (typeof MutationObserver !== 'undefined' && (
    isNative(MutationObserver) ||
    // PhantomJS and iOS 7.x
    MutationObserver.toString() === '[object MutationObserverConstructor]'
  )) {
    // use MutationObserver where native Promise is not available,
    // e.g. PhantomJS IE11, iOS7, Android 4.4
    var counter = 1;
    var observer = new MutationObserver(nextTickHandler);
    var textNode = document.createTextNode(String(counter));
    observer.observe(textNode, {
      characterData: true
    });
    timerFunc = function () {
      counter = (counter + 1) % 2;
      textNode.data = String(counter);
    };
  } else {
    // fallback to setTimeout
    /* istanbul ignore next */
    timerFunc = function () {
      setTimeout(nextTickHandler, 0);
    };
  }

  return function queueNextTick (cb, ctx) {
    var _resolve;
    callbacks.push(function () {
      if (cb) { cb.call(ctx); }
      if (_resolve) { _resolve(ctx); }
    });
    if (!pending) {
      pending = true;
      timerFunc();
    }
    if (!cb && typeof Promise !== 'undefined') {
      return new Promise(function (resolve) {
        _resolve = resolve;
      })
    }
  }
})();

``` 

在这里我们不是有两个watcher吗？它们都会被加入到队列里，但是它们会调用两次nextTick吗？

答案是不会，在这个时候nextTick只会调用一次 

有waiting这个值来标识，flushSchedulerQueue里run完所有队列里的watcher后会重置这个值
此时队列里有两个watcher，在flushSchedulerQueue内依次执行

这两个watcher对应的执行：
用户自定义的先执行，是发送一个ajax请求；
然后执行render的watcher


``` javascript
if (!isRealElement && sameVnode(oldVnode, vnode)) {
  // patch existing root node
  //已经有了真的prevnode  进入到update里
  //
   patchVnode(oldVnode, vnode, insertedVnodeQueue, removeOnly);
} else {

``` 

采用异步的方式，很聪明，如果有多个watcher在同步的代理被push进去，则这些watcher可以被一并执行，
特别是对于render的watcher，可以避免重复调用
