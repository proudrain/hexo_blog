---
title: JavaScript异步编程进化论
date: 2016-09-07 17:12:43
tags: 
 - JavaScript
---
![cover](http://oanr6klwj.bkt.clouddn.com/blog/js-async-cover.jpg)
> [*pixiv-ID: 16810195*](http://www.pixiv.net/member_illust.php?mode=medium&illust_id=16810195)

众所周知，JavaScript的执行环境是单线程的，这意味着如果有很多任务，就只能排队一个一个执行。请看下面的例子。
```javascript
console.log(1);
var X = 100000;
for(let i = 0; i < X; i++) {
}
console.log(2);
```
随着X的增大，打印1和2之间出现的时间间隔也在明显变长。对此，JavaScript采取异步来处理这种问题。
<!--more-->
```javascript
console.log(1);
setTimeout(() => {
  console.log(2)
}, 2000);
console.log(3)
```
事实上，打印的结果是1、3、2，setTimeout是一个典型的异步操作，所以JavaScript引擎执行到它时，会让它异步执行，而自己继续执行后面的任务，等到异步操作有了结果，再进行处理。
但是异步编程的方法是值得讨论的，下面我用Node.js环境来进行后续的演示。
```javascript
const http = require('http');
http.createServer((req, res) => {
  res.writeHead(200, {
    'Content-Type': 'text/plain'
  });
  //TODO
}).listen(3000, console.log('callback running at localhost:3000'));
```
我创建了一个本地服务器，**后面书写的所有代码都放在注释为`TODO`的位置执行**。
******
## Callback
早期的大多数异步操作都是通过回调函数来完成的。下面我们用delay函数来模拟异步的数据获取(就像客户端通过Ajax或是服务端操作数据库所获取的数据)。
```javascript
const delay = (msg, callback) => {
  setTimeout(() => {
    callback(msg + '\n');
  }, 0);
}
delay('delay begin', (data) => {
  res.write(data);
  delay('delay end', (data) => {
    res.write(data);
    delay('callback work', (data) => {
      res.write(data);
      delay('done', (data) => {
        res.write(data);
        res.end();
      })
    })
  })
});
```
我们发现问题的所在了，由于数据只能在回调函数内进行使用和操作，这导致了冗长的回调函数的出现，而且非常难以阅读。这就是著名的callback hell，对于需要连续进行异步操作的应用，这样的实现方案无疑是维护者的噩梦。
******
## Promise
因此，Promise应运而生，它改变了写法。请看下面的代码。
```javascript
const delay = (msg) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(msg + '\n');
    }, 0);
  });
}
delay('delay begin').then((data) => {
  res.write(data);
}).then(() => {
  return delay('delay end');
}).then((data) => {
  res.write(data);
}).then(() => {
  return delay('promise work');
}).then((data) => {
  res.write(data);
}).then(() => {
  return delay('done');
}).then((data) => {
  res.write(data);
  res.end();
});
```
这样，我们就能通过链式调用来书写异步操作，任务先后顺序清晰易懂。只是，这份代码对于维护者显然也不是那么友好，要进行修改，就要对长长的链式调用进行阅读和修改。在度过jQuery时代时，这种写法的弊端已经暴漏无疑。
******
## Generator
ES6为我们提供了更新的异步操作思路，那就是使用Generator函数。
```javascript
const delay = (msg) => {
  setTimeout(() => {
    to.next(msg + '\n');
  }, 100);
}
function* todo() {
  const begin = yield delay('delay begin');
  res.write(begin);
  const end = yield delay('delay end');
  res.write(end);
  const generator = yield delay('generator work');
  res.write(generator);
  const done = yield delay('done');
  res.write(done);
  res.end();
}
let to = todo();
to.next();
```
我们可以通过在函数声明时加上星号创建Generator函数。它允许函数在执行过程中退出和重新进入，且重新进入时的环境（例如变量等）可以继续使用。
调用Generator并不会立即执行，而是返回一个Iterator对象，每次调用这个对象的next方法，它会执行到第一个yield表达式之前，这个表达式定义了返回值以及返回状态。可以对next方法直接传入参数来规定返回值。
其实，Generator想做的就是**让异步编程越来越像同步编程**。上述代码中我们发现它确实达到了目的，但是它也有显而易见的问题，那就是我们在书写异步操作时总要牢记什么时候要调用next，且异步操作和调用Generator函数生成的实例总要放在同一作用域，耦合度很高，再加上第一次的next方法要手动调用，因此这一方案注定还有很大的提升空间。

******
## Thunk Generator
Thunk函数是一种高阶函数，它支持我们将一个**带有回调函数作为参数的多参函数转化为单参版本**。举个例子：
```javascript
//带回调函数作为参数的多参函数
const delay = (msg, callback) => {
  setTimeout(() => {
    callback(msg);
  }, 0);
};
//常规调用
delay('your msg', () => {
  //whatever
});
//Thunk化
const delay_thunk = Thunk(delay);
//Thunk化之后的调用
delay('your msg')(() => {
  //whatever
});
```
如果我们使用Thunk化之后的delay函数，那么就有希望把generator函数中执行的异步函数和next方法分离开来，并制造出一个generator自动执行器。
```javascript
const Thunk = (f) => {
  return (...args) => {
    return (callback) => {
      return f.call(this, ...args, callback);
    }
  }
};
const delay = (msg, callback) => {
  setTimeout(() => {
    callback(msg);
  }, 0);
}
const write = (pass) => {
  res.write(pass + '\n');
}
const runG = (g) => {
  const it = g();
  !function next(data){
    let result = it.next(data);
    if (!result.done) {
      result.value(next);
    }
  }();
};
const td = Thunk(delay);
const todo = function* () {
  const begin = yield td('delay begin');
  const end = yield td('delay end');
  const gValue = yield td('generator thunk value');
  const done = yield td('done');
  write(begin);
  write(end);
  write(gValue);
  write(done);
  res.end();
}
runG(todo);
```
也许最难理解的就是自动处理generator的函数`runG`了，这个函数其实通过递归层层调用generator函数的next方法。
这非常有意义，因为这下异步操作真正做到了将结果分离出来供外部进行使用和操作。
******
## Promise Generator
如果不想借助thunk函数，我们还可以使用promise，它还免去了我们手写`Thunk`函数的麻烦，请看代码。
```javascript
const delay = (msg) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(msg);
    }, 0);
  });
}
const write = (pass) => {
  res.write(pass + '\n');
}
const runG = (g) => {
  const it = g();
  !function next(data){
    let result = it.next(data);
    if (!result.done) {
      result.value.then((data) => {
        next(data);
      });
    } else {
      return result.value;
    }
  }();
}
const todo = function* () {
  const begin = yield delay('delay begin');
  const end = yield delay('delay end');
  const gValue = yield delay('generator thunk value');
  const done = yield delay('done');
  write(begin);
  write(end);
  write(gValue);
  write(done);
  res.end();
}
runG(todo);
```

******
## Async
ES7提出了新的异步方案async，不过babel之类的转换器已经支持了它，async其实就是改变了generator函数的api，同时**内置了自动执行函数**。
```javascript
const delay = (msg) => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(msg);
    }, 0);
  });
}
const write = (pass) => {
  res.write(pass + '\n');
}
async function todo() {
  const begin = await delay('delay begin');
  const end = await delay('delay end');
  const gValue = await delay('generator thunk value');
  const done = await delay('done');
  write(begin);
  write(end);
  write(gValue);
  write(done);
  res.end();
}
const autoRun = todo();
```
可以看到，就是将声明方式去掉了星号，在函数前加上了`async`前缀，并将`yield`替换为`await`，执行时，只需要进行一次调用，就能自动执行内部函数了。
******
总之，异步操作正变得越来越好理解，好使用，好维护，它目前进步的方向是变得越来越像同步操作。
如果你想了解更多：
- [阮一峰的ES6入门教程](http://es6.ruanyifeng.com/)
- [阮一峰关于Thunk函数的教程](http://www.ruanyifeng.com/blog/2015/05/thunk.html)
- [MDN的Generator函数介绍](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*)
- [ES6 Generators: Complete Series BY Kyle Simpson系列教程](https://davidwalsh.name/es6-generators)
- [Promise迷你书 BY azu](http://liubin.org/promises-book/)

