---
title: React测试
tags: javascript,react,test,jest,enzyme
grammar_cjkRuby: true
---

#### 测试React components 主要从两方面:

1. 给定一些输入 (state&props),断言应该输出的component
2. 给定一些用户的操作,断言component的行为.component可能会更姓 `state` 或者 调用父类传入的 porps-function

## Shallow rendering

当一个 compoennt 是 shallow 形式渲染的, 它并没有写进DOM.相反,它保持它的虚拟DOM的表现状态.

Shadow 渲染只会渲染当前层级的component.如果渲染方法中的component包含 children,这些children并不会被渲染.实际上,生成的虚拟DOM中只包含这些没有渲染的children的引用.

React 提供了**ReactTestUtils**这个库来shadow 渲染 component,很有用,但是用法略显繁杂.

Enzyme包装了ReactTestUtils,在其上提供了一些使用起来更方便的方法.


## mock
jest 中的mock可以inject一个module,这里完全参考官方文档[Async Examle][1].
(文档写的好啊!);
注意点 : 
1. 要mock的模块,需要在其同层建立一个__mocks__ 文件夹,里面手动新建要mock的模块
	![example __mock__][2]
2. jest.mock(' module'),这个module在test.js中require时,需要确保在其中要mock的函数要在 jest.mock命令之后
	```javascript
		jest.mock('../services/Fetch.ts')
		import '/store';// store 中使用 fetch来拉取数据
	```

##  coverage
1. statement not cover
```typescript
@observable currentTrack: ITrack|null=null
@observable private nextHrefsByTrack = new ObservableMap<string>();  
```

## jest 有一些莫名奇妙的错误
1. cant

--------


  [1]: http://facebook.github.io/jest/docs/tutorial-async.html#content
  [2]: ./images/%E9%80%89%E5%8C%BA_012.png "选区_012"