---
title: 九宫格中的奇技淫巧
date: 2017-08-02 17:18:46
tags:
  - CSS
  - Front-End
---
![cover](http://oanr6klwj.bkt.clouddn.com/blog/3x3-grid-cover.jpg)
> [*pixiv-ID: 54260293*](https://www.pixiv.net/member_illust.php?mode=medium&illust_id=54260293)

九宫格是一个比较常见的布局，从微博到QQ空间到朋友圈都离不开它。实现九宫格的文章也是一抓一大把了，但是九宫格有一个有趣的细节：当图片是4张的时候，图片呈“田字格”排列（2X2），关于这一小细节的实现，常见的做法是为4图时专门写一个布局存在一个单独的className中，在JS中检测图片的张数为4张时替换为刚才的className，那么，有没有办法使用纯CSS搞定呢？

<!--more-->

今天我们就从头开始实现一个九宫格布局。

# 关注点

我们需要关注以下细节：

1. 排列方式：在图片为4张时，2X2排列，其它情况均按九宫格从左至右排列。
2. 宽高缩放：图片应该通过容器的`background-image`属性来加载，表现为正方形，每张图片占据父容器宽度的1/3。
3. 图片间有固定宽度（例如3px）的缝隙。



# 基本排列

面对这种等比例布局时，我们选择`flex`布局，简单方便。

假设我们的HTML结构为：

```HTML
<section class="pic-list">
  <div class="pic-list-item" style="background-image:url(your/pic/address)"></div>
  <div class="pic-list-item" style="background-image:url(your/pic/address)"></div>
  <!--...(小于九个)-->
</section>
```

那么对应的flex布局的CSS为：

```css
div, body {
  margin: 0;
  padding: 0;
}
.pic-list {
  display: flex;
  flex-wrap: wrap;
}
.pic-list-item {
  flex: 0 0 33%;
}
```



# 宽高缩放

图片是按照背景图片的形式加载的，所以我们的`.pic-list-item`中是没有任何元素可以撑起div的高度，更别提自适应了，这时应该怎么办？

通过使用为`padding-top`或`padding-bottom`设置百分比的形式我们可以让一个元素的宽度自适应，百分比的比值是以**父元素的宽度**为标准的，例如在九宫格中，如果想让子元素呈正方形，只要设置`padding-top: 33%`就可以了。

因此我们修改`.pic-list-item`为：

```CSS
.pic-list-item {
  flex: 0 0 33%;
  padding-top: 33%;
  background-repeat: no-repeat;
  background-size: cover;
  background-position: center;
}
```

这样，我们就做出了一个基本的宽高自适应的九宫格了。

# 2X2的处理

要想处理2X2，就要**筛选出有且仅有4张图片的情况**。

在CSS中有一对选择器可以处理这个情况：`:first-child`和`:nth-last-child()`。

看起来很绕，让我们慢慢分析。例如：`.pic-list-item:first-child:nth-last-child(3)`，`:first-child`选择了第一个`.pic-list-item`元素，`:nth-last-child(3)`选择了属于倒数第三个`.pic-list-item`的元素，这两个加在一起，就是选择了**即使倒数第三个也是第一个`.pic-list-item`**的元素。

接下来我们只需要搭配一个`A~B`选择器，就可以选择同级下A元素之后的所有B元素了，所以代码是：

```css
.pic-list-item:first-child:nth-last-child(4),
.pic-list-item:first-child:nth-last-child(4) ~ .pic-list-item {
  /*some style*/
}
```

这样我们就选择了有且仅有4张图片时所有的`.pic-list-item`了。

此时我们只要选中这种情况下的第二个元素，给它加一个足以把第三个元素挤到第二排的margin即可，因此代码为：

```CSS
.pic-list-item:first-child:nth-last-child(4) ~ .pic-list-item:nth-child(2) {
  margin-right: 33%;
}
```

如果，你使用Scss，那就更简单了：

```SCSS
.pic-list-item:first-child:nth-last-child(4) ~ .pic-list-item {
  &:nth-child(2) {
    margin-right: 33%;
  }
}
```

# 图片间距

增加间距的常见方法是使用`margin`，但是使用margin毫无疑问会增加我们的工作量，要专门计算增加的总数，然后在父容器中做相应的调整。

使用`box-sizing:border-box`配合`border`，就可以轻松解决间距问题，也免去了我们计算的麻烦。

```CSS
.pic-list-item {
  ...
  background-clip: padding-box; /*将背景图片的边距限制在padding外边届内（即不填充border）*/
  box-sizing: border-box; /*开启border-box，将marin，padding，border算入height和width*/
  border: 1.5px solid transparent;
}
```



然后在父元素中减去多出的1.5px即可

```CSS
.pic-list {
  ...
  margin: -1.5px;
}
```

# 最终代码：

```CSS
div, body {
  margin: 0;
  padding: 0;
}
.pic-list {
  display: flex;
  flex-wrap: wrap;
  margin: -1.5px;
}
.pic-list-item {
  flex: 0 0 33%;
  padding-top: 33%;
  background-repeat: no-repeat;
  background-size: cover;
  background-position: center;
  background-clip: padding-box;
  box-sizing: border-box;
  border: 1.5px solid transparent;
}
.pic-list-item:first-child:nth-last-child(4) ~ .pic-list-item:nth-child(2) {
  margin-right: 33%;
}
```



我提供了一个[jsbin示例](http://jsbin.com/nojujaw/edit?html,css,output)，以供各位参考。