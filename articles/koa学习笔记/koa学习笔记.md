## koa学习笔记
最近因为种种原因需要使用koa框架，所以写了篇笔记记录我对koa框架源码的学习。   
虽然写完后才发现网上已经有很多类似的了。。。。   

### 简介
koa是由express原班人马打造的下一代nodejs框架，反正就是很屌啦。   
相对express框架主要改进为洋葱模型，也可以理解为一个栈，   
而express则是简单的中间件流水线式处理请求。   

为啥他们当时不直接写koa要先写个express呢？   
我认为是由于async函数的普及，基础决定上层建筑，整个koa的中间件机制都是基于async函数和promise对象实现的。  

### 中间件机制：
中间件机制是koa框架的核心，事实上koa框架的源码只有中间件机制的逻辑代码和context、request、response这三个对象的封装。    

其他的功能均由插件作为中间件来实现。   
    
下面这张图描述了中间件代码执行的流程：   

![](/articles/koa学习笔记/koa代码执行顺序.png)   

### 例子
在讨论中间件机制的实现前我们先给出一个例子：
```js 
const Koa = require('koa');
const app = new Koa();

// middleware1
app.use(async (ctx, next) => {
  console.log("middleware1 start");
  await next();
  console.log("middleware1 stop");

});

// middleware2

app.use(async (ctx, next) => {
  console.log("middleware2 start");
  await next();
  console.log("middleware2 stop");
});

// middleware3

app.use(async ctx => {
  console.log("middleware3 start and stop");
});

app.listen(8080);
```

整个调用过程的输出顺序为：
+ middleware1 start
+ middleware2 start
+ middleware3 start and stop
+ middleware2 stop
+ middleware1 stop

是不是很像一个栈？

其中中间件1和中间件2均由await next()代码将中间件处理逻辑代码一分为二，分别在不同时间点执行。  

而中间件3则没有await next（）代码，因为他已经是最末的一个中间件，所以不需要使用next调用。

### 源码分析：   
翻开koa在github 的源码，我们会发现其核心代码只有1600行，   
文件只有4个，分别为：
+ application.js:   
koa的核心代码，定义了中间件的运作机制等核心逻辑
+ context.js:   
context上下文对象的定义代码，       
这个对象的意义在于防止异步调用时上下文数据会丢失。  
+ request.js:   
request请求对象的封装代码
+ response.js:   
response答复对象的封装代码   

是的koa框架只有这几个对象以及中间件的运作机制，其他均由插入的中间件来实现。

### 实现：
koa的中间件机制基于async函数实现，这也就是为什么koa官方文档一定要告诉你如何在不同版本node中使用async函数。  

application.js文件创建的koa实例其实只对外暴露了5个接口，  
listen、use、callback、toJSON、inspect   
其中后两个接口不属于核心逻辑，我们重点讨论前三个接口。   

#### use:
use的逻辑很简单，只是把传入的中间件函数添加到本koa示例的数组中去。   

当然在此之前会判断一下传入的参数是否是函数，以及是否是generator函数，如果是则会转换成promise，这就是历史的包袱了。

#### listen：   
listen函数实质上是对node http函数调用的一层封装，   
简化后只有如下两行：   
```
listen(...args) {
  const server = http.createServer(this.callback());
  return server.listen(...args);
}
```
他首先使用node的http模块创建了http服务器，并且将this.callback()的返回值作为回调函数，然后再启动http服务器。     
  
#### callback:  
好了，这个接口就是我们整个中间件机制的核心部分了。   
下面是简化后的代码： 

```js  
callback() {
  const fn = compose(this.middleware);

  const handleRequest = (req, res) => {
    const ctx = this.createContext(req, res);
    return this.handleRequest(ctx, fn);
  };

  return handleRequest;
} 
```

在讨论compose函数前我们先讨论handleRequest函数

整个callback函数返回handleRequest函数，   
这个函数只做两件事情：   
+ 创建上下文ctx对象（创建时自动挂载req和res对象）
+ 调用koa实例的handleRequest方法，ctx和fn作为参数，并将结果返回

这里的req，res对象由原生node http模块提供。   

this.handleRequest函数简化如下：
```js
handleRequest(ctx, fnMiddleware) {
  const handleResponse = () => respond(ctx);     
  return fnMiddleware(ctx).then(handleResponse).catch(onerror);
}
```

handleRequest函数是最顶层的一次请求处理，他启动了第一个fnMiddleware，即最外层的中间件。handleRequest函数只执行一次，之后交由中间件自动链式调用。      

当最外层的中间件执行完毕后（即所有中间件执行完毕），则运行handleResponse函数，该函数封装了请求答复的相关逻辑，如```res.send()```。   

同样，handleResponse函数也只运行一次

#### compose

callback函数中koa实例的中间件数组作为参数传入compose函数，再被赋值给fn变量。    
其中compose函数对koa示例的middleware数组进行了处理，使其具有链式异步调用的能力。

compose函数简化如下：       
```js
function compose (middleware) {
  return function (context, next) {
    // last called middleware #
    let index = -1;
    let fn = middleware[i];
    return dispatch(0);

    function dispatch (i) {
        return Promise.resolve(
          fn(context, function next () {
            return dispatch(i + 1)
          })
        )
    }
  }
}
```   
调用compose函数后将会得到一个新的函数，此外层函数仅仅作为链式调用的启动配置（一些变量如index、中间件数组等）。

然后就是我们的重头戏，链式调用了。   
每一次调用dispatch函数都将调用当前的中间件函数，   
将上下文context与next函数作为参数传入，并将其返回值转换为promise对象。   
再次调用next函数时则将调用下一个中间件函数的dispatch函数。   

我只需要每次调用均保证下次调用能调用到下一个中间件函数即可，不断调用即可完成遍历。这也就是遍历器的一种应用。

### 参考资料
---
阮一峰koa教程：    
http://www.ruanyifeng.com/blog/2017/08/koa.html   
Koa中间件（middleware）实现探索：   
https://www.jianshu.com/p/a30c193f6e61   
浅析koa的洋葱模型实现：   
https://segmentfault.com/a/1190000013981513