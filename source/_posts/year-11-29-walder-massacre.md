---
title: Walder惨案
date: 2017-11-29 19:22:55
tags:
  - Haskell
---

![cover](http://oanr6klwj.bkt.clouddn.com/blog/walder-massacre.jpg)
> [*pixiv-ID: 56376665*](https://www.pixiv.net/member_illust.php?mode=medium&illust_id=56376665)

伟大的将monad带入haskell的Philip Walder曾说过一句名言：

> Monad说白了不过就是自函子范畴上的一个幺半群而已，这有什么难理解的？

于是我决定认真理解一下这句话，于是两天时间被谋杀了，史称walder惨案（雾）...但是也算是有一些收获。

本文只讨论monad的定义，对monad的用途意义等话题不做涉及。

<!--more-->

我们来理解一下这句话涉及的几个数学概念（有些概念虽然不在句中，但是对理解其它概念很有帮助）：

# 态射(Morphism)

态射是两个数学结构之间保持结构的一种抽象过程。最常见的就是函数和映射，例如在集合论中，态射就是函数。

态射满足两条**公理**：

* 满足结合律：对于任意`h . (g . f) = (h . g) . f`


* 存在恒等态射：使得对于每个对象X，存在一个态射`idx: X -> X`，使得对于每个态射`f: A -> B`都有`idb . f = f = f.idA`

我们就简单的理解为函数吧。

# 范畴(Category)

范畴，简单的讲可以看做一个集合，它有三个组成部分：

* 一个对象组成的集合
* 一个态射的集合，每个态射将两个对象联系在一起，如态射f链接了源对象A和目标对象B，那么我们就可以写为`f: A -> B`
* 态射可以进行组合，例如有态射`f: A -> B`和`g: B -> C`，那么`g . f -> A -> C`

# 函子(Functor)

函子是范畴间的一类映射，例如有范畴C和D，从C至D的函子为一映射F：

* 将每个对象`X ∈ C`映射至一对象`F(X) ∈ D`上
* 将每个态射`f: X -> Y ∈ C`映射至一态射`F(f): F(X) -> F(Y) ∈ D`上

# 自函子(Enfunctor)

自函子是一种特殊的函子，他的源范畴和目标范畴一样。也就是它映射到自身。

# 自然变换(Natrural transformation)

设有两个函子F和G都是范畴C和D之间的函子，那么对C中的任意对象A，都有在D中的F(A)和G(A)，它们之间存在函数`Ta: F(A) -> G(A)`，同样的，对于范畴C中的任意对象B在D中的映射F(B)和G(B)，它们之间存在函数`Tb: F(B) -> G(B)`；如果对于范畴C上的态射`f: A -> B`，满足`G(f) . Ta =  Tb . F(f)`，则称`t: F -> G`是函子F和G的自然变换。

# 群(Group)

群是一种代数结构，它由一个二元运算符（设为⊗）和一个集合（设为G）组成，写为(G, ⊗)，他满足**群公理**的四个要求：

* 封闭性：对于所有G中的a, b, a⊗b也在G中
* 结合性：对于所有G中的a, b, c，等式`(a ⊗ b) ⊗ c = a ⊗ (b ⊗ c)`成立
* 单位元：存在G中的一个元素e，使得所有G中的元素a，等式`(e ⊗ a) = (a ⊗ e) = e`
* 逆元：对于每个G中的a，存在G中的一个元素b使得`a ⊗ b = b ⊗ a = e`

整数在加法下就是一个常见的群。

# 幺半群(Monoid)

满足除了逆元以外的所有条件的称为幺半群。自然数群在加法下就是一个典型的幺半群。

***

好了让我们重新回味一下文章开头那句话，把它简单转换一下，得到两个信息：

* Monad是functor组成的群，所以对于任何一个Monad，存在`A -> M A`，`(A -> B) -> M A -> M B`

* 对于任意一个Monad，满足`Id ⊗ f = f = f ⊗ Id`和`(f ⊗ g) ⊗ h = f ⊗ (g ⊗ h)` 

# 举个栗子

## List Monad

让我们以List为例，看看它是如何满足这些要求的。

``` haskell
-- functor axiom 1
return :: t -> [t]
return x = [x]
-- functor axiom 2
fmap :: (a -> b) -> [a] -> [b]
fmap _ [] = []
fmap f (x:xs) = f x : fmap f xs
```

那么List首先满足了functor的要求。

我们先构造两个函数：

```haskell
concat :: [[a]] -> [a]
concat [] = []
concat (x:xs) = x ++ (concat xs)
(>=>) :: (t -> [b]) -> (b -> [a]) -> t -> [a]
(>=>) f g = \x -> concat (fmap g (f x))
```

那么设有一态射`f: a -> [a]`，则：

```haskell
  f >=> return
= \x -> concat (fmap return (f x))
= \x -> concat [(f x)]
= \x -> f x
= f
```

反过来：

```haskell
  return >=> f
= \x -> concat (fmap f (return x))
= \x -> concat (fmap f [x])
= \x -> concat [(f x)]
= \x -> f x
= f
```

于是得出结论：

`return >=> f ≡ f >=> return ≡ f`

这样我们就找到满足`Id ⊗ f = f = f ⊗ Id`的表达式，其中的⊗为`>>=`、Id为`return`。

那么要做的就是证明结合律。

为了简化证明中的表达式，先声明几个函数：

```haskell
(.) :: (t2 -> t1) -> (t -> t2) -> t -> t1
(.) f g = \x -> f (g x)
concatMap :: (a -> [b]) -> [b] -> [a]
concatMap f = \x -> concat (fmap f x)
(>>=) :: [b] -> (b -> [a]) -> [a]
xs >>= f = concatMap f xs
```

可以愉快的证明了：

```haskell
  (f >=> g) >=> h
= (\x -> concat (fmap g (f x))) >=> h
= \y -> concat (fmap h ((\x -> concat (fmap g (f x))) y))
= \y -> concat (fmap h (concat (fmap g (f y))))
= \y -> concatMap h (concatMap g (f y))
= \y -> concatMap h ((f y) >>= g)
= \y -> ((f y) >>= g) >>= h
```

另一半：

```haskell
  f >=> (g >=> h)
= f >=> (\x -> concat (fmap h (g x)))
= \y -> concat (fmap (\x -> concat (fmap h (g x))) (f y))
= \y -> concatMap (\x -> concatMap h (g x)) (f y)
= \y -> (f y) >>= (\x -> concatMap h (g x))
= \y -> (f y) >>= (\x -> (g x) >>= h)
```

于是问题变成了如何证明

`(m >>= f) >>= g == m >>= (\x -> f x >>= g)` — （1）

这里使用归纳法进行证明（感谢[@魔理沙](https://www.zhihu.com/people/marisa.moe/activities)的帮助，解法也由她给出）：

```haskell
--[]
  [] >>= f >>= g
= [] >>= g
= []
= []
--(l:r)
  (l:r) >>= f >>= g
= (f l ++ (r >>= f)) >>= g
= (f l >>= g) ++ ((r >>= f) >>= g)
= (f l >>= g) ++ (r >>= (\x -> f x >>= g)) -- inductive hypothesis
= (l:r) >>= (\x -> f x >>= g)
```

于是我们证明了结合律。

# Monad Laws

至此，我们证明了以下公式：

```haskell
f >=> return ≡ f ≡ return >=> f
(f >=> g) >=> h ≡ f >=> (g >=> h)
```

我们对第一条进行一定的转换：

```haskell
-- handle >=>
f >=> g ≡ \x -> f x >>= g
-- left
  f >=> return ≡ f
= \x -> f x >>= return ≡ f
= f x >>= return ≡ f x
= m >>= return ≡ m where m = f x -- (2)
-- right
  return >=> f ≡ f
= \x -> return x >>= f ≡ f
= return x >>= f ≡ f x  -- (3)
```

公式（2），（3）和前一节中转换的（1）一起，构成了传说中的Monad laws。

Monad laws规定了haskell中monad一定要满足的三个条件：

```haskell
-- law 1
return a >>= f ≡ f a
-- law 2
m >>= return ≡ m
-- law 3
(m >>= f) >>= g ≡ m >>= (\x -> f x >>= g)
```

这也说明了，monad laws其实约束的，就是monad的结合性和单位元，换句话说，保证了每个monad instance属于这个半幺群。因此，每个monad都要定义`>>=`和`return`这两个自然变换。

***

这样一来，我们终于明白了为什么monad是“自函子范畴上的一个幺半群”。