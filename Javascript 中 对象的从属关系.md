---
title: Javascript 中 对象的从属关系
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---

# Javascript 中 对象的从属关系

首先要确定的是 `javascript` 只有 以下 6 中基本类型

- null
- undefined
- object
- string
- number
- boolean

**其他类似 `function`,`Array`都是从属与 `object`**

## instanceOf
> 测试一个对象在其**原型链中**是否存在一个**构造函数**的prototype属性

### 语法

`object instanceof constructor`

除了`object`的基本类型外,使用 instanceof 不能检测其属性

```javascript
	
const s = '12313'

console.log(s instanceof Object);//false
```

对象的 .constructor 属性 是声明时的默认属性,但是由于 它是可写的,所以 使用 instanceof 来检测通常不是很可靠

```javascript
	var simpleStr = "This is a simple string"; 
	var myString  = new String();
	var newStr    = new String("String created with constructor");
	var myDate    = new Date();
	var myObj     = {};

	simpleStr instanceof String; // returns false, 检查原型链会找到 undefined
	myString  instanceof String; // returns true
	newStr    instanceof String; // returns true
	myString  instanceof Object; // returns true
	
```
并且,instanceof 只能用于 对象 和 函数(带有.prototype引用)之间的关系.

### isPrototypeOf()

在某条原型链中可以直接检测某个对象的原型链是否出现过.

```javascript
	class User{
  
	}

const u = new User()
console.log(User.prototype.isPrototypeOf(u)) // true
```

### Object.getPrototypeOf()

> 直接获取一个对象的的原型链

```javascript
	Object.getPrototypeOf(u) === User.prototype // true
```

###  (__proto\__)

> 一些浏览器支持 方法,用来访问内部的[[prototype]]属性

```javascript
u.__proto__ === User.prototype // true
```

# typeof
> 返回一个**字符串**,指示未经计算的操作数的类型

## 语法
`typeof operand`

![typeof][1]


除了上图提到的 Null 之所以返回的是 'object',js引擎判断一个对象是否是 `object`是看其转换为二进制时前三位是否为`000`, `null`为 `0000`,所以返回 `object`

上图中提到的Boolean,Number,String等是基本类型,不是包装类型.
没有提到的 包装类型 属于函数对象.所以

```javascript
	

console.log(typeof Array) // 'function'
console.log(typeof Boolean)//`function`
```

# Object.is
> 判断两个值是否是同一个值

## 语法

```
	Object.is(v1,v2)
```

Object.is() 会在下面这些情况下认为两个值是相同的：

 - 两个值都是 undefined
 -  两个值都是 null 
 -  两个值都是 true 或者都是 false
  - 两个值是由相同个数的字符按照相同的顺序组成的字符串 
  - 两个值指向同一个对象
  -  两个值都是数字并且
	  -   都是正零 +0 
	  -   都是负零 -0 
	  -   都是 NaN
	  -   都是除零和 NaN 外的其它同一个数字

与 `===`相似,但它不会把 -0 和 +0 这两个数值视为相同的，也不会会把两个 NaN 看成是不相等的


  [1]: ./images/%E6%B7%B1%E5%BA%A6%E6%88%AA%E5%9B%BE20170329155757.png "深度截图20170329155757"