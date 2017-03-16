---
title: Big O 
tags: Algorithm
grammar_cjkRuby: true
---


```
	function crossAdd(input) {
		var answer = [];
		for (var i = 0; i < input.length; i++) {
			var goingUp = input[i];
			var goingDown = input[input.length-1-i];
			answer.push(goingUp + goingDown);
		}
		return answer;
	}
```
这是O(n)，因为遍历所有的输入只有一层循环。

```
function makeTuples(input) {
    var answer = [];
    for (var i = 0; i < input.length; i++) {
        for (var j = 0; j < input.length; j++) {
            answer.push([input[i], input[j]]);
        }
    }
    return answer;
}
```
这是 **O(n2)**,对每一个输入,都经过外部和内部两层循环.所以在代码中寻找循环是算法分析的一个妙计.如果两层中再嵌套一层的话就是**O(n3)**

如果使用 [分治法](DAC)[1]进行编码的话,经常会见到**O(log n)**,这意味着如果你添加更多的 ==meaning as you add more terms, the increases in time as you add input diminishes==
  
  递归,就是用它自己来定义某些东西.比如当有人问你"什么是*计算机*?","一种可以用来*计算*的东西".

在计算机领域谈论递归,是在讨论一个可以调用自己的函数.这个技巧特别适用于一些特定的问题,因为在递归的不同层级也可以保持状态.



  [1]: https://www.wikiwand.com/zh/%E5%88%86%E6%B2%BB%E6%B3%95