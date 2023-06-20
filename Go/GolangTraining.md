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

