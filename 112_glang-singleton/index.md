# Golang 单例模式与sync.Once



## 背景

单例模式可以说是最简单的设计模式之一了，功能很简单：一个类型的东西只实例化一次，全局只有一个实例，并提供方法来获取该实例。

在 Golang 中变量或说明实例只初始化一次的效果通过`init`函数是可以实现的，包在被引入时就会执行一次`init`函数且无论同一包被引入多少次也都只执行一次。

不过本文主要想讨论的单例模式是第一次需要用到时才去初始化，也就是延迟初始化。

## 不太好的单例实现

```go
// bad_singleton.go

package main

import (
	"sync"
)

var svcMu sync.Mutex
var svc *Svc

type Svc struct {
	Num int
}

func GetSvc() *Svc {
	if svc == nil { // 这一步判断不是并发安全的
		svcMu.Lock()
		defer svcMu.Unlock()
		if svc == nil {
			svc = &Svc{Num: 1}
			svc = &Svc{}
			svc.Num = 1

		}
	}

	return svc
}
```

注意执行互斥锁`svcMu.Lock()`前的语句`if svc == nil `并不是并发安全的，即在多个 goroutine 并发调用的场景下，其中的一个 goroutine 正在初始化这个变量`svc`的过程中，这里别的 goroutine 判断得到`svc`不等于`nil`的结果时也并不意味着`svc`就一定完成初始化了。

因为在缺乏显式同步的情况下，编译器和CPU在能保证每个 goroutine 内满足串行一致性的基础上可以自由地重排访问内存的指令顺序。

比如`svc = &Svc{Num: 1}`这行看上去只是一条执行语句，可能重排后的一种实现是像下面这样的：

```go
svc = &Svc{}
svc.Num = 1
```

可见，不等于`nil`并不意味着就一定完成了初始化，因此上面示例是一种不太好的单例实现。

## 比较好的单例实现

```go
// good_singleton.go

package main

import (
	"sync"
)

var svcOnce sync.Once
var svc *Svc

type Svc struct {
	Num int
}

func GetSvc() *Svc {
	svcOnce.Do(func() {
		svc = &Svc{Num: 1}
	})

	return svc
}
```

`sync.Once`提供的`Do`方法无论被调用多少次都只执行传入的函数一次，那为什么说直接使用`Do`方法执行初始化而不是套一层`if svc == nil `才是比较好的做法呢，下面结合`sync.Once`源码来说明。

```go
// sync.Once 源码

package sync

import (
	"sync/atomic"
)

type Once struct {
	done uint32
	m    Mutex
}

func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 0 { // 这步是判断是否已经完成初始化的关键
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```

官方对于`sync.Once`的实现是非常短小精悍的。其中`atomic.LoadUint32(&o.done) == 0`是关键的一步，这里采用的是原子操作语句，保证了即使在并发场景下也是安全的，对数据的读写都是完整的。

当`o.done`的值为`0`时表示未进行初始化或正在初始化中，只有等于`1`时才表示初始化已经完成，即`f()`执行完成后由`defer atomic.StoreUint32(&o.done, 1)`语句给`o.done`赋值`1`；也就是`o.done`作为是否完成初始化的标识，可能的值只有前面说的两个，为`0`时则加锁并尝试初始化流程，反之则视为已完成初始化直接跳过，这样就完美兼顾了效率与并发安全。

由此可见`sync.Once`内置的初始化完成标识判断远比`if svc == nil `靠谱，因此像上面这样使用`sync.Once`实现单例模式是最推荐的方式。

## 额外推荐

实则开发中用到的设计模式经常不止一种，越是复杂大型的项目就越需要使用更多合适的模式来优化代码。

下面要推荐的是[RefactoringGuru](https://refactoringguru.cn/design-patterns/catalog)。这是我所见过最好的设计模式教程，是国外创建的一个教程网站，有中文站点，图文并茂地介绍每一种模式的结构、关系和逻辑，
最重要的是示例代码方面囊括了常见的几种主流编程语言，是个适合多数程序员学习设计模式的好地方！

下图是设计模式的目录页面（是不是很图文并茂呢）：

![调试模式教程目录](https://data.waacoo.cc/upic/program_patten.1600.jpg)

## 结语

以上为本人学习和实践的一些总结，如有错漏还请不吝赐教。

## 参考

《Go程序设计语言》9.5 延迟初始化：sync.Once [网络版](https://yar999.gitbook.io/gopl-zh/ch9/ch9-05)
[Go 单例模式讲解和代码示例](https://refactoringguru.cn/design-patterns/singleton/go/example)

