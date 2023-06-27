---
title: "【Golang】Go语言之旅"
author: "Tweakzx"
date: 2022-02-22T17:20:32+08:00
description: 学习Go语言的一些练习代码
categories: Go语言
tags: 
  - Go语言
  - 练习代码
image: https://go.dev/images/gophers/biplane.svg
draft: false
---

# 旅行起点

[Go 语言之旅 (go-zh.org)](https://tour.go-zh.org/welcome/1)

上方链接是一个Go语言学习的Playground，快点击它，开启一场Go语言之旅吧

# 旅行开始

## 练习：循环与函数

为了练习函数与循环，我们来实现一个平方根函数：用牛顿法实现平方根函数。

计算机通常使用循环来计算 x 的平方根。从某个猜测的值 z 开始，我们可以根据 z² 与 x 的近似度来调整 z，产生一个更好的猜测：

```
z -= (z*z - x) / (2*z)
```

重复调整的过程，猜测的结果会越来越精确，得到的答案也会尽可能接近实际的平方根。

在提供的 `func Sqrt` 中实现它。无论输入是什么，对 z 的一个恰当的猜测为 1。 要开始，请重复计算 10 次并随之打印每次的 z 值。观察对于不同的值 x（1、2、3 ...）， 你得到的答案是如何逼近结果的，猜测提升的速度有多快。

提示：用类型转换或浮点数语法来声明并初始化一个浮点数值：

```
z := 1.0
z := float64(1)
```

然后，修改循环条件，使得当值停止改变（或改变非常小）的时候退出循环。观察迭代次数大于还是小于 10。 尝试改变 z 的初始猜测，如 x 或 x/2。你的函数结果与标准库中的 [math.Sqrt](https://go-zh.org/pkg/math/#Sqrt) 接近吗？

（*注：* 如果你对该算法的细节感兴趣，上面的 z² − x 是 z² 到它所要到达的值（即 x）的距离， 除以的 2z 为 z² 的导数，我们通过 z² 的变化速度来改变 z 的调整量。 这种通用方法叫做[牛顿法](https://zh.wikipedia.org/wiki/牛顿法)。 它对很多函数，特别是平方根而言非常有效。）

```go
package main

import (
	"fmt"
	"math"
)

func Sqrt(x float64) (z float64) {
	z = float64(1)
	for math.Abs(z*z-x)>0.000001 {
		z -= (z*z-x)/(z*2)
	}
	return
}

func main() {
	fmt.Println(Sqrt(2))
}

```



##  练习：切片

实现 `Pic`。它应当返回一个长度为 `dy` 的切片，其中每个元素是一个长度为 `dx`，元素类型为 `uint8` 的切片。当你运行此程序时，它会将每个整数解释为灰度值（好吧，其实是蓝度值）并显示它所对应的图像。

图像的选择由你来定。几个有趣的函数包括 `(x+y)/2`, `x*y`, `x^y`, `x*log(y)` 和 `x%(y+1)`。

（提示：需要使用循环来分配 `[][]uint8` 中的每个 `[]uint8`；请使用 `uint8(intValue)` 在类型之间转换；你可能会用到 `math` 包中的函数。）

```go
package main

import "golang.org/x/tour/pic"

func Pic(dx, dy int) [][]uint8 {
	picture := make([][]uint8,dy)
	for x:=range picture{
		line := make([]uint8,dx) 
		for y:=range line{
			line[y] = uint8((x+y)/2)
		}
		picture[x] = line
	}
	return picture
}

func main() {
	pic.Show(Pic)
}
```

##  练习：映射

实现 `WordCount`。它应当返回一个映射，其中包含字符串 `s` 中每个“单词”的个数。函数 `wc.Test` 会对此函数执行一系列测试用例，并输出成功还是失败。

你会发现 [strings.Fields](https://go-zh.org/pkg/strings/#Fields) 很有帮助。

```go
package main

import (
	"golang.org/x/tour/wc"
	"strings"
)

func WordCount(s string) map[string]int {
	words := strings.Fields(s)
	ans := make(map[string]int)
	for _,w :=range words{
		//v,ok := ans[w]
		ans[w] = ans[w] + 1
	}
	return ans
}

func main() {
	wc.Test(WordCount)
}
```

## 练习：斐波纳契闭包

让我们用函数做些好玩的事情。

实现一个 `fibonacci` 函数，它返回一个函数（闭包），该闭包返回一个[斐波纳契数列](https://zh.wikipedia.org/wiki/斐波那契数列) `(0, 1, 1, 2, 3, 5, ...)`。

```go
package main

import "fmt"

// 返回一个“返回int的函数”
func fibonacci() func() int {
	first := 2 
	second := 1   //根据公式倒推出的first和second
	return func() int{
		first = second - first
		second = second + first //斐波那契公式
		return second
	}
}

func main() {
	f := fibonacci()
	for i := 0; i < 10; i++ {
		fmt.Println(f())
	}
}
```

## 练习：Stringer

通过让 `IPAddr` 类型实现 `fmt.Stringer` 来打印点号分隔的地址。

例如，`IPAddr{1, 2, 3, 4}` 应当打印为 `"1.2.3.4"`。

```go
package main

import "fmt"

type IPAddr [4]byte

// TODO: 给 IPAddr 添加一个 "String() string" 方法
func (ip IPAddr) String() string{
	return fmt.Sprintf("%v.%v.%v.%v\n",ip[0],ip[1],ip[2],ip[3])
}

func main() {
	hosts := map[string]IPAddr{
		"loopback":  {127, 0, 0, 1},
		"googleDNS": {8, 8, 8, 8},
	}
	for name, ip := range hosts {
		fmt.Printf("%v: %v\n", name, ip)
	}
}
```

## 练习：错误

从[之前的练习](https://tour.go-zh.org/flowcontrol/8)中复制 `Sqrt` 函数，修改它使其返回 `error` 值。

`Sqrt` 接受到一个负数时，应当返回一个非 nil 的错误值。复数同样也不被支持。

创建一个新的类型

```
type ErrNegativeSqrt float64
```

并为其实现

```
func (e ErrNegativeSqrt) Error() string
```

方法使其拥有 `error` 值，通过 `ErrNegativeSqrt(-2).Error()` 调用该方法应返回 `"cannot Sqrt negative number: -2"`。

**注意:** 在 `Error` 方法内调用 `fmt.Sprint(e)` 会让程序陷入死循环。可以通过先转换 `e` 来避免这个问题：`fmt.Sprint(float64(e))`。这是为什么呢？

修改 `Sqrt` 函数，使其接受一个负数时，返回 `ErrNegativeSqrt` 值。

```go
package main

import (
	"fmt"
)
type ErrNegativeSqrt float64

func (e ErrNegativeSqrt) Error() string{
	return fmt.Sprintf("cannot Sqrt negative number: %v",float64(e))
}

func Sqrt(x float64) (float64, error) {
	if x>0{
		return 0, nil
	}else{
		var e ErrNegativeSqrt
		e = ErrNegativeSqrt(x)
		return x,e
	}
}

func main() {
	fmt.Println(Sqrt(2))
	fmt.Println(Sqrt(-2))
}
```

##  练习：Reader

实现一个 `Reader` 类型，它产生一个 ASCII 字符 `'A'` 的无限流。

```go
package main

import "golang.org/x/tour/reader"

type MyReader struct{}

// TODO: 给 MyReader 添加一个 Read([]byte) (int, error) 方法
func (mr MyReader) Read(buf []byte) (int, error) {
	for i :=range buf{
		buf[i] = 'A'
	}
	return 1, nil
}

func main() {
	reader.Validate(MyReader{})
}
```

## 练习：rot13Reader

有种常见的模式是一个 [`io.Reader`](https://go-zh.org/pkg/io/#Reader) 包装另一个 `io.Reader`，然后通过某种方式修改其数据流。

例如，[`gzip.NewReader`](https://go-zh.org/pkg/compress/gzip/#NewReader) 函数接受一个 `io.Reader`（已压缩的数据流）并返回一个同样实现了 `io.Reader` 的 `*gzip.Reader`（解压后的数据流）。

编写一个实现了 `io.Reader` 并从另一个 `io.Reader` 中读取数据的 `rot13Reader`，通过应用 [rot13](http://en.wikipedia.org/wiki/ROT13) 代换密码对数据流进行修改。

`rot13Reader` 类型已经提供。实现 `Read` 方法以满足 `io.Reader`。

```go
package main

import (
	"io"
	"os"
	"strings"
)

type rot13Reader struct {
	r io.Reader
}

func ( rot rot13Reader) Read(buf []byte) (int, error){
	len,ok := rot.r.Read(buf)
	for i,v := range buf{
		switch{
			case ('a'<=v && v<='m')||('A'<=v && v<='M'):
				buf[i] = v+13
			case ('n'<=v && v<='z')||('N'<=v && v<='Z'):
				buf[i] = v-13
			default:
		}
	}
	return len,ok
}

func main() {
	s := strings.NewReader("Lbh penpxrq gur pbqr!")
	r := rot13Reader{s}
	io.Copy(os.Stdout, &r)
}
```

##  练习：图像

还记得之前编写的[图片生成器](https://tour.go-zh.org/moretypes/18) 吗？我们再来编写另外一个，不过这次它将会返回一个 `image.Image` 的实现而非一个数据切片。

定义你自己的 `Image` 类型，实现[必要的方法](https://go-zh.org/pkg/image/#Image)并调用 `pic.ShowImage`。

`Bounds` 应当返回一个 `image.Rectangle` ，例如 `image.Rect(0, 0, w, h)`。

`ColorModel` 应当返回 `color.RGBAModel`。

`At` 应当返回一个颜色。上一个图片生成器的值 `v` 对应于此次的 `color.RGBA{v, v, 255, 255}`。

```go
package main

import (
	"golang.org/x/tour/pic"
	"image"
	"image/color"
)

type Image struct{
	w,h int
	pixels [][]uint8
}

func (self Image) Bounds()(image.Rectangle){
	return image.Rect(0, 0, self.w, self.h)
}
func (self Image) ColorModel()(color.Model){
	return color.RGBAModel
}
func (self Image) At(x int ,y int)(color.Color){
	v := self.pixels[y][x]
	return color.RGBA{v,v, 255, 255}
}
func Pic(dx, dy int) [][]uint8 {
    img := make([][]uint8, dy)
    for y := 0; y < dy; y++ {
        img[y] = make([]uint8, dx)
        for x := 0; x < dx; x++ {
			img[y][x] = (uint8)(x^y)
        }
    }
    return img
}

func main() {
	m := Image{256,256,Pic(256,256)}
	pic.ShowImage(m)
}
```

## 练习：等价二叉查找树

**1.** 实现 `Walk` 函数。

**2.** 测试 `Walk` 函数。

函数 `tree.New(k)` 用于构造一个随机结构的已排序二叉查找树，它保存了值 `k`, `2k`, `3k`, ..., `10k`。

创建一个新的信道 `ch` 并且对其进行步进：

```
go Walk(tree.New(1), ch)
```

然后从信道中读取并打印 10 个值。应当是数字 `1, 2, 3, ..., 10`。

**3.** 用 `Walk` 实现 `Same` 函数来检测 `t1` 和 `t2` 是否存储了相同的值。

**4.** 测试 `Same` 函数。

`Same(tree.New(1), tree.New(1))` 应当返回 `true`，而 `Same(tree.New(1), tree.New(2))` 应当返回 `false`。

`Tree` 的文档可在[这里](https://godoc.org/golang.org/x/tour/tree#Tree)找到。

```go
package main

import (
	"fmt"
	"golang.org/x/tour/tree"
)

// Walk 步进 tree t 将所有的值从 tree 发送到 channel ch。
func Walk(t *tree.Tree, ch chan int) {
	dfs(t, ch)
	close(ch)
}
func dfs(t *tree.Tree, ch chan int) {
	if t == nil {
		return
	}
	dfs(t.Left, ch)
	ch <- t.Value
	dfs(t.Right, ch)
}

// Same 检测树 t1 和 t2 是否含有相同的值。
func Same(t1, t2 *tree.Tree) bool {
	ch1, ch2 := make(chan int), make(chan int)

	go Walk(t1, ch1)
	go Walk(t2, ch2)

	for i := range ch1 { // ch1 关闭后   for循环自动跳出
		if i != <-ch2 {
			return false
		}
	}
	return true
}
func main() {
	fmt.Println(Same(tree.New(1), tree.New(1)))
}

```

##  练习：Web 爬虫

在这个练习中，我们将会使用 Go 的并发特性来并行化一个 Web 爬虫。

修改 `Crawl` 函数来并行地抓取 URL，并且保证不重复。

*提示*：你可以用一个 map 来缓存已经获取的 URL，但是要注意 map 本身并不是并发安全的！

```go
package main

import (
	"fmt"
	"sync"
)

type Fetcher interface {
	// Fetch 返回 URL 的 body 内容，并且将在这个页面上找到的 URL 放到一个 slice 中。
	Fetch(url string) (body string, urls []string, err error)
}

// Crawl 使用 fetcher 从某个 URL 开始递归的爬取页面，直到达到最大深度。

type CrawlRecord struct{
	m   map[string]int
	mux sync.Mutex
	wg  sync.WaitGroup
}
var	cr = CrawlRecord{m: make(map[string]int)}

func Crawl(url string, depth int, fetcher Fetcher) {
	defer cr.wg.Done()
	// TODO: 并行的抓取 URL。
	// TODO: 不重复抓取页面。
        // 下面并没有实现上面两种情况：
	if depth <= 0 {
		return
	}
	
	cr.mux.Lock()
	cr.m[url]++
	cr.mux.Unlock()
	
	body, urls, err := fetcher.Fetch(url)
	if err != nil {
		fmt.Println(err)
		return
	}
	
	fmt.Printf("found: %s %q\n", url, body)

	for _, u := range urls {
		cr.mux.Lock()
		if _,ok := cr.m[u]; !ok{
			cr.wg.Add(1)
			go Crawl(u, depth-1, fetcher)
		}
		cr.mux.Unlock()
	}

	return
}

func main() {
	cr.wg.Add(1)
	Crawl("https://golang.org/", 4, fetcher)
	cr.wg.Wait()
}

// fakeFetcher 是返回若干结果的 Fetcher。
type fakeFetcher map[string]*fakeResult

type fakeResult struct {
	body string
	urls []string
}

func (f fakeFetcher) Fetch(url string) (string, []string, error) {
	if res, ok := f[url]; ok {
		return res.body, res.urls, nil
	}
	return "", nil, fmt.Errorf("not found: %s", url)
}

// fetcher 是填充后的 fakeFetcher。
var fetcher = fakeFetcher{
	"https://golang.org/": &fakeResult{
		"The Go Programming Language",
		[]string{
			"https://golang.org/pkg/",
			"https://golang.org/cmd/",
		},
	},
	"https://golang.org/pkg/": &fakeResult{
		"Packages",
		[]string{
			"https://golang.org/",
			"https://golang.org/cmd/",
			"https://golang.org/pkg/fmt/",
			"https://golang.org/pkg/os/",
		},
	},
	"https://golang.org/pkg/fmt/": &fakeResult{
		"Package fmt",
		[]string{
			"https://golang.org/",
			"https://golang.org/pkg/",
		},
	},
	"https://golang.org/pkg/os/": &fakeResult{
		"Package os",
		[]string{
			"https://golang.org/",
			"https://golang.org/pkg/",
		},
	},
}

```

# 旅行终点

你可以从[安装 Go](https://go-zh.org/doc/install/) 开始。

``` 
wget -c https://dl.google.com/go/go1.14.2.linux-amd64.tar.gz -O - | sudo tar -xz -C /usr/local
```

```
vim /etc/profile
```

``` 
export PATH=$PATH:/usr/local/go/bin
export GOPROXY=https://goproxy.cn,direct
export GO111MODULE=on
```

```
source /etc/profile
go version
```

一旦安装了 Go，Go [文档](https://go-zh.org/doc/)是一个极好的 应当继续阅读的内容。 它包含了参考、指南、视频等等更多资料。

了解如何组织 Go 代码并在其上工作，参阅[此视频](https://www.youtube.com/watch?v=XCsL89YtqCs)，或者阅读[如何编写 Go 代码](https://go-zh.org/doc/code.html)。

如果你需要标准库方面的帮助，请参考[包手册](https://go-zh.org/pkg/)。如果是语言本身的帮助，阅读[语言规范](https://go-zh.org/ref/spec)是件令人愉快的事情。

进一步探索 Go 的并发模型，参阅 [Go 并发模型](https://www.youtube.com/watch?v=f6kdp27TYZs)([幻灯片](https://talks.go-zh.org/2012/concurrency.slide))以及[深入 Go 并发模型](https://www.youtube.com/watch?v=QDDwwePbDtw)([幻灯片](https://talks.go-zh.org/2013/advconc.slide))并阅读[通过通信共享内存](https://go-zh.org/doc/codewalk/sharemem/)的代码之旅。

想要开始编写 Web 应用，请参阅[一个简单的编程环境](https://vimeo.com/53221558)([幻灯片](https://talks.go-zh.org/2012/simple.slide))并阅读[编写 Web 应用](https://go-zh.org/doc/articles/wiki/)的指南。

[函数：Go 中的一等公民](https://go-zh.org/doc/codewalk/functions/)展示了有趣的函数类型。

[Go 博客](https://blog.go-zh.org/)有着众多关于 Go 的文章和信息。

[mikespook 的博客](https://www.mikespook.com/tag/golang/)中有大量中文的关于 Go 的文章和翻译。

开源电子书 [Go Web 编程](https://github.com/astaxie/build-web-application-with-golang)和 [Go 入门指南](https://github.com/Unknwon/the-way-to-go_ZH_CN)能够帮助你更加深入的了解和学习 Go 语言。

访问 [go-zh.org](https://go-zh.org/) 了解更多内容。

