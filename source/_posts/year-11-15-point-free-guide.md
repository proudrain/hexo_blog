---
title: point-free转换指南
date: 2017-11-15 13:58:39
tags:
  - Haskell
  - FP
---

![cover](http://oanr6klwj.bkt.clouddn.com/blog/point-free-guide.jpg)
> [*pixiv-ID: 55550732*](https://www.pixiv.net/member_illust.php?mode=medium&illust_id=55550732)

本文几乎是对一个[StackOverflow回答](https://stackoverflow.com/questions/29596285/point-free-problems-in-haskell/29596461#29596461)的翻译，欢迎有能力的同学去点个赞。

point-free是一种编程风格，简单的说，就是省略函数的参数。它的定义和内容不在本文讨论的范畴之中。
理论上讲任何函数都可以被转换为point-free风格，本文讨论的内容就是如何将一个普通函数转换为point-free风格。

为了更简便的说明，本文代码使用Haskell编写，但是**读者就算不会Haskell也可以正常阅读**。但是任何其它语言都可以通过几个工具函数来达到相同的效果。

<!--more-->

# 一个前提、五个函数

一个函数要转换成point-free风格的函数，有一个前提和五个函数

前提：函数必须是curry的，也就是对于任何的

`f x y = Z` 有 `f = \x -> \y -> Z`

对于函数式语言，这几乎是标配，而对于其它范式的语言，可能要借助一些工具库(例如js中的lodash)或者手写转换函数了。

还有五个帮助函数也是必要的，没有它们，转换之路会无比艰难：

```haskell
id :: a -> a
id x = x
--
const :: a -> (b -> a)
const x = \_ -> x
--
(.) :: (b -> c) -> (a -> b) -> (a -> c)
f.g = \x -> f (g x)
--
flip :: (a -> b -> c) -> (b -> a -> c)
flip f = \y x -> f x y
--
(<*>) :: (a -> b -> c) -> (a -> b) -> (a -> c)
(<*>) f g x = f x (g x)
```

# 转换分析

函数大体都可以分为以下情况：

1. `\x -> x`：这种最简单，只要写为`id x`就好了
2. `\x -> A`且A与x无关：则可以写为`const x`
3. `\x -> A x` 且A与x无关：则可以写为A
4. `\x -> A B`：
   1. x与A和B都有关，则可以写为`(\x -> A) <*> (\x -> B)`
   2. x只与A有关，则写为 `flip (\x -> A) B`
   3. x只与B有关，则写为`A . (\x -> B)`
   4. x与AB都无关，emmmm…这函数你都能写出来，你是智障吗。
5. 当函数体内出现条件表达式：这种情况可以用一些特殊函数来帮忙处理，例如如果出现
   `\x -> if x == 1 then 0 else -1`
   这种函数，我们可以用一个帮助函数例如
   `bool f t b = if b then t else f`
   替换为
   `\x -> bool (x == 1) (-1) 0`


# 举个栗子

对于原函数：`f x y z = foo z (bar x y)` ，要把它转化为point-free风格，如果刚接触point-free的人肯定一脸懵逼，那么我们就来看看按照我们的指引来进行转化的过程吧：

```haskell
f x y z = foo z (bar x y)
f x y = \z -> foo z (bar x y)
f x y = \z -> (foo z) (bar x y) --这就是curry函数的好处
f x y = flip (\z -> foo z) (bar x y) --使用规则4.2
f x y = flip foo (bar x y) --使用规则3
f x = \y -> flip foo (bar x y)
f x = \y -> (flip foo) (bar x y)
f x = flip foo . (\y -> bar x y) --使用规则4.3
f x = flip foo . (bar x)
f = \x -> flip foo . (bar x)
f = \x -> (.) (flip foo) (bar x) --在Haskell中，a+b == (+) a b
f = \x -> ((.) (flip foo)) (bar x) --别忘了curry
f = ((.) (flip foo)) . (\x -> bar x) --使用规则4.3
f = ((.) (flip foo)) . bar
f = (.) (flip foo) . bar
```



当然了，point-free并不是适合所有场景，如果point-free完可读性反而变差了，你可能就要思考是否有这个必要了。