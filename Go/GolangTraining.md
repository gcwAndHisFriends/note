# format

用"%c"将int以ascll码的字符输出

go中,使用"%v"默认打印，使用%T打印目标类型

# 打印空值,自动初始化

```
int:打印0
string 打印空
float64 打印0
bool 打印false
```

# iota

iota只能被用在常量的赋值中，在每一个**const关键字**出现时，被重置为0，然后每出现一个常量，iota所代表的数值会自动增加1，iota可以理解成常量组中常量的计数器，不论该常量的值是什么，只要有一个常量，那么iota 就加1。

```go
const (
	a = iota // 0
	b = iota // 1
	c = iota // 2
)
-------------
const (
	a = iota // 0
	b        // 1
	c        // 2
)
--------------
const (
	a = iota // 0
	b        // 1
	c        // 2
)

const (
	d = iota // 0
	e        // 1
	f        // 2
)
---------------
const (
	_ = iota      // 0
	b = iota * 10 // 1 * 10
	c = iota * 10 // 2 * 10
)
----------------
const (
	_  = iota             // 0
	KB = 1 << (iota * 10) // 1 << (1 * 10)
	MB = 1 << (iota * 10) // 1 << (2 * 10)
	GB = 1 << (iota * 10) // 1 << (3 * 10)
	TB = 1 << (iota * 10) // 1 << (4 * 10)
)
```

# rune

遍历一个字符串时，得到的数据类型是`rune`而不是`byte`

`rune`和`byte`两者在ASCII范围内的大小写字母的ASCLL码是相同的。

# go中的switch

Go里面switch默认相当于每个case最后带有break，匹配成功后不会自动向下执行其他case，而是跳出整个switch, 但是可以使用**fallthrough**强制执行后面**一个**的case代码。

当下一个case里没有fallthrough时，执行完就会跳出

swith可以包含多个元素

```go
switch "Jenny" {
	case "Tim", "Jenny":
		fmt.Println("Wassup Tim, or, err, Jenny")
	case "Marcus", "Medhi":
		fmt.Println("Both of your names start with M")
	case "Julian", "Sushant":
		fmt.Println("Wassup Julian / Sushant")
	}
```

# 函数缺省参数

```go
package main

import "fmt"

func main() {
	n := average(43, 56, 87, 12, 45, 57)
    //
    data := []float64{43, 56, 87, 12, 45, 57}
	n := average(data...)
    
	fmt.Println(n)
}

func average(sf ...float64) float64 {
	fmt.Println(sf)
	fmt.Printf("%T \n", sf)
	var total float64
	for _, v := range sf {
		total += v
	}
	return total / float64(len(sf))
}
```

# 函数闭包

`increment`实际上是一个闭包，它捕获了`wrapper`函数中的变量`x`。

接下来，我们连续两次调用`increment`函数，并将其返回值打印出来。由于`increment`是一个闭包，它会记住上次调用后的`x`的值，并在每次调用时对其进行自增操作。

可以把闭包看作一个对象

```go
package main

import "fmt"

func wrapper() func() int {
	var x int
	return func() int {
		x++
		return x
	}
}

func main() {
	increment := wrapper()
	fmt.Println(increment())//1
	fmt.Println(increment())//2
}

```

# make

```go
greeting := make([]string, 3, 5)
```

其中3是使用长度,5是预留长度

访问

```
greeting[3]
```

会导致panic

# map的删除操作

如果键不存在，不会报错

```go
delete(myGreeting, "two")
```

# json 

注意，如果首字母不大写，就无法转换

```go
package main

import (
	"encoding/json"
	"fmt"
)

type person struct {
	First       string
	Last        string
	Age         int
	notExported int
}

func main() {
	p1 := person{"James", "Bond", 20, 007}
	bs, _ := json.Marshal(p1)
	fmt.Println(bs)
	fmt.Printf("%T \n", bs)
	fmt.Println(string(bs))//{"First":"James","Last":"Bond","Age":20}
}
```



# struct tag

在struct中的每一个field后面添加一段额外的注释或者说明，来**引导struct的encoding到某种格式中**

```go
type Person struct {
    FirstName  string `json:"first_name"`
    LastName   string `json:"last_name"`
    MiddleName string `json:"middle_name,omitempty"`
}
```

指定encoding到json后,各个field在json中的key名称

```go
p = Person{FirstName: "chen", LastName:"he"}
解析后
{
  "first_name":"chen",
  "last_name":"he"
}
```

如果给结构体字段添加了`json:"-"`标签之后，它将被视为一个非导出字段，因此在序列化过程中将被忽略掉。

# sync.Waitgroup

等待一组 goroutine 的结束

```go
var wg sync.WaitGroup
func main() {
	
	wg.Add(2)
	go foo()
	go bar()
	wg.Wait()
}

func foo() {
	for i := 0; i < 45; i++ {
		fmt.Println("Foo:", i)
	}
	wg.Done()
}

func bar() {
	for i := 0; i < 45; i++ {
		fmt.Println("Bar:", i)
	}
	wg.Done()
}
```

# atomic库

## add

```go
import (
   "sync/atomic"
)
atomic.AddInt64(&counter, 1)
```

有

```go
func AddInt32(addr *int32, delta int32) (new int32)
func AddInt64(addr *int64, delta int64) (new int64)
func AddUint32(addr *uint32, delta uint32) (new uint32)
func AddUint64(addr *uint64, delta uint64) (new uint64)
func AddUintptr(addr *uintptr, delta uintptr) (new uintptr)
```

## cas

```go
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
func CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)
func CompareAndSwapPointer(addr *unsafe.Pointer, old, new unsafe.Pointer) (swapped bool)
func CompareAndSwapUint32(addr *uint32, old, new uint32) (swapped bool)
func CompareAndSwapUint64(addr *uint64, old, new uint64) (swapped bool)
func CompareAndSwapUintptr(addr *uintptr, old, new uintptr) (swapped bool)
```

比较当前 addr 地址对应值是不是等于 old，等于的话就更新成 new，并返回 true，不等于的话返回 false。

**CAS 本身并未实现失败的后的处理机制，只不过我们最常用的处理方式是重试而已**。

## swap

Swap 会返回旧值

```go
func SwapInt32(addr *int32, new int32) (old int32)
func SwapInt64(addr *int64, new int64) (old int64)
func SwapPointer(addr *unsafe.Pointer, new unsafe.Pointer) (old unsafe.Pointer)
func SwapUint32(addr *uint32, new uint32) (old uint32)
func SwapUint64(addr *uint64, new uint64) (old uint64)
func SwapUintptr(addr *uintptr, new uintptr) (old uintptr)
```

## Load

Load 方法会取出 addr 地址中的值

```
func LoadInt32(addr *int32) (val int32)
func LoadInt64(addr *int64) (val int64)
func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)
func LoadUint32(addr *uint32) (val uint32)
func LoadUint64(addr *uint64) (val uint64)
func LoadUintptr(addr *uintptr) (val uintptr)
```

## Store 

将一个值存到指定的 addr 地址中去

```go
func StoreInt32(addr *int32, val int32)
func StoreInt64(addr *int64, val int64)
func StorePointer(addr *unsafe.Pointer, val unsafe.Pointer)
func StoreUint32(addr *uint32, val uint32)
func StoreUint64(addr *uint64, val uint64)
func StoreUintptr(addr *uintptr, val uintptr)
```

上面的几种方法只支持基本的几种类型，因此，atomic 还提供了一个 Value 类型，它可以实现对任意类型（结构体）原子的存取操作。

```go
type Value struct {
    v interface{}
}

func (v *Value) Load() (x interface{})
func (v *Value) Store(x interface{})
```



# 锁定核数

```go
runtime.GOMAXPROCS(1)
```

