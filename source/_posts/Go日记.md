---
title: Go日记
comments: true
tags:
  - GAP
  - Go
  - Tips
categories: Go
abbrlink: '2766'
date: 2019-11-26 09:11:49
---

 记录下休学期间学习Go语言的一些想法、笔记和踩过的一些坑。

<!-- More -->

# 安装下载

因为一些奇怪的原因我分别在Windows和Linux子系统上安装了Go。

## Windows

[下载地址](<https://golang.org/dl/>) 笔者现安装的版本是`go version go1.13.4 windows/amd64`

安装完成后默认会在环境变量 Path 后添加 Go 安装目录下的 bin 目录 `C:\Go\bin\`，并添加环境变量 GOROOT，值为 Go 安装根目录 `C:\Go\`因为笔者电脑内存不够，于是放在了D盘，于是修改一通环境变量，最主要就是GOROOT目录下存在go.exe以及你的代码放置区域要存在GOPATH里。

## Linux

*强烈建议不要直接apt install而是去官网下载最新的版本手动安装*

因为之前C++课程使用的VSCode接WSL确实用得舒服一点，就直接在WSL环境下ubuntu使用Go
第一次apt直接安装`sudo apt install golang-go`版本为`go version go1.10.4 linux/amd64`然后笔者觉得还是统一下会比较好一点（强迫症）于是去官网下了Linux的1.13版本解压在本地安装，然后因为卑微的C盘，于是把go文件夹放在了D盘，由于之Windows版本的Go放在D盘就给Linux版本文件夹重命名了一下，修改PATH：
`export GOPATH=/mnt/e/Program/Go`
`export GOROOT=/mnt/d/Go_linux`
`export PATH=$PATH:$GOROOT/bin:$GOPATH/bin`
`source /etc/profile`

## 检查

至此理论上就能跑了，不放心可以用`go version`检查版本`go env`检查环境变量。

# Hello World

## 基础结构

用hello world信仰开头：
```Go
package main
import "fmt"
func main() {
    fmt.Printf("Hello, world 你好，世界 καλημ ́ρα κóσμ こんにちはせかい\n")
}
```

Go程序是通过**package**来组织的`package <pkgName>`这一行告诉我们当前文件属于哪个包，包名**main**表明它是一个可独立运行的包，在编译后会产生可执行文件。除了main包之外，其它的包最后都会生成***.a**文件（也就是包文件）并放置在`$GOPATH/pkg/$GOOS_$GOARCH`中。
每个可独立运行的Go程序，必定包含一个`package main`其中必定包含一个无参无return的入口函数**main**。

Go使用UTF-8字符串和标识符。

## 变量

使用`var`关键字是Go最基本的定义变量方式，Go把变量类型放在变量名后面：

```Go
//初始化“variableName”的变量为“value”值，类型是“type”
var variableName type = value
//定义三个变量，分别初始化相应值，编译器会根据初始化值自动推导相应类型
var vname1, vname2, vname3 = v1, v2, v3
/*
    :=这个符号直接取代了var和type，但只能用在函数内部
    定义全局变量一般还是猜用var方式来
*/
vname1, vname2, vname3 := v1, v2, v3
```

`_`是个特殊的变量名，任何赋予它的值都会被丢弃。eg.我们将值35赋予b，并同时丢弃34：

```
_, b := 34, 35
```

**Go对于已声明但未使用的变量会在编译阶段报错**，eg.声明了`i`但未使用。

```Go
package main

func main() {
    var i int
}
```

### 常量

在Go程序中，常量可定义为数值、布尔值或字符串等类型。

```Go
const constantName = value
//如果需要，也可以明确指定常量的类型：
const Pi float32 = 3.1415926
```

Go 常量和一般程序语言不同的是，可以指定相当多的小数位数(例如200位)， 若指定給float32自动缩短为32bit，指定给float64自动缩短为64bit，详情参考[链接](http://golang.org/ref/spec#Constants)

### 基础类型

#### Boolean

```Go
//在Go中，布尔值的类型为bool，值是true或false，默认为false。
var isActive bool  // 全局变量声明
var enabled, disabled = true, false  // 忽略类型的声明
func test() {
    var available bool  // 一般声明
    valid := false      // 简短声明
    available = true    // 赋值操作
}
```

#### 数值类型

Go同时支持`int`和`uint`，两种类型长度相同，但具体长度取决于编译器的实现。
Go里面也有直接定义好位数的类型：`int8`, `int16`, `int32(rune)`, `int64`和`uint8(byte)`, `uint16`, `uint32`, `uint64`。
**不同类型的变量之间不允许互相赋值或操作！**

浮点数的类型有`float32`和`float64`两种（没有`float`类型），默认是`float64`。

复数默认类型是`complex128`（64位实数+64位虚数）也有`complex64`(32位实数+32位虚数)
复数的形式为`RE + IMi`，其中`RE`是实数部分，`IM`是虚数部分，而最后的`i`是虚数单位。

#### 字符串

```Go
//Go中的字符串都是采用UTF-8字符集编码
var frenchHello string  // 声明变量为字符串的一般方法
var emptyString string = ""  // 声明了一个字符串变量，初始化为空字符串
func test() {
    no, yes, maybe := "no", "yes", "maybe"  // 简短声明，同时声明多个变量
    frenchHello = "Bonjour"  // 常规赋值
}
```

在Go中字符串不能当初char数组修改，例如下面的代码编译时会报错：cannot assign to s[0]

```Go
var s string = "hello"
s[0] = 'c'
```

真的需要修改，要将字符串 s 转换为 []byte 类型，修改后再转回 string 类型：

```Go
s := "hello"
c := []byte(s)  // 将字符串 s 转换为 []byte 类型
c[0] = 'c'
s2 := string(c)  // 再转换回 string 类型
fmt.Printf("%s\n", s2)
```

Go中可以使用`+`操作符来连接两个字符串
所以字符串虽不能更改，但可进行切片操作，故修改字符串也可写为：

```Go
s := "hello"
s = "c" + s[1:] // 字符串虽不能更改，但可进行切片操作
fmt.Printf("%s\n", s)
```

如果要声明一个多行的字符串怎么办？可以通过`` `来声明：

```go
m := `hello
    world`
```

`` ` 括起的字符串为Raw字符串，即字符串在代码中的形式就是打印时的形式，它没有字符转义，换行也将原样输出。例如本例中会输出：

```go
hello
    world
```

#### ERROR

Go内置有一个`error`类型，专门用来处理错误信息，Go的`package`里面还专门有一个包`errors`来处理错误：

```Go
err := errors.New("emit macho dwarf: elf header corrupted")
if err != nil {
    fmt.Print(err)
}
```

#### 分组声明

```Go
import "fmt"
import "os"

const i = 100
const pi = 3.1415
const prefix = "Go_"

var i int
var pi float32
var prefix string
```

等价于：

```Go
import(
    "fmt"
    "os"
)

const(
    i = 100
    pi = 3.1415
    prefix = "Go_"
)

var(
    i int
    pi float32
    prefix string
)
```

#### 代码规范

- 大写字母开头的变量是可导出的，也就是其它包可以读取的，是公有变量；
- 小写字母开头的就是不可导出的，是私有变量。
- 大写字母开头的函数相当于`class`中的带`public`关键词的公有函数；
- 小写字母开头的函数相当于`private`关键词的私有函数。

### 内建类型

#### array

```Go
var arr [10]int  // 声明了一个int类型的数组
arr[0] = 42      // 数组下标是从0开始的
arr[1] = 13      // 赋值操作
fmt.Printf("The first one is %d\n", arr[0])  // 获取数据，返回42
fmt.Printf("The last one is %d\n", arr[9]) // 未赋值默认返回0
```

数组间的赋值是**值的赋值**，即当把一个数组作为参数传入函数的时候，传入的其实是该数组的副本，而不是它的指针。如果要使用指针，那么就需要用到后面介绍的`slice`类型了。

```Go
a := [3]int{1, 2, 3} // 简短声明了一个长度为3的int数组
b := [10]int{1, 2, 3} // 简短声明，前三个元素初始化为1、2、3，其它默认为0
c := [...]int{4, 5, 6} // 采用`...`的方式Go会自动根据元素个数来计算长度
```

```Go
// 声明了一个二维数组，该数组以两个数组作为元素，其中每个数组中又有4个int类型的元素
doubleArray := [2][4]int{[4]int{1, 2, 3, 4}, [4]int{5, 6, 7, 8}}
// 上面的声明可以简化，直接忽略内部的类型
easyArray := [2][4]int{{1, 2, 3, 4}, {5, 6, 7, 8}}
```

#### slice

初始定义数组时并不知道数组长度，在Go里面这种数据结构叫slice。
slice并不是真正意义上的动态数组，而是一个引用类型。slice总是指向一个底层array。

```Go
var fslice []int
slice := []byte {'a', 'b', 'c', 'd'}
```

```Go
// 声明一个含有10个元素元素类型为byte的数组
var ar = [10]byte {'a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j'}
// 声明两个含有byte的slice
var a, b []byte
a = ar[2:5] // a含有的元素: ar[2]、ar[3]和ar[4]
b = ar[3:5] // b的元素是：ar[3]和ar[4]
```

array方括号内写明数组长度或使用`...`自动计算长度，声明slice时，**方括号内没有任何字符**。

- slice的默认开始位置是0，`ar[:n]`等价于`ar[0:n]`
- slice的默认结束位置是数组长度，`ar[n:]`等价于`ar[n:len(ar)]`
- 如果从一个数组里面直接获取slice，可以这样`ar[:]`等价于`ar[0:len(ar)]`

slice是引用类型，所以当引用改变其中元素的值时，其它的所有引用都会改变该值。

slice内置函数：
- `len` 获取slice的长度
- `cap` 获取slice的最大容量
- `append` 向slice里面追加一个或者多个元素，然后返回一个和slice一样类型的slice
- `copy` 函数copy从源slice的src中复制元素到目标dst，并且返回复制的元素的个数

注：`append`函数会改变slice所引用的数组的内容，从而影响到引用同一数组的其它slice。 
但当slice中没有剩余空间（即`(cap-len) == 0`）时，此时将动态分配新的数组空间。返回的slice数组指针将指向这个空间，而原数组的内容将保持不变；其它引用此数组的slice则不受影响。

```Go
var array [10]int
slice := array[2:4] // slice的容量10-2，即8
slice = array[2:4:7] // 第三个参数可以指定容量
// 容量为7-2，即5。这样新的slice就没办法访问array最后三个元素
```

#### map
map也就是Python中字典的概念

```Go
// 声明一个key是字符串，值为int的字典,这种方式的声明需要在使用之前使用make初始化
var numbers map[string]int
// 另一种map的声明方式
numbers := make(map[string]int)
numbers["one"] = 1  //赋值
numbers["ten"] = 10 //赋值
numbers["three"] = 3
fmt.Println("第三个数字是: ", numbers["three"]) // 读取数据
// 打印出来如:第三个数字是: 3
```

使用map过程中需要注意的几点：

- map是无序的，每次打印出的map会不一样，它不能通过index获取，而必须通过key获取
- map的长度是不固定的，也就是和slice一样，也是一种引用类型，如果两个map同时指向一个底层，那么一个改变，另一个也相应的改变
- 内置的`len`函数同样适用于map，返回map拥有的key的数量
- map的值很方便修改，通过`numbers["one"]=11`可以把key为one的字典值改为11
- map和其他基本型别不同，它不是thread-safe，在多个go-routine存取时，必须使用mutex lock机制

map的初始化可以通过`key:val`的方式初始化值，同时map内置有判断是否存在`key`的方式

```Go
// 初始化一个字典
rating := map[string]float32{"C":5, "Go":4.5, "Python":4.5, "C++":2 }
// map有两个返回值，第二个返回值，如果不存在key，那么ok为false，如果存在ok为true
csharpRating, ok := rating["C#"]
if ok {
    fmt.Println("C# is in the map and its rating is ", csharpRating)
} else {
    fmt.Println("We have no rating associated with C# in the map")
}
delete(rating, "C")  // 删除key为C的元素
```

#### make、new操作（TODO）

内建函数new和make是两个用于内存分配的原语，简单说new只分配内存，make用于slice，map，和channel的初始化。在Go语言中，如果一个局部变量在函数返回后仍然被使用，这个变量会从heap，而不是stack中分配内存。内建函数make(T, args)与new(T)的用途不一样。它只用来创建slice，map和channel，并且返回一个初始化的(而不是置零)，类型为T的值（而不是*T）。之所以有所不同，是因为这三个类型的背后引用了使用前必须初始化的数据结构。例如，slice是一个三元描述符，包含一个指向数据（在数组中）的指针，长度，以及容量，在这些项被初始化之前，slice都是nil的。对于slice，map和channel，make初始化这些内部数据结构，并准备好可用的值。记住make只用于map，slice和channel，并且不返回指针。要获得一个显式的指针，使用new进行分配，或者显式地使用一个变量的地址。

## 流程控制

Go中流程控制分三大类：条件判断，循环控制和无条件跳转。

### if

Go里面`if`条件判断语句中不需要括号，如下代码所示

```Go
if x > 10 {
    fmt.Println("x is greater than 10")
} else {
    fmt.Println("x is less than 10")
}
```

if语句里允许声明一个变量，变量作用域只能在该条件逻辑块内，有点类似python的`for i in range(100):`

```Go
// 计算获取值x,然后根据x返回的大小，判断是否大于10。
if x := computedValue(); x == 10 {
    fmt.Println("x is equal to 10")
} else if x > 10 {
    fmt.Println("x is greater than 10")
} else {
    fmt.Println("x is less than 10")
}
```

### for

```Go
package main
import "fmt"
func main(){
    sum := 0;
    for index:=0; index < 10 ; index++ {
        sum += index
    }
    fmt.Println("sum is equal to ", sum)
}
// 输出：sum is equal to 45
```

有时我们可以省略一点

```Go
sum := 1
for ; sum < 1000;  {
    sum += sum
}
```

甚至更省略一点，看起来就像个while，配合continue和break用风味更佳

```Go
sum := 1
for sum < 1000 {
    sum += sum
}
```

`for`配合`range`可以用于读取`slice`和`map`的数据（真的很像python啊）

```Go
for k,v:=range map {
    fmt.Println("map's key:",k)
    fmt.Println("map's val:",v)
}
```

由于 Go 支持 “多值返回”, 而对于“声明而未被调用”的变量, 编译器会报错, 于是用`_`来丢弃不需要的返回值 

```Go
for _, v := range map{
    fmt.Println("map's val:", v)
}
```

### switch

```Go
i := 10
switch i {
case 1:
    fmt.Println("i is equal to 1")
case 2, 3, 4:
    fmt.Println("i is equal to 2, 3 or 4")
case 10:
    fmt.Println("i is equal to 10")
default:
    fmt.Println("All I know is that i is an integer")
}
```

Go里面`switch`默认相当于每个`case`最后带有`break`，匹配成功后不会自动向下执行其他case，而是跳出整个`switch`, 但是可以在case最后加上`fallthrough`强制执行后面的case代码。

## 函数

### 声明

```Go
func funcName(input1 type1, input2 type2) (output1 type1, output2 type2) {
    //这里是处理逻辑代码
    return value1, value2
}
```

来个实例：

```Go
package main
import "fmt"

func SumAndProduct(A, B int) (int, int) {
    return A+B, A*B
}

func main() {
    x := 3
    y := 4

    xPLUSy, xTIMESy := SumAndProduct(x, y)

    fmt.Printf("%d + %d = %d\n", x, y, xPLUSy)
    fmt.Printf("%d * %d = %d\n", x, y, xTIMESy)
}
```

改进意见：函数的声明还可以更人性化，可读性更强一点：

```Go
func SumAndProduct(A, B int) (add int, Multiplied int) {
    add = A+B
    Multiplied = A*B
    return
}
```

### 变参

接受变参的函数是有着不定数量的参数的。为了做到这点，首先需要定义函数使其接受变参：

```Go
func myfunc(arg ...int) {}
```

`arg ...int`告诉Go这个函数接受不定数量的参数。注意，这些参数的类型全部是`int`。
在函数体中，变量`arg`是一个`int`的`slice`：

```Go
for _, n := range arg {
    fmt.Printf("And the number is: %d\n", n)
}
```

### 传值与传指针

当我们传一个参数值到被调用函数里面时，实际上是传了这个值的一份copy，当在被调用函数中修改参数值的时候，调用函数中相应实参不会发生任何变化，因为数值变化只作用在copy上。

而想直接传这个值本身就需要用到指针。我们知道，变量在内存中是存放于一定地址上的，修改变量实际是修改变量地址处的内存。只有`add1`函数知道`x`变量所在的地址，才能修改`x`变量的值。所以我们需要将`x`所在地址`&x`传入函数，并将函数的参数的类型由`int`改为`*int`，即改为指针类型，才能在函数中修改`x`变量的值。此时参数仍然是按copy传递的，只是copy的是一个指针。

```Go
package main

import "fmt"

//简单的一个函数，实现了参数+1的操作
func add1(a *int) int { // 请注意，
    *a = *a+1 // 修改了a的值
    return *a // 返回新值
}

func main() {
    x := 3
    fmt.Println("x = ", x)  // 应该输出 "x = 3"
    x1 := add1(&x)  // 调用 add1(&x) 传x的地址
    fmt.Println("x+1 = ", x1) // 应该输出 "x+1 = 4"
    fmt.Println("x = ", x)    // 应该输出 "x = 4"
}
```

这样，我们就达到了修改`x`的目的。那么到底传指针有什么好处呢？

- 传指针使得多个函数能操作同一个对象。
- 传指针比较轻量级 (8bytes),只是传内存地址，我们可以用指针传递体积大的结构体。如果用参数值传递的话, 在每次copy上面就会花费相对较多的系统开销（内存和时间）。所以当你要传递大的结构体的时候，用指针是一个明智的选择。
- Go语言中`channel`，`slice`，`map`这三种类型的实现机制类似指针，所以可以直接传递，而不用取地址后传递指针。（注：若函数需改变`slice`的长度，则仍需要取地址传递指针）

### defer

Go语言中有种不错的设计，即延迟（defer）语句，你可以在函数中添加多个defer语句。当函数执行到最后时，这些defer语句会按照逆序执行，最后该函数返回。特别是当你在进行一些打开资源的操作时，遇到错误需要提前返回，在返回前你需要关闭相应的资源，不然很容易造成资源泄露等问题。

如下代码所示，我们一般写打开一个资源是这样操作的：

```Go
func ReadWrite() bool {
    file.Open("file")
	// 做一些工作
    if failureX {
        file.Close()
        return false
    }
    if failureY {
        file.Close()
        return false
    }
    file.Close()
    return true
}
```

而使用`defer`则会显得优雅很多，在`defer`后指定的函数会在函数退出前调用。

```Go
func ReadWrite() bool {
    file.Open("file")
    defer file.Close()
    if failureX {
        return false
    }
    if failureY {
        return false
    }
    return true
}
```

如果有很多调用`defer`，那么`defer`是采用后进先出模式，所以如下代码会输出`4 3 2 1 0`

```Go
for i := 0; i < 5; i++ {
    defer fmt.Printf("%d ", i) // 4 3 2 1 0
}
```

### 函数作为值、类型

在Go中函数也是一种变量，我们可以通过`type`来定义它

```go
type typeName func(input1 inputType1 , input2 inputType2 [, ...]) (result1 resultType1 [, ...])
```

```Go
package main

import "fmt"

type testInt func(int) bool // 声明了一个函数类型

func isOdd(integer int) bool {
    if integer%2 == 0 {
        return false
    }
    return true
}

func isEven(integer int) bool {
    if integer%2 == 0 {
        return true
    }
    return false
}

// 声明的函数类型在这个地方当做了一个参数
func filter(slice []int, f testInt) []int {
    var result []int
    for _, value := range slice {
        if f(value) {
            result = append(result, value)
        }
    }
    return result
}

func main(){
    slice := []int {1, 2, 3, 4, 5, 7}
    fmt.Println("slice = ", slice)
    odd := filter(slice, isOdd)    // 函数当做值来传递了
    fmt.Println("Odd elements of slice are: ", odd)
    even := filter(slice, isEven)  // 函数当做值来传递了
    fmt.Println("Even elements of slice are: ", even)
}
```

函数当做值和类型在我们写一些通用接口的时候非常有用，通过上面例子我们看到`testInt`这个类型是一个函数类型，然后两个`filter`函数的参数和返回值与`testInt`类型是一样的，但是我们可以实现很多种的逻辑，这样使得我们的程序变得非常的灵活。

### Panic和Recover

Go没有像Java那样的异常机制，它不能抛出异常，而是使用了`panic`和`recover`机制。一定要记住，你应当把它作为最后的手段来使用，也就是说，你的代码中应当没有，或者很少有`panic`的东西。

Panic

> 是一个内建函数，可以中断原有的控制流程，进入一个令人恐慌的流程中。当函数`F`调用`panic`，函数F的执行被中断，但是`F`中的延迟函数会正常执行，然后F返回到调用它的地方。在调用的地方，`F`的行为就像调用了`panic`。这一过程继续向上，直到发生`panic`的`goroutine`中所有调用的函数返回，此时程序退出。恐慌可以直接调用`panic`产生。也可以由运行时错误产生，例如访问越界的数组。

Recover

> 是一个内建的函数，可以让进入令人恐慌的流程中的`goroutine`恢复过来。`recover`仅在延迟函数中有效。在正常的执行过程中，调用`recover`会返回`nil`，并且没有其它任何效果。如果当前的`goroutine`陷入恐慌，调用`recover`可以捕获到`panic`的输入值，并且恢复正常的执行。

下面这个函数演示了如何在过程中使用`panic`

```Go
var user = os.Getenv("USER")

func init() {
    if user == "" {
        panic("no value for $USER")
    }
}
```

下面这个函数检查作为其参数的函数在执行时是否会产生`panic`：

```Go
func throwsPanic(f func()) (b bool) {
    defer func() {
        if x := recover(); x != nil {
            b = true
        }
    }()
    f() //执行函数f，如果f中出现了panic，那么就可以恢复回来
    return
}
```

### `main`函数和`init`函数

Go里面有两个保留的函数：`init`函数（能够应用于所有的`package`）和`main`函数（只能应用于`package main`）。这两个函数在定义时不能有任何的参数和返回值。虽然一个`package`里面可以写任意多个`init`函数，但这无论是对于可读性还是以后的可维护性来说，都强烈建议用户在一个`package`中每个文件只写一个`init`函数。

Go程序会自动调用`init()`和`main()`，所以你不需要在任何地方调用这两个函数。每个`package`中的`init`函数都是可选的，但`package main`就必须包含一个`main`函数。

程序的初始化和执行都起始于`main`包。如果`main`包还导入了其它的包，那么就会在编译时将它们依次导入。有时一个包会被多个包同时导入，那么它只会被导入一次（例如很多包可能都会用到`fmt`包，但它只会被导入一次，因为没有必要导入多次）。当一个包被导入时，如果该包还导入了其它的包，那么会先将其它包导入进来，然后再对这些包中的包级常量和变量进行初始化，接着执行`init`函数（如果有的话），依次类推。等所有被导入的包都加载完毕了，就会开始对`main`包中的包级常量和变量进行初始化，然后执行`main`包中的`init`函数（如果存在的话），最后执行`main`函数。下图详细地解释了整个执行过程：

![main函数引入包初始化流程图](https://astaxie.gitbooks.io/build-web-application-with-golang/zh/images/2.3.init.png?raw=true)

### import

我们在写Go代码的时候经常用到import这个命令用来导入包文件，而我们经常看到的方式参考如下：

```Go
import(
    "fmt"
)
```

然后我们代码里面可以通过如下的方式调用

```Go
fmt.Println("hello world")
```

上面这个fmt是Go语言的标准库，其实是去`GOROOT`环境变量指定目录下去加载该模块，当然Go的import还支持如下两种方式来加载自己写的模块：

1. 相对路径

   import “./model” //当前文件同一目录的model目录，但是不建议这种方式来import

2. 绝对路径

   import “shorturl/model” //加载gopath/src/shorturl/model模块

上面展示了一些import常用的几种方式，但是还有一些特殊的import，让很多新手很费解，下面我们来一一讲解一下到底是怎么一回事

1. 点操作

   我们有时候会看到如下的方式导入包

   ```
    import(
        . "fmt"
    )
   ```

   这个点操作的含义就是这个包导入之后在你调用这个包的函数时，你可以省略前缀的包名，也就是前面你调用的fmt.Println("hello world")可以省略的写成Println("hello world")

2. 别名操作

   别名操作顾名思义我们可以把包命名成另一个我们用起来容易记忆的名字

   ```
    import(
        f "fmt"
    )
   ```

   别名操作的话调用包函数时前缀变成了我们的前缀，即f.Println("hello world")

3. _操作

   这个操作经常是让很多人费解的一个操作符，请看下面这个import

   ```Go
    import (
        "database/sql"
        _ "github.com/ziutek/mymysql/godrv"
    )
   ```

   _操作其实是引入该包，而不直接使用包里面的函数，而是调用了该包里面的init函数。

   ## struct

   ```Go
   type person struct {
       name string
       age int
   }
   
   var P person  // P现在就是person类型的变量了
   
   P.name = "Astaxie"  // 赋值"Astaxie"给P的name属性.
   P.age = 25  // 赋值"25"给变量P的age属性
   fmt.Printf("The person's name is %s", P.name)  // 访问P的name属性.
   ```

   除了上面这种P的声明使用之外，还有另外几种声明使用方式：

   - 1.按照顺序提供初始化值

     P := person{"Tom", 25}

   - 2.通过`field:value`的方式初始化，这样可以任意顺序

     P := person{age:24, name:"Tom"}

   - 3.当然也可以通过`new`函数分配一个指针，此处P的类型为*person

     P := new(person)

