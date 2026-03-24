---
title: "Go Slice源码解析"
date: 2025-03-01
categories: [Go, 源码分析]
tags: [go, slice, source-code]
---

# slice 切片

Golang中的切片是类似于数组结构的，不过数组是定长的，切片是动态的，也被称为动态数组（其长度不固定，可以向切片中追加元素，还支持动态扩容的能力）；其实切片的底层数据结构就是数组；

## 1 数据结构

![image-20260324170857692](/assets/img/posts/image-20260324170857692.png)

```go
type slice struct {
    array unsafe.Pointer  // 元素指针(指向起点)
	len   int // 长度
	cap   int // 容量
}
```

由于底层数组是一片连续的内存空间，所以我们可以将切片理解成一片连续的内存空间加上长度与容量的标识。

切片其实是对底层数组的封装，在runtime过程中可以动态的修改底层数组的大小，当底层数组不满足切片长度要求时，会重写开辟一段连续的空间然后让array指向开头，并同时跟新len和cap；注意：一个底层数组可能会被多个array指针指向；

## 2 初始化

*   声明但不初始化

    ```go
    var s []int
    ```

    只声明了一个切片的类型，但是未初始化即未做内存分配操作，这个s字面量此时是一个空指针 nil；

    这个切片后续可以使用 append 进行内存分配操作，追加元素；但是不能使用下标操作；

*   使用 make

    make初始化有三个参数：元素类型、长度(len)、容量(cap)

    可以省略第三个参数，表示len和cap一样

    ```go
    s := make([]int, 0, 10)
    ```

*   初始化同时赋值

    ```go
    s := []int{1, 2, 3}
    ```

    这样初始化的话len和cap一样，都为初始化元素的数量

*   截取

    ```go
    s := []int{1, 2, 3, 4, 5}
    s1 := s[1:3]
    s2 := s[:5]
    s3 := s[1:] 等价于 s3 := s[1:len(s)]
    s4 := s[:]  等价于 s4 := s[0:len(s)]
    ```

    截取操作其实复用的是同一片内存空间中的同一份数据，比如s1 s2 s3 s4 复用的全是 s 的内存空间数据：

![image-20260324170908473](/assets/img/posts/image-20260324170908473.png)

## 3 切片作为参数传递

我们知道在Go中函数参数的传递都是值传递，切片作为函数参数传递也遵循这个规则，但切片的底层是一个`unsafe.Pointer` 指向的数组头地址，对这个地址进行值传递复制那么对于整个切片来说就是引用传递，因为两个指针指向了同一个底层数组地址；

除非函数中的切片重新被初始化或有扩容操作，否则一直指向同一个切片，且通过下标修改元素会影响到原切片数据；

```go
func main() {
	slice1 := make([]int, 3, 5)
	change(slice1)
}

func change(slice2 []int) {
	fmt.Println("change", slice2, len(slice2), cap(slice2))
}
```

![image-20260324170921467](/assets/img/posts/image-20260324170921467.png)

**使用下标修改：**

若是使用下标修改 `slice2` 的元素，也会影响到 `slice1` 对应下标为止的元素

```go
func main() {
	slice1 := make([]int, 3, 5)
	slice1[0] = 1
	slice1[1] = 11
	slice1[2] = 111

	change(slice1)
}

func change(slice2 []int) {
	fmt.Println("change", slice2, len(slice2), cap(slice2))

	slice2[0] = 2

	fmt.Println("change", slice2, len(slice2), cap(slice2))
}

/*
输出：
change [1 11 111] 3 5
change [2 11 111] 3 5
*/
```

**使用append修改：**

例子1：

```go
func main() {
   //创建一个长度和容量均为3的切片
   arr := []int{1,2,3}
   fmt.Println(arr) // [1 2 3]

   addNum(arr)      

   fmt.Println(arr) // [1,2,3]
}

func addNum(sli []int){

   sli = append(sli,  4)
   fmt.Println(sli) // [1,2,3,4]
}
```

因为原数组是通过 `arr := []int{1,2,3}` 的方式创建的，所以对应的len和cap都是一样的；这种情况下使用append必定会导致sli切片触发扩容机制重新分配一块内存空间，所以修改sli的不会影响到arr的数据；

例子2：

```go
func main() {
	arr := make([]int, 3, 4) //创建一个长度为3，容量为4的切片
	fmt.Println(arr, len(arr), cap(arr)) //[0 0 0] 3 4

	addNum(arr)

	fmt.Println(arr, len(arr), cap(arr)) //[0 0 0] 3 4
}

func addNum(sli []int) {
	sli = append(sli, 4)
	fmt.Println(sli, len(sli), cap(sli)) //[0 0 0 4] 4 4
}
```

sli的append其实影响到了原切片的内容，但是由于arr的len没变还是3，所以打印不出来变化后的值，实际是影响到了；

## 4 扩容机制

触发的时机：当 slice 当前长度 len 与容量 cap 相等时，下一次 append 操作就会引发依次切片扩容；

核心扩容代码：

```go
// 参数：
// oldPtr: 老切片的array
// newLen: 新切片的len
// oldCap: 老切片的cap
// num: 本次扩容的大小
func growslice(oldPtr unsafe.Pointer, newLen, oldCap, num int, et *_type) slice {
    ... 
    
    if et.Size_ == 0 {
		return slice{unsafe.Pointer(&zerobase), newLen, newLen}
	}
    
    newcap := nextslicecap(newLen, oldCap)
    
    var overflow bool
	var lenmem, newlenmem, capmem uintptr
	noscan := !et.Pointers()
	switch {
	case et.Size_ == 1:
		lenmem = uintptr(oldLen)
		newlenmem = uintptr(newLen)
		capmem = roundupsize(uintptr(newcap), noscan)
		overflow = uintptr(newcap) > maxAlloc
		newcap = int(capmem)
	case et.Size_ == goarch.PtrSize:
		lenmem = uintptr(oldLen) * goarch.PtrSize
		newlenmem = uintptr(newLen) * goarch.PtrSize
		capmem = roundupsize(uintptr(newcap)*goarch.PtrSize, noscan)
		overflow = uintptr(newcap) > maxAlloc/goarch.PtrSize
		newcap = int(capmem / goarch.PtrSize)
	case isPowerOfTwo(et.Size_):
		var shift uintptr
		if goarch.PtrSize == 8 {
			// Mask shift for better code generation.
			shift = uintptr(sys.TrailingZeros64(uint64(et.Size_))) & 63
		} else {
			shift = uintptr(sys.TrailingZeros32(uint32(et.Size_))) & 31
		}
		lenmem = uintptr(oldLen) << shift
		newlenmem = uintptr(newLen) << shift
		capmem = roundupsize(uintptr(newcap)<<shift, noscan)
		overflow = uintptr(newcap) > (maxAlloc >> shift)
		newcap = int(capmem >> shift)
		capmem = uintptr(newcap) << shift
	default:
		lenmem = uintptr(oldLen) * et.Size_
		newlenmem = uintptr(newLen) * et.Size_
		capmem, overflow = math.MulUintptr(et.Size_, uintptr(newcap))
		capmem = roundupsize(capmem, noscan)
		newcap = int(capmem / et.Size_)
		capmem = uintptr(newcap) * et.Size_
	}

    ...
    
    var p unsafe.Pointer
	if !et.Pointers() {
		p = mallocgc(capmem, nil, false)
		memclrNoHeapPointers(add(p, newlenmem), capmem-newlenmem)
	} else {
		p = mallocgc(capmem, et, true)
		if lenmem > 0 && writeBarrier.enabled {
			bulkBarrierPreWriteSrcOnly(uintptr(p), uintptr(oldPtr), lenmem-et.Size_+et.PtrBytes, et)
		}
	}
	memmove(p, oldPtr, lenmem)

	return slice{p, newLen, newcap}
}

// 参数:
// newLen: 新切片的len
// odlCap: 老切片的cap
func nextslicecap(newLen, oldCap int) int {
	newcap := oldCap
	doublecap := newcap + newcap
	if newLen > doublecap {
		return newLen
	}

	const threshold = 256
	if oldCap < threshold {
		return doublecap
	}
	for {
        newcap += (newcap + 3*threshold) >> 2 // => 等价于: newcap = newcap + (1/4*newcap + 3/4*threshold)

		if uint(newcap) >= uint(newLen) {
			break
		}
	}

	if newcap <= 0 {
		return newLen
	}
	return newcap
}
```

主要分析下`nextslicecap` 方法的实现：

*   如果新切片的长度大于老切片容量的两倍，则直接采用新切片长度作为预期的新切片容量
*   如果老切片的容量小于256，则直接采用老切片容量的2倍作为新切片的容量
*   如果老切片的容量大于256，在老切片容量的基础上循环累加 (1/4*newcap + 192(3/4*threshold)) ；直到得到的新容量已经大于预期的新容量为止

> go 1.18 版本之前的扩容机制: <br>
> 当原切片容量小于 1024 时, 新切片的容量变为原来的 2 倍; 原切片容量大于 1024 时, 新切片的容量需要 for 循环递加, 每次增加 25%, 直到大于期望预期容量即可, 然后再做一个内存对齐操作.

## 5 切片的几个常用技巧

**删除某个指定下标的元素：**

```go
func delSlice() {
    s := []int{1, 2, 3, 4 , 5}
    index := 2 // 删除下标
    s = append(s[:index], s[index+1:]...)
}
```

**删除头部元素：**

```go
func delSlice() {
    s := []int{1, 2, 3, 4 , 5}
    s = [1:]
}
```

**删除尾部元素：**

```go
func delSlice() {
    s := []int{1, 2, 3, 4 , 5}
    s = [:len(s)-1]
}
```

**删除全部元素：**

```go
func delSlice() {
    s := []int{1, 2, 3, 4 , 5}
    s = [:0]
}
```

**for...range...地址：**

```go
func main() {
    m := make(map[int]*int)
    arr := []int{1, 2, 3, 4, 5}
    for i, v := range arr {
        m[i] = &v
    }
    for k, v := range m {
        fmt.Println(k, *v)
    }
}

/*
输出：
3 5
1 5
0 5
2 5
4 5
*/
```

为什么会出现这种情况呢，因为range每次取值操作都是在做一次值拷贝，相当于每次赋值给`m[i]`的都是v的地址，但是v每次都会变化，所以导致之前被赋值的`m[i]`中的值也被修改了，上面的写法也可以改写成这样：

```go
func main() {
    m := make(map[int]*int)
    arr := []int{1, 2, 3, 4, 5}
    for i, v := range arr {
        v = arr[i]  // <=
        m[i] = &v
    }
    for k, v := range m {
        fmt.Println(k, *v)
    }
}
```