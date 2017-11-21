---
title: SKI组合子演算入门
date: 2017-11-21 18:40:19
tags:
  - Haskell
  - FP
---

![cover](http://oanr6klwj.bkt.clouddn.com/blog/ski-guide.jpg)
> [*pixiv-ID: 39597155*](https://www.pixiv.net/member_illust.php?mode=medium&illust_id=39597155)

什么是SKI组合子演算？

> SKI组合子演算是一个计算系统，它是无类型版本的Lambda演算的简约。这个系统声称在Lambda演算中所有运算都可以用三个组合子S、K和I来表达。
>
> ​	--[维基百科](https://en.wikipedia.org/wiki/SKI_combinator_calculus)

通过[组合子演算](https://en.wikipedia.org/wiki/Combinatory_logic)，我们只需要几个预定义的组合子(S、K、I)互相apply，就可以达到[图灵完备](https://en.wikipedia.org/wiki/Turing_completeness)。

<!--more-->

# 定义

所谓的SKI，一般来说可以这样定义：

`I x -> x`
`K x y -> x`
`S x y z -> x z (y z)`

# 推导

让我们从一个简单的lambda开始，感受一下SKI组合子的威力。

首先是常用的compose函数。假设有：

`compose = \g -> \f -> \x -> g (f x)`

那么我们可以做如下转换：

1. 转化`g (f x)`为`(K g x) (f x)`
2. 由于`(K g x) (f x)`满足 `x z (y z)`的形式，所以可简化为`S (K g) f x`
3. 现在compose变成了`\g -> \f -> \x -> S (K g) f x`，那么其实可以直接消去x变为`\g -> \f -> S (K g) f`
4. 同理可消去f，变为`\g -> S (K g)`
5. 转化`S (K g)`为`(K S g) (K g)`
6. 与步骤2同理，简化为`S (K S) K g`
7. 于是得到`compose = S (K S) K`

所有的lambda表达式都可以用这种方式，转化为SKI组合子。

总结一下：

1. 对于任意`f = \x -> x`, `f = I`
2. 对于任意`f = \x -> A`且A与x无关, `f = K A `
3. 对于任意`f = \x -> A B`, `f = S (\x -> A) (\x -> B)`

由于所有lambda表达式都可以转换为SKI组合子演算，所以递归、算数计算、布尔逻辑和数据结构这些自然也不在话下了。

# 转化为SK

一个有趣的事实：

```haskell
I
=> \x -> x
=> \x -> K x _
=> \x -> K x (K x)
=> \x -> S K K x
=> S K K
```

这说明了其实I组合子可以用SKK来表示，所以SKI组合子演算可以转化为SK组合子演算。

# 动手试试

看上去是不是很厉害的样子，来自己动手试试这道[codewars题目](https://www.codewars.com/kata/5a02dccf32b8b988120000da)吧。
