---
title: 实现温度曲线过程中的思考
date: 2016-07-15T15:17:16.000Z
tags:
  - Front-End
  - JavaScript
---
![cover](http://oanr6klwj.bkt.clouddn.com/blog/temperature-curve-cover.png)
> [*pixiv-ID: 50140585*](http://www.pixiv.net/member_illust.php?mode=medium&illust_id=50140585)

上周接到的需求是实现QQ浏览器中的一个温度曲线。具体来说就是传入一组数据，数据包括6天的当日最高温度和最低温度，由此拟合出一条曲线。

曲线使用canvas进行绘制，因此有原生的贝塞尔曲线api可以使用。我们应该最大化的利用这个先天优势，避免过多的数学计算。

<!--more-->

在这个需求中，有两个很有趣的点：

- 如果出现天气变化很小的情况：比如前四天气温都为22°~23°，第五天的变为23°~24°，那么曲线中这一变化是否应该表现的非常明显？
- 给定的温度是一个区间，那么如何在这个区间中用一条**平滑**曲线较为准确的描绘出变化？

需求的设计稿如下：
![design curve](http://oanr6klwj.bkt.clouddn.com/blog/temperature-curve-design.png)

经过思考，个人认为处理上述两个问题的办法是:

> - 如果温度变化情况很小，那么曲线的变化也不宜过大，以免给长期用户造成困扰。
> - 这条曲线不采用描点-连线-折线转曲线的方式来描绘，也不采用点拟合曲线的方式来进行，理由如下：

>   - 描点-连线-折线转曲线面对的一个很棘手的问题就是对于曲线曲度的把握，而且需要进行的向量计算可能会对性能产生一些影响。
>   - 点拟合曲线是一个稳妥的方式，但是需要面对的问题是超大的计算量。而且对于气温变化大的城市很可能出现曲线偏离其中一点很远的情况。

> - 综上所述，我采取了一种折线拟合曲线的方式，这个方式非常有趣，灵感来自[这里](https://github.com/hongru/Canvas-Tattle/issues/19)

> - 但是这个办法有一个很大的问题就是：书写上下温度值的时候如何确定在线上的点的Y坐标，我们需要一个算法去解决它。

因此代码核心分为三部分：

1. 第一部分是如何让曲线变化根据温差大小进行适度的变化，温差大变化大，温差小变化小。且能够容纳足够的温度变化区间。
2. 第二部分是如何把折线拟合为平滑曲线。
3. 第三部分是确立算法求解特定点的Y坐标。

******

处理第一部分的办法是根据给定的数据计算出一个参考区间，然后根据这个参考区间来描绘曲线。例如，现在的温度数据是上图设计图所示的温度数据，那么理想的温度区间就是28~10。因此，首先要**计算合理温度区间**。其次，区间计算出来之后，我们在canvas上的坐标和温度在这个区间中的位置是成比例的，因此要**计算温度点对应canvas画布中的位置**

```javascript
var cfg = {
  canvaswidth: 660, //画布宽度
  canvasheight: 228,  //画布高度
  blankx: 0, //曲线横向留白
  blankt: 30, //文字横向留白
  blanky: 35  //曲线纵向留白
};
var setting = {
  linewidth: 1, //曲线粗细
  linecolor: '#fff', //曲线颜色
  base: 5, //区间基准值
  textsize: 24, //文字大小
  textoffset: 34, //文字与曲线距离
  textcolor: '#fff', //文字颜色
  font: 'arial' //文字字体
}
var getdata = function(data) {
  var _getdata = [],
      _avarage;
  for (var _i = 0; _i < data.length; _i++) {
    _avarage = (data[_i][0] + data[_i][1]) / 2;
    _getdata.push(_avarage);
  }
  return _getdata;
}
var choseRank = function(getData) {
  var _max = -Infinity,
      _min = Infinity,
      _res,
      _cur,
      _rank;
  for (var _i = 0; _i < getData.length; _i++) {
    _cur = getData[_i];
    if (_cur > _max) {
      _max = _cur;
    }
    if (_cur < _min) {
      _min = _cur;
    }
  }
  _res = _max - _min;
  _rank = Math.round(_res / setting.base) + 1;
  return _rank * setting.base;
}
```

处理第二部分的办法就来自于上述那个链接中的灵感。假设我们有点A、B、C、D、E。那么我们以A为起点，B、C的中点为终点，B为参考点做贝塞尔曲线;然后接着以C、D的中点为终点，C为参考点做贝塞尔曲线，以此类推。
```javascript
var draw_MAIN = function(canvas, data) {
  var _dataGot = getData(data);
  var _rank = choseRank(_dataGot);
  var _totalPoint = _dataGot.length;
  var _sum = getSum(_dataGot);
  var _avr = Math.round(_sum / _totalPoint);
  var _low = _avr - _rank / 2,
      _high = _avr + _rank / 2,
      _interval = (cfg.canvasWidth - cfg.blankX * 2) / (_totalPoint - 1),
      _intervalT = (cfg.canvasWidth - cfg.blankT * 2) / (_totalPoint - 1),
      _curSupport,
      _cur,
      _next;
  var _curPoint = cfg.blankX;
  var getY = function(i) {
    return cfg.canvasHeight - cfg.blankY - (_dataGot[i] - _low) *
           (cfg.canvasHeight - cfg.blankY * 2) / _rank;
  }
  if (canvas.getContext) {
    var _ctx = canvas.getContext('2d');
    _ctx.imageSmoothingEnabled = true;
    _ctx.strokeStyle = setting.lineColor;
    _ctx.lineWidth = setting.lineWidth;
    _ctx.lineCap = 'round';
    _ctx.beginPath();
    _ctx.moveTo(_curPoint, getY(0));
    for (var _i = 0; _i < _dataGot.length - 1; _i++) {
      _cur = getY(_i + 1);
      _next = getY(_i + 2);
      _curSupport = getSupportPoint(_cur, _next);
      if (_i < _dataGot.length - 2) {
        _ctx.quadraticCurveTo(_curPoint + _interval, _cur,
          _curPoint + _interval * 1.5, _curSupport);
      } else if (_i >= _dataGot.length - 2) {
        _ctx.quadraticCurveTo(_curPoint, getY(_i),
          _curPoint + _interval, _cur);
      }
      _curPoint += _interval;
      _ctx.stroke();
    }
  }
}
```

这时我们需要一个算法来计算出bezier-curve上特定点的坐标，canvas没有给我们提供相应的api，需要我们**根据bezier函数**自己来完成，这里我放上两个参考链接

> [贝塞尔曲线初探](http://www.cnblogs.com/jay-dong/archive/2012/09/26/2704188.html)
> [stackoverflow上的问题](http://stackoverflow.com/questions/14174252/how-to-find-out-y-coordinate-of-specific-point-in-bezier-curve-in-canvas)

最后得出的函数为：

```javascript
var getYValues = function(start, control, end, X) {
  var _q1,
      _q2,
      _q3,
      _t,
      _y;
  _q1 = 2 * start.x - 2 * control.x;
  _q2 = Math.sqrt(5 * start.x * start.x - 10 * start.x * control.x +
    start.x * end.x - start.x * X + 2 * control.x * X +
    4 * control.x * control.x );
  _q3 = 2 * start.x - 4 * control.x + 2 * end.x;
  if (_q3 !== 0) {
    _t = (_q1 + _q2) / _q3;
  } else {
    _t = ( X - start.x) / (2*control.x - 2*start.x);
  }
  if (_t < 0) {
    _t = -1 * _t;
  }
  _y = start.y * (1 - _t) * (1 - _t) + control.y * 2 * (1 - _t)
    * _t + end.y * _t * _t;
  return _y;
}
```

需要注意的是canvas绘制曲线会产生锯齿，除了开启浏览器抗锯齿以外，按照二倍图来绘制然后再缩小也是行之有效的方法，这里给出一个笨办法：
```javascript
var setScale = function(num, cfg, setting) {
  cfg.canvasWidth *= num;
  cfg.canvasHeight *= num;
  cfg.blankX *= num;
  cfg.blankT *= num;
  cfg.blankY *= num;
  setting.lineWidth *= num;
  setting.base *= num;
  setting.textSize *= num;
  setting.textOffSet *= num;
}
```

******
所以最终代码为:
```javascript
function drawTemperature(canvas, data) {
  var cfg = {
    canvasWidth: 660, //画布宽度
    canvasHeight: 228,  //画布高度
    blankX: 0, //曲线横向留白
    blankT: 30, //文字横向留白
    blankY: 35  //曲线纵向留白
  };
  var setting = {
    lineWidth: 1, //曲线粗细
    lineColor: '#fff', //曲线颜色
    base: 5, //区间基准值
    textSize: 24, //文字大小
    textOffSet: 34, //文字与曲线距离
    textColor: '#fff', //文字颜色
    font: 'arial' //文字字体
  }
  var getData = function(data) {
    var _getData = [],
        _avarage;
    for (var _i = 0; _i < data.length; _i++) {
      _avarage = (data[_i][0] + data[_i][1]) / 2;
      _getData.push(_avarage);
    }
    return _getData;
  }
  var choseRank = function(getData) {
    var _max = -Infinity,
        _min = Infinity,
        _res,
        _cur,
        _rank;
    for (var _i = 0; _i < getData.length; _i++) {
      _cur = getData[_i];
      if (_cur > _max) {
        _max = _cur;
      }
      if (_cur < _min) {
        _min = _cur;
      }
    }
    _res = _max - _min;
    _rank = Math.round(_res / setting.base) + 1;
    return _rank * setting.base;
  }
  var getYValues = function(start, control, end, X) {
    var _q1,
        _q2,
        _q3,
        _t,
        _y;
    _q1 = 2 * start.x - 2 * control.x;
    _q2 = Math.sqrt(5 * start.x * start.x - 10 * start.x * control.x +
      start.x * end.x - start.x * X + 2 * control.x * X +
      4 * control.x * control.x );
    _q3 = 2 * start.x - 4 * control.x + 2 * end.x;
    if (_q3 !== 0) {
      _t = (_q1 + _q2) / _q3;
    } else {
      _t = ( X - start.x) / (2*control.x - 2*start.x);
    }
    if (_t < 0) {
      _t = -1 * _t;
    }
    _y = start.y * (1 - _t) * (1 - _t) + control.y * 2 * (1 - _t)
      * _t + end.y * _t * _t;
    return _y;
  }
  var setScale = function(num, cfg, setting) {
    cfg.canvasWidth *= num;
    cfg.canvasHeight *= num;
    cfg.blankX *= num;
    cfg.blankT *= num;
    cfg.blankY *= num;
    setting.lineWidth *= num;
    setting.base *= num;
    setting.textSize *= num;
    setting.textOffSet *= num;
  }
  var getSum = function(getData) {
    var _sum = 0;
    for (var _i = 0; _i < getData.length; _i++) {
      _sum = _sum + getData[_i];
    }
    return _sum;
  }
  var getSupportPoint = function(prev, next) {
    return (prev + next) / 2;
  }
  var draw_MAIN = function(canvas, data) {
    setScale(2, cfg, setting);
    var _dataGot = getData(data);
    var _rank = choseRank(_dataGot);
    var _totalPoint = _dataGot.length;
    var _sum = getSum(_dataGot);
    var _avr = Math.round(_sum / _totalPoint);
    var _low = _avr - _rank / 2,
        _high = _avr + _rank / 2,
        _interval = (cfg.canvasWidth - cfg.blankX * 2) / (_totalPoint - 1),
        _intervalT = (cfg.canvasWidth - cfg.blankT * 2) / (_totalPoint - 1),
        _start = {},
        _control = {},
        _end = {},
        _Y,
        _curSupport,
        _cur,
        _next;
    var _curPoint = cfg.blankX;
    var _curPointT = cfg.blankT;
    var getY = function(i) {
      return cfg.canvasHeight - cfg.blankY - (_dataGot[i] - _low) *
             (cfg.canvasHeight - cfg.blankY * 2) / _rank;
    }
    if (canvas.getContext) {
      var _ctx = canvas.getContext('2d');
      _ctx.imageSmoothingEnabled = true;
      _ctx.strokeStyle = setting.lineColor;
      _ctx.lineWidth = setting.lineWidth;
      _ctx.lineCap = 'round';
      _ctx.font = setting.textSize + 'px ' + setting.font;
      _ctx.fillStyle = setting.textColor;
      _ctx.textAlign = 'center';
      _ctx.textBaseline = 'middle';
      _ctx.fillText(data[0][1] + "°", _curPointT,
        getY(0) - setting.textOffSet);
      _ctx.fillText(data[0][0] + "°", _curPointT,
        getY(0) + setting.textOffSet);
      _ctx.beginPath();
      _ctx.moveTo(_curPoint, getY(0));
      for (var _i = 0; _i < _dataGot.length - 1; _i++) {
        _cur = getY(_i + 1);
        _next = getY(_i + 2);
        _curSupport = getSupportPoint(_cur, _next);
        if (_i === 0) {
          _start.x = _curPoint;
          _start.y = getY(0);
          _control.x = _curPoint + _interval;
          _control.y = _cur;
          _end.x = _curPoint + _interval * 1.5;
          _end.y = _curSupport;
          _Y = getYValues(_start, _control, _end, _curPoint + _interval);
        } else if (_i < _dataGot.length - 2) {
          _start.x = _curPoint + _interval * 0.5;
          _start.y = getSupportPoint(getY(_i), getY(_i + 1));
          _control.x = _curPoint + _interval;
          _control.y = _cur;
          _end.x = _curPoint + _interval * 1.5;
          _end.y = _curSupport;
          _Y = getYValues(_start, _control, _end, _control.x);
        } else if (_i >= _dataGot.length - 2) {
          _Y = _cur;
        }
        if (_i < _dataGot.length - 2) {
          _ctx.quadraticCurveTo(_curPoint + _interval, _cur,
            _curPoint + _interval * 1.5, _curSupport);
        } else if (_i >= _dataGot.length - 2) {
          _ctx.quadraticCurveTo(_curPoint, getY(_i),
            _curPoint + _interval, _cur);
        }
        _curPoint += _interval;
        _ctx.stroke();
        _curPointT += _intervalT;
        _ctx.fillText(data[_i + 1][1] + "°", _curPointT,
          _Y - setting.textOffSet);
        _ctx.fillText(data[_i + 1][0] + "°", _curPointT,
          _Y + setting.textOffSet);
      }
    }
  }
  draw_MAIN(canvas,data);
}
```

