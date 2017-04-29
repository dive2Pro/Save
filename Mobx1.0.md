---
title: Mobx1.0
tags: 源码,mobx,React
grammar_cjkRuby: true
---

> mobx的源码是用typescript写的,代码的许多技巧和结构都值得好好阅读一番. 

# index.ts

作为lib的入口,只向外暴露主要的使用接口方法.

```typescript
export {
	isObservable,
	isObservableObject,
	isObservableArray,
	isObservableMap,
	observable,
	extendObservable,
	asReference,
	asFlat,
	asStructure,
	observe,
	autorun,
	autorunUntil,
	autorunAsync,
	expr,
	transaction,
	toJSON,
	// deprecated, add warning?
	isObservable as isReactive,
	map,
	observable as makeReactive,
	extendObservable as extendReactive,
	autorunUntil as observeUntil,
	autorunAsync as observeAsync
} from './core';

```

core中包含了大部分日常使用的方法,用的最多的就是 observable

```typescript
export function observable(target:Object, key:string, baseDescriptor?:PropertyDescriptor):any;
export function observable<T>(value: T[]): IObservableArray<T>;
export function observable<T, S extends Object>(value: ()=>T, thisArg?: S): IObservableValue<T>;
export function observable<T extends string|number|boolean|Date|RegExp|Function|void>(value: T): IObservableValue<T>;
export function observable<T extends Object>(value: T): T;
export function observable(v:any, keyOrScope?:string | any) {
    ...
}
```
使用了typescript的函数重载,精确度最高的放在顶部.

在重载函数中 `value`值的类型有 函数,基本类型,和Object,但最顶端的observable中没有`value`参数,所以在函数体中是用`arguments`来获取参数的值

> 下面的代码都在 observble{ 这里 };

最上层的判断:
```typescript?linenums&fancy=1,2
    if (typeof arguments[1] === "string")
        return  observableDecorator apply(null, arguments);
    switch (arguments.length) {
        case 0:
            throw new Error("[mobservable.observable] Please provide at least one argument.");
        case 1:
            break;
        case 2:
            if (typeof v === "function")
                break;
            throw new Error("[mobservable.observable] Only one argument expected.");
        default:
            throw new Error("[mobservable.observable] Too many arguments. Please provide exactly one argument, or a function and a scope.");
    }
```

mobx支持 [decorator(装饰器)](https://tslang.cn/docs/handbook/decorators.html),@observable 的第二个参数值为目标成员的名字,如果是以装饰器模式使用`observable`,则就走[高亮行](#observableDecorator)方法
 

然后就是检查参数,函数重载中只有最下层的参数有效,所以这里检查的是v的类型.

----

如果目标值已经是Observable对象直接返回
```typescript
if (isObservable(v))
        return v;
```

---
v的值可以有多种类型,数组,对象,基本类型.数组和对象中又可以有不同的数组和对象作为成员,为了明确它们的子对象是否也需要进行observable,所以需要进行类型表明.这个控制权在使用者手中,在外部调用为`observable.asStruct`

[ValueMode](#valueMode)有四种类型;


```typescript
  let [mode, value] = getValueModeFromValue(v, ValueMode.Recursive);//>>== 获取指定的类型
  const sourceType = mode === ValueMode.Reference ? ValueType.Reference : getTypeOfValue(value); 
```
```typescript
//  返回值的类型
export function getTypeOfValue(value): ValueType {
	if (value === null || value === undefined)
		return ValueType.Reference;
	if (typeof value === "function")
		return value.length ? ValueType.ComplexFunction : ValueType.ViewFunction;
	if (Array.isArray(value) || value instanceof ObservableArray)
		return ValueType.Array;
	if (typeof value == 'object')
		return isPlainObject(value) ? ValueType.PlainObject : ValueType.ComplexObject;
	return ValueType.Reference; // safe default, only refer by reference..
}

```

这样 `ValueType` 和`ValueMode`都拿到:

```typescript
switch(sourceType) {
        case ValueType.Reference:
        case ValueType.ComplexObject:
            return toGetterSetterFunction(new ObservableValue(value, mode, null));
        case ValueType.ComplexFunction:
            throw new Error("[mobservable.observable] To be able to make a function reactive it should not have arguments. If you need an observable reference to a function, use `observable(asReference(f))`");
        case ValueType.ViewFunction: {
            const context = {
                name: value.name,
                object: value
            };
            return toGetterSetterFunction(new ObservableView(value, keyOrScope, context, mode === ValueMode.Structure));
        }
        case ValueType.Array:
        case ValueType.PlainObject:
            return makeChildObservable(value, mode, null);
    }
```
`new ObservableValue`和 `new ObservableView`都是要转化的类型,这里先不去看,
[makeChildObservable](#makeChildObservable)处理数组和普通的对象(普通对象形如 {value:"val"},复杂的形如 new Obj();),代码如下:
```typescript
export function makeChildObservable(value, parentMode:ValueMode, context) {
	let childMode: ValueMode;
	if (isObservable(value))
		return value;

	switch (parentMode) {
		case ValueMode.Reference:
			return value;
		case ValueMode.Flat:
			assertUnwrapped(value, "Items inside 'asFlat' canont have modifiers");
			childMode = ValueMode.Reference;
			break;
		case ValueMode.Structure:
			assertUnwrapped(value, "Items inside 'asStructure' canont have modifiers");
			childMode = ValueMode.Structure;
			break;
		case ValueMode.Recursive:
			[childMode, value] = getValueModeFromValue(value, ValueMode.Recursive);
			break;
		default:
			throw "Illegal State"; 
	}

	if (Array.isArray(value))
		return createObservableArray(value.slice(), childMode, context);
	if (isPlainObject(value))
		return extendObservableHelper(value, value, childMode, context);
	return value;
}
```
`createObservableArray`为静态函数,返回 `new ObservableArray(initialValues, mode, context)`;

`extendObservableHelper`:

```typescript?linenums&fancy=2
export function extendObservableHelper(target, properties, mode: ValueMode, context: IContextInfoStruct):Object {
	var meta = ObservableObject.asReactive(target, context, mode);// 返回 `new ObservableObject(target,context,mode)`
	for(var key in properties)
	    if (properties.hasOwnProperty(key)) {
		meta.set(key, properties[key]);
	}
	return target;
}
```
,[toGetterSetterFunction](#toGetterSetterFunction)接受一个Observable实例,将其用一个新对象来进行**代理**,
```typescript
export function toGetterSetterFunction<T>(observable: ObservableValue<T> | ObservableView<T>):IObservableValue<T> {
	var f:any = function(value?) {
		if (arguments.length > 0)
			observable.set(value);
		else
			return observable.get();
	};
	f.$mobservable = observable;
	f.observe = function(listener, fire) {
		return observable.observe(listener, fire);
	}
	f.toString = function() {
		return observable.toString();
	}
	return f;
}
```


---




observableDecorator 

```typescript

function observableDecorator(target:Object, key:string, baseDescriptor:PropertyDescriptor) {
    if (arguments.length < 2 || arguments.length > 3)
        throw new Error("[mobservable.@observable] A decorator expects 2 or 3 arguments, got: " + arguments.length);
 
    const isDecoratingGetter = baseDescriptor && baseDescriptor.hasOwnProperty("get");
    const descriptor:PropertyDescriptor = {};
    let baseValue = undefined;
	
	
    if (baseDescriptor) {
    // 获取定义的 value值,value的值定义的方法不同
	if (baseDescriptor.hasOwnProperty('get'))
            baseValue = baseDescriptor.get;
        else if (baseDescriptor.hasOwnProperty('value'))
            baseValue = baseDescriptor.value;
        else if ((<any>baseDescriptor).initializer) { // For babel
            baseValue = (<any>baseDescriptor).initializer();
            if (typeof baseValue === "function")
                baseValue = asReference(baseValue);
        }
    }

    if (!target || typeof target !== "object")
        throw new Error(`The @observable decorator can only be used on objects`);
    if (isDecoratingGetter) {
        if (typeof baseValue !== "function")
            throw new Error(`@observable expects a getter function if used on a property (in member: '${key}').`);
        if (descriptor.set) // 如果也有set那么会导致回环调用
            throw new Error(`@observable properties cannot have a setter (in member: '${key}').`);
        if (baseValue['length'] !== 0)
            throw new Error(`@observable getter functions should not take arguments (in member: '${key}').`);
    }

    descriptor.configurable = true;
    descriptor.enumerable = true;
    descriptor.get = function() {
        // the getter might create a reactive property lazily, so this might even happen during a view.
        withStrict(false, () => {
		// 获取this的$mobservable 中的数据
            ObservableObject.asReactive(this, null,ValueMode.Recursive).set(key, baseValue);
        });
        return this[key];
    };
    descriptor.set = isDecoratingGetter 
        ? throwingViewSetter(key)
        : function(value) { 
            ObservableObject.asReactive(this, null,ValueMode.Recursive)
			// 如果观察的是一个函数,标记value为引用值
                .set(key, typeof value === "function" ? asReference(value) : value);
        }
    ;
    if (!baseDescriptor) {
        Object.defineProperty(target, key, descriptor); // For typescript
    } else { 
        return descriptor;
    }
}

```





# dnode

> 之后就进入observableView和observablevalue中处理逻辑,这两个类都继承自 `DataNode`

## DataNode

> 被观察者,不能观察其他Node


`DataNode` 管理属性的观察者对象,在自身状态改变时去通知这些观察者.

定义的属性: 
```typescript
    id = ++mobservableId;
    state: NodeState = NodeState.READY;
    observers: ViewNode[] = [];       //  观察者们  是**ViewNode**类型的
    protected isDisposed = false;            // 标记是否可以被垃圾回收,没有其他的观察者
    externalRefenceCount = 0;      //  外部的对自身的依赖数量,类似标记垃圾回收法,如果值大于0,自身不会进入 sleep状态
    context: IContextInfoStruct;    // 上下文
```


改变外部的依赖数量:只在 autonrun中使用.   而实际上autonrun内部使用的是ObservableView对象.
```typescript
setRefCount(delta:number) {
        this.externalRefenceCount += delta;
    }
```

添加/删除观察者:通用方法
```typescript
addObserver(node:ViewNode) {
        this.observers[this.observers.length] = node;
    }
removeObserver(node:ViewNode) {
        var obs = this.observers, idx = obs.indexOf(node);
        if (idx !== -1)
            obs.splice(idx, 1);
    }	
```



改变状态,在子类中调用
```typescript
  markStale() {
        if (this.state !== NodeState.READY)
            return; // stale or pending; recalculation already scheduled, we're fine..
        this.state = NodeState.STALE;
        if (transitionTracker) // 用于全局的状态改变跟踪,mobxreact-devtools这些工具就依赖这个
            reportTransition(this, "STALE");
        this.notifyObservers();
    }
	
markReady(stateDidActuallyChange:boolean) {
        if (this.state === NodeState.READY)
            return;
        this.state = NodeState.READY;
        if (transitionTracker)
            reportTransition(this, "READY", true, this["_value"]);
        this.notifyObservers(stateDidActuallyChange);
    }
```

发送改变状态, 
```typescript
    notifyObservers(stateDidActuallyChange:boolean=false) {
        var os = this.observers.slice();// slice()
        for(var l = os.length, i = 0; i < l; i++)
            os[i].notifyStateChange(this, stateDidActuallyChange);
    }
```

函数被调用时,就尝试将自身放入 [mobservableViewStack](#mobservableViewStack)栈尾的observing中.
```typescript?linenums&fancy=4,10
 public notifyObserved() {
        var ts = global.__mobservableViewStack, l = ts.length;
        if (l > 0) {
            var deps = ts[l-1].observing, depslength = deps.length;
            // this last item added check is an optimization especially for array loops,
            // because an array.length read with subsequent reads from the array
            // might trigger many observed events, while just checking the latest added items is cheap
            // (n.b.: this code is inlined and not in observable view for performance reasons)
            if (deps[depslength -1] !== this && deps[depslength -2] !== this)
                deps[depslength] = this;
        }
    }
```


释放内存
```typescript
 public dispose() {
        if (this.observers.length)
            throw new Error("[mobservable] Cannot dispose DNode; it is still being observed");
        this.isDisposed = true;
    }
```







```typescript
export enum NodeState {
    STALE,     // One or more depencies have changed but their values are not yet known, current value is stale
    PENDING,   // All dependencies are up to date again, a recalculation of this node is ongoing or pending, current value is stale
    READY,     // Everything is bright and shiny
};
```

---




# observableValue

>  在core中,当 ValueType为 `Reference`或`ComplexObject`时被指定调用.
 

```typescript
constructor(protected value:T, protected mode:ValueMode, context: IContextInfoStruct){
        super(context);
        const [childmode, unwrappedValue] = getValueModeFromValue(value, ValueMode.Recursive);
        // If the value mode is recursive, modifiers like 'structure', 'reference', or 'flat' could apply
        if (this.mode === ValueMode.Recursive)
            this.mode = childmode;
        this._value = this.makeReferenceValueReactive(unwrappedValue);
    }
	 // 将值转化
	  private makeReferenceValueReactive(value) {
        return makeChildObservable(value, this.mode, this.context);
    }
```

改变值时,会将新的值转化,同时发送状态改变事件,通知观察者们
```typescript 
  set(newValue:T) {
        assertUnwrapped(newValue, "Modifiers cannot be used on non-initial values.");
        checkIfStateIsBeingModifiedDuringView(this.context);
        const changed = this.mode === ValueMode.Structure ? !deepEquals(newValue, this._value) : newValue !== this._value;
        // Possible improvement; even if changed and structural, apply the minium amount of updates to alter the object,
        // To minimize the amount of observers triggered.
        // Complex. Is that a useful case?
        if (changed) {
            var oldValue = this._value;
            this.markStale();
            this._value = this.makeReferenceValueReactive(newValue);
            this.markReady(true);
            this.changeEvent.emit(this._value, oldValue);
        }
        return changed;
    }

```
被调用就尝试入栈
```typescript 
 get():T {
        this.notifyObserved();
        return this._value;
    }
```

---

## ViewNode

> 核心, 可以观察其他Node,自己也可以被观察

属性:

```typescript 
     isSleeping = true; // isSleeping: 没有被观察,所以也不会去追踪观察的对象 
    hasCycle = false;  // cycler观察自身,true为出错
    observing: DataNode[] = [];       // 追踪的node. 值依赖于这些node
    private prevObserving: DataNode[] = null; // 之前观测中的nodes. 用来确定观测树的改变
    private dependencyChangeCount = 0;     //依赖(追踪)的node的值改变数量. 如果 > 0,  需要重新计算
    private dependencyStaleCount = 0;      //  依赖(追踪)的node的值将要改变的数量.  
```



```typescript 
 setRefCount(delta: number) {
        var rc = this.externalRefenceCount += delta; // 该值>0 则不会sleeep
        if (rc === 0)
            this.tryToSleep();
        else if (rc === delta) // a.k.a. rc was zero.
            this.wakeUp();
    }
```

移除追踪的node
```typescript 
   removeObserver(node: ViewNode) {
        super.removeObserver(node);
        this.tryToSleep();
    }
// ,如果没有其他node或者本身的依赖为0时,将自身从自己的观察者中移除
    tryToSleep() {
        if (!this.isSleeping && this.observers.length === 0 && this.externalRefenceCount === 0) {
            for (var i = 0, l = this.observing.length; i < l; i++)
                this.observing[i].removeObserver(this);
            this.observing = [];
            this.isSleeping = true;
        }
    }

```


唤醒
```typescript 
 wakeUp() {
        if (this.isSleeping) {
            this.isSleeping = false;
            this.state = NodeState.PENDING; // 正在计算,意味者本身的观察者的dependencyStaleCount+1
            this.computeNextState();
        }
    }

```

计算下一个状态
```typescript 
  computeNextState() {
        this.trackDependencies(); // 开辟新的mobxobservableStack空间
        if (transitionTracker)
            reportTransition(this, "PENDING");
        var hasError = true;
        var stateDidChange;
        try {
            withStrict(this.externalRefenceCount === 0,// 检查是否允许运行在严格模式修改状态
			() => {
                stateDidChange = this.compute(); // 抽象方法,需要子类实现
            });
            hasError = false;
        } finally {
            if (hasError)
            // TODO: merge with computable view, use this.func.toString
                console.error(`[mobservable.view '${this.context.name}'] There was an uncaught error during the computation of ${this.toString()}`);
            // TODO: merge with computable view, so this is correct:
            (<any>this).isComputing = false
            this.bindDependencies();  // 计算完成后,绑定新的依赖
            this.markReady(stateDidChange);//设置状态,已经完成计算
        }
    }
	
 private trackDependencies() {
        this.prevObserving = this.observing;
        this.observing = [];
        global.__mobservableViewStack[global.__mobservableViewStack.length] = this;
    }

```

```typescript 
   private bindDependencies() {
        global.__mobservableViewStack.length -= 1; // 移除栈尾的Node

        var [added, removed] = quickDiff(this.observing, this.prevObserving); // 比较并获取状态改变后的观察者对象,用来计算依赖树,保持最小更新
        this.prevObserving = null;

        this.hasCycle = false;
        for (var i = 0, l = added.length; i < l; i++) {
            var dependency = added[i];
            if (dependency instanceof ViewNode && dependency.findCycle(this)) {
                this.hasCycle = true;
                // don't observe anything that caused a cycle, or we are stuck forever!
                this.observing.splice(this.observing.indexOf(added[i]), 1);
                dependency.hasCycle = true; // for completeness sake..
            } else {
                added[i].addObserver(this);
            }
        }
        // remove observers after adding them, so that they don't go in lazy mode to early
        for (var i = 0, l = removed.length; i < l; i++)
            removed[i].removeObserver(this);
    }

```

makeready后被调用,通知自己的观察者,或者说,自己的观察者是主体 :   `observers.notifyStateChange(this,b)`
```typescript 
  // the state of something we are observing has changed..
    notifyStateChange(observable: DataNode, stateDidActuallyChange: boolean) {
        if (observable.state === NodeState.STALE) {
            if (++this.dependencyStaleCount === 1)//依赖还未改变的数量
                this.markStale();
        } else { // not stale, thus ready since pending states are not propagated
            if (stateDidActuallyChange)
                this.dependencyChangeCount += 1;//自己的依赖是否有状态的变化 && +1
            if (--this.dependencyStaleCount === 0) { // all dependencies are ready // 所以依赖是否都计算完成
                this.state = NodeState.PENDING;
                schedule(() => { 
                    if (this.dependencyChangeCount > 0) // 依赖都计算完成
                        this.computeNextState();  // 计算自己(就是状态改变的node的观察者)的下一个状态
                    else
                    // 完成工作,但是没有改变,通知自己的观察者 
                        this.markReady(false);  
                    this.dependencyChangeCount = 0;
                });
            }
        }
    }

```


----

## ObservableObject











## ValueMode


```typescript
export enum ValueMode {
	Recursive, // 如果value是普通对象,它及它的将来的子值也是reactive的.
	Reference, // Treat this value always as a reference, without any further processing.
	Structure, // Similar to recursive. However, this structure can only exist of plain arrays and objects.
				// No observers will be triggered if a new value is assigned (to a part of the tree) that deeply equals the old value.
	Flat       // If the value is an plain object, it will be made reactive, and so will all its future children.
}
```


## mobservableViewStack

> 把对象转化成 reactive

```typescript
   constructor(){
   ...


   Object.defineProperty(target, "$mobservable", {
   			enumerable: false,
   			configurable: false,
   			value: this  // 在 asReactive中代理使用
   		});
   }

   	static asReactive(target, context:IContextInfoStruct, mode:ValueMode):ObservableObject {
   		if (target.$mobservable)
   			return target.$mobservable;
   		return new ObservableObject(target, context, mode);
   	}


```

设置值的函数 重写:

```typescript
set(propName, value) {
		if (this.values[propName])
			this.target[propName] = value; // the property setter will make 'value' reactive if needed.
		else
			this.defineReactiveProperty(propName, value);//转化该值
	}
```


将键值对对象转化为$mobservable的reactive属性

```typescript

private defineReactiveProperty(propName, value) {
		let observable: ObservableView<any>|ObservableValue<any>;
		let context = {
			object: this.context.object,
			name: `${this.context.name || ""}.${propName}`
		};

		if (typeof value === "function" && value.length === 0)
			observable = new ObservableView(value, this.target, context, false);
		else if (value instanceof AsStructure && typeof value.value === "function" && value.value.length === 0)
			observable = new ObservableView(value.value, this.target, context, true);
		else
			observable = new ObservableValue(value, this.mode, context);

		this.values[propName] = observable;

		// 劫持propName属性的get,set方法
		Object.defineProperty(this.target, propName, {
			configurable: true,
			enumerable: observable instanceof ObservableValue,
			get: function() {
				return this.$mobservable ? this.$mobservable.values[propName].get() : undefined;
			},
			set: function(newValue) {
				const oldValue = this.$mobservable.values[propName].get();
				this.$mobservable.values[propName].set(newValue);
				this.$mobservable._events.emit(<IObjectChange<any, any>>{
					type: "update",
					object: this,
					name: propName,
					oldValue
				});
			}
		});

		this._events.emit(<IObjectChange<any, any>>{
			type: "add",
			object: this.target,
			name: propName
		});
	}
```

----

# ObservableView

> 继承自 ViewNode


get/set

```typescript
get():T {
       ...
        if (this.isSleeping) {
            if (isComputingView()) {
                //  有Node依赖于这个的计算值
                this.wakeUp(); // note: wakeup triggers a compute
                this.notifyObserved(); // 入栈,成为其他的依赖
            } else {
                //  不在其他的mobx栈中,只是更新一下值
                this.wakeUp();
                this.tryToSleep();
            }
        } else {
            // we are already up to date, somebody is just inspecting our current value
            this.notifyObserved();
        }

        if (this.hasCycle)
            throw new Error(`[mobservable.view '${this.context.name}'] Cycle detected`);
        return this._value;
    }
 // 计算属性不允许设置值
 set(x) {
        throwingViewSetter(this.context.name)();
    }
```

`compute` 将传入的 `func` 运行一遍获取返回值作为新值


```typescript

 compute() {
        // this cycle detection mechanism is primarily for lazy computed values; other cycles are already detected in the dependency tree
        if (this.isComputing)
            throw new Error(`[mobservable.view '${this.context.name}'] Cycle detected`);
        this.isComputing = true;
        const newValue = this.func.call(this.scope);
        this.isComputing = false;
        const changed = this.compareStructural ? !deepEquals(newValue, this._value) : newValue !== this._value;
        if (changed) {
            const oldValue = this._value;
            this._value = newValue;
            this.changeEvent.emit(newValue, oldValue);
            return true;
        }
        return false;
    }
```


对计算值的观测,是其他重要方法,比如`autorun`的实现基础:

```typescript
    observe(listener:(newValue:T, oldValue:T)=>void, fireImmediately=false):Lambda {
        this.setRefCount(+1); // 保持唤醒状态
        if (fireImmediately)
            listener(this.get(), undefined);
        var disposer = this.changeEvent.on(listener);
        // 返回的值再次调用就会被清除
        return once(() => {
            this.setRefCount(-1);
            disposer();
        });
    }

```


---

# ObservableArray

> 用自己的实现替换数组的方法

参与工作的有两大类:

   - ObservableArrayAdministration: $mobservable , 和reactive有关,处理状态(比如:长度,内容)的改变事件
   - ObservableArray: 数组类,在其原型上重写所有的数组方法

`ObservableArray`是继承自`StubArray`,它的定义很简单:

```typescript
export class StubArray {
}

StubArray.prototype = [];
```


劫持数组的部分原型函数,这些不会改变数组自身:

```typescript
/**
 * Wrap function from prototype
 */
[
    "concat",
    "every",
    "filter",
    "forEach",
    "indexOf",
    "join",
    "lastIndexOf",
    "map",
    "reduce",
    "reduceRight",
    "slice",
    "some",
].forEach(funcName => {
    var baseFunc = Array.prototype[funcName];
    Object.defineProperty(ObservableArray.prototype, funcName, {
        configurable: false,
        writable: true,
        enumerable: false,
        value: function() {
            this.$mobservable.notifyObserved();// 将自身放入mobxStacks中
            return baseFunc.apply(this.$mobservable.values, arguments);
        }
    });
});

```


## `ObservableArray`

构造:
```typescript
   constructor(initialValues:T[], mode:ValueMode, context: IContextInfoStruct) {
        super();
        Object.defineProperty(this, "$mobservable", {
            enumerable: false,
            configurable: false,
            value : new ObservableArrayAdministration(this, mode, context) // 和其绑定
        });

        if (initialValues && initialValues.length)
            this.replace(initialValues);            // 初始化值
    }

    // spliceWithArray 用来处理所有的数组拼接,并修改其长度并发出通知
     replace(newItems:T[]) {
            return this.$mobservable.spliceWithArray(0, this.$mobservable.values.length, newItems);
        }
```

数组中所有会改变自身的函数都被重写,且都会调用`spliceWithArray`函数:

```typescript
 clear(): T[] {
        return this.splice(0);
    }

    replace(newItems:T[]) {
        return this.$mobservable.spliceWithArray(0, this.$mobservable.values.length, newItems);
    }
    splice(index:number, deleteCount?:number, ...newItems:T[]):T[] {
        switch(arguments.length) {
            case 0:
                return [];
            case 1:
                return this.$mobservable.spliceWithArray(index);
            case 2:
                return this.$mobservable.spliceWithArray(index, deleteCount);
        }
        return this.$mobservable.spliceWithArray(index, deleteCount, newItems);
    }

    push(...items: T[]): number {
        this.$mobservable.spliceWithArray(this.$mobservable.values.length, 0, items);
        return this.$mobservable.values.length;
    }

    pop(): T {
        return this.splice(Math.max(this.$mobservable.values.length - 1, 0), 1)[0];
    }

    shift(): T {
        return this.splice(0, 1)[0]
    }

    unshift(...items: T[]): number {
        this.$mobservable.spliceWithArray(0, 0, items);
        return this.$mobservable.values.length;
    }

    reverse():T[] {
        return this.replace(this.$mobservable.values.reverse());
    }

    sort(compareFn?: (a: T, b: T) => number): T[] {
        return this.replace(this.$mobservable.values.sort.apply(this.$mobservable.values, arguments));
    }

    remove(value:T):boolean {
        var idx = this.$mobservable.values.indexOf(value);
        if (idx > -1) {
            this.splice(idx, 1);
            return true;
        }
        return false;
    }
 ```

## ObservableArrayAdministration

内部的 `spliceWithArray`是核心,


```typescript

spliceWithArray(index:number, deleteCount?:number, newItems?:T[]):T[] {
        var length = this.values.length;
        ... // 计算数量

        if (newItems === undefined)
            newItems = [];
        else  //将子类转化为reactive
            newItems = <T[]> newItems.map((value) => this.makeReactiveArrayItem(value));

        var lengthDelta = newItems.length - deleteCount;
        this.updateLength(length, lengthDelta); //  更新数组长度
        var res:T[] = this.values.splice(index, deleteCount, ...newItems);
        // 通知拼接的事件
        this.notifySplice(index, res, newItems);
        return res;
    }
```




目前只做到自身的长度改变时发出事件,有一个疑问,那么如果某个子类的值的改变,在哪里响应?
类中有这个方法`notifyChildUpdate`,肯定有关,发现是在

```typescript
// 劫持 set/get函数
function createArrayBufferItem(index:number) {
   // 生成PropertyDescriptor到ENUMERABLE_PROPS数组中
    var prop = ENUMERABLE_PROPS[index] = {
        enumerable: true,
        configurable: true,
        set: function(value) {
            const impl = this.$mobservable;
            const values = impl.values;
            assertUnwrapped(value, "Modifiers cannot be used on array values. For non-reactive array values use makeReactive(asFlat(array)).");
            if (index < values.length) {
                checkIfStateIsBeingModifiedDuringView(impl.context);
                var oldValue = values[index];
                var changed = impl.mode === ValueMode.Structure ? !deepEquals(oldValue, value) : oldValue !== value;
                if (changed) {
                    values[index] = impl.makeReactiveArrayItem(value);
                    impl.notifyChildUpdate(index, oldValue); // 这里被调用
                }
            }
            else if (index === values.length)
                this.push(impl.makeReactiveArrayItem(value));
            else
                throw new Error(`[mobservable.array] Index out of bounds, ${index} is larger than ${values.length}`);
        },
        get: function() {
            const impl = this.$mobservable;
            if (impl && index < impl.values.length) {
                impl.notifyObserved();
                return impl.values[index];
            }
            return undefined;
        }
    };
    Object.defineProperty(ObservableArray.prototype, "" + index, {
        enumerable: false,
        configurable: true,
        get: prop.get,
        set: prop.set
    });
}

```

同时,updateLength的调用链上也有这个方法:

```typescript
 private updateLength(oldLength:number, delta:number) {
        if (delta < 0) {
            checkIfStateIsBeingModifiedDuringView(this.context);
            for(var i = oldLength + delta; i < oldLength; i++)
                delete this.array[i]; // bit faster but mem inefficient:
                //Object.defineProperty(this, <string><any> i, notEnumerableProp);
        } else if (delta > 0) {
            checkIfStateIsBeingModifiedDuringView(this.context);
            if (oldLength + delta > OBSERVABLE_ARRAY_BUFFER_SIZE)
                reserveArrayBuffer(oldLength + delta);// 通知改变
            // funny enough, this is faster than slicing ENUMERABLE_PROPS into defineProperties, and faster as a temporarily map
            for (var i = oldLength, end = oldLength + delta; i < end; i++)
                Object.defineProperty(this.array, <string><any> i, ENUMERABLE_PROPS[i])//拿到改变该位置的PropertyDescriptor
        }
    }
```

在mobx中数组初始长度为 1000,这样是出于什么考虑?性能:在修改值时已经在reactive中,日常使用的
大部分数组不超过该长度时,不需要重新来转化

```typescript
  /**
    * This array buffer contains two lists of properties, so that all arrays
    * can recycle their property definitions, which significantly improves performance of creating
    * properties on the fly.
    *
    */
var OBSERVABLE_ARRAY_BUFFER_SIZE = 0;
var ENUMERABLE_PROPS : PropertyDescriptor[] = [];

function reserveArrayBuffer(max:number) {
    for (var index = OBSERVABLE_ARRAY_BUFFER_SIZE; index < max; index++)
        createArrayBufferItem(index);
    OBSERVABLE_ARRAY_BUFFER_SIZE = max;
}
// 直接调用该函数,分配长度
reserveArrayBuffer(1000);

```




# 主要方法的实现

## autorun

使用方法:

```typescript
autorun(()=>{
 回调函数内需要有observable元素
})
```

源码 :

```typescript
function autorun(){

   ...
  // 回调函数会在 compute中被调用,期间开辟的新栈供依赖绑定

  const observable = new ObservableView(unwrappedView, scope, {
        object: scope || view,
        name: view.name
    }, mode === ValueMode.Structure);
    // 保持唤醒状态
    observable.setRefCount(+1);
    const disposer = once(() => {
        observable.setRefCount(-1);
    });
    (<any>disposer).$mobservable = observable;

    return disposer;

}
```






