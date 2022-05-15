---
title: "Go slice 的创建与扩容"
date: 2022-05-15T02:05:48+08:00
draft: false
tags: ["Go"]
keywords: ["Go", "slice", "创建", 扩容]
---


### 内部数据结构

```
type slice struct {
  array unsafe.Pointer  // 指向底层数组的指针
  len   int            // 长度
  cap   int            // 容量
}
```

### 创建

* var 直接声明

  ```
  var ints []int
  // 这种创建方式只声明了切片结构，但没有声明底层数组，所以其底层结构如下
  type slice struct {
    array unsafe.Pointer  // nil
    len   int            // 0
    cap   int            // 0
  }
  ```

* make 创建

  ```
  ints := make([]int, 2, 5)
  // 通过make创建的slice会开辟一个底层数组，并把这个底层数组初始化为整形的默认值 0， 所以其底层结构如下
  type slice struct {
    array unsafe.Pointer  // 指向一个长度为5，默认值为0的数组
    len   int            // 2
    cap   int            // 5
  }
  
  ```

* new 创建

  ```
  ints := new([]int)
  // 通过new创建的slice，底层结构与var创建的一样，但new返回的是slice结构的起始地址
  type slice struct {
    array unsafe.Pointer  // nil
    len   int            // 0
    cap   int            // 0
  }
  // 此时，slice还没有底层数组，像 (*ints)[0] = 1这样的操作是会报panic的
  // 通过append的方式添加元素，它就会给slice开辟一个底层数组
  ```

  

### 扩容策略

1、如果oldCap * 2 < newCap, 那么就直接扩容到newCap

2、如果大于newCap，那么就看元素个数，如果oldLen < 1024, 就直接扩到 oldLen * 2

3、如果oldLen >= 1024, 那么就先扩个1/4， 即 oldLen * (1 + 1/4)

4、扩容的数量确定了，就该分配对应大小的内存了，内存的计算方式为：newCap * 元素大小

5、从系统预先创建的内存块中选取合适大小的内存块(runtime/sizeclasses.go中定义)作为实际分配的内存

6、实际分配的内存 / 元素大小 = 实际扩容后的容量

eg:

```
package main

import "fmt"

func main() {
	s := []int{1,2}
	s = append(s,4,5,6)
	fmt.Printf("len=%d, cap=%d",len(s),cap(s))
}
```

结合上面的例子来分析源码：

```
// s 一次性添加3个元素，那么所需的newCap变为5，所以这里的入参为：slice类型 int，slice结构 s，所需容量 5
func growslice(et *_type, old slice, cap int) slice {
    // ……
    newcap := old.cap
	doublecap := newcap + newcap
  // 所需容量 > oldCap * 2
	if cap > doublecap {
		newcap = cap //新容量直接变为5
	} else {
		if old.len < 1024 {
			newcap = doublecap
		} else {
			for newcap < cap {
				newcap += newcap / 4
			}
		}
	}
	// ……
	
  // 新容量 * 元素大小，int类型大小为8字节，所以这里入参为：5 * 8 = 40字节的容量
	capmem = roundupsize(uintptr(newcap) * ptrSize) //roundupsize后，最终返回 48
	newcap = int(capmem / ptrSize) // 48 / 8 = 6，所以最终的newCap为6，长度为5
}

// 为什么需要roundupsize？在Go语言中申请内存并不是直接与操作系统交涉，而是与语言自身实现的内存管理模块交涉，内存管理模块会提前和操作系统申请一批内存，分成常用的规格管理起来，我们申请内存时，它会为我们匹配到足够大且最接近的规格。 像上面的例子中每个元素的大小为8字节，所以需要分配5 * 8 = 40字节大小的内存，而实际申请时会匹配到48字节，所以实际扩容后的容量为6。
func roundupsize(size uintptr) uintptr {
	if size < _MaxSmallSize {
		if size <= smallSizeMax-8 { //40 < 32768 - 8
			return uintptr(class_to_size[size_to_class8[divRoundUp(size, smallSizeDiv)]])//class_to_size[size_to_class8[5]]，根据索引取出值48
		} else {
			return uintptr(class_to_size[size_to_class128[divRoundUp(size-smallSizeMax, largeSizeDiv)]])
		}
	}
	if size+_PageSize < size {
		return size
	}
	return alignUp(size, _PageSize)
}

func divRoundUp(n, a uintptr) uintptr {
  return (n + a - 1) / a  // (40 + 8 - 1) / 8 = 5
}

var class_to_size = [_NumSizeClasses]uint16{0, 8, 16, 24, 32, 48, 64, 80, 96, 112, 128, 144, 160, 176, 192, 208, 224, 240, 256, 288, 320, 352, 384, 416, 448, 480, 512, 576, 640, 704, 768, 896, 1024, 1152, 1280, 1408, 1536, 1792, 2048, 2304, 2688, 3072, 3200, 3456, 4096, 4864, 5376, 6144, 6528, 6784, 6912, 8192, 9472, 9728, 10240, 10880, 12288, 13568, 14336, 16384, 18432, 19072, 20480, 21760, 24576, 27264, 28672, 32768}

var size_to_class8 = [smallSizeMax/smallSizeDiv + 1]uint8{0, 1, 2, 3, 4, 5, 5, 6, 6, 7, 7, 8, 8, 9, 9, 10, 10, 11, 11, 12, 12, 13, 13, 14, 14, 15, 15, 16, 16, 17, 17, 18, 18, 19, 19, 19, 19, 20, 20, 20, 20, 21, 21, 21, 21, 22, 22, 22, 22, 23, 23, 23, 23, 24, 24, 24, 24, 25, 25, 25, 25, 26, 26, 26, 26, 27, 27, 27, 27, 27, 27, 27, 27, 28, 28, 28, 28, 28, 28, 28, 28, 29, 29, 29, 29, 29, 29, 29, 29, 30, 30, 30, 30, 30, 30, 30, 30, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 31, 32, 32, 32, 32, 32, 32, 32, 32, 32, 32, 32, 32, 32, 32, 32, 32}

```










