# go中的map不支持并发读写

常见的解决方案是加锁+分片化

Go1.9 起支持 `sync.Map`

# nil亦可调用函数

```go
type T struct{}

func (t *T) Hello() string {
	if t == nil {
		fmt.Println("脑子进煎鱼了")
		return ""
	}

	return "煎鱼进脑子了"
}

func main() {
	var t *T
	t.Hello()
```

并不会导致panic，而是输出"脑子进煎鱼了"

调用函数的指向**不是由该表达式的特定运行时值来决定**

实际上，在 Go 中，表达式 `Expression.Name` 的语法，所调用的函数完全由 `Expression` 的类型决定。

具体如下：

```golang
func (p *Sometype) Somemethod (firstArg int) {}
```

本质上是：

```golang
func SometypeSomemethod(p *Sometype, firstArg int) {}
```

上述入参 `p *Sometype` 是有具体上下文类型的，自然而然也就能调用到相应的方法。如果是没有任何上下文类型的，例如：`nil.Somemethod` 方法来调用，那肯定就是无法运行的