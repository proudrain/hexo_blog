---
title: MongoDB权限管理
date: 2016-10-25 15:12:34
tags:
 - Database
---
![cover](http://oanr6klwj.bkt.clouddn.com/blog/mongo-admin-cover.png)
> [*pixiv-ID: 58984559*](http://www.pixiv.net/member_illust.php?mode=medium&illust_id=58984559)

MongoDB数据库默认是不需要用户名和登陆密码的，在实际环境中，我们需要设置不同的权限方便开发者操作数据库但要避免开放所有权限引起的危险，因此如果使用mongodb，了解它的权限该如何分配是十分重要的。
本文以Mac上使用homebrew安装的MongoDB^3.2.10版本为准，mongodb版本更新可能会导致命令的变化和不可用。

<!--more-->

先开启数据库命令行，并创建admin数据库
```shell
$ mongo
use admin
```

创建用户的格式为：
```javascript
db.createUser({
  user: 'your username',
  pwd: 'your password',
  customData: {
    "description": "your description"
  },
  roles: [
    {
      role: "your role",
      db: "the data base to set the user"
    }
  ]
})
```
其中cunstomData字段是可选的，可以不填。
roles支持为一个用户分配多个角色，用数组的形式来表示。
其中role字段的值由MongoDB提供，目前支持的有：
> - 数据库用户角色（Database User Roles）read、readWrite
> - 数据库管理角色（Database Administration Roles）dbAdmin、dbOwner、userAdmin
> - 管理员组（Cluster Administration Roles）clusterAdmin、clusterManager、clusterMonitor、hostManager
> - 备份还原角色组（Backup and Restoration Roles）backup、restore
> - 所有数据库角色（All-Database Roles）readAnyDatabase、readWriteAnyDatabase、userAdminAnyDatabase、dbAdminAnyDatabase
> - 超级管理员（Superuser Roles）root
> - 内部角色（Internal Role）\_system（一般不建议设置）

> 每个角色的具体职责请查阅[官网](http://docs.mongoing.com/manual-zh/reference/built-in-roles.html)
> 建立完用户后，可以去相应数据库下进行验证，例如下面我将建立一个test数据库的user用户，他的权限为`dbOwner`，然后进行验证。

```javascript
db.createUser({
  user: 'test',
  pwd: 'testpwd',
  roles: [
    {
      role: 'dbOwner',
      db: 'test'
    }
  ]
})
use test
db.auth('test', 'testpwd')
```

然后我们需要在MongoDB的默认配置文件中开启安全认证，Homebrew安装的MongoDB的配置文件位置在`/usr/local/etc/mongod.conf`，我们在其中加入
```shell
security:
  authorization: "enabled"
```

之后重启服务
`$ brew services restart mongodb`

之后登陆mongodb就需要用户密码了
- 使用shell登陆
  `$ mongo db -u username -p`
- node中的连接
  `mongodb.connect('mongodb://user:pwd@host:port/db')`

db是要登陆的数据库的名字，登陆后进行权限不够的操作将会被拒绝，大大提升了安全性。

之后当我们要为某个数据库添加新的用户时，进行的操作如下：

1. 使用管理员账户登录mongodb
2. 切换至要添加用户的数据库
3. 添加用户
4. 使用`db.auth({username, pwd})`来认证用户
5. 退出管理员账户，用新创建的用户登录该数据库进行检查

几个常用的命令：

- 查看已存在用户（需要管理员权限）
  `db.system.users.find()`
- 删除一个用户（需要超级管理员权限）
  `db.system.users.remove({user: username})`
- 查看当前数据库的用户
  `show users`


