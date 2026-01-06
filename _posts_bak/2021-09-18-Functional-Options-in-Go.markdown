---
title: Functional Options in Go
author: sunao
tag: ["#go","#设计模式"]
---


functional options(功能选项)可以使对象的初始化更灵活，更直观，实现高度可配置。

举例说明
```go
type House struct {
	Material     string // 材料
	HasFireplace bool   // 是否有防火墙
	Floors       int    // 楼层数
}
```

一般的初始化
```go
const (
	defaultFloors       = 2
	defaultHasFireplace = true
	defaultMaterial     = "wood"
)

func main(){
	h := &House{
		Material:     defaultMaterial,
		HasFireplace: defaultHasFireplace,
		Floors:       defaultFloors,
	}
	return h
}
```
实际上你可能为了初始化直接写一个函数
```go
func New(material string,hasFireplace bool,floors int) *House {
	return &House{
		Material: material,
		HasFireplace: hasFireplace,
		Floors: floors,
	}
}
```
假如这时发现有些初始化不需要用到有些参数

那你可能还需要再写几个new，何况go还没有重载机制
```go
func NewHouseA(floors int) *House {
	return &House{Floors: floors,}
}
func NewHouseB(hasFireplace bool) *House {
	return &House{HasFireplace: hasFireplace,}
}
```

现在我们用选项模式重构这部分代码

先定义一个函数类型
```go
type HouseOption func(*House)
```
然后写几个匿名函数
```go
func WithConcrete(n string) HouseOption {
	return func(h *House) {
		h.Material = n
	}
}

func WithoutFireplace() HouseOption {
	return func(h *House) {
		h.HasFireplace = false
	}
}

func WithFloors(floors int) HouseOption {
	return func(h *House) {
		h.Floors = floors
	}
}
```
上面的这几组函数，传入参数，返回一个函数，这个函数会设置house其中的成员变量

- 我们调用 WithFloors(3)

- 返回值是 func(h *House) {h.Floors = 3}的函数


最后写一个新的初始化函数
```go
func NewHouse(opts ...HouseOption) *House {
	h := &House{
		Material:     defaultMaterial,
		HasFireplace: defaultHasFireplace,
		Floors:       defaultFloors,
	}
	for _, opt := range opts {
		opt(h)
	}
	return h
}
```
opts是可变参数，可以传入一个好几个HouseOption类型的参数

之前定义了三个匿名函数，就是返回的HouseOption类型的函数，所以可以直接传入

-----
现在我们可以通过NewHouse函数重新初始化对象
```go
func main() {
	house1 = NewHouse(
		WithConcrete("wood"),
		WithoutFireplace(),
		WithFloors(3),
	)

    house2 = NewHouse(
        WithFloors(7),
		WithConcrete("unknown"),
	)
}
```
重构的效果是显而易见的，高度可配置化，甚至不用考虑顺序。

## Uber对功能选项的建议

Uber在功能选项的实现建议就有所不同，他们不使用匿名函数，转而使用接口实现。

参考[Uber关于选项模式的建议](https://github.com/xxjwxc/uber_go_guide_cn#%E5%88%9D%E5%A7%8B%E5%8C%96%E7%BB%93%E6%9E%84%E4%BD%93)

我们来修改上面的例子，该用接口实现

先定义接口
```go
type Option interface {
	apply(*House)
}
```
然后对house的每个成员变量都添加如下的代码
```go
// house中的material字段
type materialOption string

func (m materialOption) apply(house *House) {
	house.Material = string(m)
}

func WithMaterial(m string) materialOption {
	return materialOption(m)
}

// house中的floors字段
type FloorsOption int

func (f FloorsOption) apply(house *House) {
	house.Floors = int(f)
}

func WithFloor(n int) FloorsOption {
	return FloorsOption(n)
}
```
现在我们就可以修改原有的初始化方法

原先传入的是函数类型，现在是接口类型

```go
func NewHouse(opts ...Option) *House {
	h := &House{
		Material:     defaultMaterial,
		HasFireplace: defaultHasFireplace,
		Floors:       defaultFloors,
	}
	for _, opt := range opts {
		opt.apply(h)
	}
	return h
}
```

按原文的话说
> Note that there's a method of implementing this pattern with closures.

> but we believe that the pattern above provides more flexibility for authors and is easier to debug and test for users. 

> In particular, it allows options to be compared against each other in tests and mocks, versus closures where this is impossible. Further, it lets options implement other interfaces, including fmt.Stringer which allows for user-readable string representations of the options.

在某些情况，定义接口可能是更灵活的方式，但具体还没遇到= =

## 参考

https://github.com/uber-go/guide/blob/master/style.md#functional-options

https://coolshell.cn/articles/21146.html

https://www.sohamkamani.com/golang/options-pattern/


