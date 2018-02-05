title: better-scroll
speaker: zhangjun
url: https://github.com/greenzhang/better-scroll-slide
transition: cards
files: /js/demo.js,/css/demo.css

[slide]

# better-scroll源码分享

[slide]

# 背景 {:&.flexbox.vleft}
## 移动端滚动条的痛点

[slide style="background-image:url('/img/bg1.png')"]
## 元标签
----
```javascript
<meta name="viewport" content="width=device-width, 
initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0" >

width：控制 viewport 的大小，可以指定的一个值，如果 600，或者特殊的值，
如 device-width 为设备的宽度（单位为缩放为 100% 时的 CSS 的像素）。
height：和 width 相对应，指定高度。
initial-scale：初始缩放比例，也即是当页面第一次 load 的时候缩放比例。
maximum-scale：允许用户缩放到的最大比例。
minimum-scale：允许用户缩放到的最小比例。
user-scalable：用户是否可以手动缩放
```

[slide]
## viewport的概念
- 1 layout viewport
- 2 visual viewport

[slide]
----

## Demo
![qrcode](../imgs/1517794336.png 'qrcode')


[slide]

* 没有滚动条
* 滚动不流畅

[slide]

**需要解决方案！**

[slide]
![better scroll](../imgs/better-scroll.png 'better scroll')

[slide]
```
| |—— util 工具库
| |    |____ dom : 对DOM操作的库，例如获取兼容浏览器的css前缀，获取元素相对位置，节点插入等
| |    |____ ease : 贝塞尔曲线
| |    |____ env : 运行环境
| |    |____ index : 导出当前util的所有库
| |    |____ lang : 拷贝函数及高精度时间
| |    |____ momentum : 滚动动量/回弹动画的库
| |    |____ debug : 控制台告警
| |    |____ raf : requestAnimateFrame函数
| |    |____ const : 定义的常量
| |
| |—— scroll : 核心功能
| |    |____ core : 处理滚动三大事件和处理滚动动画相关的函数。
| |    |____ event : 事件监听
| |    |____ init : 初始化：参数配置、DOM 绑定事件、调用其他扩展插件
| |    |____ pulldown : 下拉刷新功能
| |    |____ pullup : 上拉加载更多功能
| |    |____ scrollbar : 生成滚动条、滚动时实时计算大小与位置
| |    |____ snap : 扩展的轮播图及一系列操作
| |    |____ wheel : 扩展的picker插件操作
```

[slide]

* init模块
* core模块

[slide]
原型链扩展
```javascript

 BScroll.prototype._transitionEnd = ...

import { initMixin } from './scroll/event'
.
.
.
```

最后再统一合并并 export 出来

```js
initMixin(BScroll)
.
.
.

export default BScroll
```

[slide]
init模块

[slide]

```JavaScript
BScroll.prototype._init = function(el, options) {
    this._handleOptions(options) //处理配置参数

    // init private custom events
    this._events = {} //初始化自定义事件缓存，自定义事件的实现在src/scroll/event.js中
    //scroll 横轴坐标。
    this.x = 0 
    //scroll 纵轴坐标。
    this.y = 0 
    //判断 scroll 滑动结束后相对于开始滑动位置的方向（左右）。-1 表示从左向右滑，1 表示从右向左滑，0 表示没有滑动。
    this.directionX = 0 
    //判断 scroll 滑动结束后相对于开始滑动位置的方向（上下）。-1 表示从左向下滑，1 表示从右向上滑，0 表示没有滑动。
    this.directionY = 0 
    //添加原生事件监听
    this._addDOMEvents() 
    //根据配置参数初始化betterScroll的额外功能
    this._initExtFeatures() 
    //监听是否处于过渡状态
    this._watchTransition() 
    //会检测 scroller 内部 DOM 变化，自动调用 refresh 方法重新计算来保证滚动的正确性。
    if (this.options.observeDOM) {
        this._initDOMObserver() 
    }
    // 重新计算容器和可滚动元素（this.scroller)宽高，可滚动最大距离，并重置滚动元素的位置
    this.refresh()
    //不关心
    if (!this.options.snap) {
        this.scrollTo(this.options.startX, this.options.startY) 
    }
    //开启滚动功能
    this.enable() 
}
```
[slide]

- 将用户传入的配置参数与默认参数一起进行合并配置
- 将初始化的自定义事件缓存,并定义了滚动过程中滚动元素的位置和方向(扩展x,y,directionX和directionY用于存放滚动过程中x轴y轴上的位置和方向)
- **对原生的事件进行监听(mouse,touch,resize,orientationchange)和监听是否处于过渡状态**
- 根据配置需求,初始化其他功能(如snap/wheel之类)
- **初始化observer**
- **refresh方法,当滚动组件内部dom变化的时候,重新计算滚动容器的高宽,滚动条的位置,并将滚动元素重置**
- 初始化滚动元素的位置

[slide]
- eventListener
- watchTransition 
- refresh

[slide]
事件监听

[slide]
```js
target.addEventListener('type', fn, false / true)
```
[slide]
handleEvent()

[slide]
tips:
- 如果事件监听没有被正确的 remove,可能会产生内存泄漏的问题
- 如果被绑定事件的DOM被移除了，也可能会产生内存泄漏问题

![qrcode](../imgs/1517796378.png 'qrcode')


[slide]

```html
<style>
    #container {
        min-height: 60px;
    }

    #target {
        padding: 5px;
        border: 1px solid green;
    }
</style>
<div id="container">
    <span id="target">这是目标 span</span>
</div>
<button type="button" id="remove">删除</button>
<button type="button" id="recover">恢复</button>
<script>
    const container = document.querySelector("#container");
    const target = document.querySelector("#target");
    target.addEventListener("click", () => {
       	console.log("the span is clicked");
    });

    document.querySelector("#remove").addEventListener("click", () => {
        alert("removed");
        target.remove();
    });

    document.querySelector("#recover").addEventListener("click", () => {
        alert("recovered");
        container.appendChild(target);
    });
</script>
```
[slide]
watch isInTransition

[slide]

```JavaScript
BScroll.prototype._watchTransition = function() {
    if (typeof Object.defineProperty !== 'function') {
        return
    }
    let me = this
    let isInTransition = false
    Object.defineProperty(this, 'isInTransition', {
        get() {
            return isInTransition
        },
        set(newVal) {
            isInTransition = newVal
            // fix issue #359
            let el = me.scroller.children.length
                ? me.scroller.children
                : [me.scroller]
            let pointerEvents = isInTransition && !me.pulling ? 'none' : 'auto'
            for (let i = 0; i < el.length; i++) {
                el[i].style.pointerEvents = pointerEvents
            }
        }
    })
}
```

[slide]
Object.defineprototy

[slide]
refresh()

[slide]
获取容器和滚动元素的宽高，计算最大滚动距离。

当DOM结构变化后应该调用重新计算
（比如在scroll container中异步的加载DOM或者插入新的DOM）

[slide]
![scroll-position](../imgs/scroll_position.png 'scroll-position')

[slide]
core模块

[slide]
- scrollTo
- start
- move
- end
[slide]
```javascript
  BScroll.prototype.scrollTo = function (x, y, time = 0, easing = ease.bounce) {
    // 设置isInTransition
    this.isInTransition = this.options.useTransition && time > 0 && (x !== this.x || y !== this.y)

    // 配置是否使用css3动画transitiond的标记 如果设置中使用transiation并且过渡时间为0
    if (!time || this.options.useTransition) {
      // 设置需要进行css transition的css样式
      this._transitionProperty()
      // 过渡的计算方法,比如linear，ease，cubic-bezier(0.1, 0.7, 1.0, 0.1),steps(4, end)等等 各种贝塞尔曲线等
      this._transitionTimingFunction(easing.style)
      // 修改当前滚动对象的动画运行时间
      this._transitionTime(time)
      // 设置transform：translate，这是better-scroll实现滑动的原理
      // 通过子元素不断改变transform:translate，来达到浏览器滚动的效果
      this._translate(x, y)

      // 如果probetype设置为3 则使用requestAnimateFrame在浏览器每次刷新时触发scroll事件，然后在回调中判断过渡状态是否结束
      if (time && this.options.probeType === 3) {
        this._startProbe()
      }

      if (this.options.wheel) {
        if (y > 0) {
          this.selectedIndex = 0
        } else if (y < this.maxScrollY) {
          this.selectedIndex = this.items.length - 1
        } else {
          this.selectedIndex = Math.round(Math.abs(y / this.itemHeight))
        }
      }
    } else {
      this._animate(x, y, time, easing.fn)
    }
  }
```
[slide]

scrollTo中的核心方法_translate
```javascript
BScroll.prototype._translate = function(x, y) {
    if (this.options.useTransform) {
        // this.translateZ是从硬件加速那儿来的
        this.scrollerStyle[style.transform] = `translate(${x}px,${y}px)${
            this.translateZ
        }`
    } else {
      ...
    }
    //wheel相关 不关心
    if (this.options.wheel) {
       ...
    }

    // 这里设置了this.x, this.y
    this.x = x
    this.y = y

    if (this.indicators) {
        for (let i = 0; i < this.indicators.length; i++) {
            this.indicators[i].updatePosition()
        }
    }
}
```
通过设置transform实现元素位置的变化

[slide]
start函数

- 判断 PC 端只允许左键点击
- 判断是否可操作、是否未销毁,没有就直接跳出函数
- 设置 transition 运动时间，不传参数默认为0
- 若上一次滚动还在继续时，此时触发了_start，则停止到当前滚动位置
- BS还提供了beforeScrollStart事件,方便和外面做交互
[slide]
```javascript
BScroll.prototype._start = function(e) {
    /************ 1、判断 PC 端只允许左键点击 不是鼠标左键，直接退出 ***************/
    let _eventType = eventType[e.type]
    if (_eventType !== TOUCH_EVENT) {
        ...
    }
    /************ 2、判断是否可操作、是否未销毁,没有就直接跳出函数 ***************/
    if (!this.enabled ||this.destroyed ||(this.initiated && this.initiated !== _eventType)) {
        return
    }
    this.initiated = _eventType

    if (this.options.preventDefault &&!preventDefaultException(e.target, this.options.preventDefaultException)) {
        e.preventDefault() //阻止浏览器默认行为
    }
    //初始化数据
    this.moved = false
    // 记录开始位置到结束位置之间的距离  手指或鼠标的滑动距离
    this.distX = 0
    this.distY = 0
    this.directionX = 0 // 滑动方向（左右） -1 表示从左向右滑，1 表示从右向左滑，0 表示没有滑动。
    this.directionY = 0 // 滑动方向（上下） -1 表示从上往下滑，1 表示从下往上滑，0 表示没有滑动。
    this.movingDirectionX = 0 // 滑动过程中的方向(左右) -1 表示从左向右滑，1 表示从右向左滑，0 表示没有滑动。
    this.movingDirectionY = 0 // scroll 滑动过程中的方向(上下）-1 表示从上往下滑，1 表示从下往上滑，0 表示没有滑动。
    this.directionLocked = 0 //
    /************ 3、设置 transition 运动时间，不传参数默认为0 ***************/
    this._transitionTime() // 初始化过渡时间为0
    this.startTime = getNow() // 初始化开始时间
    ...
    /************ 4、若上一次滚动还在继续时，此时触发了_start，则停止到当前滚动位置 ***************/
    this.stop() // 如果此时处于惯性滚动过程中，停止惯性滚动。
    // 略
    ... 
    /************ 5、BS还提供了beforeScrollStart事件,方便和外面做交互 ***************/
    this.trigger('beforeScrollStart')
}
```
[slide]
```javascript
BScroll.prototype.stop = function() {
    // useTransition表示使用css3的transition
    //  isInTransition表示正处在动画过程中
    if (this.options.useTransition && this.isInTransition) {
        this.isInTransition = false
        // getComputedPosition返回的是transform的属性值
        let pos = this.getComputedPosition()
        this._translate(pos.x, pos.y)
        if (this.options.wheel) {
            this.target = this.items[Math.round(-pos.y / this.x)]
        } else {
            this.trigger('scrollEnd', {
                x: this.x,
                y: this.y
            })
        }
        this.stopFromTranstion = true
        // 使用的不是css3的transition isAnimating是用js判断是否在滚动过程中
    } else if (!this.options.useTransition && this.isAnimating) {
        this.isAnimating = false
        this.trigger('scrollEnd', {
            x: this.x,
            y: this.y
        })
        this.stopFromTranstion = true
    }
}
```

[slide]
非常高端的从矩阵中计算出当前位置的函数... 看看就好
```javascript
  BScroll.prototype.getComputedPosition = function () {
    var matrix = window.getComputedStyle(this.scroller, null);
    var x = void 0;
    var y = void 0;

    if (this.options.useTransform) {
      matrix = matrix[style.transform].split(')')[0].split(', ');
      x = +(matrix[12] || matrix[4]);
      y = +(matrix[13] || matrix[5]);
    } else {
      x = +matrix.left.replace(/[^-\d.]/g, '');
      y = +matrix.top.replace(/[^-\d.]/g, '');
    }

    return {
      x: x,
      y: y
    };
  };
```
[slide]
move函数

- 计算鼠标或者手指的滑动距离
- 根据横向纵向滑动距离锁定一个滚动方向（前提 freeScroll === false)
- 根据滑动距离计算滚动距离，并执行滚动
- 派发"scroll"事件
- 判断手指滑动到视窗(visual viewport)边缘，执行_end()

[slide]
代码太长不看
[slide]
end函数
- 如果滚动超出边界，重置滚动位置，并退出函数
- 如果是一次轻拂操作，退出函数
- 计算整个滑动过程的时间和距离
- 判断是否满足惯性滚动条件，并计算惯性滚动距离和时间
- 执行惯性滚动，并退出
[slide]
代码就看一点点
[slide]
```javascript
 // 当快速在屏幕上滑动一段距离的时候，会根据滑动的距离和时间计算出动量，并生成滚动动画
    // 因为用户可能在某一个瞬间滚动一小段距离，所以这个时候应该是app native那种滚动一大段的动画，而不是再根据手指的位移来计算this.scroller的位置了
    if (
        this.options.momentum &&
        duration < this.options.momentum &&
        (absDistY > this.options.momentumLimitDistance ||
            absDistX > this.options.momentumLimitDistance)
    ) {
        // bounce: 当滚动超过边缘的时候会有一小段回弹动画。设置为 true 则开启动画
        // momentum return {destination, duration}
        let momentumX = this.hasHorizontalScroll
            ? momentum(
                  this.x,
                  this.startX,
                  duration,
                  this.maxScrollX,
                  this.options.bounce ? this.wrapperWidth : 0,
                  this.options
              )
            : { destination: newX, duration: 0 }
        let momentumY = this.hasVerticalScroll
            ? momentum(
                  this.y,
                  this.startY,
                  duration,
                  this.maxScrollY,
                  this.options.bounce ? this.wrapperHeight : 0,
                  this.options
              )
            : { destination: newY, duration: 0 }
        newX = momentumX.destination
        newY = momentumY.destination
        time = Math.max(momentumX.duration, momentumY.duration)
        this.isInTransition = true
    } else {
        if (this.options.wheel) {
            newY = Math.round(newY / this.itemHeight) * this.itemHeight
            time = this.options.wheel.adjustTime || 400
        }
    }
```
[slide]
最后scrollEnd这个事件是通过监听transitionend来解决的
[slide]
```javascript
  BScroll.prototype._transitionEnd = function (e) {
    if (e.target !== this.scroller || !this.isInTransition) {
      return
    }

    this._transitionTime() //默认设置过渡时间0
    if (!this.pulling && !this.resetPosition(this.options.bounceTime, ease.bounce)) {
      this.isInTransition = false
      this.trigger('scrollEnd', {
        x: this.x,
        y: this.y
      })
    }
  }
```
[slide]


```javascript
  BScroll.prototype.resetPosition = function () {
    var time = arguments.length > 0 && arguments[0] !== undefined ? arguments[0] : 0;
    var easeing = arguments.length > 1 && arguments[1] !== undefined ? arguments[1] : ease.bounce;

    var x = this.x;
    var roundX = Math.round(x);
    if (!this.hasHorizontalScroll || roundX > 0) {
      x = 0;
    } else if (roundX < this.maxScrollX) {
      x = this.maxScrollX;
    }

    var y = this.y;
    var roundY = Math.round(y);
    if (!this.hasVerticalScroll || roundY > 0) {
      y = 0;
    } else if (roundY < this.maxScrollY) {
      y = this.maxScrollY;
    }

    if (x === this.x && y === this.y) {
      return false;
    }

    this.scrollTo(x, y, time, easeing);

    return true;
  };
```
- dis>0，向下或者向右滚动超过边界值，会重置为0。
- dis<maxScrollX/Y，向上或者向左滚动超出边界值，会重置为maxScrollX/Y。