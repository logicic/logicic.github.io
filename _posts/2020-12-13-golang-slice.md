---
layout:     post
title:      golang-slice
subtitle:   slice分析
date:       2020-12-13
author:     logicic
catalog: true
tags:
    - golang
---



# golang slice分析

## Reference：

1. https://docs.kilvn.com/go-internals/ref2.html
2. https://github.com/golang/go/blob/master/src/runtime/slice.go
3. https://github.com/qcrao/Go-Questions/blob/master/%E6%95%B0%E7%BB%84%E5%92%8C%E5%88%87%E7%89%87/%E5%88%87%E7%89%87%E4%BD%9C%E4%B8%BA%E5%87%BD%E6%95%B0%E5%8F%82%E6%95%B0.md
4. unsafe：https://zhuanlan.zhihu.com/p/67852800

**本文根据：是什么，为什么，怎么做来描述**

### golang源码slice的定义

```go
type slice struct {
	array unsafe.Pointer 
	len   int
	cap   int
}
```

- 指针： unsafe包下的所有变量都是不安全的，因为给予了非常大的权限。比如此处的指针可以对内存随意的操作，如果不是对变量内存的产生释放流程十分了解的话，不建议使用，平常go中日常日常的使用的指针是被阉割过的，权限仅有一些，比如可以方便编程时的作为函数参数进行址传而不是拷贝副本，对指针的数学运算是不允许的。如果要想对内存地址进行数学运算的话：

  需要把pointer转换成 uintptr，对 uintptr 类型进行数学运算，再转换成 pointer 类型。

![uintptr-pointer-T](https://user-images.githubusercontent.com/81942428/115107532-fb50f200-9f9d-11eb-9dde-dad3244b9b23.png)

```go
var pointA *int
var val int = 12
pointA = &val
pointA++  // err
```

- 长度：slice的有效数据大小，即可以程序可以读出的长度

- 容量：slice的可以存放数据大小，即当需要增加数据的时候可以增加len，而不用new空间
- 长度和容量的区别：



### 为什么会有slice产生，和go的array的区别

[go中的array和c中的array的区别](https://blog.csdn.net/wangkai_123456/article/details/72511037)

**Arrays are values, not implicit pointers as in C.**

- Go数组做参数时, 需要被检查长度.

```go
func use_array(args [4]int) {
	args[1] = 100
}

func main() {
	var args = [5]int{1, 2, 3, 4, 5}
	use_array(args) // cannot use args (type [5]int) as type [4]int in argument to use_array
	fmt.Println(args)
}
```

- go数组变量名不等于数组开始指针

```go
func use_array(args [5]int) {
	args[1] = 100
}

func main() {
	var args = [5]int{1, 2, 3, 4, 5}
	use_array(args)
  fmt.Println(args) // print: [1 2 3 4 5] 并没有更改结果
}

// 要想能够做到func能够更改变量数据需要添加取址符&
func use_array(args *[4]int) {
	args[1] = 100 //但是使用还是和C一致,不需要别加"*"操作符
}

func main() {
	var args = [4]int{1, 2, 3, 4}
	use_array(&args) // 数组名已经不是表示地址了, 需要使用"&"得到地址
  fmt.Println(args) // print: [1 100 3 4]
}
```

- 如果p2array为指向数组的指针，*p2array不等于p2array[0]

```go
import (
	"fmt"
)

func main() {
	var p2array *[3]int
	p2array = new([3]int)
  fmt.Printf("%x\n", *p2array+1) // err : invalid operation: *p2array + 1 (mismatched types [3]int and int)
}

func main() {
	var p2array *[3]int
	p2array = new([3]int)
  fmt.Printf("%x\n", *p2array) // print: [0 0 0]
}
```

- 获取数组中某一位置的地址方法

```go
func use_array(args *[4]int) {
	args[1] = 100                            //但是使用还是和C一致,不需要别加"*"操作符
	fmt.Printf("%v\n", unsafe.Pointer(args)) //获取数组地址方法1
}

func main() {
	var args = [4]int{1, 2, 3, 4}
	use_array(&args)             // 数组名已经不是表示地址了, 需要使用"&"得到地址
	fmt.Printf("%v\n", &args[0]) //获取数组地址方法2
	fmt.Println(args)
}
/*
0xc000014020
0xc000014020
[1 100 3 4]
*/

// 如果想通过方法1获取别的位置的地址，需要uintptr，这个就是相当于c中的*void，随意使用没有任何限制的指针
func use_array(args *[4]int) {
	args[1] = 100                            //但是使用还是和C一致,不需要别加"*"操作符
	b := uintptr(unsafe.Pointer(args))+8
	ab := unsafe.Pointer(b)
	fmt.Printf("%v\n", ab) //获取数组地址方法1
}

func main() {
	var args = [4]int{1, 2, 3, 4}
	use_array(&args)             // 数组名已经不是表示地址了, 需要使用"&"得到地址
	fmt.Printf("%v\n", &args[1]) //获取数组地址方法2
	fmt.Println(args)
}
/*
output：
./prog.go:11:8: possible misuse of unsafe.Pointer
Go vet exited.

0xc00010c008
0xc00010c008
[1 100 3 4]
*/
```



[slice和go-array的区别](https://github.com/qcrao/Go-Questions/blob/master/%E6%95%B0%E7%BB%84%E5%92%8C%E5%88%87%E7%89%87/%E6%95%B0%E7%BB%84%E5%92%8C%E5%88%87%E7%89%87%E6%9C%89%E4%BB%80%E4%B9%88%E5%BC%82%E5%90%8C.md)

数组是定长的，长度定义好之后，不能再更改。在 Go 中，数组是不常见的，因为其长度是类型的一部分，限制了它的表达能力，比如 [3]int 和 [4]int 就是不同的类型。而slice则非常灵活，它可以动态地扩容。slice的类型和长度无关。**因为go限制了指针的能力，所以go的array不能像c那样方便使用，所以才产生了slice，方便大家使用。（我猜）**

![slice-struct](https://user-images.githubusercontent.com/81942428/115107763-71a22400-9f9f-11eb-8eb5-5ef987b7bc33.png)

**注意，底层数组是可以被多个 slice 同时指向的，因此对一个 slice 的元素进行操作是有可能影响到其他 slice 的。**



### 怎么使用slice呢？使用场景是什么？注意事项

由于：**底层数组是可以被多个 slice 同时指向的，因此对一个 slice 的元素进行操作是有可能影响到其他 slice 的。**所以如果使用不注意的话，会产生许多与程序猿本意相悖的结果。

```go
package main

import "fmt"

func main() {
	slice := []int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
	s1 := slice[2:5]
	s2 := s1[2:6:7]

	s2 = append(s2, 100)
	s2 = append(s2, 200)

	s1[2] = 20

	fmt.Println(s1)
	fmt.Println(s2)
	fmt.Println(slice)
}

/*
output：
[2 3 20]
[4 5 6 7 100 200]
[0 1 2 3 20 5 6 7 100 9]
*/
```

`s1` 从 `slice` 索引2（闭区间）到索引5（开区间，元素真正取到索引4，也可以理解为长度，从0开始数5个数），长度为3，容量默认到数组结尾，为8。 `s2` 从 `s1` 的索引2（闭区间）到索引6（开区间，元素真正取到索引5），容量到索引7（开区间，真正到索引6），为5。

![slice-assignment](https://user-images.githubusercontent.com/81942428/115107577-5a166b80-9f9e-11eb-8552-cf64ae0f854c.png)





slice，s1，s2共用一块内存空间，只是index的起始位置和范围是不同的。比如，s1中，index=3时，是存放着5的，不过s1这个struct只能访问到index=3。

```go
s2 = append(s2, 100)
// s2 len： 5 cap： 5，只是更新了len，没有涉及内存
```

![slice-append-s2](https://user-images.githubusercontent.com/81942428/115107647-e58ffc80-9f9e-11eb-8599-a4d37e7f61ab.png)



因为共用内存地址，所以对于s1暂时没什么问题，但是对于slice来说，如果这不是程序猿的本意的话，那就太糟糕了！这也是为什么go限制了指针的原因！

```go
// 再次append
s2 = append(s2, 100)
// s2 len:6 cap: 10
```

![slice-append-s2-again](https://user-images.githubusercontent.com/81942428/115107733-51726500-9f9f-11eb-96a9-03521854be1b.png)

当cap超过原有的struct所维护的内存空间时，s2会另起炉灶，重新new一块空间，此时，slice和s1还是共用同一块内存空间，s2使用的是另一块空间，即，对s2的操作不会影响到slice和s1，反之亦然。



**那这就牵扯出slice 的扩容问题，即，扩容的触发条件和扩容的大小**

[slice的扩容](https://github.com/qcrao/Go-Questions/blob/master/%E6%95%B0%E7%BB%84%E5%92%8C%E5%88%87%E7%89%87/%E5%88%87%E7%89%87%E7%9A%84%E5%AE%B9%E9%87%8F%E6%98%AF%E6%80%8E%E6%A0%B7%E5%A2%9E%E9%95%BF%E7%9A%84.md)

一般都是在向 slice 追加了元素之后，才会引起扩容。追加元素调用的是 `append` 函数。

先来看看 `append` 函数的原型：

```go
func append(slice []Type, elems ...Type) []Type
```

append 函数的参数长度可变，因此可以追加多个值到 slice 中，还可以用 `...` 传入 slice，直接追加一个切片。

```go
slice = append(slice, elem1, elem2)
slice = append(slice, anotherSlice...)

// 需要注意的是
sliceB = append(sliceA, elem1) //sliceA和sliceB是两个不同的变量，只不过它包含的指针指向了同一块地址
// 例如
var sliceA = []int{1, 2}  //len: 2, cap: 2 print: [1,2]
sliceB := append(sliceA, 100) // sliceB len:3 cap:3 print: [1,2,100] sliceA len:2 cap:3 print: [1,2]
```

为了减少频繁扩容对性能的影响，每次扩容不会只增加刚刚需要的那一部分，而是会预留一部分buffer，以减少扩容的次数，而扩大的容量就得内存的占用大小和扩容的次数平衡。

go的扩容源码

```go
/*
目标： newcap ,这是新的cap，最终slice的cap
输入： old slice ， 这是所要扩容的slice； cap ： 想要扩容的期望值，需要比它大，append的时候会得到所想要的
*/
func growslice(et *_type, old slice, cap int) slice {
	...

	newcap := old.cap
	doublecap := newcap + newcap	// 两倍
	if cap > doublecap {
		newcap = cap		// 如果期望的cap比两倍
	} else {
		if old.cap < 1024 {
			newcap = doublecap
		} else {
			// Check 0 < newcap to detect overflow
			// and prevent an infinite loop.
			for 0 < newcap && newcap < cap {
				newcap += newcap / 4
			}
			// Set newcap to the requested cap when
			// the newcap calculation overflowed.
			if newcap <= 0 {
				newcap = cap
			}
		}
	}
  ...
  // 内存对齐
  ...
}
```

综上，有可能是

1. 大于等于两倍的old.cap
2. 大于等于1.25倍的old.cap



[slice作为参数](https://github.com/qcrao/Go-Questions/blob/master/%E6%95%B0%E7%BB%84%E5%92%8C%E5%88%87%E7%89%87/%E5%88%87%E7%89%87%E4%BD%9C%E4%B8%BA%E5%87%BD%E6%95%B0%E5%8F%82%E6%95%B0.md)

不管slice作为参数是指针还是值，修改了slice内部指针所指向的地址中的值都会生效。

```go
package main

func main() {
	s := []int{1, 1, 1}
	f(s)
	fmt.Println(s)
}

func f(s []int) {
	// i只是一个副本，不能改变s中元素的值
	/*for _, i := range s {
		i++
	}
	*/

	for i := range s {
		s[i] += 1
	}
}

// [2 2 2]
```



指针和值之间的区别：

```go
package main

import "fmt"

func myAppend(s []int) []int {
	// 这里 s 虽然改变了，但并不会影响外层函数的 s
	s = append(s, 100) // append前后s是不一样的
	return s
}

func myAppendPtr(s *[]int) {
	// 会改变外层 s 本身
	*s = append(*s, 100)
	return
}

func main() {
	s := []int{1, 1, 1}
	newS := myAppend(s) 

	fmt.Println(s)
	fmt.Println(newS)

	s = newS

	myAppendPtr(&s)
	fmt.Println(s)
}

/*
[1 1 1]
[1 1 1 100]
[1 1 1 100 100]
*/
```