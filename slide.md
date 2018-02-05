### scroll 插件出现的背景

#### viewport

当我们在写移动端页面的时候,经常会看到这样的一个元素信息

```javascript
<meta name="viewport" content="width=device-width, initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0" >
```

这是一个移动端专属的元标签,用于定义 viewport 的各种行为.在早期用户使用智能移动设备来访问网页的时候,由于屏幕太小这个原因,用户并不能得到很好的体验,所以 apple 最早引入了此项特性,用于解决移动端页面展示问题,随后安卓厂商大批跟进.
此项特性其实就是创建一个虚拟窗口(layout viewport),这个窗口接近桌面浏览器(980px),所以 pc 上的网页基本可以在手机上呈现,只是元素看上去很小,一般默认可以通过手动缩放网页.
而我们手机本身的浏览器就扮演了 visual viewport,可以放大缩小来看清细节
![layout viewport](./img/layout_viewport.jpg 'layout_viewport')
![visual viewport](./img/visual_viewport.jpg 'visual_viewport')

然后问题就来了, 因为我们的手机有一个叫 viewport 的东西,所以在布局的时候,fixed 属性就有问题了,因为 fixed 属性是相对于整个页面的固定位置,然而页面本身没有变化,只不过在 viewport 下变大了,然后我们移动的是 viewport, 而网页本身并没有滚动.所以我们移动 viewport 并不会让我们 fixed 属性跟着变化.所以最早 ios 和安卓都是不支持 fixed 属性的.
虽然这个问题后来被修复了,但实际上到了今天,fixed 属性都是有问题的,并不好用.
并且在早期,移动端浏览器是不支持非 body 元素滚动条的,当我们需要在局部的一个固定高度的容器内进行滚动内容是做不到.
如果想取巧头尾固定使用 overflow:auto 来实现滚动条,理论上是没问题的, 但是这个 overflow 属性在移动端依然有问题,下面就是一个小 demo

```html
<html xmlns="http://www.w3.org/1999/xhtml">
<head>
  <title></title>
  <meta name="viewport" content="width=device-width,initial-scale=1.0, minimum-scale=1.0, maximum-scale=1.0, user-scalable=no">
  <style type="text/css">
    body, div, dl, dt, dd, ul, ol, li, h1, h2, h3, h4, h5, h6, pre, code, form, fieldset, legend, input, textarea, p, blockquote, th, td, hr, button, article, aside, details, figcaption, figure, footer, header, hgroup, menu, nav, section { margin: 0; padding: 0; }
    body { font: normal 14px/1.5 "Arial" , "Lucida Grande" ,Verdana, "Microsoft YaHei" , "hei"; -webkit-font-smoothing: antialiased; color: #000; background: #f5f5f5; }
    header { position: absolute; top: 0; left: 0; width: 100%; height: 48px; background-color: #1491c5; }
    footer { position: absolute; bottom: 0; left: 0; width: 100%; height: 48px; background-color: #1491c5; }
    h1 { display: block; font-size: 2em; font-weight: bold; font-weight: 500; text-align: center; color: White; }
    #body { background: #fff; border: 1px solid #cfcfcf; width: 96%; height: 100%; margin: 50px auto; padding: 4px; }
  </style>
</head>
<body>
  <header id="header">
    <h1>
      Header</h1>
  </header>
  <div id="body">
    body
  </div>
  <footer>
    <h1>
      Footer</h1>
  </footer>
</body>
</html>
```

demo 中有如下几个问题

* 没有滚动条
* 滚动不流畅
更严重的问题是，滚动条的性能在低端安卓机器上非常差，在我实验的使用安卓4.4系统里，多数情况下是无法达到30fps，肉眼可见的卡顿直接说明了用户体验的糟糕(ios系统需要使用webkit-overflow-scrolling来让滚动更平滑)。
当目前的业务变得更加复杂的时候,比如上拉下拉刷新,模仿IOS的picker之类的需求被提出后,移动端急切的需要滚动条插件来解决这些奇怪的兼容性问题和满足业务的要求.这也是 scroll 插件诞生的背景.

### 源码分析

#### 使用须知

如果我们要使用 better-scroll 的滚动,那就要满足一个条件,即内容的高度必须  大于容器的高度,这样我们  才可以通过使用滚动条来看到超出的部分.
![better scroll](./img/better-scroll.png 'better scroll')

#### 源码结构

目录结构

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

可以看到 better-scroll 提供了相当多的扩展功能,比如轮播图/picker/上拉下拉刷新等功能,我们这里不对扩展功能进行讲解,重点分析核心的滚动部分.

作者将 better-scroll 的原型链通过 mixin 的方式分成不同的功能部分并进行扩展

```javascript
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

可以看到入口文件其实就只做了一件事,创建 BScroll 的构造函数,其中调用了 init 方法,那我们先来分析 init()

```javascript
BScroll.prototype._init = function(el, options) {
    this._handleOptions(options) //处理配置参数

    // init private custom events
    this._events = {} //初始化自定义事件缓存，自定义事件的实现在src/scroll/event.js中

    this.x = 0 //scroll 横轴坐标。
    this.y = 0 //scroll 纵轴坐标。
    this.directionX = 0 //判断 scroll 滑动结束后相对于开始滑动位置的方向（左右）。-1 表示从左向右滑，1 表示从右向左滑，0 表示没有滑动。
    this.directionY = 0 //判断 scroll 滑动结束后相对于开始滑动位置的方向（上下）。-1 表示从左向下滑，1 表示从右向上滑，0 表示没有滑动。

    this._addDOMEvents() //添加原生事件监听

    this._initExtFeatures() //根据配置参数初始化betterScroll的额外功能

    this._watchTransition() //监听是否处于过渡状态

    if (this.options.observeDOM) {
        this._initDOMObserver() //会检测 scroller 内部 DOM 变化，自动调用 refresh 方法重新计算来保证滚动的正确性。
    }

    this.refresh() // 重新计算容器和可滚动元素（this.scroller)宽高，可滚动最大距离，并重置滚动元素的位置

    if (!this.options.snap) {
        this.scrollTo(this.options.startX, this.options.startY) // 滚动到初始位置
    }

    this.enable() //开启滚动功能
}
```

在 init 模块中,作者做了如下一些事情.

    1.将用户传入的配置参数与默认参数一起进行合并配置
    2.将初始化的自定义事件缓存,并定义了滚动过程中滚动元素的位置和方向(扩展x,y,directionX和directionY用于存放滚动过程中x轴y轴上的位置和方向)
    3.**对原生的事件进行监听(mouse,touch,resize,orientationchange)和监听是否处于过渡状态**
    4.根据配置需求,初始化其他功能(如snap/wheel之类)
    5.**初始化observer**
    6.**refresh方法,当滚动组件内部dom变化的时候,重新计算滚动容器的高宽,滚动条的位置,并将滚动元素重置**
    6.初始化滚动元素的位置

重点讲解的部分  就是高亮的三块

better-scroll 是一个滚动插件，所以必定要监听原生的事件。通常而言，我们写一个事件的监听函数，一般会像如下的方式编写

```js
target.addEventListener('type', fn, false / true)
```

这其中第二个参数 fn 我们都是写成函数的形式去处理，但这样有一个问题，在你要注销这个监听事件的时候，你必须要传递一个一模一样的函数进去才可以。在我们需要监听大量的事件的时候，这种做法就不可行了。在[MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/EventTarget/addEventListener)的文档中，其实还有另一种写法，即传入一个具有 handleEvent 属性的对象就行，这样在我们需要移除事件监听的时候，只要将方法中的 addEventListener 修改为 removeEventListener 就行，非常优雅。此处的代码实现可以直接参考[addEvent](https://github.com/ustbhuangyi/better-scroll/blob/master/src/util/dom.js#L39)和[removeEvent](https://github.com/ustbhuangyi/better-scroll/blob/master/src/util/dom.js#L43)方法。
better-scroll 这里，是将 this[直接传入](https://github.com/ustbhuangyi/better-scroll/blob/master/src/scroll/init.js#L172)，因为 better-scroll 已经在原型中定义了[handleEvent](https://github.com/ustbhuangyi/better-scroll/blob/master/src/scroll/init.js#L333)方法。

tips:
如果事件监听没有被正确的 remove,可能会产生内存泄漏的问题

初始化模块中另一个  的属性`isInTransition`，是用来处理处于过度动画的时候，禁用鼠标事件(为了解决 #359 issue)。这里是使用了 es5 的 Object.defineprototy 来实现的对对象进行双向绑定功能(与 vue 的  基本实现方式一致,所以在 ie8 以下无法使用).

```js
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

初始化中最后一个重要的方法就是 refresh()方法，他是用来在 DOM 结构发生变化的时候保证滚动效果正常的方法,主要是获取容器和滚动元素的宽高，计算最大滚动距离。

```js
BScroll.prototype.refresh = function() {
    // 通过getBoundingClientRect来获取容器的宽高
    let wrapperRect = getRect(this.wrapper)
    this.wrapperWidth = wrapperRect.width
    this.wrapperHeight = wrapperRect.height

    // 通过getBoundingClientRect来获取滚动元素的宽高

    let scrollerRect = getRect(this.scroller)
    this.scrollerWidth = scrollerRect.width
    this.scrollerHeight = scrollerRect.height

    const wheel = this.options.wheel
    if (wheel) {
        this.items = this.scroller.children
        this.options.itemHeight = this.itemHeight = this.items.length
            ? this.scrollerHeight / this.items.length
            : 0
        if (this.selectedIndex === undefined) {
            this.selectedIndex = wheel.selectedIndex || 0
        }
        this.options.startY = -this.selectedIndex * this.itemHeight
        this.maxScrollX = 0
        this.maxScrollY = -this.itemHeight * (this.items.length - 1)
    } else {
        //计算最大的横向滚动距离
        this.maxScrollX = this.wrapperWidth - this.scrollerWidth
        //计算最大的纵向滚动距离
        this.maxScrollY = this.wrapperHeight - this.scrollerHeight
    }

    this.hasHorizontalScroll = this.options.scrollX && this.maxScrollX < 0
    this.hasVerticalScroll = this.options.scrollY && this.maxScrollY < 0

    if (!this.hasHorizontalScroll) {
        this.maxScrollX = 0
        this.scrollerWidth = this.wrapperWidth
    }

    if (!this.hasVerticalScroll) {
        this.maxScrollY = 0
        this.scrollerHeight = this.wrapperHeight
    }
    //滚动结束时间
    this.endTime = 0
    //横向滚动方向， 0表示没有滚动
    this.directionX = 0
    //纵向滚动方向， 0表示没有滚动
    this.directionY = 0
    this.wrapperOffset = offset(this.wrapper)
    //触发"refresh"事件
    this.trigger('refresh')
    //  重置滚动元素位置
    this.resetPosition()
}
```

在 web 页面中,坐标系参考是在  元素的左上方的,所以如果我们向上/向左滚动的话,那 maxScrollX/maxScrollY 必定是负值,所以只要判断他们的值的正负,就能说明元素是否可以滚动.
![scroll-position](./img/scroll_position.png 'scroll-position')

在 DOM 结构改变后，scroller 需要获取容器和滚动元素的宽高，重新计算最大的滚动距离等。

以上就完成了初始化部分的分析.接下来就是滚动的核心部分,讲解手指在屏幕上 touch start/touch move/touch end 三个时间点 better-scroll 都做了什么事情.
better-scroll 为了实现较好的用户体验,在用户滑动屏幕后加入了惯性滚动部分,所以滑动和滚动在 Better-scroll 中指的是两个不同的部分.

core 文件中,最核心的部分就是 scrollTo()方法,他是模拟滚动的直接实现,在这个函数的内部主要有两个操作_translate(x,y)和_startProbe().其他操作是设置过度动画时间/设置过渡动画加速度曲线/兼容浏览器设置 transition 前缀等

```javascript
BScroll.prototype.scrollTo = function(x, y, time = 0, easing = ease.bounce) {
    // useTransition是否使用css3 transition,isInTransition表示是否在滚动过程中
    // this.x表示translate后的位置或者初始化this.x = 0
    this.isInTransition =
        this.options.useTransition && time > 0 && (x !== this.x || y !== this.y)

    //如果存在过渡动画（time>0即有动画），且指定过渡过程中也要派发"scroll"事件
    if (!time || this.options.useTransition) {
        this._transitionProperty() // 设置使用transform
        this._transitionTimingFunction(easing.style) // 设置过渡动画加速度曲线，暂不关心
        this._transitionTime(time) // 设置过渡时间
        this._translate(x, y) // 滚动的最直接实现

        // time存在protoType表示不仅在屏幕滑动的时候， momentum 滚动动画运行过程中实时派发 scroll 事件
        if (time && this.options.probeType === 3) {
            //实现在浏览器每次刷新时派发"scroll"事件
            this._startProbe()
        }

        // wheel用于picker组件设置,不关心
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

在_translate()方法中主要做了两件事：设置 tranform，实现元素位置变化,这样给人的视觉就和浏览器的滚动一样；记录滚动位置。如果只有_translate()方法会实现页面位置变化，但不会有过渡动画。

```javascript
BScroll.prototype._translate = function(x, y) {
    if (this.options.useTransform) {
        // this.translateZ是从硬件加速那儿来的
        this.scrollerStyle[style.transform] = `translate(${x}px,${y}px)${
            this.translateZ
        }`
    } else {
        x = Math.round(x)
        y = Math.round(y)
        // 使用绝对定位控制位置，我们主要关心transform的情况 此处不关心
        this.scrollerStyle.left = `${x}px`
        this.scrollerStyle.top = `${y}px`
    }
    //wheel相关 不关心
    if (this.options.wheel) {
        const { rotate = 25 } = this.options.wheel
        for (let i = 0; i < this.items.length; i++) {
            let deg = rotate * (y / this.itemHeight + i)
            this.items[i].style[style.transform] = `rotateX(${deg}deg)`
        }
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

在_startProbe()方法中,主要是通过 requestAnimationFrame 请求浏览器在每次刷新时执行回调 probe()派发"scroll"事件，同时在回调中判断，如果过渡状态结束，不再请求执行回调。这里还调用了 getComputedPosition()方法，主要是用于获取此时的滚动位置

```javascript
BScroll.prototype._startProbe = function() {
    cancelAnimationFrame(this.probeTimer)
    this.probeTimer = requestAnimationFrame(probe)

    let me = this

    function probe() {
        if (!me.isInTransition) {
            return
        }
        // getComputedPosition返回的transform的属性值
        let pos = me.getComputedPosition()
        // 派发scroll事件，并且将Pos参数传入进去
        me.trigger('scroll', pos)
        me.probeTimer = requestAnimationFrame(probe)
    }
}
```

#### 自定义事件

自定义事件是通过经典的 Pub-sub 设计模式来实现的,这个比较简单,无论是原理还是源码在网上都有很多.

```javascript
export function eventMixin(BScroll) {
    BScroll.prototype.on = function(type, fn, context = this) {
        // 如果这个类型的事件不存在，那么this._events[type] = []
        if (!this._event[type]) {
            this._events[type] = []
        }

        this._events[type].push([fn, context])
    }

    BScroll.prototype.once = function(type, fn, context = this) {
        let fired = false

        function magic() {
            this.off(type, magic)

            if (!fired) {
                fired = true
                fn.apply(context, arguments)
            }
        }
        // 将参数中的回调函数挂载在magic对象的fn属性上，
        // 为了执行off方法的时候，暴露对应的函数方法
        magic.fn = fn

        this.on(type, magic)
    }

    // 移除事件
    BScroll.prototype.off = function(type, fn) {
        let _events = this._events[type]
        if (!_events) {
            return
        }

        let count = _events.length
        while (count--) {
            // 移除通过on或者once绑定的回调函数
            if (
                _events[count][0] === fn ||
                (_events[count][0] && _events[count][0].fn === fn)
            ) {
                _events[count][0] = undefined
            }
        }
    }

    BScroll.prototype.trigger = function(type) {
        let events = this._events[type]
        if (!events) {
            return
        }

        let len = events.length
        let eventsCopy = [...events]
        for (let i = 0; i < len; i++) {
            let event = eventsCopy[i]
            let [fn, context] = event
            if (fn) {
                fn.apply(context, [].slice.call(arguments, 1))
            }
        }
    }
}
```

下面就是最核心的环节,弄清楚手指在屏幕上滑动开始/滑动中/滑动结束的三个事件点,better-scroll 都做了什么事情.
bs 在初始化的时候,监听了一些原生事件

```javascript
BScroll.prototype.handleEvent = function(e) {
    switch (e.type) {
        // code...
        case 'touchend':
        case 'mouseup':
        case 'touchcancel':
        case 'mousecancel':
            this._end(e)
            break
        case 'orientationchange':
        case 'resize':
            this._resize()
            break
        case 'transitionend':
        case 'webkitTransitionEnd':
        case 'oTransitionEnd':
        case 'MSTransitionEnd':
            this._transitionEnd(e)
            break
        // code...
        case 'wheel':
        case 'DOMMouseScroll':
        case 'mousewheel':
            this._onMouseWheel(e)
            break
    }
}
```

这样写即可以兼容 PC 端、移动端、各个浏览器 事件.

先看一下_start()方法

```javascript
BScroll.prototype._start = function(e) {
    /************ 1、判断 PC 端只允许左键点击 不是鼠标左键，直接退出 ***************/
    let _eventType = eventType[e.type]
    if (_eventType !== TOUCH_EVENT) {
        if (e.button !== 0) {
            return
        }
    }
    // 未开启滚动，已销毁都直接退出
    if (
        !this.enabled ||
        this.destroyed ||
        (this.initiated && this.initiated !== _eventType)
    ) {
        return
    }
    this.initiated = _eventType

    if (
        this.options.preventDefault &&
        !preventDefaultException(e.target, this.options.preventDefaultException)
    ) {
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

    this._transitionTime() // 初始化过渡时间为0
    this.startTime = getNow() // 初始化开始时间

    if (this.options.wheel) {
        // 不关心
        this.target = e.target
    }

    this.stop() // 如果此时处于惯性滚动过程中，停止惯性滚动。

    let point = e.touches ? e.touches[0] : e
// 在 _end 方法中用于计算快速滑动 flick
    this.startX = this.x
    this.startY = this.y
    this.absStartX = this.x
    this.absStartY = this.y
    this.pointX = point.pageX // 鼠标相对于页面位置
    this.pointY = point.pageY

    this.trigger('beforeScrollStart')
}
```

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

如果没有处于惯性滚动过程中，这个方法什么都不会做（我们只考虑使用 transition 的情况）。如果处于惯性滚动过程中，会获取元素当前位置，重新调用_translate()设置最终滚动位置为当前位置。为什么说是重新调用？因为由于过渡动画的存在，上一次的滚动还没有结束，即还没有滚动要目标位置，我们现在又重新设置目标位置为当前位置。同时调用 stop()前调用了_transitionTime()设置过渡时间为 0，所以过渡动画会马上结束。最终的效果就是，当我们手指接触屏幕的时候，原来的惯性滚动马上停止。
总结而言
- 判断 PC 端只允许左键点击
- 判断是否可操作、是否未销毁,没有就直接跳出函数
- 设置 transition 运动时间，不传参数默认为0
- 若上一次滚动还在继续时，此时触发了_start，则停止到当前滚动位置
- BS还提供了beforeScrollStart事件,方便和外面做交互

在_move()中,bs 做了如下一些事情

- 计算鼠标或者手指的滑动距离
- 根据横向纵向滑动距离锁定一个滚动方向（前提 freeScroll === false)
- 根据滑动距离计算滚动距离，并执行滚动
- 派发"scroll"事件
- 判断手指滑动到视窗(visual viewport)边缘，执行\_end()

```javascript
BScroll.prototype._move = function (e) {
    ... // 是否开启滚动，是否已销毁的判断，阻止浏览器默认行为操作

    /************下面部分计算滑动距离************/
    let point = e.touches ? e.touches[0] : e
    let deltaX = point.pageX - this.pointX
    let deltaY = point.pageY - this.pointY

    this.pointX = point.pageX // 缓存手指/鼠标坐标
    this.pointY = point.pageY

    this.distX += deltaX
    this.distY += deltaY

    let absDistX = Math.abs(this.distX)
    let absDistY = Math.abs(this.distY)
    /************上面部分计算滑动距离************/

    let timestamp = getNow()

    // 需要移动至少 (momentumLimitDistance) 个像素来启动滚动
    if (timestamp - this.endTime > this.options.momentumLimitTime && (absDistY < this.options.momentumLimitDistance && absDistX < this.options.momentumLimitDistance)) {
      return
    }

    /****************下面部分锁定滚动方向******************************/
    // 如果没有开启freeScroll，会根据手指滑动方向，锁定一个滚动方向
    if (!this.directionLocked && !this.options.freeScroll) {
      /** 
       *  absDistX：横向移动的距离
       *  absDistY：纵向移动的距离
       *  触摸开始位置移动到当前位置：横向距离 > 纵向距离，说明当前是在往水平方向(h)移动，反之垂直方向(v)移动。
       */
      if (absDistX > absDistY + this.options.directionLockThreshold) {
        this.directionLocked = 'h'		// lock horizontally
      } else if (absDistY >= absDistX + this.options.directionLockThreshold) {
        this.directionLocked = 'v'		// lock vertically
      } else {
        this.directionLocked = 'n'		// no lock
      }
    }
    /****若当前锁定（h | v）方向， eventPassthrough 为 锁定方向 相反方向的话则阻止默认事件  ****/

    if (this.directionLocked === 'h') {
      if (this.options.eventPassthrough === 'vertical') {
        e.preventDefault() //取消滚动锁定方向的浏览器默认行为
      } else if (this.options.eventPassthrough === 'horizontal') {
        this.initiated = false //如果滚动锁定方向和eventPassthrough方向相同，不允许滚动，并直接退出
        return
      }
      deltaY = 0 // 滚动锁定的另一个方向，手指滑动距离重置为0
    } else if (this.directionLocked === 'v') {
      if (this.options.eventPassthrough === 'horizontal') {
        e.preventDefault()
      } else if (this.options.eventPassthrough === 'vertical') {
        this.initiated = false
        return
      }
      deltaX = 0
    }
    /****************上面部分锁定滚动方向******************************/

    /************以下部分计算新滚动坐标**********/
    deltaX = this.hasHorizontalScroll ? deltaX : 0
    deltaY = this.hasVerticalScroll ? deltaY : 0
    this.movingDirectionX = deltaX > 0 ? DIRECTION_RIGHT : deltaX < 0 ? DIRECTION_LEFT : 0
    this.movingDirectionY = deltaY > 0 ? DIRECTION_DOWN : deltaY < 0 ? DIRECTION_UP : 0

    let newX = this.x + deltaX
    let newY = this.y + deltaY

    // Slow down or stop if outside of the boundaries
    // 如果超过边界的话，滚动到超过边界时速度要变慢,fre设置了 bounce 参数就让它回弹过来
    if (newX > 0 || newX < this.maxScrollX) {
      if (this.options.bounce) {
        newX = this.x + deltaX / 3
      } else {
        newX = newX > 0 ? 0 : this.maxScrollX
      }
    }
    if (newY > 0 || newY < this.maxScrollY) {
      if (this.options.bounce) {
        newY = this.y + deltaY / 3
      } else {
        newY = newY > 0 ? 0 : this.maxScrollY
      }
    }

    // 在 _start 方法中设置的 moved 变量，现在使用到了
    if (!this.moved) {
      this.moved = true
      this.trigger('scrollStart') //触发"scrollStart"事件
    }

    this._translate(newX, newY) //滚动到指定位置
    /***********以上部分计算新的滚动坐标**********/

    /***********下面这部分负责滑动过程中派发"scroll"事件**********/
    //大于 momentumLimitTime 是不会触发flick\momentum，赋值是为了重新计算，能够在_end函数中触发flick\momentum 
    if (timestamp - this.startTime > this.options.momentumLimitTime) {
      this.startTime = timestamp
      this.startX = this.x
      this.startY = this.y
    //   滚动比较慢的情况下
      if (this.options.probeType === 1) {
        this.trigger('scroll', {
          x: this.x,
          y: this.y
        })
      }
    }
    // 实时触发scroll
    if (this.options.probeType > 1) {
      this.trigger('scroll', {
        x: this.x,
        y: this.y
      })
    }
    /***********上面这部分负责滑动过程中派发"scroll"事件**********/

    /**********下面是如果滑动到视窗的四个边缘执行滑动结束****************/
    let scrollLeft = document.documentElement.scrollLeft || window.pageXOffset || document.body.scrollLeft
    let scrollTop = document.documentElement.scrollTop || window.pageYOffset || document.body.scrollTop

    let pX = this.pointX - scrollLeft //距离可视窗口左边缘距离
    let pY = this.pointY - scrollTop //距离可视窗口上边缘距离

    if (pX > document.documentElement.clientWidth - this.options.momentumLimitDistance || pX < this.options.momentumLimitDistance || pY < this.options.momentumLimitDistance || pY > document.documentElement.clientHeight - this.options.momentumLimitDistance
    ) {
      this._end(e) // 滑动结束
    }
    /**********上面是如果滑动到视窗的四个边缘执行滑动结束****************/
  }
```

在_end()方法中,Bs 做了如下的事情

* 如果滚动超出边界，重置滚动位置，并退出函数
* 如果是一次轻拂操作，退出函数
* 计算整个滑动过程的时间和距离
* 判断是否满足惯性滚动条件，并计算惯性滚动距离和时间
* 执行惯性滚动，并退出

```javascript
BScroll.prototype._end = function(e) {
    // this.enabled判断scroller 是否处于启用状态
    if (
        !this.enabled ||
        this.destoryed ||
        eventType[e.type] !== this.initiated
    ) {
        return
    }
    this.initiated = false

    if (
        this.options.preventDefault &&
        !preventDefaultException(e.target, this.options.preventDefaultException)
    ) {
        e.preventDefault()
    }

    this.trigger('touchEnd', {
        x: this.x,
        y: this.y
    })

    let preventClick = this.stopFromTranstion
    this.stopFromTranstion = false

    // 下拉刷新
    if (this.options.pullDownRefresh && this._checkPullDown()) {
        return
    }

    // rest if we are outside of the boundaries
        // 如果this.scroller越界，设置回弹效果

    if (this.resetPosition(this.options.bounceTime, ease.bounce)) {
        return
    }
    this.isInTransition = false
    // 确保最后的位置是最靠近的
    let newX = Math.round(this.x)
    let newY = Math.round(this.y)

    if (!this.moved) {
        if (this.options.wheel) {
            if (
                this.target &&
                this.target.className === this.options.wheel.wheelWrapperClass
            ) {
                let index = Math.abs(Math.round(newY / this.itemHeight))
                let _offset = Math.round(
                    (this.pointY +
                        offset(this.target).top -
                        this.itemHeight / 2) /
                        this.itemHeight
                )
                this.target = this.items[index + _offset]
            }
            this.scrollToElement(
                this.target,
                this.options.wheel.adjustTime || 400,
                true,
                true,
                ease.swipe
            )
        } else {
            // preventClick = false
            if (!preventClick) {
                // tap被点击的区域派发点击tap事件
                if (this.options.tap) {
                    tap(e, this.options.tap)
                }
                // 点击派发click事件，并且给Event加一个属性_constructed,为true
                if (this.options.click) {
                    click(e)
                }
            }
        }
        this.trigger('scrollCancel')
        return
    }

    this.scrollTo(newX, newY)

    let deltaX = newX - this.absStartX
    let deltaY = newY - this.absStartY
    this.directionX =
        deltaX > 0 ? DIRECTION_RIGHT : deltaX < 0 ? DIRECTION_LEFT : 0
    this.directionY =
        deltaY > 0 ? DIRECTION_DOWN : deltaY < 0 ? DIRECTION_UP : 0
    //计算整个滑动过程的距离和时间

    this.endTime = getNow()

    let duration = this.endTime - this.startTime
    let absDistX = Math.abs(newX - this.startX)
    let absDistY = Math.abs(newY - this.startY)

    // 判断是否是一次轻拂操作
    if (
        this._events.flick &&
        duration < this.options.flickLimitTime &&
        absDistX < this.options.flickLimitDistance &&
        absDistY < this.options.flickLimitDistance
    ) {
        this.trigger('flick')
        return
    }

    let time = 0
    // start momentum animation if needed
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

    let easing = ease.swipe
    if (this.options.snap) {
        let snap = this._nearstSnap(newX, newY)
        this.currentPage = snap
        time =
            this.options.snapSpeed ||
            Math.max(
                Math.max(
                    Math.min(Math.abs(newX - snap.x), 1000),
                    Math.min(Math.abs(newY - snap.y), 1000)
                ),
                300
            )
        newX = snap.x
        newY = snap.y

        this.directionX = 0
        this.directionY = 0
        easing = ease.bounce
    }

    if (newX !== this.x || newY !== this.y) {
        // 改变easing方法，当scroller超出边界的时候
        if (
            newX > 0 ||
            newX < this.maxScrollX ||
            newY > 0 ||
            newY < this.maxScrollY
        ) {
            easing = ease.swipeBounceTime
        }
        this.scrollTo(newX, newY, time, easing)//执行惯性滚动过渡动画
        return
    }

    if (this.options.wheel) {
        this.selectedIndex = Math.round(Math.abs(this.y / this.itemHeight))
    }
    this.trigger('scrollEnd', {
        x: this.x,
        y: this.y
    })
}
```

可以看到在滑动结束的时候，better-scroll 会判断是否需要惯性滚动/滑动，计算要滑动到哪里。至于怎么计算的，可以查看 src/scroll/momentum.js 文件。

而在滚动结束后,bs 也有监听 transitionend

```JavaScript
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
最后的方法中会判断resetPosition

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
可以看到resetPosition()方法只做了一件事情，调用scrollTo()方法重置滚动元素位置。什么情况下需要重置滚动元素位置呢？先说一下我们在滚动过程中一个合理的滚动距离dis，应该满足maxScrollX/Y <= dis <= 0。所以滚动的距离如果超出了边界值：

dis>0，向下或者向右滚动超过边界值，会重置为0。
dis<maxScrollX/Y，向上或者向左滚动超出边界值，会重置为maxScrollX/Y。
如果滚动距离没有超过边界值，那么resetPosition不会做任何事情，保持原来的位置。