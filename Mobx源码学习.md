---
title: Mobx源码学习
tags: mobx,source code,github
grammar_cjkRuby: true
---

# 0.4v

## 一个对象需要依赖另一个对象,关于计算属性 computed

对象 o1 : order.price
对象 o2: order.priceWithVat

``` javascript
var vat = mobservable.value(0.20);

var order = {};
order.price = mobservable.value(10),
order.priceWithVat = mobservable.value(() => order.price() * (1 + vat()));

order.priceWithVat.observe((price) => console.log("New price: " + price));

order.price(20);
// Prints: New price: 24
vat(0.10);
// Prints: New price: 22
```
#### o2 需要知道的

> 1. o1 什么时候改变?
> 2. o1 改变后的值
> 3. 如果o2也被依赖?

``` javascript
mobservable.value(() => order.price() * (1 + vat()));
```
传入mobx是函数就会设置为计算属性.改变 DNode的nextState 的指向;

```javascript
	observe(listener:(newValue:T, oldValue:T)=>void, fireImmediately=false):Lambda {
        this.dependencyState.setRefCount(+1); // awake
     	...
    }
```
这个会设置当前externalRefenceCount+1,在第一次observe的时候就会去调用wakeup方法.如果是计算属性的话会设置sleep为false,state 为PENDING.并且去调用computeNextState.

``` javascript
computeNextState() {
        this.trackDependencies();
        var stateDidChange = this.nextState();// 重要!!!
        this.bindDependencies();
        this.markReady(stateDidChange);
    }
```
这个方法非常重要,基本上所有关于本DNode的依赖关系都在这个方法完成计算.
来一个个看,这些方法做了什么

``` javascript
private trackDependencies() {
        this.prevObserving = this.observing;//正在依赖的DNode
        DNode.trackingStack[DNode.trackingStack.length] = [];
    }
```
设置prevObserving,因为要重新计算依赖.
DNode.trackingStack 是一个栈结构,用来处理DNode之间的依赖关系.
`this.nextState()`会指向 `compute`方法,检查是否有依赖改变

```javascript
    compute() {
        var newValue:U;
        try {
            // this cycle detection mechanism is primarily for lazy computed values; other cycles are already detected in the dependency tree
            if (this.isComputing)
                throw new Error("Cycle detected");
            this.isComputing = true;
            **newValue = this.func.call(this.scope);** 
            this.hasError = false;
        } catch (e) {
            this.hasError = true;
            console.error(this + "Caught error during computation: ", e);
            if (e instanceof Error)
                newValue = e;
            else {
                newValue = <U><any> new Error("MobservableComputationError");
                (<any>newValue).cause = e;
            }
        }
        this.isComputing = false;
        if (newValue !== this._value) {
            var oldValue = this._value;
            this._value = newValue;
            this.changeEvent.emit(newValue, oldValue);
            return true;
        }
        return false;
    }
```
**`  **newValue = this.func.call(this.scope);`**这段会去调用计算属性依赖的值.
会这么用
`order.priceWithVat = mobservable.value(() => order.price() * (1 + vat()));`
然后走get()方法 => 
如果是不是依赖属性的对象

```javascript
get():T {
        this.dependencyState.notifyObserved();
        return this._value;
    }
```
依赖属性的方法 
```javascript
 get():U {
        if (this.isComputing)
            throw new Error("Cycle detected");
    	var state = this.dependencyState;
        if (state.isSleeping) {
            if (DNode.trackingStack.length > 0) {
                // somebody depends on the outcome of this computation
                state.wakeUp(); // note: wakeup triggers a compute
                state.notifyObserved();
            } else {
                // nobody depends on this computable; so compute a fresh value but do not wake up
                this.compute();
            }
        } else {
            // we are already up to date, somebody is just inspecting our current value
            state.notifyObserved();
        }

        if (state.hasCycle)
            throw new Error("Cycle detected");
        if (this.hasError) {
            if (mobservableStatic.debugLevel) {
                console.trace();
                warn(`${this}: rethrowing caught exception to observer: ${this._value}${(<any>this._value).cause||''}`);
            }
            throw this._value;
        }
        return this._value;
    }
```
都会调用` this.dependencyState.notifyObserved();`方法
```javascript
 public notifyObserved() {
        var ts = DNode.trackingStack, l = ts.length;
        if (l > 0) {
            var cs = ts[l - 1], csl = cs.length;
            // this last item added check is an optimization especially for array loops,
            // because an array.length read with subsequent reads from the array
            // might trigger many observed events, while just checking the last added item is cheap
            if (cs[csl -1] !== this && cs[csl -2] !== this)
                cs[csl] = this;
        }
    }
``
拿到DNode.trackingStack,把自己放入当前栈这个集合的最后一个.

随后回到`computeNextState()`方法接着调用 `this.bindDependencies()`

```typescript

private bindDependencies() {
        this.observing = DNode.trackingStack.pop(); **将之前入栈的集合取出**

        if (this.isComputed && this.observing.length === 0 && mobservableStatic.debugLevel > 1 && !this.isDisposed) {
            console.trace();
            warn("You have created a function that doesn't observe any values, did you forget to make its dependencies observable?");
        }

        var [added, removed] = quickDiff(this.observing, this.prevObserving);// **计算依赖的不同**
        this.prevObserving = null;

        for(var i = 0, l = removed.length; i < l; i++)
            removed[i].removeObserver(this); // 取消之前依赖保留的对当前DNode的引用

        this.hasCycle = false;
        for(var i = 0, l = added.length; i < l; i++) {
            if (this.isComputed && added[i].findCycle(this)) {// **删除死循环调用**
                this.hasCycle = true;
                // don't observe anything that caused a cycle, or we are stuck forever!
                this.observing.splice(this.observing.indexOf(added[i]), 1);
                added[i].hasCycle = true; // for completeness sake..
            } else {
                added[i].addObserver(this); // **添加到依赖中**
            }
        }
    }
```

至此所有的依赖完成计算,然后如果有依赖发生改变,就会重新走一遍流程,完成新一轮的计算.


![MindMap][1]

.....


### watch
响应观测的对象,那么就要把观测的对象和本方法联系起来,也就是加入依赖追踪.

```javascript
mobx.watch(()=>mobx.value(),func);
watch = function (func,fn){
	const d = new DNode(true)  // 新建一个DNode
	let vatal
	d.nextState= function(){
	 vatal = func();         // 将依赖入栈,获取值
	 d.nextState = function(){ // 	   //  当依赖的值发生变化后的响应
	   d.depose();       // 删除依赖 -- 以便重新建立依赖
	   fn();               // 调用回调
	   return false;  // 返回 false -- 不去追踪依赖本Dnode 的值
	 }
	 return false;
	}
	
	d.computeNextState(); // 计算依赖
	return [vatal,()=>d.dispose()]
}

```

## defineObservableProperty

```
	function defineObservableProperty(obj,key,value){
		const atom = createObservableValue(value) // 对值包装
			definePropertyForObservable(obj,key,atom) // 作为属性加入对象
}

	function definePropertyForObservable(obj,key,atom){
		Object.defineProperty(obj,key,{
			get function(){
				return atom()
			},
			set function(v){
				atom(v)
			},
			enumerable:true,
			configurable:true;
		}
	}

```

## schedule

* ready 
	 ```javascript
	 DNode. makeReady(){
	 ...
	    if (this.observers.length === 0) // otherwise, let one of the observers do that :)
            Scheduler.scheduleReady();}
	 ```
	 每次依赖完成后触发
* batch
   ```JavaScript
   mobservable.batch(() => { 
  	order.price = 10;
  	order.amount = 3;
  	order.amount = 5;
	})
	order.total// 总是在batch完成后调用
   ```
   
   
   # ReactMixin!!!!
   

``` javascript
mobservableStatic.ReactObservingMixin = {
    componentWillMount: function() {
        var baseRender = this.render;
        this.render = function() {
            if (this._watchDisposer)
                this._watchDisposer();
            var[rendering, disposer] = mobservableStatic.watch(() => baseRender.call(this), () => {
                this.forceUpdate();
            });
            this._watchDisposer = disposer;
            return rendering;
        }
    },
    
    componentWillUnmount: function() {
        if (this._watchDisposer)
            this._watchDisposer();
    }
}
```
这里就是引入React的最初实现！
```javascript
var[rendering, disposer] = mobservableStatic.watch(() => baseRender.call(this), () => {
                this.forceUpdate();
            });
```
所以：
1.  render是函数，所以中间引用的各项依赖就会DNode入栈
2.  扩展render函数，渲染完成后其实render就是个observes
3.  每次依赖的数据改变，直接调用watch方法，直接render，从而不和react的生命周期有关，数据改变，重新渲染！
4.  mobx帮你做了数据的比对，改变就触发，所以粒度越细渲染越高效 -- [React的最佳实践][2]

ES5 React 的使用
[什么是mixin？][3]
```javascript
React.createClass({
mixin:[mobx.ReactObservingMixin]
})
```
ES6 和react的使用
```javascript
mobservableStatic.ObservingComponent = function(componentClass) {
    var baseMount = componentClass.componentWillMount;
    var baseUnmount = componentClass.componentWillUnmount;
    componentClass.prototype.componentWillMount = function() {
        mobservableStatic.ObserverMixin.componentWillMount.apply(this, arguments);
        return baseMount && baseMount.apply(this, arguments);
    };
    componentClass.prototype.componentWillUnmount = function() {
        mobservableStatic.ObserverMixin.componentWillUnmount.apply(this, arguments);
        return baseUnmount && baseUnmount.apply(this, arguments);
    };
    componentClass.prototype.shouldComponentUpdate = mobservableStatic.ObserverMixin.shouldComponentUpdate;
    return componentClass;
};
```

## Ts , 清理命名空间
> all internal methods / classes are now hidden in '_' namespace for clean top level api

like this :

``` typescript
namespace mobservable {

    export function array<T>(values?:T[]): _.ObservableArray<T> {
        return new _.ObservableArray(values);
    }

    export namespace _ {
        // Workaround to make sure ObservableArray extends Array
        class StubArray {
        }
        StubArray.prototype = [];
    
        export class ObservableArray<T> extends StubArray implements Mobservable.IObservableArray<T> {
            [n: number]: T;
    
```



  [1]: ./images/Screenshot%20from%202017-03-19%2022-17-35.png "Screenshot from 2017-03-19 22-17-35"
  [2]: https://mobx.js.org/best/react-performance.html
  [3]: https://segmentfault.com/a/1190000003016446