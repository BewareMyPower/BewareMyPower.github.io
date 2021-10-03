---
title: 重新学习 Golang
date: 2021-10-03 20:46:01
tags:
---

## 前言

第一次接触 Go 还是 2018 年在 360 实习的时候，当时调研一个 Kafka 监控工具 [Burrow](https://github.com/linkedin/Burrow)，还贡献了我在开源社区的 [第一个 PR](https://github.com/linkedin/Burrow/pull/439/files)。到后来正式入职前，还特地过了遍 The Way of Go，再后来也基本搁置了，偶尔帮业务排查下问题的时候会简单写个例子。当时还在用 GOPATH，现在 Burrow 官方文档都表示最低支持 Golang 1.11 和 Go module 的管理方式了。总的来说感觉目前由于云原生的火热，Go 的使用确实比较广，国庆刚好休息下，就重新看看了。

## Get Started

### 基本概念

- module（模块）：一组 package 的集合，可以直接从版本控制仓库或者模块代理服务器上下载。module 由 go.mod 文件中的 module path 以及 module 依赖决定。module 根目录是包含 go.mod 文件的目录，main module 是包含这个目录的模块，并且在该目录下可以执行 go 命令。
- module path（模块路径）：仅作为 package 的 import path 的前缀，表明了 go 命令应该在哪去下载。
- package（包）：同一个目录下的一组源文件的集合。
- package path：也就是 package 的 import path。是 module path 加上包含该 package 的子目录的路径。例如 `"golang.org/x/net"` 在目录 `"html"` 下面包含一个 package，这个 package 的路径是 `"golang.org/x/net/html"`。

### 第一个程序

首先指定一个 module path（这里是 `example.com/user/hello`）创建 go.mod 文件：

```bash
$ go mod init example.com/user/hello
go: creating new go.mod: module example.com/user/hello
go: to add module requirements and sums:
        go mod tidy
```

可以发现生成了 go.mod 文件，内容为：

```go
module example.com/user/hello

go 1.16
```

然后创建 main.go，包含以下代码：

```go
package main

import "fmt"

func main() {
	fmt.Println("hello")
}
```

使用 `go` 工具安装并运行程序：

```bash
$ go install example.com/user/hello
$
```

该命令会构建 `hello` 命令并生成二进制文件 `hello`，安装到 `GOPATH` 目录（可通过 `go env GOPATH` 查看，默认是 `$HOME/go`）的 `bin/` 子目录下。如果 `GOBIN` 已经设置，则会安装到 `GOBIN` 下面。

也可以直接用以下方式将其导入环境变量：

```bash
export PATH=$PATH:$(dirname $(go list -f '{{.Target}}' .))
```


设置 Go 环境变量的方式：

```bash
$ go env -w GOBIN=$PWD
```

这样就会安装到当前目录（注意这只是示例，而且会永久生效，最好改到需要的路径）。如果要取消 Go 环境变量，使用 `-u` 选项即可：

```bash
$ go env -u GOBIN
```

### 从 module 中导入 package

创建子目录 *morestrings/*，然后在该目录下创建文件 *reverse.go*：

```go
// Package morestrings implements additional functions to manipulate UTF-8
// encoded strings, beyond what is provided in the standard "strings" package.
package morestrings

// ReverseRunes returns its argument string reversed rune-wise left to right.
func ReverseRunes(s string) string {
	r := []rune(s)
	for i, j := 0, len(r)-1; i < len(r)/2; i, j = i+1, j-1 {
		r[i], r[j] = r[j], r[i]
	}
	return string(r)
}
```

进入该目录，运行 `go build`，可以发现不会生成文件，但实际上已经编译到本地 build cache 里了。

现在修改 *hello.go* 内容，导入刚才的 package：

```go
package main

import (
	"fmt"

	"example.com/user/hello/morestrings"
)

func main() {
	fmt.Println(morestrings.ReverseRunes("!oG ,olleH"))
}
```

然后重新 `go install example.com/user/hello` 编译。 

### 调用外部 package 的代码

Go 可以导入版本控制系统比如 Git 的 package 源码，`go` 工具使用这个属性去从远程仓库自动下载 package，比如：

```go
package main

import (
	"fmt"

	"example.com/user/hello/morestrings"
	"github.com/google/go-cmp/cmp"
)

func main() {
	fmt.Println(morestrings.ReverseRunes("!oG ,olleH"))
	fmt.Println(cmp.Diff("Hello World", "Hello Go"))
}
```

然后运行：

```bash
$ go mod tidy
go: finding module for package github.com/google/go-cmp/cmp
go: found github.com/google/go-cmp/cmp in github.com/google/go-cmp v0.5.6
```

然后查看 *go.mod* 内容，可以发现它新加了一行：

```go
require github.com/google/go-cmp v0.5.6
```

因为 `go mod tidy` 命令会添加缺失的 module requirements，同时移除不再使用的模块 requirements。在导入新 module 时，会从网上下载。

module 依赖会自动下载到 `$GOPATH/pkg/mod` 目录下面，下载内容会被所有其他 module 共享，要删除所有下载的 module，可以传递 `-modcache` 标志给 `go clean`：

```bash
$ go clean -modcache
$
```

### 测试

一般在包目录下面会创建单元测试，比如 *morestrings/reverse.go*，一般会创建 *morestrings/reverse_test.go*。

```go
package morestrings

import "testing"

func TestReverseRunes(t *testing.T) {
	cases := []struct {
		in, want string
	}{
		{"Hello, world", "dlrow ,olleH"},
		{"Hello, 世界", "界世 ,olleH"},
		{"", ""},
	}
	for _, c := range cases {
		got := ReverseRunes(c.in)
		if got != c.want {
			t.Errorf("ReverseRunes(%q) == %q, want %q", c.in, got, c.want)
		}
	}
}
```

然后使用 *go test* 命令运行测试即可：

```bash
$ cd morestrings/
$ go test
PASS
ok  	example.com/user/hello/morestrings	0.401s
```

### 总结

基于 go mod 的方法，创建 go.mod 指定当前的 **模块路径**：

```bash
go mod init <module-path>
```

模块路径一般是多级的，比如 *github.com/google/go-cmp/cmp*，可以发现项目是在 https://github.com/google/go-cmp/，那么模块路径就是 *github.com/google/go-cmp*，也就是项目下载地址，而包路径则是 `cmp` 目录。

然后代码中如果引入了外部依赖，需要在 *go.mod* 里导入依赖：

```bash
go mod tidy
```

如果依赖在本地不存在，它会去下载依赖。不仅在 *go.mod* 中添加依赖模块路径和版本，还会在 *go.sum* 文件中引入模块元数据（tag 或 commit 信息，Base64 编码的 Hash 码）。

在代码中导入 **包** 的方式是：

```go
import "<module-path>/<package-path>"
```

包路径即包目录的相对路径。如果包路径是多级的，比如 *dir1/dir2/dir3*，那么包名是最后一级 *dir3*。

构建即 `go build`，安装即 `go install`，可以使用 `go env` 相关命令设置环境变量，比如 `GOBIN` 配置安装目录。

一般对于模块下的 *xxx.go* 一般创建对应的测试文件 *xxx_test.go*，导入 *testing* 包然后使用相关对象比如 `T` 进行验证。在模块目录下运行 `go test` 即可。

## Tour of Go 笔记

https://tour.golang.org/welcome/1

### 基础知识

Go 没有继承，也没有 `public`/`private` 这种关键字，导入一个包时，包内的函数/类/变量等，只有大写字母开头的才可以导入。

函数参数的定义基本是较新语言的 `<value> <type>` 的顺序，此外连续参数可以合并，比如 `x int, y int` 可以合并为 `x, y int`。Go 可以返回多值，这使得交换操作可以写成 `x, y = y, x` 这种形式。

函数的返回值也可以被命名，这样就不用显式 return 返回值了，但还是需要 return。

```go
func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return
}
```

> 对于这个例子可能不太好，但是对于那种多返回值，且一个返回值在很多分支之前都是同样的情况，可能比较适用。

Go 的基本类型：

- `bool`
- 普通整型：`int`/`uint`，或者加上位数的后缀，比如 `uint8`，位数包括8，16，32，64。不带位数的话，能存储的位数取决于平台，最多为 64。
- `uintptr`：指针类型，也是整型。Go 的指针不像 C 一样支持数学运算，从而保证安全。
- `byte`：`uint8` 的别名，代表一个字节
- `rune`：`int32` 的别名，它表示一个 Unicode 码点，因为对于 ASCII 字符串而言，基本单位是 `int8`（也就是 `char`），但是 Go 对字符串只支持 Unicode 编码，因此没有使用 `char` 关键字，而是使用 `rune`，代表一个 Unicode 字符。因此在 Go 里，字符和字节不会混淆。
- 浮点数：`float32`，`float64`
- 复数：`complex64`，`complex128`。

> Go 保证变量不赋予初值的话，默认是**零值**。数值类型都是 0，布尔类型是 false，字符串是空字符串 `""`（注意不是 **空值** `nil`）。

Go 的变量初始化可以推断类型，整型默认为 `int`，浮点数类型默认为 `float64`，如果没有初始值，则必须显式注明类型。此外初始化还支持 `:=` 这种形式（但是必须在函数体内使用），比如：

```go
var x = 10
var y int
y = x + 10
z := 30
```

常量（`const` 修饰）因为必须有初值，所以基本可以省略类型声明。

Go 的格式化打印不是很灵活，只支持 C 风格的格式化字符串（使用 `fmt.Printf`）或者直接打印任意类型（使用 `fmt.Println` 或 `fmt.Print`）。但是在 C 风格打印时，Go 支持 `%T` 打印类型名，`%v` 打印任何类型，还能用 `%#v` 对类型本身进行修饰，比如：

```go
fmt.Printf("%T x = %v, %T y = %#v\n", x, x, y, y)
```

打印的是 `int x = 10, string y = "hello"`，注意如果不用 `%#v` 的话，不会给字符串加上引号。

Go 的类型转换有点像函数调用，比如 `T(v)` 将 `v` 转换成类型 `T`。Go 没有隐式类型转换，因此即使将 `int32` 转换成 `int` 也要显式进行，比如：

```go
var x int32 = 1
var y int = int(x)
```

Go 的循环只支持 `for` 循环，循环体必须有大括号，而初始化（可选），条件判断以及后置语句（可选）的部分则不用。`if` 也类似。比如：

```java
  for i := 0; i < 100; i++ {
		if i%3 == 0 {
			fmt.Println(i)
		}
	}
```

无限循环可以缩写为

```go
for {
  // ...
}
```

`if` 也支持初始化，并且初始化语句的值可以在 `else` 分支使用，比如：

```go
	var x int = 100
	if y := x / 2; y%2 == 0 {
		fmt.Println(y)
	} else {
		fmt.Println(y)
	}
```

注意在 `for` 和 `if` 的初始化语句里，只能用 `:=` 的形式，因此要声明类型必须对 `:=` 右边的表达式进行类型转换。另外，Go 虽然也支持 `else if`，但这种情况下，最好使用 `switch`。Go 的 `switch` 默认不会 fallthrough，因此不需要对每个分支加 `break`。

`switch` 比较灵活，其实更类似把 `if` 语句做个包装，比如：

```go
	switch x := 1; true { // switch <initialize>; <variable>
	case x < 0:
		fmt.Println("x < 0")
	case x == 0:
		fmt.Println("x = 0")
	case x > 0:
		fmt.Println("x > 0")
	}
```

这里的 `true` 可以省略（因为默认的 `<variable>` 就是 `true`），比如上面代码可以改为 `switch x := 1; {`。这里我这么写实际上想表达，上面语句是将 `true` 与下面的 `case` 后接的表达式（比如 `x < 0`）求值依次进行比较。

> 这次重新学习的时候，差点以为 Go 的 `switch` 语句像 Scala，Rust 的模式匹配一样强大，实际上，虽然也比较强大就是了。

最后 Go 关键字里最关键的 `defer`，类似于 C++ 的析构，举个例子：

```go
func main() {
	x := 100
	defer fmt.Printf("defer %v\n", x)
	fmt.Println("main")
}
```

打印：

```
main
defer 100
```

因为 `defer` 语句会在脱离当前作用域时执行。

### 指针和结构体

Go 也有指针类型，但是它更多的只是作为对象的地址，不像 C 一样支持算术运算。因为 Go 没有所谓的引用类型，都是传值，对于复杂结构，传指针起到了传递引用的作用，并且避免了对象拷贝。示例：

```go
	var p *int
	i := 10
	p = &i
	fmt.Printf("i = %v (%v)\n", *p, p)
```

Go 的函数可以像值一样直接传递，而不是像 C 一样需要传递函数指针。Go 的闭包也就是匿名函数，函数签名里少了函数名。闭包可以访问函数体之外的变量，类似 C++ 的 lambda 表达式捕获所有引用（`[&]`）。

Go 有结构体，语法和 C 类似。这里有个特殊的语法，Go 的结构体指针（假如为 `p`）可以直接访问字段，比如 `p.X`，而不用 `(*p).X`。 这是为了简单实现『方法』而用的。不同于 Java 这种面向对象语言，它没有方法，对于方法的定义，是在函数定义的 `func` 以及参数列表之间插入结构体的值或指针（也就是所谓的 **接收者**），在这个前提下，针对结构体指针访问字段的语法就能大量简化代码，比如：

```go
type Vertex struct {
	X, Y int
}

// 这里也可以写为 func (v Vertex) ToString() string，即将值而非指针作为接收者（但大多数情况下没必要）
func (v *Vertex) ToString() string {
	return fmt.Sprintf("Vertex{X: %d, Y: %d}", v.X, v.Y)
}

func main() {
	v1 := Vertex{1, 2} // 依次初始化
  // 这里用到了结构体部分初始化的语法，即用 Name: Value 的语法列出部分字段
	v2 := Vertex{X: 1} // 仅初始化 X，Y 为默认值 0
	fmt.Printf("v1: %s, v2: %s\n", v1.ToString(), v2.ToString())
}
```

Go 也没有继承，但是可以通过匿名字段的方式实现继承，也就是结构体嵌入。（似乎官方 go tour 教程里并没有讲这个）

```go
type Base struct {
	i int
}

func (b *Base) String() string {
	return fmt.Sprintf("Base: %d", b.i)
}

type Derived struct {
	Base // 匿名字段

	s string
}

// 如果不实现 String() 方法，那么 Derived 对象也能调用父类的 String() 方法
func (d *Derived) String() string {
	return fmt.Sprintf("Derived: %s, %s", d.Base.String(), d.s) // d.Base 直接访问父类对象
}
```

### 数组，切片，映射

数组，切片，映射都是引用语义，因此想要修改内部时，不必传递指针。默认值都是 `nil`，也就是底层为空，但是像 `len` 和 `cap` 都能成功调用并返回 0。

Go 的数组也类似 C，即是定长数组比如 `[n]T`，表示数组元素类型为 `T`，数量为 `n` ，也可以采用大括号初始化的方式：

```go
x := [5]int{1, 1, 2, 3, 5}
```

注意，Go 的数组不像 C 一样可以推断出数组长度，如果你写成了下面这样：

```go
x := []int{1, 1, 2, 3, 5}
```

此时 `x` 的类型是切片（`[]int`）而非数组（`[5]int`）。相比数组而言，切片可以动态增长，类似 C++ 的 `vector`，因此对于切片 `x`，可以用 `cap(x)` 取得切片 `x` 的容量，`len(x)` 取得切片的长度，以及用 `make` 进行初始化（`make(<type>, <len>)` 或者 `make(<type>, <len>, <cap>)`，用 `append` 添加元素。但这些都是 Go 的内置函数。

数组和切片都支持 `x[low:high]` 这种形式的部分引用，`low` 缺省为 0，`high` 缺省为 `len(x)`（数组或切片的长度）。而 `x[low:high]` 本身类型其实也是切片，也就是说数组可以方便地转换成切片。如果 `high` 超出切片长度上限但没有超出容量上限，会扩充切片长度到 `high`。

```go
func printSlice(s []int) {
	fmt.Printf("%v len=%d cap=%d\n", s, len(s), cap(s))
}

func main() {
	s := make([]int, 5, 10)
	printSlice(s)
	s = s[:10]
	printSlice(s)
	s = s[:20] // 超出容量上限，引发 panic
}
```

输出：

```
[0 0 0 0 0] len=5 cap=10
[0 0 0 0 0 0 0 0 0 0] len=10 cap=10
panic: runtime error: slice bounds out of range [:20] with capacity 10
```

> panic 类似其他语言的 **异常**。Go 对错误处理的态度是，它认为异常（panic）默认是不可恢复的，对于可恢复的错误，一般使用错误码或字符串保存。当然，万不得已要恢复异常，可以用 recover 内置函数。不像 Java 这种用 unchecked exception 和 checked exception 来区分这两种错误，而且很多时候写代码的人都没注意好。

如果要扩充容量，那么就必须用内置的 `append` 函数，它可以添加一个或多个元素。可以用以下代码查看切片的扩容策略。

```go
func printSliceMetadata(s []int) {
	fmt.Printf("len=%d cap=%d\n", len(s), cap(s))
}

func main() {
	s := make([]int, 1)
	for i := 0; i < 100; i++ {
		s = append(s, i)
		printSliceMetadata(s)
	}
}
```

对于切片（以及映射），可以用 `range` 来进行遍历：

```go
	s := []int{1, 1, 2, 3, 5}
	for i, v := range s {
		fmt.Printf("%d %v\n", i, v)
	}
```

注意，`range` 遍历映射返回 `(key, value)` 比较直观，但是对于切片，实际上也是返回一对值 `(index, value)`，这样方便操作下标。如果不想取得值或下标，可以用 `_` 隐去，比如下列代码就是只打印切片 `s` 的值。

```go
	for _, v := range s {
		fmt.Printf("%v\n", v)
	}
```

至于映射，直接用个 demo 表示吧，顺便复习下之前的知识。当前项目的模块路径为 *com.example/temp*，新建 *wrapper/map.go*，内容如下（实现了类似 Java `Map` 的接口）：

```go
package wrapper

import (
	"errors"
	"fmt"
)

type Map struct {
	internalMap map[int]string
}

func NewMap(m map[int]string) Map {
	return Map{m}
}

func (m *Map) ContainsKey(key int) bool {
	_, ok := m.internalMap[key]
	return ok
}

func (m *Map) Put(key int, value string) {
	m.internalMap[key] = value
}

func (m *Map) Get(key int) (value string, err error) {
	value, ok := m.internalMap[key]
	if !ok {
		err = errors.New(fmt.Sprintf("Map doesn't contain key %v", key))
	}
	return
}

func (m *Map) Remove(key int) (value string, err error) {
	value, ok := m.internalMap[key]
	if ok {
		delete(m.internalMap, key)
	} else {
		err = errors.New(fmt.Sprintf("Map doesn't contain key %v", key))
	}
	return
}
```

然后 *main.go* 如下：

```go
package main

import (
	"fmt"

	"com.example/temp/wrapper"
)

func checkKey(m *wrapper.Map, key int) {
	value, err := m.Get(key)
	if err == nil {
		fmt.Printf("Get %d: %s\n", key, value)
	} else {
		fmt.Println(err.Error())
	}
}

func main() {
	m := wrapper.NewMap(map[int]string{1: "hello"})
	fmt.Println(m.ContainsKey(1))
	fmt.Println(m.ContainsKey(2))
	checkKey(&m, 2)
	m.Put(2, "world")
	checkKey(&m, 2)
}
```

运行结果：

```
true
false
Map doesn't contain key 2
Get 2: world
```

> Remove 方法的示例和 Get 类似，在 *main.go* 里就不写了，总之套路就是返回正确值和 `error` 接口。很多其他语言用户觉得这种处理很丑陋，但其实我觉得还好。入乡随俗。

### 接口

基于对象编程只需要 **结构体** 和 **方法** 即可，但是面向对象编程则需要 **结构体** 和 **接口** 结合。接口的语法其实就是在 `type <interface-name> interface` 代码块中定义一系列函数，但是不需要 `func` 前缀。还是用个 demo 比较明确。

```go
// 1. 二员操作符接口
type BinaryOperation interface {
	Calculate(x, y int) int
}

// 2. 加操作和乘操作的结构体，并且都实现了 Calculate 方法（函数参数和返回类型都一样）
type AddOperation struct{}
type ProductOperation struct{}

func (op AddOperation) Calculate(x, y int) int {
	return x + y
}

func (op ProductOperation) Calculate(x, y int) int {
	return x * y
}

func runBinaryOperation(op BinaryOperation, x int, y int) int {
	return op.Calculate(x, y)
}

func main() {
  // 3. 因此加操作和乘操作都可以视为二员操作符
	x := runBinaryOperation(AddOperation{}, 2, 3)
	y := runBinaryOperation(ProductOperation{}, 2, 3)
	// ...
}
```

Duck typing 使得 Golang 实现多态非常灵活，无需像 Java 一样显式 `implements` 某个接口，不然无法被转换成对应的接口类型。典型的接口在我们之前用 `errors.Error` 接口保存错误信息时已经看到了。此外，Golang 的 `Stringer` 接口用于将任意结构体转换成字符串，从而可以被 `%v` 格式化打印。这点类似于 Java 的 `toString()` 方法。

需要注意的是，如果是将结构体指针作为方法的接收者，那么必须要通过指针访问才会被视为该接口，比如：

```go
type Vertex struct {
	X, Y int
}

func (v *Vertex) String() string {
	return fmt.Sprintf("Point(%d, %d)", v.X, v.Y)
}
```

这里实现了接口 `Stringer` 的是 `*Vertex`，而不是 `Vertex`。因此对于以下调用：

```go
	v := Vertex{Y: 100}
	fmt.Println(v)
	fmt.Println(&v)
```

输出：

```
{0 100}
Point(0, 100)
```

因此很多时候，直接像这样初始化：

```go
	v := &Vertex{Y: 100} // v 是结构体指针
```

另一方面，duck typing 也使得 Go 可以用 `interface{}` 类型表示任意类型（类似 Java 的 `Object`）。由于 Golang（目前为止，1.17）没有泛型，使得它要对某种类型进行抽象，只能传递 `interface{}`，比如上述代码就限定了参数类型都是 `int`。比如在 C++ 中可以定义这样的接口：

```c++
template <typename T>
struct BinaryOperation {
    T Calculate(T x, T y);
};
```

但是 Golang 里，只能这么干了：

```go
type BinaryOperation interface {
  Calculate(x, y interface{}) interface{}
}
```

但是这么实现很蛋疼，因为虽然任何对象都可以转换成 `interace{}`，但是 `interface{}` 不能直接转换为其他类型，比如我基于新的 `BinaryOperation` 接口尝试写下面这段代码：

```go
func (op AddOperation) Calculate(x, y interface{}) interface{} {
	return int(x) + int(y)
}
```

编译会报错：

> cannot convert x (type interface {}) to type int: need type assertion

在 Golang 里需要进行类型断言，`t := i.(T)` 使得 `i` 的实际类型为 `T` 时将其转换成 `T` 类型的变量 `t`，如果类型不匹配将会触发 panic。而 `t, ok := i.(T)` 则可以避免 panic，类型不匹配时 `ok` 为 `false`。因此上述的实现可能变成：

```go
func (op AddOperation) Calculate(x, y interface{}) interface{} {
	return x.(int) + y.(int)
}

func main() {
	sum := AddOperation{}.Calculate(1, 2)
	fmt.Println(sum.(int))
}
```

如果要支持多种类型，比如 `float`，就得利用`t, ok := i.(T)` 的形式了，但是可以用 `switch` 进行简化：

```go
func (op AddOperation) Calculate(x, y interface{}) interface{} {
	switch vx := x.(type) {
	case int:
		if vy, ok := y.(int); ok {
			return vx + vy
		} else {
			panic("x is int while y is not int")
		}
	case float64:
		if vy, ok := y.(float64); ok {
			return vx + vy
		} else {
			panic("x is int while y is not int")
		}
	default:
		panic("x is not int or float64")
	}
}
```

但是写起来还是很复杂……而且似乎不能写成：

```go
switch vx, vy := x.(type), y.(type) {
```

这种形式。

我可以理解这么做的原因，因为在没有类型检查的情况下向下转型很危险。C++ 的模板相对一般语言的泛型比较特殊，可以简单地写成：

```c++
template <typename T>
struct AddOperation {
  T add(T x, T y) { return x + y; }
};
```

但如果 `T` 的类型不支持 `operator+`，那么编译错误会非常庞大，这也是 C++20 引入 constrains 和 concepts 的原因。

### Goroutine

首先可以把 goroutine 当成轻量级线程（Fiber）来使用，这里为求方便，不去纠结名词细节，**下文中若不特别说明，『线程』直接代指『goroutine』**。总之起一个线程语法很简单，就是 `go` 后接函数调用，甚至可以是匿名函数。

```go
go func (s string) {
  fmt.Println(s);
}("hello") // 定义匿名函数 func (s string)，并传递参数 "hello"
```

因此类似其他语言，Golang 的线程同步也可以用锁和条件变量之类，具体见 [`sync` 包](https://pkg.go.dev/sync@go1.17.1)。一般更多地是推荐用 channel（我更喜欢用『管道』这个翻译）进行同步。Channel 其实就是个 FIFO 队列，对于一个 channel（假如为 `ch`）：

- `<- ch`：**读操作**。从 channel 中去取一个值，这个操作具有返回值，可以将其赋予一个变量，比如 `v := <-ch`。
- `ch <- v`：**写操作**。将值 `v` 放入管道。

其实就类似 setter 和 getter 嘛。Channel 类型一般是 `chan T`，`T` 为 channel 元素的类型，直接用 `make(chan T)` 或 `make(chan T, n)` 的方式创建，指定 `n` 则代表 channel 带缓冲区（前提是 `n` 大于 0）。如果 channel 满了（或者没缓冲区），则写操作会阻塞直到读操作完成；如果 channel 为空，则读操作会阻塞直到 channel 有元素。这也是基于 channel 进行线程同步的基础。当然，对于单线程而言，这两个操作会直接触发 panic。就类似 Java 的 `ArrayBlockingQueue` 在队列为空时进行 `remove` 或者队列为满时进行 `add` 一样。但是 Golang channel 的特殊之处在于，多线程环境下会将 `remove`/`add` 的语义改成 `take`/`put` 的语义。

此外，channel 可以进行 `range` 操作，其实就是一个语法糖，代替在无限循环中进行读操作直到 channel 被关闭。对于一个 channel `ch`，关闭操作（`close(ch)`）意味着 `ch` 之后不再可用（无法进行读写，否则会 panic）。

Golang 还提供了 `select` 语句，这个名字基本上就意味着它和 I/O 多路复用的场景。I/O 多路复用本质上是可以同时等待多个文件描述符的事件（包括读就绪，写就绪，错误就绪）。`select` 即可以等待多个 channel 的读写事件。

这里举个例子：

```go
func eventLoop(c, quit chan int) {
	cnt := 1
	flag := true
	for flag {
		select {
		case c <- cnt: // 写事件，c 未满即可触发
			fmt.Printf("Push %d\n", cnt)
			cnt++
		case <-quit: // 读事件，quit 未空即可触发
			fmt.Println("Quit")
			flag = false
			break // 跳出 select block
		default: // 其他事件都未就绪时执行
			time.Sleep(500 * time.Millisecond)
		}
	}
}

func main() {
	ch := make(chan int, 2)
	quit := make(chan int)

	go func() {
		time.Sleep(300 * time.Millisecond)
		<-ch // 取出元素，使得 eventLoop 的 case c <- cnt 可以就绪
    time.Sleep(100 * time.Millisecond)
		quit <- 0 // 触发 eventLoop 的 case <-quit
	}()
	eventLoop(ch, quit)
}
```

多次运行，可以发现输出结果有两种，一种是

```
Push 1
Push 2
Quit
```

另一种是

```
Push 1
Push 2
Push 3
Quit
```

原因是在 push 1 和 2 之后，`c <- cnt` 和 `<- quit` 都处于阻塞状态，因此进入 `default` 分支等待 500 ms。在此期间，子线程中的 `<-ch` 和 `quit <- 0` 使得两个事件都就绪了。此时 `select` 语句触发的顺序不是固定的。

在做等价二叉查找树练习的时候，发现一些 channel 被忽视的用法。Golang 不允许声明不被使用的变量，因此仅仅是想判断 channel 是否还有多余值时，可以用：

```go
for range ch {
  fmt.Println("There's a value in channel")
}
```

另外，默认读取空 channel 时会阻塞，但是有一种非阻塞的方式，和之前的类似：

```go
elem, ok := <-ch // 如果 ch 暂时没有元素，则 ok 为 false，elem 为 nil
```

## 下一步

总的来说 Go 的语言设计最大的优点就是作为一门广泛使用的非脚本语言，上手很快，就这一两天把 Tour of Go 过了遍，感觉直接上手写代码是足够了。

不过感觉也可以抽时间看看：

- [Effective Go](https://golang.org/doc/effective_go)：感觉类似 Effective C++，各种 tips。
-  [Uber Go 语言编码规范](https://github.com/xxjwxc/uber_go_guide_cn)：也就是编码上最佳实践类型的。
- [Debugging Go Code with GDB](https://golang.org/doc/gdb)：毕竟除了 Java 系语言我都不用 IDE。另外官网推荐了 [dlv](https://github.com/go-delve/delve) 进行调试。

当然，也有两个词典性质的网站：

- [Go 语言标准](https://golang.org/ref/spec)
- [Go 标准库](https://pkg.go.dev/std)
