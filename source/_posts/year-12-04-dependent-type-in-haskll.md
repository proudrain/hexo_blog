---
title: 在Haskell中模拟dependent type
date: 2017-12-04 14:25:40
tags:
  - Haskell
  - FP
---

![cover](http://oanr6klwj.bkt.clouddn.com/blog/dependent-type-in-haskell.jpg)
> [*pixiv-ID: 61127101*](https://www.pixiv.net/member_illust.php?mode=medium&illust_id=61127101)

Dependent type是一种依赖于值的类型，它增强了类型表达能力，让我们可以构造更强大的类型定义，无论用在定理证明还是用在减少程序出错的可能都是极好的。

本文我们讨论在Haskell中如何合理使用扩展模拟Dependent type。

<!--more-->

# 入门级List

让我们先从一个简单的List开始，看看没有dependent type的时候，会发生什么。

先定义自然数和一个List结构：

```haskell
data Nat = Zero | Succ Nat
data List a = Nil | Cons a (List a)
-- example
one = Succ Zero -- 1
two = Succ $ Succ Zero -- 2
listOne :: List Nat
listOne = Cons Zero $ Nil -- [0]
listOne :: List Nat
listTwo = Cons (Succ Zero) $ Cons Zero Nil -- [1, 0]
```

此时让我们定义一个`index`函数，根据下标返回List中的元素：

```haskell
listIndex :: Nat -> List a -> a
listIndex Zero (Cons x _) = x
listIndex (Succ n) (Cons _ xs) = listIndex n xs
-- example
listIndex Zero listTwo => Succ Zero
listIndex one listTwo => Zero
```

此时会产生一个问题：

```haskell
listIndex Zero Nil -- 编译通过
listIndex two listOne -- 编译通过
```

可以看到，无论是去空数组中取值，还是下标超过了List的总长度这种明显是错误的代码，依然被编译了，一直到运行期才会产生异常。

# 升级版List—GADT

现在让我们在类型层面上解决这个问题。我们定先义一个新的List，让它有记录List长度的功能。

```haskell
data Vec a n = VNil a n | VCons a n
```

但是这个太弱了，几乎是换汤不换药：a和n都是可以由手动控制的，万一制造出了`VNil a (Succ Zero)`的类型，编译器只能默默接受。仔细观察，`VNil`和`VCons`都是函数，那么它们的类型其实可以写出：

```haskell
VNil :: Vec a Zero
VCons :: a -> Vec a n -> Vec a (Succ n)
```

所以我们需要一个叫[GADT](https://en.wikibooks.org/wiki/Haskell/GADT)的扩展来添加第一层限制，它可以让我们在类型定义中约束返回的类型。

``` haskell
{-# LANGUAGE GADTs #-}
data Zero
data Succ n
data Nat = Zero | Succ Nat
data Vec a n where
  Nil  :: Vec a Ze
  Cons :: a -> Vec a n -> Vec a (Succ n)
-- example
vecOne :: Vec Nat (Succ Zero)
vecOne = VCons Zero VNil -- [0]
vecTwo :: Vec Nat (Succ (Succ Zero))
vecTwo = VCons Zero $ VCons (Succ Zero) VNil -- [1, 0]
```

这样一来，我们可以通过类型直接约束List的长度了。你也许注意到了，我们为了在类型定义中添加约束而将`Nat`类型拆分了，这样看上去很蠢，其实我们可以借助一个叫做[DataKinds](https://downloads.haskell.org/~ghc/7.8.4/docs/html/users_guide/promotion.html)的扩展来让编译器自动帮我们做这件事：

```haskell
{-# LANGUAGE DataKinds #-}
data Nat = Zero | Succ Nat
-- 可以代替
data Zero
data Succ n
data Nat = Zero | Succ n
```

当然，此时编译器会提醒你在接下来的类型中使用`'Zero`和`'Succ`来指向那个type constructor来避免歧义：

```haskell
data Vec a n where
  VNil  :: Vec a 'Zero
  VCons :: a -> Vec a n -> Vec a ('Succ n)
```

我们还有最后一个不顺眼的地方：那就是`Vec`构造器的第二个参数`n`其实应该是`Nat`类型的，不可能是其它类型，所以我们希望也给它指定好类型，这时候我们就需要[KindSignatures](https://downloads.haskell.org/~ghc/7.8.1-rc1/docs/html/users_guide/kind-polymorphism.html)扩展了。

```haskell
data Vec :: * -> Nat -> * where
  VNil  :: Vec a 'Zero
  VCons :: a -> Vec a n -> Vec a ('Succ n)
```

OK，这样就得到了我们的升级版List，它能够通过类型直观的表现出它的长度。但是它还没有解决我们上一节的问题，那就是如何用类型约束index函数的参数：

```haskell
vecIndex :: Nat -> Vec x b -> x
vecIndex Zero (VCons x _) = x
vecIndex (Succ n) (VCons _ xs) = vecIndex n xs
```
问题依然存在着。


# 终极版List—Type Families

现在让我们把注意力集中到`vecIndex :: Nat -> Vec x b -> x`这里，既然我们已经用函数来表示出了类型，那么其实只要约束`Nat`小于 `b`就好了，这时我们需要[TypeFamilies](https://wiki.haskell.org/GHC/Type_families)扩展来让类型可以重载，这样我们就可以为类型定义一些二元运算了（要支持运算符的话还要添加一个[TypeOperators](https://downloads.haskell.org/~ghc/7.8.3/docs/html/users_guide/data-type-extensions.html)扩展）：

```haskell
type family ((a :: Nat) :< (b :: Nat)) :: Bool
type instance m :< 'Zero = 'False
type instance 'Zero :< 'Succ n = 'True
type instance ('Succ m) :< ('Succ n) = m :< n
```
现在还有一个问题，`Nat`和`b`是不能够比较的，因此我们需要创造一个能够比较的版本：

```haskell
data SNat a where
  SZero :: SNat 'Zero
  SSucc :: SNat a -> SNat ('Succ a)
```

现在我们可以重写我们的`vecIndex`了：

```haskell
vecIndex :: ((a :< b) ~ 'True) => SNat a -> Vec x b -> x
vecIndex SZero (VCons x _) = x
vecIndex (SSucc n) (VCons _ xs) = vecIndex n xs
-- example
vecIndex (SSucc SZero) vecTwo --成功
vecIndex (SSucc $ SSucc SZero) vecTwo --报错
```

非常安全。

***

现在你可以试试这道题目来自己体会一下了：

[codewars-Singletons](https://www.codewars.com/kata/54750ed320c64c64e20002e2)