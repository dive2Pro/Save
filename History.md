---
title: History
tags: window,html,browser,react,react-router
grammar_cjkRuby: true
---
 
 > 一直使用react-router来作为单页面的路由,项目中需要一个功能-记录某个页面跳转时当前的浏览位置-,发现v4版本并没有react-scrooll的支持. 趁这个机会来学习一下history和路由的原理知识.

[history](https://developer.mozilla.org/zh-CN/docs/DOM/Manipulating_the_browser_history)是属于浏览器实现DOM接口的window对象的属性,用于记录当前浏览的访问历史,以及控制浏览器在记录中移动.

# 基本方法

## 1. go (number)

go接受一个数字参数,控制浏览器加载记录的索引值.
- -n 后退n个页面
-  0  重新加载当前页面
-  +n 前进n个页面
-  window.history.length : 查看历史记录的长度

## 2. pushState,replaceState
> 添加或修改历史记录条文

它们允许你逐条地添加和修改历史记录条目。这些方法可以协同[window.onpopstate](https://developer.mozilla.org/zh-CN/docs/Web/API/Window/onpopstate)事件,相当于一个回调函数.


###  pushState
MDN的定义如下:

pushState()有三个参数：一个状态对象、一个标题（现在会被忽略），一个可选的URL地址。下面来单独考察这三个参数的细节：

- 状态对象（state object） — 一个JavaScript对象，与用pushState()方法创建的新历史记录条目关联。无论何时用户导航到新创建的状态，**popstate事件都会被触发**，并且**事件对象的state属性都包含历史记录条目的状态对象的拷贝**。

任何可序列化的对象都可以被当做状态对象。因为FireFox浏览器会把状态对象保存到用户的硬盘，这样它们就**能在用户重启浏览器之后被还原**，我们强行限制状态对象的大小为640k。如果你向pushState()方法传递了一个超过该限额的状态对象，该方法会抛出异常。如果你需要存储很大的数据，建议使用sessionStorage或localStorage。

- 标题（title） — FireFox浏览器目前会忽略该参数，虽然以后可能会用上。考虑到未来可能会对该方法进行修改，**传一个空字符串会比较安全**。或者，你也可以传入一个简短的标题，标明将要进入的状态。

- 地址（URL） — 新的历史记录条目的地址。浏览器不会在调用pushState()方法后加载该地址，但之后，可能会试图加载，例如用户重启浏览器。新的URL不一定是绝对路径；如果是**相对路径，它将以当前URL为基准**；传入的**URL与当前URL应该是同源的**，否则，pushState()会抛出异常。该参数是可选的；不指定的话则为文档当前URL。

### replaceState
> history.replaceState()操作类似于history.pushState()，不同之处在于replaceState()方法会修改当前历史记录条目而并非创建新的条目。


#### 读取当前状态

在页面加载时，可能会包含一个非空的状态对象。这种情况是会发生的，例如，如果页面中使用pushState()或replaceState()方法设置了一个状态对象，然后用户重启了浏览器。当**页面重新加载**时，页面会触发`onload`事件，但不会触发`popstate`事件。但是，**如果你读取 history.state 属性，你会得到一个与  popstate 事件触发时得到的一样的状态对象**。



# 开源项目

> 既然window.history只提供了这些基本的方法,而开源出来的项目肯定是为了来解决某种问题,那么工作中都遇到了哪些问题需要解决呢?这些开源项目又解决了这些问题中的哪部分呢?


## [History](https://github.com/ReactTraining/history)
> react-traning开源的项目,react-router更新迭代但是底层一直是使用它来作为浏览器记录控制

在原有的基础上增强的有:

1. 浏览器兼容
2. popstate的事件管理
3. popstate的事件确认

## 设计模式:
###  观察者模式

  >  js中的观察者模式通过闭包和返回函数可以达到非常方便的解绑动作

block,和listener都会触发下面的方法,window对popstate事件的监听,这里通过计算listener的数量,确定是否继续window监听
```javascript
	const checkDOMListeners = (delta) => {
  	  listenerCount += delta

	    if (listenerCount === 1) {
  	    addEventListener(window, PopStateEvent, handlePopState) //全局的

  	    if (needsHashChangeListener)
  	      addEventListener(window, HashChangeEvent, handleHashChange)
  	  } else if (listenerCount === 0) {
	      removeEventListener(window, PopStateEvent, handlePopState)

	      if (needsHashChangeListener)
	        removeEventListener(window, HashChangeEvent, handleHashChange)
	    }
  }

```

 block和listener的函数结构差不多,这里以添加listener为例:
```javascript
 const appendListener = (fn) => {
    let isActive = true
	// 闭包引用变量
    const listener = (...args) => {
      if (isActive)
        fn(...args)
    }

    listeners.push(listener)
  // 返回的是一个闭包
    return () => {
      isActive = false
      // listener被闭包闭住,本函数被调用即完成解绑操作
      listeners = listeners.filter(item => item !== listener)
    }
  }
```
popstate 事件被触发,handlePopState被调用

```javascript
const handlePop = (location) => {
    if (forceNextPop) {
      forceNextPop = false
      setState()
    } else {
      const action = 'POP'
	
      transitionManager.confirmTransitionTo(location, action, 
      // 这是 window.confirm函数       它是这样的:
      /**
	*  第二个参数包裹住window.confim的返回
        *  const getConfirmation = (message, callback) =>
        *                     callback(window.confirm(message))
	**/
      getUserConfirmation, 
      // 这是confirm函数的回调函数 也就是callback
      (ok) => {
        if (ok) {
          setState({ action, location })//setState则是通知观察者
        } else {
          revertPop(location)
        }
      })
    }
  }
 // ----------------------
 const confirmTransitionTo = (location, action, getUserConfirmation, callback) => {
 	// prompt 是 block中传入的函数
    if (prompt != null) {
      const result = typeof prompt === 'function' ? prompt(location, action) : prompt

      if (typeof result === 'string') {
        if (typeof getUserConfirmation === 'function') {
        // callback 作为参数 window.confirm选择后被执行
          getUserConfirmation(result, callback)
        } else { 
          callback(true)
        }
      } else {
        // Return false from a transition hook to cancel the transition.
        callback(result !== false)
      }
    } else {
      callback(true)
    }
  }
```










