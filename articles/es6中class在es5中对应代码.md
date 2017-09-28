## es6中class在es5中对应代码
---
```js
class Packet extends EventEmitter{
      constructor(){
        super();
        this.value = 5;
      }

      getSubValue(){
        console.log(this.value);
      }
```