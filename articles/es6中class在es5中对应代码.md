## es6中class在es5中对应代码
---
```js
class Super{
  constructor(){
    this.superValue = 'this is father';
  }

  getSuperValue(){
    console.log(this.superValue);
  }
}

class Sub extends Super{
  constructor(){
    super();
    this.subValue = 'this is child';
  }

  getSubValue(){
    console.log(this.subValue);
  }
}
```

以上是es6中class写法下的继承，表面上看起来和java相同。但这其实是一种语法糖，我们需要知道他在es5中对应代码才能更好知道继承的原理。

```js
function Super(){
    this.superValue = 'this is father';
}

Super.prototype.getSuperValue= function(){
    console.log(this.superValue);
}

function Sub(){
    Super.call(this);
    this.subValue = 'this is child';
}

Sub.prototype = new Super();

Sub.prototype.getSubValue = function(){
    console.log(this.subValue);
}

```
