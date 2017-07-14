---
title: 130行实现Express风格的Node.js框架
date: 2017-01-08 16:03:16
tags:
  - Back-End
  - Node.js
---
![cover](http://oanr6klwj.bkt.clouddn.com/blog/express-core.png)
> [*pixiv-ID: 59561981*](http://www.pixiv.net/member_illust.php?mode=medium&illust_id=59561981)

很多时候我们使用[Express](http://expressjs.com/)，只是用到了它方便的路由和中间件系统。其实这个功能我们用一百多行代码可以轻松实现，且没有任何依赖，而不必专门引入Express。

<!--more-->
<!-- toc -->

我们先来分析一下需求，我们要做的是一个路由系统，书写的方式为：
```javascript
//路由系统
app.method(path, handler);
//例如
app.get('/', (req, res) => {
  //do something
});
app.post('/user', (req, res) => {
  //do something
});
//中间件
app.use('/blog', (req, res, next) => {
  if(/*校验通过*/) {
    next();
  } else {
    //校验不能通过的错误信息
  }
});
//模式匹配
app.get('/blog/:id', (req, res) => {
  const id = req.params.id;
});
//监听启动服务
app.listen(port, host);
```
# 一个简单的Node.js服务器
在开始之前，我们先看看普通的Node.js服务器是什么样的：
```javascript
const http = require('http');
http.createServer((req, res) => {
  //do something
}).listen(port, host, callback);
```
每当http请求到来，就会执行回调函数(即do something位置)的代码。那么我们要做的就是**实现一个路由池，当请求到来的时候通过httpServer的回调函数遍历路由池，选择匹配的路由，执行响应的逻辑**。一个路由包括三个属性：请求方法(method)，请求路径(path)和处理函数(handler)。
# 实现路由池
路由池是一个对象数组，我们要定义好如何添加路由。按照Express的API，我们通过`app.method(path, handler)`来添加路由。
```javascript
const app = {};
const routes = [];
['get', 'post', 'put', 'delete', 'options', 'all'].forEach((method) => {
  app[method] = (path, fn) => {
    routes.push({method, path, fn});
  };
});
```
现在，我们调用app的get、post、put、delete、options, all方法时，就会添加一个路由对象到routes数组中了。例如：
```javascript
app.get('/', (req, res) => {
  res.end('hello world');
});
//此时routes为
[{
  method: 'get', 
  path: '/', 
  fn: (req, res)=>{
    res.end('hello world');
  }
}]
```
# 路由池的遍历
路由池的遍历很简单，通过循环遍历数组即可。

```javascript
const passRouter = (method, path) => {
  let fn;
  for(let route of routes) {
    if((route.path === path 
      || route.path === '*')
      && (route.method === method
      || route.method === 'all')) {
      //匹配到了符合的路由
      //路由的method为all时匹配所有请求的方法
      //路由path为*时匹配所有请求的路径
      fn = route.fn;
    }
  }
  if(!fn) {
    fn = (req, res) => {
      res.end(`Cannot ${method} ${pathname}.`);
    }
  }
  return fn;
}
```
这样我们就写好了遍历router的函数，现在要做的就是把它添加到server中。
```javascript
http.createServer((req, res) => {
  //获取请求的方法
  const method = req.method.toLowerCase()
  //解析url
  const urlObj = url.parse(req.url, true)
  //获取path部分
  const pathname = urlObj.pathname
  //遍历路由池
  const router = passRouter(method, pathname);
  router(req, res);
}).listen(port, host, callback);
```
我们可以把创建server的方法放在app对象中，把这个方法和app一起暴露出去。
```javascript
app.listen = (port, host) => {
  http.createServer((req, res) => {
    const method = req.method.toLowerCase()
    const urlObj = url.parse(req.url, true)
    const pathname = urlObj.pathname
    const router = passRouter(method, pathname);
    router(req, res);
  }).listen(port, host, () => {
    console.log(`Server running at ${host}\:${port}.`)
  });
}
```
这样，我们只要调用`app.listen(port, host)`，就可以创建服务器了。

# 添加中间件
什么是中间件？中间件是请求到达匹配的路由前经过的一层逻辑，这层逻辑可以对请求进行过滤、修改等操作。举个例子：
```javascript
app.use('/blog', (req, res, next) => {
  if(req.username) {
    next();
  } else {
    res.writeHead(404, {'Content-Type': 'text/html'});
    res.end('对不起，你没有相应权限');
  }
});
```
在这个例子中，每当请求`/blog`这个路径的时候，请求都会经过这个中间件，只有request对象有username这个方法时，请求才能继续向后传递，否则就会返回一个404信息。
要实现中间件也很简单，我们把中间件与get, post等方法一样看成是一种路由即可。于是问题的核心就变成了**由于中间件中使用next函数来确认请求通过了中间件，我们不再能通过`for..in`遍历的方法来遍历路由池了。**
如果你对ES6足够熟悉，那么这个next方法一定能让你想起一个很有趣的新语法：generator函数。

## 使用generator函数来遍历数组
[generator函数](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/function*)是一种生成器函数，允许我们在退出函数后重新进入之前的状态（可以理解为一个状态机），我们可以用它实现函数式中的惰性求值特性，用这种办法来遍历数组，举个例子：
```javascript
const lazy = function* (arr) {
  yield* arr;
}
const lazyArray = lazy([1, 2, 3]);
lazy.next(); // {value: 1, done: false}
lazy.next(); // {value: 2, done: false}
lazy.next(); // {value: 3, done: false}
lazy.next(); // {value: undefined, done: true}
```

## 重写路由遍历函数

那么我们现在可以重写路由遍历的函数了，需要注意的是，中间件匹配过程中是可以匹配子目录的，例如`/path`可以匹配到`/path/a`、`/path/a/b/c`这些目录。
```javascript
//lazy函数，使数组可被惰性求值
const lazy = function* (arr) {
  yield* arr;
}
//路由遍历
const passRouter = (routes, method, path) => (req, res) => {
  const lazyRoutes = lazy(routes);
  (function next () {
    //当前遍历状态
    const it = lazyRoutes.next().value;
    if (!it) {
      //已经遍历所有路由，没有匹配的路由，停止遍历
      res.end(`Cannot ${method} ${pathname}`)
      return;
    } else if (it.method === 'use' 
      && (it.path === '/'
      || it.path === path
      || path.startsWith(it.path.concat('/')))) {
      //匹配到了中间件
      it.fn(req, res, next);
    } else if ((it.method === method
      || it.method === 'all')
      && (it.path === path
      || it.path === '*')) {
      //匹配到了路由
      it.fn(req, res);
    } else {
      //继续匹配
      next();
    }
  }());
};
```
这样我们就得到了一个可以添加中间件的路由系统。

# 模式匹配
## 匹配路由
模式匹配是每个后端框架必不可少的功能之一。他允许我们匹配一类路由，例如`/blog/:id`可以匹配类似`/blog/123`、`/blog/qw13`之类的一系列请求路径。
既然是模式匹配，那么肯定少不了正则表达式了。我们以`/blog/:id`为例，想要匹配一系列这样的路由，只要请求的路径能够通过正则表达式`/^\/blog\/\w[^\/]+$/`即可。
**也就是说，我们把路由中的:whatever替换成正则表达式`\w[^\/]+`就能匹配到相应的路由了。**
JavaScript中提供了`new Exp`来把字符串转换为正则表达式
因此转化的步骤为：
- 将路由中模式匹配的部分转换为`\w[^\/]+`
- 用替换好的字符串生成正则表达式
- 用这一正则表达式匹配请求路径，判断是否匹配

实现：
```javascript
//转换模式为相应正则表达式
const replaceParams = (path) => new RegExp(`\^${path.replace(/:\w[^\/]+/g, '\\w[^\/]+')}\$`);
//判断模式是否吻合
//...在passRouter函数中最后一个else之前添加一层if else
} else if ( it.path.includes(':')
  && (it.method === method
  || it.method === 'all')
  && (replaceParams(it.path).test(path))) {
  //匹配成功
} else {
  next();
}
```

## 转换匹配到的路径为相应对象
匹配成功后我们需要把模式转为对象以便调用:
```javascript
//匹配成功时逻辑
let index = 0;
//分割路由
const param2Array = it.path.split('/');
//分割请求路径
const path2Array = path.split('/');
const params = {};
param2Array.forEach((path) => {
  if(/\:/.test(path)) {
    //如果是模式匹配的路径，就添加入params对象中
    params[path.slice(1)] = path2Array[index]
  }
  index++
})
req.params = params
it.fn(req, res);
```
我们把params对象加入了req对象中，调用时很方便，例如：
`/blog/:id`在调用时为`const id = req.params.id`。

# 静态文件处理
请求时如果请求了静态文件，我们的服务器还没有做出处理，这点很不合理，我们需要添加静态文件处理逻辑。
```javascript
//常用的静态文件格式
const mime = {
  "html": "text/html",
  "css": "text/css",
  "js": "text/javascript",
  "json": "application/json",
  "gif": "image/gif",
  "ico": "image/x-icon",
  "jpeg": "image/jpeg",
  "jpg": "image/jpeg",
  "png": "image/png"
}
//处理静态文件
function handleStatic(res, pathname, ext) {
  fs.exists(pathname, (exists) => {
    if(!exists) {
      res.writeHead(404, {'Content-Type': 'text/plain'})
      res.write('The request url' + pathname + 'was not found on this server')
      res.end()
    } else {
      fs.readFile(pathname, (err, file) => {
        if(err) {
          res.writeHead(500, {'Content-Type': 'text/plain'})
          res.end(err)
        } else {
          const contentType = mime[ext] || 'text/plain'
          res.writeHead(200, {'Content-Type': contentType})
          res.write(file)
          res.end()
        }
      })
    }
  })
}
```
然后我们找到app.listen函数，添加判断静态文件的逻辑。
```javascript
let _static = 'static' //默认静态文件夹位置
//更改静态文件夹的函数
app.setStatic = (path) => {
  _static = path;
};
//...server回调函数中内容
const method = req.method.toLowerCase()
const urlObj = url.parse(req.url, true)
const pathname = urlObj.pathname
//获取后缀
const ext = path.extname(pathname).slice(1)
//如果有后缀，则是静态文件
if(ext) {
  handleStatic(res, _static + pathname, ext)
} else {
  passRouter(_routes, method, pathname)(req, res)
}
```
至此，我们已经实现了一个完整的后端路由控制器，有中间件功能，静态文件处理和模式匹配功能。

# 一个彩蛋
有时我们希望node应用从命令行退出时不是直接退出，而是向我们输出一些信息（比如道个别），就像这样：
```bash
^C
Good Day!
```
这一功能借助node中[process模块](https://nodejs.org/api/process.html)的SIGINT事件也可以轻松实现，我们只需要在创建server成功的回调函数加上几行就可以了：
```javascript
http.createServer(/*...*/).listen(port, host, () => {
  console.log(`Server running at ${host}\:${port}.`)
  //添加的代码：
  process.stdin.resume();
  process.on('SIGINT', function() {
    console.log('\n');
    console.log('Good Day!');
    process.exit(2);
  });
});
```

现在，我们的退出小彩蛋也完成了。

---
完整代码放在[我的gist上](https://gist.github.com/Lovin0730/00a9ac8082d1caa74f64bbb96aa09939)。

至此，我们就完成了整个应用，如果重量级的框架对你来说比较多余，就试试自己动手实现吧。
