---
title: "Golang排序"
date: 2022-03-07T14:12:47+08:00
draft: false
categories: ["Go"]
tags: ["源码", "排序"]
---

## 问题

golang的排序函数非常强大，对于数组切片，可以很方便地进行排序，那么他是怎么实现的呢？如果对于一个struct切片，要想根据其中一个field从大到小的顺序进行排列可以实现吗？怎么实现？

## 探索

首先进入包内看一下方法实现：
```go
// 实现了Interface接口的类型可以被此包中的routines排序
// 这些方法通过整数索引引用基础集合的元素
type Interface interface {
    // Len 集合中的元素个数
    Len() int
    
    // Less 反映了索引i处元素是否必须排列在索引j处元素之前
    //
    // 如果Less(i, j)和Less(j, i)均返回false
    // 那么索引i处元素和索引j处元素视为相等
    // Sort 可能在最终的结果中以任意顺序排列相等的元素（不稳定）
    // 稳定 即保留了相同元素的原始输入顺序
    //
    // Less必须描述一个传递排序：
    //  - 如果Less(i, j)和Less(j, k)均返回true，那么Less(i, k)也必须返回true。
    //  - 如果Less(i, j)和Less(j, k)军返回false，那么Less(i, k)也必须返回false。
    //
    // 注意浮点型比较（float32或float64类型的值上的<运算符）
    // 当涉及非数字not-a-number (NaN) 值时，它不是传递排序
    // 对于浮点型值的正确实现参见 Float64Slice.Less
    Less(i, j int) bool
    
    // Swap 交换索引i处和索引j处的值
    Swap(i, j int)
}


// Sort 对数据排序
// 通过调用data.Len来判断n，并且调用O（n*log(n)）次的data.Less和data.Swap。排序不保证稳定
func Sort(data Interface) {
	n := data.Len()
	quickSort(data, 0, n, maxDepth(n))
}
```

通过上面的源码，可以知道：
1. 要想使用sort.Sort进行排序，被排序元素必须实现sort.Interface接口
2. sort.Sort排序不稳定

接着看`quickSort`的实现源码
```go

// maxDepth 返回由快速排序切换到堆排序的阈值，即 2*ceil(lg(n+1))
func maxDepth(n int) int {
	var depth int
	for i := n; i > 0; i >>= 1 {
		depth++
	}
	return depth * 2
}

// Quicksort, loosely following Bentley and McIlroy,
// ``Engineering a Sort Function,'' SP&E November 1993.

// medianOfThree moves the median of the three values data[m0], data[m1], data[m2] into data[m1].
func medianOfThree(data Interface, m1, m0, m2 int) {
    // sort 3 elements
    if data.Less(m1, m0) {
        data.Swap(m1, m0)
    }
    // data[m0] <= data[m1]
    if data.Less(m2, m1) {
        data.Swap(m2, m1)
        // data[m0] <= data[m2] && data[m1] < data[m2]
        if data.Less(m1, m0) {
            data.Swap(m1, m0)
        }
    }
    // now data[m0] <= data[m1] <= data[m2]
}


func doPivot(data Interface, lo, hi int) (midlo, midhi int) {
	m := int(uint(lo+hi) >> 1) // Written like this to avoid integer overflow.
	if hi-lo > 40 {
		// Tukey's ``Ninther,'' median of three medians of three.
		s := (hi - lo) / 8
		medianOfThree(data, lo, lo+s, lo+2*s)
		medianOfThree(data, m, m-s, m+s)
		medianOfThree(data, hi-1, hi-1-s, hi-1-2*s)
	}
	medianOfThree(data, lo, m, hi-1)

	// Invariants are:
	//	data[lo] = pivot (set up by ChoosePivot)
	//	data[lo < i < a] < pivot
	//	data[a <= i < b] <= pivot
	//	data[b <= i < c] unexamined
	//	data[c <= i < hi-1] > pivot
	//	data[hi-1] >= pivot
	pivot := lo
	a, c := lo+1, hi-1

	for ; a < c && data.Less(a, pivot); a++ {
	}
	b := a
	for {
		for ; b < c && !data.Less(pivot, b); b++ { // data[b] <= pivot
		}
		for ; b < c && data.Less(pivot, c-1); c-- { // data[c-1] > pivot
		}
		if b >= c {
			break
		}
		// data[b] > pivot; data[c-1] <= pivot
		data.Swap(b, c-1)
		b++
		c--
	}
	// If hi-c<3 then there are duplicates (by property of median of nine).
	// Let's be a bit more conservative, and set border to 5.
	protect := hi-c < 5
	if !protect && hi-c < (hi-lo)/4 {
		// Lets test some points for equality to pivot
		dups := 0
		if !data.Less(pivot, hi-1) { // data[hi-1] = pivot
			data.Swap(c, hi-1)
			c++
			dups++
		}
		if !data.Less(b-1, pivot) { // data[b-1] = pivot
			b--
			dups++
		}
		// m-lo = (hi-lo)/2 > 6
		// b-lo > (hi-lo)*3/4-1 > 8
		// ==> m < b ==> data[m] <= pivot
		if !data.Less(m, pivot) { // data[m] = pivot
			data.Swap(m, b-1)
			b--
			dups++
		}
		// if at least 2 points are equal to pivot, assume skewed distribution
		protect = dups > 1
	}
	if protect {
		// Protect against a lot of duplicates
		// Add invariant:
		//	data[a <= i < b] unexamined
		//	data[b <= i < c] = pivot
		for {
			for ; a < b && !data.Less(b-1, pivot); b-- { // data[b] == pivot
			}
			for ; a < b && data.Less(a, pivot); a++ { // data[a] < pivot
			}
			if a >= b {
				break
			}
			// data[a] == pivot; data[b-1] < pivot
			data.Swap(a, b-1)
			a++
			b--
		}
	}
	// Swap pivot into middle
	data.Swap(pivot, b-1)
	return b - 1, c
}


func quickSort(data Interface, a, b, maxDepth int) {
	for b-a > 12 { // 当元素个数小于等于12时采用希尔排序
		if maxDepth == 0 { // 当阈值等于0时采用堆排序
			heapSort(data, a, b)
			return
		}
		maxDepth--
		mlo, mhi := doPivot(data, a, b)
		// 
		// Avoiding recursion on the larger subproblem guarantees
		// a stack depth of at most lg(b-a).
		if mlo-a < b-mhi {
			quickSort(data, a, mlo, maxDepth)
			a = mhi // i.e., quickSort(data, mhi, b)
		} else {
			quickSort(data, mhi, b, maxDepth)
			b = mlo // i.e., quickSort(data, a, mlo)
		}
	}
	if b-a > 1 {
		// Do ShellSort pass with gap 6
		// It could be written in this simplified form cause b-a <= 12
		for i := a + 6; i < b; i++ {
			if data.Less(i, i-6) {
				data.Swap(i, i-6)
			}
		}
		insertionSort(data, a, b)
	}
}
```

从上到下看下来，让人叹为观止！这里的快速排序并非常规的快排，而是根据拆分的程度采用不同的排序方法：
1. 当集合中元素个数大于12时，用快排拆分
2. 当阈值为0时，切换为堆排序
3. 当集合中元素小于等于12是用希尔排序，间隙为6
4. 当个数小于6时，采用插入排序

## 解答

回到上面的问题，例：一个班级有多个学生，现在要通过学生的成绩从高到底一次排列，如何实现？
1. 申明一个Student结构体
2. 申明一个实质为[]Student的Students type
3. 为Students实现`Len`、`Less`、`Swap`方法
4. 调用sort.Sort
```go
package main

import (
	"fmt"
	"math/rand"
	"sort"
	"strconv"
)

type Student struct {
	Name  string
	Score int
}

type Students []Student

func main() {

	var students Students
	for i := 0; i < 10; i++ {
		students = append(students, Student{
			Name:  strconv.Itoa(i),
			Score: rand.Intn(100),
		})
	}

	fmt.Println(students)
	sort.Sort(students)
	fmt.Println(students)
}

func (s Students) Len() int {
	return len(s)
}

func (s Students) Less(i, j int) bool {
	return s[i].Score > s[j].Score
}

func (s Students) Swap(i, j int) {
	s[i], s[j] = s[j], s[i]
}
```
注意当要实现从大到小时`Less`的实现

最终输出：
```shell
[{0 81} {1 87} {2 47} {3 59} {4 81} {5 18} {6 25} {7 40} {8 56} {9 0}]
[{1 87} {0 81} {4 81} {3 59} {8 56} {2 47} {7 40} {6 25} {5 18} {9 0}]
```
