---
title: 使用MySQL配合Node.js进行简单session校验
date: 2016-09-04 23:47:32
tags:
  - Back-End
  - Node.js
---
![cover](http://oanr6klwj.bkt.clouddn.com/blog/nodejs-mysql-session-cover.jpg)
> [*pixiv-ID: 52610961*](http://www.pixiv.net/member_illust.php?mode=medium&illust_id=52610961)

session是web应用保持会话的一种相对安全有效又简单的方式，通常的做法是在每当客户端发起请求，服务端就会在返回的响应头信息中添加一个字段，同时将这个字段携带的信息记录在服务端。这样，当下次客户端发起请求的时候，服务端只要检查客户端携带的cookie，将它与服务端存储的数据做对比。就可以得知客户端当前状态，从而确保会话的连接了。

<!--more-->

例如：当我第一次注册登录某网站，它记录了我的信息，并告诉我：“欢迎来到本网站！”，然后我不小心关掉了浏览器，再打开这个网站时惊喜的发现不用自己手动再登录一次了，但是当我三天没有访问它，它就会让我重新登录。这些就是通过session来完成的。

实现一个session的方式分为三步：
> - 检查一个客户端请求是否携带约定字段。
> - 若没带，说明是第一次访问。若带了，则根据该字段携带的值判断session是否超时（若超时可能需要重新登录等处理）。然后删除旧session，生成新的session。
> - 将新生成的session的识别字段及所携带的值写入响应的头信息中的cookie中，返回客户端。

******
由于Node.js项目可能需要使用cluster，这导致session不能存储在缓存中，因此需要一个数据库，redis非常适合，但是由于项目刚好使用了MySQL，因此使用MySQL来存储session。在开始编写session之前，我们需要先封装几个简单的数据库操作。为此，我使用了一个node-mysql模块来帮助我进行工作。
```javascript
var mysql = require('mysql');
const pool = mysql.createPool({
  host: 'yourhost',
  user: 'youruser',
  database: 'yourdb',
  password: '*****',
  port: '3306',
  connectionLimit: 100
});
const createSchema = (schema, name) => {
  return {
    name: name,
    options: schema
  }
};
const sessionSchema = createSchema(
  `(
    id VARCHAR(100) primary key,
    expire VARCHAR(50)
  )`, 'session'
);
const operation = (connection, table) => {
  this.createTable = (callback) => {
    const action = 'CREATE TABLE IF NOT EXISTS ' +
      table.name + ' ' + table.options;
    connection.query(action, callback);
  };
  this.dropTable = (callback) => {
    const action =  'DROP TABLE ' + table.name;
    connection.query(action, callback);
  };
  this.insertItem = (data, callback) => {
    const action =  'INSERT INTO ' + table.name + ' SET ?';
    connection.query(action, data, callback);
  };
  this.selectById = (id, callback) => {
    const action =  'SELECT * FROM ' + table.name + ' WHERE id=' + pool.escape(id);
    connection.query(action, callback);
  };
  this.updateById = (id, data, callback) => {
    const action = 'UPDATE ' + table.name + ' SET ? WHERE id=' + pool.escape(id);
    connection.query(action, data, callback);
  };
  this.deleteById = (id, callback) => {
    const action = 'DELETE FROM ' + table.name + ' WHERE id=' + pool.escape(id);
    connection.query(action, callback);
  };
  this.release = () => {
    connection.release();
  };
  return this;
};
const connectPool = (callback, table) => {
  pool.getConnection((err, connection) => {
    if(err) {
      console.log('[query] - :' + err);
      return;
    }
    console.log('[connection connect] succeed!');
    const connect = operation(connection, table);
    callback(connect);
  });
};
const defaultCallback = (err, result) => {
  if (err) {
    console.log(err);
  } else {
    console.log(result);
  }
};
const initTable = () => {
  const createSessionTable = connectPool((connect) => {
    connect.createTable(defaultCallback);
    connect.release();
  }, sessionSchema);
};
const manager = (mode) => {
  const common = (callback) => {
    return connectPool((connect) => {
      callback(connect);
      connect.release();
    }, mode);
  };
  return {
    create: () => {
      common((connect) => {
        connect.createTable(defaultCallback);
      });
    },
    add: (data) => {
      common((connect) => {
        connect.insertItem(data, defaultCallback);
      });
    },
    delete: (id) => {
      common((connect) => {
        connect.deleteById(id, defaultCallback);
      });
    },
    get: (id, callback) => {
      common((connect) => {
        connect.selectById(id, (err, result)=>{
          if (err) {
            console.log(err);
          } else {
            callback(result);
          }
        });
      });
    },
    update: (data) => {
      common((connect) => {
        connect.updateById(data.id, data, defaultCallback);
      });
    },
  }
}
initTable();
const sessionManager = manager(sessionSchema);
```
这样我们就封装好了一个sessionManager，用来对session表进行增删查改的操作，这样封装的好处是如果有了新的table，只要在原代码的基础上新增相关schema即可，不需要修改和删除代码。
******
接下来我们可以进行服务端session校验的编写了。
> - 确定session的时效`EXPIRES`和约定字段`KEY`，并用`parseCookie`解析请求中的cookie。
> - 使用`hackHead`方法改写response方法使其响应客户端时能够携带session。
> - 使用两个promise`checkExist`和`sessionMiddleWare`来让异步操作更有条理，对session是否存在以及删除超时ssession，生成新session进行处理。
> - 把后续操作放在`handler`中进行，这样就可以保证客户端是和我们保持通信的状态了。

```javascript
const http = require('http');
const KEY = 'session_id';
const EXPIRES = 20 * 60 * 1000;
const generate = function() {
  var session = {};
  session.id = (new Date()).getTime() + Math.random().toFixed(2);
  session.expire = (new Date()).getTime() + EXPIRES;
  sessionManager.add(session);
  return session;
}
const checkExist = (id) => {
  var promise = new Promise((resolve, reject) => {
    sessionManager.get(id, (session) => {
      if (session) {
        resolve(session);
      }
    });
  });
  return promise;
}
const parseCookie = (req) => {
  if (req.headers.cookie){
    let cookies = {};
    req.headers.cookie.split(';').forEach((cookie) => {
      let parts = cookie.split('=');
      cookies[ parts[0].trim() ] = ( parts[1] || '').trim();
    })
    return cookies;
  } else {
    return null;
  }
}
const hackHead = (sessions, req, res) => {
  var writeHead = res.writeHead;
  res.writeHead = (status, headers) => {
    let cookies = req.headers.cookie;
    let session = KEY + '=' + sessions.id;
    cookies = Array.isArray(cookies) ?
      cookies.concat(session) :
      [cookies, session];
    res.setHeader('Set-Cookie', cookies);
    res.writeHead = writeHead;
    return res.writeHead(status, headers);
  }
  return res.writeHead;
}
const sessionMiddleWare = (req, res) => {
  var promise = new Promise((resolve, reject) => {
    const cookies = parseCookie(req);
    if(!cookies || !cookies[KEY]) {
      const session = generate(user);
      res.writeHead = hackHead(session, req, res);
      const output = [req, res];
      resolve(output);
    } else {
      checkExist(cookies[KEY]).then((session) => {
        let newSession;
        if (session[0]){
          session = session[0];
          if (session.expire > (new Date()).getTime()) {
            newSession = session;
            newSession.expire = (new Date()).getTime() + EXPIRES;
            manager.update(newSession);
          } else {
            manager.delete(session.id);
            newSession = generate(user);
          }
        } else {
          newSession = generate(user);
        }
        return newSession;
      }).then((session) => {
        res.writeHead = hackHead(session, req, res);
        const output = [req, res];
        resolve(output);
      });
    }
  });
  return promise;
}
const checkSession = (req, res, handler) => {
  sessionMiddleWare(req, res).then((output) => {
    handler(output[0], output[1]);
  });
}
```
