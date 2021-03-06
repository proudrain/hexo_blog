---
title: 250行实现一个简单的MVVM
date: 2016-12-19 10:09:08
tags:
 - Front-End
 - JavaScript
---

![cover](http://oanr6klwj.bkt.clouddn.com/blog/mvvm-cover.jpg)
> [*pixiv-ID: 14402942*](http://www.pixiv.net/member_illust.php?mode=medium&illust_id=14402942)

MVVM这两年在前端届掀起了一股热潮，火热的Vue和Angular带给了开发者无数的便利，本文将实现一个简单的MVVM，用200多行代码探索MVVM的秘密。您可以[先点击本文的JS Bin查看效果](https://jsbin.com/juqeduf/edit?console,output)

<!--more-->
<!-- toc -->

# 什么是MVVM？
MVVM是一种程序架构设计。把它拆开来看应该是**Model-View-ViewModel**。

## Model
Model指的是数据层，是纯净的数据。对于前端来说，它往往是一个简单的对象。例如：
```javascript
{
  name: 'mirone',
  age: 20,
  friends: ['singleDogA', 'singleDogB'],
  details: {
    type: 'notSingleDog',
    tags: ['fff', 'sox']
  }
}
```
数据层是我们需要渲染后呈现给用户的数据，数据层本身是可变的。数据层不应该承担逻辑操作和计算的功能。

## View
View指视图层，是直接呈现给用户的部分，简单的来说，对于前端就是HTML。例如上面的数据层，它对应的视图层可能是：
```html
<div>
  <p>
    <b>name: </b>
    <span>mirone</span>
  </p>
  <p>
    <b>age: </b>
    <span>20</span>
  </p>
  <ul>
    <li>singleDogA</li>
    <li>singleDogB</li>
  </ul>
  <div>
    <p>notSingleDog</p>
    <ul>
      <li>fff</li>
      <li>sox</li>
    </ul>
  </div>
</div>
```
当然视图层是可变的，你完全可以在其中随意添加元素。这不会改变数据层，只会改变视图层呈现数据的方式。**视图层应该和数据层完全分离。**

## ViewModel
既然视图层应该和数据层分离，那么我们就需要设计一种结构，让它们建立起某种联系。当我们对Model进行修改的时候，ViewModel就会把修改自动同步到View层去。同样当我们修改View，Model同样被ViewModel自动修改。

可以看出，如何设计能够高效自动同步View与Model的ViewModel是整个MVVM框架的核心和难点。

# MVVM的原理

## 差异
不同的框架对于MVVM的实现是不同的。

### 数据劫持
Vue的实现方式，对数据（Model）进行劫持，当数据发生变动时，数据会触发劫持时绑定的方法，对视图进行更新。

### 脏检查机制
Angular的实现方式，当发生了某种事件（例如输入），Angular会检查新的数据结构和之前的数据结构是否发生了变动，来决定是否更新视图。

### 发布订阅模式
Knockout的实现方式，实现了一个发布订阅器，解析时会在对应视图节点绑定订阅器，而在数据上绑定发布器，当修改数据时，就出发了发布器，视图收到后进行对应更新。

## 相同点
但是还是有很多相同点的，它们都有三个步骤：

- 解析模版
- 解析数据
- 绑定模版与数据

### 解析模版
何谓模版？我们可以看一下主流MVVM的API：
```html
<!-- Vue -->
<div id="mobile-list">
  <h1 v-text="title"></h1>
  <ul>
    <li v-for="item in brands">
      <b v-text="item.name"></b>
      <span v-show="showRank">Rank: {{item.rank}}</span>
    </li>
  </ul>
</div>
<!-- Angular -->
<ul>
  <li ng-repeat="phone in phones">
    {{phone.name}}
    <p>{{phone.snippet}}</p>
  </li>
</ul>
<!-- Knockout -->
<tbody data-bind="foreach: seats">
  <tr>
    <td data-bind="text: name"></td>
    <td data-bind="text: meal().mealName"></td>
    <td data-bind="text: meal().price"></td>
  </tr>    
</tbody>
```
可以看到它们都定义了自己的模版关键字，这一模块的作用就是根据这些关键字解析模版，将模版对应到期望的数据结构。

### 解析数据
Model中的数据经过劫持或绑定发布器来解析。数据解析器的编写要考虑VM的实现方式，但是无论如何解析数据只要做好一件事：定义数据变动时要通知的对象。解析数据时应保证数据解析后的一致性，对于每种数据解析后暴露的接口应该保持一致。

### 绑定模版与数据
这一部分定义了数据结构以何种方式和模版进行绑定，就是传说中的“双向绑定”。绑定之后我们直接对数据进行操作时，应用就能自动更新视图了。数据和模版往往是多对多的关系，而且不同的模版更新数据的方式往往不同。例如有的是改变标签的文本节点，有的是改变标签的className。

# 动手实现MVVM
经过一番分析，来动手实现MVVM吧。

## 期望效果
对于我的MVVM，我希望对应一个数据结构：
```javascript
let data = {
  title: 'todo list',
  user: 'mirone',
  todos: [
    {
      creator: 'mirone',
      content: 'write mvvm'
      done: 'undone',
      date: '2016-11-17',
      members: [
        {
          name: 'kaito'
        }
      ]
    }
  ]
}
```
我可以对应的编写模版：
```html
<div id="root">
  <h1 data-model="title"></h1>
  <div>
    <div data-model="user"></div>
    <ul data-list="todos">
      <li data-list-item="todos">
        <p data-class="todos:done" data-model="todos:creator"></p>
        <p data-model="todos:date"></p>
        <p data-model="todos:content"></p>
        <ul data-list="todos:members">
          <li data-list-item="todos:members">
            <span data-model="todos:members:name"></span>
          </li>
        </ul>
      </li>
    </ul>
  </div>
</div>
```
然后通过调用：
```javascript
new Parser('#root', data)
```
就可以完成mvvm的绑定，之后可以直接操作data对象来对View进行更改。

## 解析模版
模版的解析其实是一个树的遍历过程。

### 遍历
众所周知，DOM是一个树状结构，这也是为什么它被称为“DOM树”。
对于树的遍历，只要递归，便能很轻松的完成一个深度优先遍历，请看代码：
```javascript
function scan(node) {
  console.log(node)
  for(let i = 0; i < node.children.length; i++) {
    const _thisNode  = node.children[i]
    console.log(_thisNode)
    if(_thisNode.children.length) {
      scan(_thisNode)
    }
  }
}
```
这个函数遍历了一个DOM节点，依次打印遍历得到的节点。

### 遍历不同结构
知道了如何遍历一个DOM树，那么我们如何获取需要分析的DOM树？
根据之前的构想，我们需要这么几种标识：
- `data-model`——用于将DOM的文本节点替换为制定内容
- `data-class`——用于将 DOM的className替换为制定内容
- `data-list`——用于标识接下来将出现一个列表，列表为制定结构
- `data-list-item`——用于标识列表项的内部结构
- `data-event`——用于为DOM节点绑定指定事件

简单的归类一下：`data-model`、`data-class`和`data-event`应该是一类，它们都只影响当前节点；而`data-list`和`data-item`作为列表应该要单独考虑。那么我们可以这样遍历：
```javascript
function scan(node) {
  if(!node.getAttribute('data-list')) {
    for(let i = 0; i < node.children.length; i++) {
      const _thisNode = node.children[i]
      parseModel(node)
      parseClass(node)
      parseEvent(node)
      if(_thisNode.children.length) {
        scan(_thisNode)
      }
    }
  } else {
    parseList(node)
  }
}
function parseModel(node) {
  //TODO:解析Model节点
}
function parseClass(node) {
  //TODO:解析className
}
function parseEvent(node) {
  //TODO:解析事件
}
function parseList(node) {
  //TODO: 解析列表
}
```
这样我们就搭好了遍历器的大概框架

### 不同结构的处理方法
`parseModel`，`parseClass`和`parseEvent`的处理方式比较相似，唯一值得注意的就是对于嵌套元素的处理，回忆一下我们的模版设计：
```html
<!--遇到嵌套部分-->
<div data-model="todos:date"></div>
```
这里的`todos:date`其实大大方便了我们解析模版，因为它展示了当前数据在Model结构中的位置。
```javascript
//event要有一个eventList,大概结构为：
const eventList = {
  typeWriter: {
    type: 'input', //事件的种类
    fn: function() {
      //事件的处理函数，函数的this代表函数绑定的DOM节点
    }
  }
}
function parseEvent(node) {
  if(node.getAttribute('data-event')) {
    const eventName = node.getAttribute('data-event')
    node.addEventListener(eventList[eventName].type, eventList[eventName].fn.bind(node))
  }
}
//根据在模版中的位置解析模版，这里的Path是一个数组，代表了当前数据在Model中的位置
function parseData(str, node) {
  const _list = str.split(':')
  let _data,
    _path
  let p = []
  _list.forEach((key, index) => {
    if(index === 0) {
      _data = data[key]
      p.push(key)
    } else {
      _path = node.path[index-1]
      p.push(_path)
      _data = _data[_path][key]
      p.push(key)
    }
  })
  return {
    path: p,
    data: _data
  }
}
function parseModel(node) {
  if(node.getAttribute('data-model')) {
    const modelName = node.getAttribute('data-model')
    const _data = parseData(modelName, node)
    if(node.tagName === 'INPUT') {
      node.value = _data.data
    } else {
      node.innerText = _data.data
    }
  }
}
function parseClass(node) {
  if(node.getAttribute('data-class')) {
    const className = node.getAttribute('data-class')
    const _data = parseData(className, node)
    if(!node.classList.contains(_data.data)) {
      node.classList.add(_data.data)
    }
  }
}
```

接下来解析列表，我们遇到列表时，应该先递归找出列表项的结构
```javascript
parseListItem(node) {
  let target
  !function getItem(node) {
    for(let i = 0; i < node.children.length; i++) {
      const _thisNode = node.children[i]
      if(node.path) {
        _thisNode.path = node.path.slice()
      }
      parseEvent(_thisNode)
      parseClass(_thisNode)
      parseModel(_thisNode)
      if(_thisNode.getAttribute('data-list-item')) {
        target = _thisNode
      } else {
        getItem(_thisNode)
      } 
    }
  }(node)
  return target
}
```
之后在用这个列表项来按需拷贝出一定数量的列表项，并填充数据
```javascript
function parseList(node) {
  const _item = parseListItem(node)
  const _list = node.getAttribute('data-list')
  const _listData = parseData(_list, node)
  _listData.data.forEach((_dataItem, index) => {
    const _copyItem = _item.cloneNode(true)
    if(node.path) {
      _copyItem.path = node.path.slice()
    }
    if(!_copyItem.path) {
      _copyItem.path = []
    }
    _copyItem.path.push(index)
    scan(_copyItem)
    node.insertBefore(_copyItem, _item)
  })
  node.removeChild(_item)
}
```
这样我们就完成了模版的渲染，scan函数会扫描模版对模版进行渲染

## 解析数据
解析了模版之后，我们就要研究如何进行数据解析了，这里我采用劫持数据的方法来进行。

### 普通对象的劫持
如何劫持数据？一般对数据的劫持都是通过Object.defineProperty方法进行的，先看一个小例子：
```javascript
var obj = {
  name: 'mi'
}
function observe(obj, key) {
  let old = obj[key]
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function() {
      return old
    },
    set: function(now) {
      if(now !== old) {
        console.log(`${old} ---> ${now}`)
        old = now
      }
    }
  })
}
observe(obj, 'name')
obj.name = 'mirone'
//输出结果：
//"mi ---> mirone"
```
这样我们就通过object.defineProperty进行了数据劫持，如果我们想自定义劫持数据时发生的操作，只要添加一个回调函数参数即可：
```javascript
function observer(obj, k, callback) {
  let old = obj[k]  
  Object.defineProperty(obj, k, {
    enumerable: true,
    configurable: true,
    get: function() {
      return old
    },
    set: function(now) {
      if(now !== old) {
        callback(old, now)
      }
      old = now
    }
  })
}
```

### 嵌套对象的劫持
对于对象中的对象，我么还需要多进行一个步骤，使用递归来劫持对象中的对象：
```javascript
//实现一个observeAllKey函数，劫持该对象的所有属性
function observeAllKey(obj, callback) {
  Object.keys(obj).forEach(function(key){
    observer(obj, key, callback)
  })
}
function observer(obj, k, callback) {
  let old = obj[k]
  if (old.toString() === '[object Object]') {
    observeAllKey(old, callback)
  } else {
    //...同前文，省略
  }
}
```

### 对象中数组的劫持
对于对象中的数组，我们使用重写数组的prototype的方法来劫持它
```javascript
function observeArray(arr, callback) {
  const oam = ['push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse']
  const arrayProto = Array.prototype
  const hackProto = Object.create(Array.prototype)
  oam.forEach(function(method){
    Object.defineProperty(hackProto, method, {
      writable: true,
      enumerable: true,
      configurable: true,
      value: function(...arg) {
        let old = arr.slice()
        let now = arrayProto[method].call(this, ...arg)
        callback(old, this, ...arg)
        return now
      },
    })
  })
  arr.__proto__ = hackProto
}
```
写完劫持数组的函数后，将它添加进主函数：
```javascript
function observer(obj, k, callback) {
  let old = obj[k]
  if(Object.prototype.toString.call(old) === '[object Array]') {
    observeArray(old, callback)
  } else if (old.toString() === '[object Object]') {
    observeAllKey(old, callback)
  } else {
    //...
  }
}
```

### 处理路径参数
之前我们所有的方法都是面对单个key值的，回想一下我们的模版，有很多例如`todos:todo:member`这样的路径，我们应该允许传入一个路径数组，根据路径数组来监听指定的对象数据
```javascript
function observePath(obj, path, callback) {
  let _path = obj
  let _key
  path.forEach((p, index) => {
    if(parseInt(p) === p) {
      p = parseInt(p)
    }
    if(index < path.length - 1) {
      _path = _path[p]
    } else {
      _key = p
    }
  })
  observer(_path, _key, callback)
}
```
之后再将它添加进主函数：
```javascript
function observer(obj, k, callback) {
  if(Object.prototype.toString.call(k) === '[object Array]') {
    observePath(obj, k, callback)
  } else {
    let old = obj[k]
    if(Object.prototype.toString.call(old) === '[object Array]') {
      observeArray(old, callback)
    } else if (old.toString() === '[object Object]') {
      observeAllKey(old, callback)
    } else {
      //...
    }
  }
}
```
这样，我们就完成了监听函数。

## 多对一监视的实现
有可能绑定某个数据结构的节点不止一个，有时我们需要修改完成时同时通知所有节点，那么我们还需要一个单独的模块负责通知所有节点，我们称之为Register模块，负责针对不同模块注册不同的一个或多个回调函数。

### 监视者的实现
```javascript
class Register {
  constructor() {
    //存放所有回调对象，回调对象由三个key组成：obj, key, fn，其中fn应该是一个数组，放着所有发生变化时要执行的回调函数
    this.routes = []
  }
  //添加一个回调
  regist(obj, k, fn) {
    const _i = this.routes.find(function(el) {
      if((el.key === k || el.key.toString() === k.toString()) 
        && Object.is(el.obj, obj)) {
        return el
      }
    })
    if(_i) {
      //如果已经存在该obj和key组成的对象
      _i.fn.push(fn)
    } else {
      //如果尚不存在
      this.routes.push({
        obj: obj,
        key: k,
        fn: [fn]
      })
    }
  }
  //解析结束时调用，绑定所有回调
  build() {
    this.routes.forEach((route) => {
      observer(route.obj, route.key, route.fn)
    })
  }
}
```

### observer模块的修改
由于现在一个key可能对应多个回调操作了，需要对observer进行修改：
```javascript
function observer(obj, k, callback) {
  //...与前文相同
  if(now !== old) {
    callback.forEach((fn) => {
      fn(old, now)
    })
  }
}
function observerArray(arr, callback) {
  //...与前文相同
  //将原来的callback(old, this, ...arg)替换为
  callback.forEach((fn) => {
    fn(old, this, ...arg)
  })
}
```

## 绑定模版与数据
现在，我们要在解析过程中添加对数据的监视了，还记得之前的parse系列函数吗？
```javascript
const register = new Register()
function parseModel(node) {
  if(node.getAttribute('data-model')) {
    //...之前逻辑不变
    register.regist(data, _data.path, function(old, now) {
      if(node.tagName === 'INPUT') {
        node.value = now
      } else {
        node.innerText = now
      }
      //添加console便于调试
      console.log(`${old} ---> ${now}`)
    })
  }
}
function parseClass(node) {
  if(node.getAttribute('data-class')) {
    //...
    register.regist(data, _data.path, function(old, now) {
      node.classList.remove(old)
      node.classList.add(now)
      console.log(`${old} ---> ${now}`)
    })
  }
}
//当列表发生变化时，为了简单直接重新渲染了当前列表
function parseList(node) {
  //...
  register.regist(data, _listData.path, () => {
    while(node.firstChild) {
      node.removeChild(node.firstChild)
    }
    const _listData = parseData(_list, node)
    node.appendChild(_item)
    _listData.data.forEach((_dataItem, index) => {
      const _copyItem = _item.cloneNode(true)
      if(node.path) {
        _copyItem.path = node.path.slice()
      }
      if(!_copyItem.path) {
        _copyItem.path = []
      }
      _copyItem.path.push(index)
      scan(_copyItem)
      node.insertBefore(_copyItem, _item)
    })
    node.removeChild(_item)
  })
}
//当模版解析结束后绑定所有事件
register.build()
```

至此我们就基本完成了一个简单的MVVM，之后我进行了一点细微的细节优化，源码放在
[我的Gist上](https://gist.github.com/Lovin0730/c356e18f0199482f7671c3442fe5c153)。各位也可以去[本教程的JSBin](https://jsbin.com/juqeduf/edit?console,output)查看效果。