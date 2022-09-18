---
title: "Golang Slices 的使用与剖析"
date: "2015-03-10"
toc: false # Enable Table of Contents for specific page
categories:
  - "Blog"
tags:
  - "golang"
  - "source code"
author: "Rory"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
canonicalURL: "https://canonical.url/to/page"
disableHLJS: false # to disable highlightjs
disableShare: false
hideSummary: true
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "<image path/url>" # image path/url
    alt: "<alt text>" # alt text
    caption: "<text>" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: true # only hide on current single page
editPost:
    URL: "https://github.com/gongzili456/0x00.wtf/content"
    Text: "Suggest Changes" # edit text
    appendFilePath: true # to append file path to Edit link
---

Golang 的 slice 与 array 两种方式来操作一组数据。他们有很多相似的地方，更多的是不同，本文将要一步一步的分析。
<!--more-->


## Array

Slice 类型是 array 类型的高级抽象层，要理解 Slices 就必须先弄懂 array。

一个 array 类型的定义包括长度和元素类型两个属性。例如下面的例子，`[4]int` 意思就是长度为 4 且元素是`int`类型的数组。数组的长度是不可变的，且是可以被索引的。可以通过 `s[n]` 的方式来访问数组 s 的第 n 个元素，n 从 0 开始。

```golang
var a [4]int
a[0] = 1
i := a[0]
// i == 1
```

Array 的元素会被默认初始化成该类型的 `零值`，所以不需要被显式的初始化。

```golang
// a[2] == 0， int 类型的零值就是 0
```

在内存中的描述类似与这样： ![](images/go-slices-usage-and-internals_slice-array-300x39.png) Golang 的 Array 是值类型而非指针类型，一个数组的变量表示整个数组，意味着数组在传递的时候，是传递的原数组的拷贝，他们的内存地址是不同的。 声明 Array 的时候需要指定长度(len)

```
b := [2]string{"Penn", "Teller"}
```

也可以隐式的指定

```
b := [...]string{"Penn", "Teller"}
```

这两种方式 b 都是长度为 2 的 string 数组。

## Slice

Array 的缺点就是长度不可变且是值传递，这就导致数组的扩容不便，与空间的浪费。所以 Golang 又提供了 Slice 类型解决这些问题。

Slice 的声明并需要显式的指定长度：

```
letters := []string{"a", "b", "c", "d"}
```

另一种方式就是利用内置的 `make` 函数构建

```
func make([]T, len, cap) []T
```

其中的 `T` 是元素的类型，`len` 是长度，可选参数 `cap` 是容纳能力。关于 `cap` 的作用会在后面解释。

```
var s []byte
s = make([]byte, 5, 5)
// s == []byte{0, 0, 0, 0, 0}
```

当 `cap` 不传的时候，默认为 `len` 的值：

```
s := make([]byte, 5)
```

想要读取 Slice 的长度和容纳能力则需要使用内建的 `len` 和 `cap` 两个函数：

```
len(s) == 5
cap(s) == 5
```

Slice 的零值是 `nil`，一个 nil 的 Slice 其 `len` 和 `cap` 两个函数都返回 0；

Slice 可以从 slice 或 array 切边(slicing)而来，可以用表达式 `b[1:4]` 来创建一个包含从索引 1 到 3 的 slice。

```
b := []byte{'g', 'o', 'l', 'a', 'n', 'g'}
// b[1:4] == []byte{'o', 'l', 'a'}, sharing the same storage as b
```

切边(slicing)操作两边的数字都是可选的：

```
// b[:2] == []byte{'g', 'o'}
// b[2:] == []byte{'l', 'a', 'n', 'g'}
// b[:] == b
```

同样可以从一个 Array 类型创建 slice

```
x := [3]string{"Лайка", "Белка", "Стрелка"}
s := x[:] // a slice referencing the storage of x
```

## Slice 的内部构造

Slice 由一个数组指针，长度(length)，容纳能力 (capacity) 组成，Slice 的长度不能超过容纳能力，即 len <= cap。 ![](images/go-slices-usage-and-internals_slice-struct-300x115.png) 上面例子中的变量 `s`, 通过 `make([]byte, 5)` 创建，那么它的结构就如下图所示： ![](images/go-slices-usage-and-internals_slice-1-300x112.png) 当对 slice 重新切片(re-slicing)的时候：

```
s = s[2:4]
```

![](images/go-slices-usage-and-internals_slice-2-300x111.png)

切片操作(slicing) 只是重建指向原始数组的指针，并不会拷贝值。这也是 slice 的操作比 array 高效的原因。所以，修改 slice 的元素操作其实是对原始 slice 的修改并重建指针的操作，称之为重新切片(re-slice)。

```
d := []byte{'r', 'o', 'a', 'd'}
e := d[2:]
// e == []byte{'a', 'd'}
e[1] = 'm'
// e == []byte{'a', 'm'}
// d == []byte{'r', 'o', 'a', 'm'}
```

上面我们创建了一个 length 小于 capacity 的 slice，现在可以通过 re-slicing 对其扩容：

```
s = s[:cap(s)]
```

![](images/go-slices-usage-and-internals_slice-3-300x111.png)

Slice 不能扩容到超过他的 capacity 值，当造作索引超出 slice 的索引范围时，就会抛出运行时 panic 错误。

## Slices 的扩容（copy 与 append 函数）

想要增加 slice 的容量(capacity)，必须新建一个更大容量的 slice，然后将原来 slice 的数据拷贝到新的 slice。可以简单的通过 for 循环来拷贝 slice 的数据。

```
t := make([]byte, len(s), (cap(s)+1)*2) // +1 in case cap(s) == 0
for i := range s {
        t[i] = s[i]
}
s = t
```

Golang 内置了 `copy` 函数可以更加方便的拷贝数据。其签名如下：

```
func copy(dst, src []T) int
```

`copy` 函数支持两个 slice 之间的数据拷贝，但只能从小的拷贝至大的，否则会出错。值得注意的是新建的 slice 与原来的 slice 同指向同一个原始数组。上面的 for 循环代码可以简化为：

```
t := make([]byte, len(s), (cap(s)+1)*2)
copy(t, s)
s = t
```

当要向 slice 尾部追加元素的时候，就需要考虑 slice 的容量(capacity)问题，如果容量不足就需要对其扩容：

```
func AppendByte(slice []byte, data ...byte) []byte {
    m := len(slice)
    n := m + len(data)
    if n > cap(slice) { // if necessary, reallocate
        // allocate double what's needed, for future growth.
        newSlice := make([]byte, (n+1)*2)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:n]
    copy(slice[m:n], data)
    return slice
}
```

利用自定义的 `AppendByte` 函数对 \[\]byte 类型的 slice 扩容：

```
p := []byte{2, 3, 5}
p = AppendByte(p, 7, 11, 13)
// p == []byte{2, 3, 5, 7, 11, 13}
```

如果要对 \[\]string 类型的 slice 扩容该怎么办呢？Golang 内置了一个 `append` 函数来做这件事情。

```
func append(s []T, x ...T) []T
```

支持泛型的 append 函数的使用非常简单

```
a := make([]int, 1)
// a == []int{0}
a = append(a, 1, 2, 3)
// a == []int{0, 1, 2, 3}
```

同样地，将一个 slice 追加(append) 到另一个 slice 也很方便，可以使用 `...` 操作符号，将第二个参数进行展开即可。

```
a := []string{"John", "Paul"}
b := []string{"George", "Ringo", "Pete"}
a = append(a, b...) // equivalent to "append(a, b[0], b[1], b[2])"
// a == []string{"John", "Paul", "George", "Ringo", "Pete"}
```

## GC 问题

上面介绍了 slice 只是数组的索引，一个完整的 array 会在内存中持续存在，直到不再被引用。其实我们需要的数据往往是一小部分的 slice，这就造成了空间的浪费。比如下面这个场景：从一个文件中搜寻数字。

```
var digitRegexp = regexp.MustCompile("[0-9]+")

func FindDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    return digitRegexp.Find(b)
}
```

上面的代码返回了一个 \[\]byte 的指针，其指向的是整个文件内容。从这个 slice 开始引用这个原始数组开始，垃圾回收器(GC)就不能对这块内容进行释放了，从来造成了空间浪费。 一种解决办法是 copy 需要的数据到新的 slice 并返回。

```
func CopyDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    b = digitRegexp.Find(b)
    c := make([]byte, len(b))
    copy(c, b)
    return c
}
```

同样也可以用 append 来解决。

```
func CopyDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    b = digitRegexp.Find(b)
    var c []byte
    c = append(c, b)
    return c
}
```

## 总结

- array 的长度不可变，slice 的长度是可变的。
- slice 可以通过 make 函数创建
- slice 可以通过 copy 函数拷贝数据，也可以通过 append 函数追加数据，并扩容。
- slice 包含长度(len),容量(capacity)和一个指针指向原始数组
- slice 可以被 re-slicing
- re-slicing 的过程只是对原始数组的指针重建，所以更加高效
- 通过 slice\[n\] 修改 slice 的元素的值，其实是对原始数组的修改，并重建指针的过程。
- slice 使用时，要避免无法 GC 的问题。
