---
title: CORS和Cookies
date: 2016-08-27 17:02:06
tags:
  - Back-End
  - Node.js
---
![cover](http://oanr6klwj.bkt.clouddn.com/blog/cors-and-cookies-cover.jpg)
> [*pixiv-ID: 56852068*](http://www.pixiv.net/member_illust.php?mode=medium&illust_id=56852068)

本次的项目是为一个短期活动搭建Node.js服务端，部署在服务器上后发现同事在自己的本机使用需要携带Cookie的AJAX请求调用接口的时候,受到了同源策略的限制。因此我们需要在服务端对跨域请求开放许可。

<!--more-->

> CORS是一个W3C标准， 全称是“跨域资源共享”（Cross-origin resource sharing）。
> 它允许了浏览器向跨源服务器发送AJAX请求。（受限于[同源策略](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy)，一般的AJAX不能跨域）。

跨域AJAX请求的发送源需要在服务端的许可名单之中，因此需要服务端对请求源开放许可。

本次项目中的服务端语言是Node.js，一般来说开放许可的做法为响应的头信息添加一些字段。

```javascript
res.setHeader('Access-Control-Allow-Origin', *);
//可以接受的源，*表示任意
res.setHeader('Access-Control-Allow-Headers', 'X-Requested-With');
//预检的时候使用，指明请求中可以使用的自定义HTTP请求头
res.setHeader('Access-Control-Allow-Methos', 'PUT, POST, GET, DELETE, OPTIONS');
//预检的时候使用，指明资源可以被请求的方式
```

但是如果在请求的时候，需要携带Cookie时，我们就需要在**服务端和客户端**两端做一些处理

> 在客户端，我们需要给`XMLHTTPRequest`对象开启`withCredentials`属性，例如下面我封装的一个简单的AJAX请求。

```javascript
function AJAX(method, URL, async, type, callback, data) {
    var xhr = new XMLHttpRequest();
    xhr.responseType = type;
    //开启withCredentials属性
    xhr.withCredentials = true;
    xhr.onreadystatechange = function() {
        if (xhr.readyState == 4 && xhr.status == 200) {
            var getRes = xhr.response;
            callback(getRes);
        }
    };
    xhr.onerror = function (e) {
        console.error(e);
    };
    xhr.open(method, URL, async);
    if (data) {
      xhr.setRequestHeader("Content-Type", "application/x-www-form-urlencoded");
      xhr.send(data);
    } else {
      xhr.send();
    }
}
```

> 在服务端，我们需要为响应的头信息添加`Access-Control-Allow-Credentials`。**这时服务端必须制定请求的域名，不再能使用'*'**, 我们需要针对不同访问设定跨源允许域，这时HTTP请求头的`Referer`值就很好用了，当在一个域下发起一个CORS请求时，HTTP请求头的`Referer`会自动设置为页面域，此时只要我们在服务端根据`Referer`值构造出相应的`Access-Control-Allow-Origin`值即可。

```javascript
let origin = req.headers.referer;
if (origin) {
  origin = origin.match(/^http:\/\/+[a-zA-Z0-9\.]+(\:[0-9]+)?/);
}
origin = origin ? origin[0] : 'your url';
res.setHeader('Access-Control-Allow-Origin', origin);
res.setHeader('Access-Control-Allow-Credentials', 'true');
```

这样我们就可以尽情使用带Cookie的跨域了。

[更多关于CORS可以阅读本篇](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Access_control_CORS)
[使用Referer的灵感来自这里](http://blog.hellofe.com/web/2014/12/28/the-CORS-protocol/)
