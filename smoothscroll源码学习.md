---
title: smoothscroll源码阅读
tags: html,scroll,source code
grammar_cjkRuby: true
---

> 项目中需要一个滑动到顶部的功能,查阅资料发现有三种不同的滑动方法,而这些原生的滑动是生硬的,所以这里找到了一个开源项目,发现写的思路清晰,所以这里做个记录.


## 原生滑动

```html
<div id='app'> 
    <div id='inner'>
      innerContent
    </div>
  </div>
  <button id='scrollTo'>
    windowscrollTo
  </button>
```
```css
#inner{
  width:100%;
  height:1000px;
  background:lightBlue;
  font-size:40px;
  
}
#app{
  overflow:auto;
  /* height:90vh; */
}
```

滑动方式:
1. scroll || scrollTo

scrollTo方法只存在window中,接受(x,y)两个参数,表示将当前窗口移动到目标位置,像这样:

```javascript
	var stBtn = document.querySelector('#scrollTo')
	var app = document.querySelector('#app')
 
	stBtn.addEventListener('click',function(){
  		window.scrollTo(0,50); 
	});
```
![enter description here][1]
对于Element,只能通过修改其scrollTop或者scrollLeft来达到同样的目的.**注意的是**,Element本身需要满足以下条件:

  -  有overflow,overflow-x,overflow-y属性中一种,且值必须为:auto,scroll
  -  它的scrollbar必须显示且可以滑动
       
 ![enter description here][2]

2. scrollBy
    window在目前坐标的基础上,改变(x,y)相应的坐标
```javascript

scrollByBtn.addEventListener('click',function(){
  window.scrollBy(0,50);  
});

```
![enter description here][3]
Element同样只能通过修改scrollTop来实现,不做演示

3. [scrollIntoView](https://developer.mozilla.org/zh-CN/docs/Web/API/Element/scrollIntoView)
     > 让当前的元素滚动到浏览器窗口的可视区域内,这是Element独有方法
具体可查看文档,这里只做演示:
![enter description here][4]



# [pollyfill](https://github.com/iamdustan/smoothscroll)
> 劫持原生方法,计算目标坐标和当前位置的距离,并计算余弦函数来设定每次requestAnimationFrame的距离,达到流畅动画的效果

- 支持全局形式和模块化形式

```javascript

(function(w, d, undefined) {
  'use strict';
  // polyfill
  function polyfill() { ...}

  if (typeof exports === 'object') {
    // commonjs
    module.exports = { polyfill: polyfill };
  } else {
    // global
    polyfill();
  }
})(window, document);
```

如果需要smooth动画,最终都会调用这个方法:
```javascript
 function smoothScroll(el, x, y) {
      var scrollable; //滑动的组件
      var startX; 
      var startY;
      var method; // 滑动方法
      var startTime = now(); // 记录开始时间,用于余弦函数的计算

      // define scroll context
      if (el === d.body) {
        scrollable = w;
        startX = w.scrollX || w.pageXOffset;
        startY = w.scrollY || w.pageYOffset;
        method = original.scroll;
      } else {
        scrollable = el;
        startX = el.scrollLeft;
        startY = el.scrollTop;
        method = scrollElement;
      }

      // scroll looping over a frame
      step({
        scrollable: scrollable,
        method: method,
        startTime: startTime,
        startX: startX,
        startY: startY,
        x: x,
        y: y
      });
    }
```

step,动画滑动方法:

```javascript
function step(context) {
      var time = now();
      var value;
      var currentX;
      var currentY;
      var elapsed = (time - context.startTime) / SCROLL_TIME; 

      // avoid elapsed times higher than one
      elapsed = elapsed > 1 ? 1 : elapsed;

      // apply easing to elapsed time
      value = ease(elapsed);

      currentX = context.startX + (context.x - context.startX) * value;
      currentY = context.startY + (context.y - context.startY) * value;

      context.method.call(context.scrollable, currentX, currentY);

      // scroll more if we have not reached our destination
      if (currentX !== context.x || currentY !== context.y) {
        w.requestAnimationFrame(step.bind(w, context));
      }
    }
```

- 如果在闭包中调用 window方法,需要进行绑定
`        const now = w.performance && w.performance.now ? w.performance.now : Date.now`这时在其他地方调用会出现
`Uncaught TypeError: Illegal invocation`
需要改成这样:
`        const now = w.performance && w.performance.now ? w.performance.now.bind(w.performance) : Date.now`

-  Element.scrollIntoView 需要将节点展现在视口中,需要计算
	-    当前父节点的确立坐标
	-    节点的确立坐标 	
	-    当前节点的可滑动的父节点的确立坐标
	
```javascript
	 function findScrollableParent(el) {
          var isBody;
          var hasScrollableSpace;
      	  var hasVisibleOverflow;
	      do {
	        el = el.parentNode;
	        // set condition variables
	        isBody = el === d.body;
	        hasScrollableSpace =
	          el.clientHeight < el.scrollHeight ||
	          el.clientWidth < el.scrollWidth;
	        hasVisibleOverflow =
	          w.getComputedStyle(el, null).overflow === 'visible';
	      } while (!isBody && !(hasScrollableSpace && !hasVisibleOverflow));
	
          isBody = hasScrollableSpace = hasVisibleOverflow = null;

      return el;
    }

```


  [1]: ./images/scrollTo%280,50%29.gif "scrollTo(0,50)"
  [2]: ./images/scrollTop=50.gif "scrollTop=50"
  [3]: ./images/scrollBy%280,50%29.gif "scrollBy(0,50)"
  [4]: ./images/scrollIntoView%28%7Bblock: "endquot;, behavior: quot;smoothquot;}).gif quot;scrollIntoView({block: quot;endquot;, behavior: quot;smoothquot;})"
