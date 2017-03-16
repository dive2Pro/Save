---
title: Data Structures
tags:  Interfaces,Implementations
grammar_cjkRuby: true
---
# Interfaces

 ## Set
 
 
 1.  数组的add,remove,contains,toList
 2.  不保证顺序
 3.  去重,数据的唯一性

## Map

1. 类似js的对象.
2. 一组set/collection作为keys并且有这些key所关联的值.
3. 不同于js的objects,Map没有prototypes,inheritance,methods等
4. 键值唯一


## Stack
> Last-In First-Out
![enter description here][1]
类似的例子比如激活函数的时候:

``` javascript
function double(x) { return 2 * x; }
function squareAndAddFive(y) { return square(y) + 5; }
function square(z) { return z * z; }

function maths(num) {
    var answer = double(num);
    answer = squareAndAddFive(answer);
    return answer;
}

maths(5);
```
### js是如何控制这段代码的

1. maths 使用, js将 maths 入栈
2. 在maths内部,double使用,js将double入栈
3. double完成,返回10,js将double推出栈
4. 回到maths,squareAndAddFive被调用,将其入栈
5. 在squareAndAddFive内部,square被调用,js将其入栈

目前的栈内顺序:
square
squareAndAddFive
maths
main

6. square返回100 ,出栈
7. squareAndAddFive 返回105
8. maths完成调用 返回105


## Queue
>First-In First-Out
![enter description here][2]


# Implementations

## [Array List][3]

与java中的ArrayList不同的是,js将集合中的元素删除后,需要手动去调整元素的位置

``` javascript
const arr=[1,2,3,4,5,6];
delete arr[3];
console.log(arr)//[1, 2, 3, undefined, 5, 6]
arr.splice(3,1)
console.log(arr)//[1, 2, 3, 5, 6]
```
所以常用array.splice方法来替代delete.

## [LinkedList][4]
![enter description here][5]

如图,LinkedList内部元素都有两个属性,value和指向下一个node的next.
和ArrayList不同的是,
添加和删除头尾节点非常快,只需要改变next节点的值.**O(1)**
但调用.get方法的话,内部会遍历直至找到目标值.**O(n)**
和Arraylist刚好相反.


## Binary Search Tree
![enter description here][6]


  [1]: http://btholt.github.io/four-semesters-of-cs/img/stack.png
  [2]: http://btholt.github.io/four-semesters-of-cs/img/queue.png
  [3]: https://docs.oracle.com/javase/8/docs/api/java/util/ArrayList.html
  [4]: https://docs.oracle.com/javase/8/docs/api/java/util/LinkedList.html
  [5]: http://btholt.github.io/four-semesters-of-cs/img/linkedlist.gif
  [6]: http://btholt.github.io/four-semesters-of-cs/img/bst.png