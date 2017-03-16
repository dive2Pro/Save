---
title: Sorting Algorithms
tags: Sort,Algorithm,Bubble Sort,Insertion Sort,Quick Sort
grammar_cjkRuby: true
---

 

 1.  [Bubble Sort](#bubble-sort)


# [Bubble Sort][1]
 ![enter description here][2]
冒泡排序,最容易理解也最符合人自然想法的排序算法.
但是也属于这个领域效率最低的算法的一种.
在冒泡排序中,循环比较数组来比较每个位置的和其下一个位置的数字的大小.如果这两个数字不相等就交换位置.不停的遍历直到在最后的循环中没有数字产生交换操作.
至于冒泡的Big o是多少? 这里有一个内部循环来检查位置上数字的大小,一个外部循环来检测是否有交换发生所以是**O(n2)**.很不效率,所以在**任何情况下都不应该使用冒泡排序**.

``` javascript
	function bubbleSort(nums){
		var swaped=true;
		do(){
			swaped=false;
			for(var i =0; i<nums.length-1;i++){
				if(nums[i]<nums[i+1]){
					var temp = nums[i];
					nums[i]=nums[i+1];
					nums[i+1]=temp;
				}
			}
		}while(swaped);
	 return nums;
	}
```

# [Insertion Sort][3]
 ![enter description here][4]
插入排序要比冒泡排序复杂一些,但是在一些情况下效率要高一些:当一个list中已经有一定的顺序存在.


 ```javascript
   function insertionSort(nums){
   let i,k,temp;
   for(i=0;i<nums.length;i++){
   		temp = nums[i];
		k = i-1;
		for(;k>=0&&nums[k]>temp;){
			k--;
			nums[k+1]=nums[k]
		}
		nums[k] = temp; 
   	}
   }
 ```
它的Big O?两层循环很显然是 **O(n2)**,但是如果该数组已经很接近有序排列的话,效率可以达到**O(n)**;


# [Merge Sort][5]

![enter description here][6]
第一个分治法[递归]算法!也是最容易理解的递归算法.

基本思路是该函数将一个大的数组第一步先分为两块,然后进行递归,并将递归后的返回值进行归并,而每层递归返回的都是已经排序好的数组.

归并算法的效率非常高,事实上,Array.prototype.sort内部经常使用的就是归并算法(视js引擎和要排序的数组的类型而定).归并算法不会改变输入的数组.

归并算法的  Big o 是 **O(log n)**. 
空间复杂度是**O(n)**,因为它需要开辟一段和输入数组大小的空间来保存排序的数组.

``` javascript
	function mergeSort(nums){
		const len = nums.length;
		if(len<2) return nums;
		const mid = len/2;
		function merge(left,right){
			const final =[];
			// 合
			while(left.length&&right.length){
				// 比较
				const v = right[0]>left[0]? left.shift():right.shift();
				final.push(v);
			}
			return final.concat(left.concat(right));
		}
		// 分
		let left =nums.slice(0,mid);
		let right = nums.slice(mid);
		return merge(mergeSort(left),mergeSort(right));
	}
```

# [Quick Sort][7]
![enter description here][8]


快速排序是最有用并且也是最强大的算法之一.
属于分治法的一种,但是又略有不同

``` javascript
let nums = [10,2,111,13,8,6,5,1,3,7,4,9];

function inplaceQuickSort(nums){
    function swap(arr,i,k){
      const temp = arr[i];
      arr[i]=arr[k];
      arr[k]=temp;
      
    }
    function partition(arr,left,right){
      let index = left;
      const pivot = arr[right];
      for(let i =left;i<right;i++){
        if(arr[i]<pivot){
          swap(arr,index++,i);
        }
      }
      swap(arr,index,right);
      return index;
    }
    function sort(arr,left,right){ 
      if(left>=right)return;
      const index = partition(arr,left,right);
      // 注意边界问题,中间的已经是两边的最值了
      sort(arr,left,index-1);
      sort(arr,index+1,right);
    }
    sort(nums,0,nums.length-1);
}
```

快速排序也是**O(nlog n)**,但常规的也最多需要**O(n)**的空间,可能要比归并排序需要的内存少,
**inplace**原地快速排序,不需要额外的内存.
但是也有自己的缺点,如果传入的是一个已经排序完成的数组,它依然要递归比较,效率就不高.


  [1]: https://www.wikiwand.com/zh/%E5%86%92%E6%B3%A1%E6%8E%92%E5%BA%8F
  [2]: http://btholt.github.io/four-semesters-of-cs/img/bubble.gif
  [3]: https://www.wikiwand.com/zh/%E6%8F%92%E5%85%A5%E6%8E%92%E5%BA%8F
  [4]: http://btholt.github.io/four-semesters-of-cs/img/insertion.gif
  [5]: https://www.wikiwand.com/zh/%E5%BD%92%E5%B9%B6%E6%8E%92%E5%BA%8F
  [6]: http://btholt.github.io/four-semesters-of-cs/img/merge.gif
  [7]: https://www.wikiwand.com/zh/%E5%BF%AB%E9%80%9F%E6%8E%92%E5%BA%8F
  [8]: http://btholt.github.io/four-semesters-of-cs/img/quick2.gif