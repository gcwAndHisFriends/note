# 第 2 章语言可用性的强化

## 2.1 常量

nullptr 出现的目的是为了替代 NULL。在某种意义上来说，传统 C++ 会把 NULL、0 视为同一种东 西，这取决于编译器如何定义 NULL，有些编译器会将 NULL 定义为 ((void*)0)，有些则会直接将其定义 为 0。

C++ 不允许直接将 void * 隐式转换到其他类型。但如果编译器尝试把 NULL 定义为 ((void*)0)， 那么在下面这句代码中：

char *ch = NULL;

没有了 void * 隐式转换的 C++ 只好将 NULL 定义为 0。而这依然会产生新的问题，将 NULL 定义 成 0 将导致 C++ 中重载特性发生混乱。考虑下面这两个 foo 函数：

 void foo(char*); 

void foo(int); 

那么 foo(NULL); 这个语句将会去调用 foo(int)，从而导致代码违反直觉。

为了解决这个问题，C++11 引入了 nullptr 关键字，专门用来区分空指针、0。

nullptr 的类型 为 nullptr_t，能够隐式的转换为任何指针或成员指针的类型，也能和他们进行相等或者不等的比较。

## constexpr

C++ 本身已经具备了常量表达式的概念，比如 1+2, 3*4 这种表达式总是会产生相同的结果并且没 有任何副作用。如果编译器能够在编译时就把这些表达式直接优化并植入到程序运行时，将能增加程序 的性能。一个非常明显的例子就是在数组的定义阶段：

```c++
#include <iostream>
#define LEN 10
int len_foo() {
    int i = 2;
    return i;
}
constexpr int len_foo_constexpr() {
	return 5;
}
constexpr int fibonacci(const int n) {
	return n == 1 || n == 2 ? 1 : fibonacci(n-1)+fibonacci(n-2);
}
int main() {
    char arr_1[10]; // 合法
    char arr_2[LEN]; // 合法
    int len = 10;
    // char arr_3[len]; // 非法
    const int len_2 = len + 1;
    constexpr int len_2_constexpr = 1 + 2 + 3;
    // char arr_4[len_2]; // 非法
    char arr_4[len_2_constexpr]; // 合法
    // char arr_5[len_foo()+5]; // 非法
    char arr_6[len_foo_constexpr() + 1]; // 合法
    std::cout << fibonacci(10) << std::endl;
    // 1, 1, 2, 3, 5, 8, 13, 21, 34, 55
    std::cout << fibonacci(10) << std::endl;
    return 0;
}

```

上面的例子中，char arr_4[len_2] 可能比较令人困惑，因为 len_2 已经被定义为了常量。为什么 char arr_4[len_2] 仍然是非法的呢？这是因为 C++ 标准中数组的长度必须是一个**常量表达式**，而对 于 len_2 而言，这是一个 **const 常数**，而不是一个**常量表达式**，因此（即便这种行为在大部分编译器中 都支持，但是）它是一个非法的行为

注意，现在大部分编译器其实都带有自身编译优化，很多非法行为在编译器优化的加持下会 变得合法，若需重现编译报错的现象需要使用老版本的编译器。

C++11 提供了 constexpr 让用户显式的声明函数或对象构造函数在编译期会成为常量表达式，这 个关键字明确的告诉编译器应该去验证 len_foo 在编译期就应该是一个常量表达式。

此外，constexpr 修饰的函数可以使用递归：

```c++
constexpr int fibonacci(const int n) {
	return n == 1 || n == 2 ? 1 : fibonacci(n-1)+fibonacci(n-2);
}
```

从 C++14 开始，constexpr 函数可以在内部使用**局部变量、循环和分支**等简单语句，例如下面的 代码在 C++11 的标准下是不能够通过编译的：

```c++
constexpr int fibonacci(const int n) {
    if(n == 1) return 1;
    if(n == 2) return 1;
    return fibonacci(n-1) + fibonacci(n-2);
}
```

为此，我们可以写出下面这类简化的版本来使得函数从 C++11 开始即可用：

```c++
constexpr int fibonacci(const int n) {
	return n == 1 || n == 2 ? 1 : fibonacci(n-1) + fibonacci(n-2);
}
```

## if/switch 变量声明强化

在传统 C++ 中，变量的声明虽然能够位于任何位置，甚至于 for 语句内能够声明一个临时变量 int，但始终没有办法在 if 和 switch 语句中声明一个临时的变量。

```c++
#include <iostream>
#include <vector>
#include <algorithm>
int main() {
    std::vector<int> vec = {1, 2, 3, 4};
    // 在 c++17 之前
    const std::vector<int>::iterator itr = std::find(vec.begin(), vec.end(), 2);
    if (itr != vec.end()) {
    	*itr = 3;
    }
    // 需要重新定义一个新的变量
    const std::vector<int>::iterator itr2 = std::find(vec.begin(), vec.end(), 3);
    if (itr2 != vec.end()) {
    	*itr2 = 4;
    }
    // 将输出 1, 4, 3, 4
    for (std::vector<int>::iterator element = vec.begin(); element != vec.end(); ++element)
    std::cout << *element << std::endl;
}
```

在上面的代码中，我们可以看到 itr 这一变量是定义在整个 main() 的作用域内的，这导致当我们 需要再次遍历整个 std::vectors 时，需要重新命名另一个变量。C++17 消除了这一限制，使得我们可 以在 if（或 switch）中完成这一操作：

```c++
if (const std::vector<int>::iterator itr = std::find(vec.begin(), vec.end(), 3);itr != vec.end()) {
	*itr = 4;
}

```

## 初始化列表

初始化是一个非常重要的语言特性，最常见的就是在对象进行初始化时进行使用。在传统 C++ 中， 不同的对象有着不同的初始化方法，例如普通数组、POD （Plain Old Data，即没有构造、析构和虚函 数的类或结构体）类型都可以使用 {} 进行初始化，也就是我们所说的初始化列表。而对于类对象的初始 化，要么需要通过拷贝构造、要么就需要使用 () 进行。这些不同方法都针对各自对象，不能通用。

为了解决这个问题，C++11 首先把初始化列表的概念绑定到了类型上，并将其称之为 std::initializer_list， 允许构造函数或其他函数像参数一样使用初始化列表，这就为类对象的初始化与普通数组和 POD 的初 始化方法提供了统一的桥梁，例如：

```c++
#include <initializer_list>
#include <vector>
class MagicFoo {
public:
    std::vector<int> vec;
    MagicFoo(std::initializer_list<int> list) {
    for (std::initializer_list<int>::iterator it = list.begin();
        it != list.end(); ++it)
        vec.push_back(*it);
    }
};
int main() {
    // after C++11
    MagicFoo magicFoo = {1, 2, 3, 4, 5};
    std::cout << "magicFoo: ";
    for (std::vector<int>::iterator it = magicFoo.vec.begin(); it != magicFoo.vec.end(); ++it) std::cout << *it << std::endl;
}
```

这种构造函数被叫做初始化列表构造函数，具有这种构造函数的类型将在初始化时被特殊关照。 初始化列表除了用在对象构造上，还能将其作为普通函数的形参，例如：

```c++
public:
void foo(std::initializer_list<int> list) {
    for (std::initializer_list<int>::iterator it = list.begin(); it != list.end(); ++it) vec.push_back(*it);
}
magicFoo.foo({6,7,8,9});
```

其次，C++11 还提供了统一的语法来初始化任意的对象，例如：

```c++
Foo foo2 {3, 4};
```

## 结构化绑定

结构化绑定提供了类似其他语言中提供的多返回值的功能。在容器一章中，我们会学到 C++11 新 增了 std::tuple 容器用于构造一个元组，进而囊括多个返回值。但缺陷是，C++11/14 并没有提供一种 简单的方法直接从元组中拿到并定义元组中的元素，尽管我们可以使用 std::tie 对元组进行拆包，但 我们依然必须非常清楚这个元组包含多少个对象，各个对象是什么类型，非常麻烦。

C++17 完善了这一设定，给出的结构化绑定可以让我们写出这样的代码：

```c++
#include <iostream>
#include <tuple>
std::tuple<int, double, std::string> f() {
	return std::make_tuple(1, 2.3, "456");
}
int main() {
    auto [x, y, z] = f();
    std::cout << x << ", " << y << ", " << z << std::endl;
    return 0;
}

```

## auto

auto 不能用于函数传参，因此下面的做法是无法通过编译的 因此下面的做法是无法通过编译的（考虑重载的问题，我们 应该使用模板）：

```
int add(auto x, auto y);
```

此外，auto 还不能用于推导数组类型：

```c++
auto auto_arr2[10] = {arr}; 
```

## decltype

decltype 关键字是为了解决 auto 关键字只能对变量进行类型推导的缺陷而出现的。它的用法和 typeof 很相似：

```
decltype(表达式)
```

```c++
auto x = 1;
auto y = 2;
decltype(x+y) z;
```

你已经在前面的例子中看到 decltype 用于推断类型的用法，下面这个例子就是判断上面的变量 x, y, z 是否是同一类型：

```c++
if (std::is_same<decltype(x), int>::value)
	std::cout << "type x == int" << std::endl;
if (std::is_same<decltype(x), float>::value)
	std::cout << "type x == float" << std::endl;
if (std::is_same<decltype(x), decltype(z)>::value)
	std::cout << "type z == type x" << std::endl;
```

其中，std::is_same 用于判断 T 和 U 这两个类型是否相等。输出结果为：

type x == int 

type z == type x

## 尾返回类型推导

你可能会思考，在介绍 auto 时，我们已经提过 auto 不能用于函数形参进行类型推导，那么 auto 能不能用于推导函数的返回类型呢？还是考虑一个加法函数的例子，在传统 C++ 中我们必须这么写：

```c++
template<typename R, typename T, typename U>
R add(T x, U y) {
	return x+y
}
```

这样的代码其实变得很丑陋，因为程序员在使用这个模板函数的时候，必须明确指出返回类型。但 事实上我们并不知道 add() 这个函数会做什么样的操作，获得一个什么样的返回类型。

在 C++11 中这个问题得到解决。虽然你可能马上会反应出来使用 decltype 推导 x+y 的类型，写 出这样的代码： 

```c++
decltype(x+y) add(T x, U y)
```

但事实上这样的写法并不能通过编译。这是因为在编译器读到 decltype(x+y) 时，x 和 y 尚未被定 义。为了解决这个问题，C++11 还引入了一个叫做尾返回类型（trailing return type），利用 auto 关键 字将返回类型后置：

```c++
template<typename T, typename U>
auto add2(T x, U y) -> decltype(x+y){
    return x + y;
}
```

令人欣慰的是从 C++14 开始是可以直接让普通函数具备返回值推导，因此下面的写法变得合法：

```c++
template<typename T, typename U>
auto add3(T x, U y){
	return x + y;
}
```

## if constexp

如果我们把这一特性引入到条件判断中去，让代码在编译时就完成分支判断， 岂不是能让程序效率更高？C++17 将 constexpr 这个关键字引入到 if 语句中，允许在代码中声明常量 表达式的判断条件，考虑下面的代码：

```c++
#include <iostream>
template<typename T>
auto print_type_info(const T& t) {
    if constexpr (std::is_integral<T>::value) {
        return t + 1;
    } else {
        return t + 0.001;
    }
}
int main() {
    std::cout << print_type_info(5) << std::endl;
    std::cout << print_type_info(3.14) << std::endl;
}
```

在编译时，实际代码就会表现为如下：

```c++
int print_type_info(const int& t) {
	return t + 1;
}
double print_type_info(const double& t) {
	return t + 0.001;
}
int main() {
    std::cout << print_type_info(5) << std::endl;
    std::cout << print_type_info(3.14) << std::endl;
}
```

## 区间 for 迭代

```c++
int main() {
    std::vector<int> vec = {1, 2, 3, 4};
    if (auto itr = std::find(vec.begin(), vec.end(), 3); itr != vec.end()) *itr = 4;
    
    for (auto element : vec)
   		std::cout << element << std::endl; // read only
    for (auto &element : vec) {
    	element += 1; // writeable
    }
    for (auto element : vec)
    	std::cout << element << std::endl; // read only
}

```

## 2.5 模板

## 外部模板

传统 C++ 中，模板只有在使用时才会被编译器实例化。换句话说，只要在每个编译单元（文件）中 编译的代码中遇到了被完整定义的模板，都会实例化。这就产生了重复实例化而导致的编译时间的增加。 并且，我们没有办法通知编译器不要触发模板的实例化。

为此，C++11 引入了外部模板，扩充了原来的强制编译器在特定位置实例化模板的语法，使我们能 够显式的通知编译器何时进行模板的实例化：

```c++
template class std::vector<bool>; // 强行实例化
extern template class std::vector<double>; // 不在该当前编译文件中实例化模板
```

## 类型别名模板

在了解类型别名模板之前，需要理解『模板』和『类型』之间的不同。仔细体会这句话：模板是用来 产生类型的。在传统 C++ 中，typedef 可以为类型定义一个新的名称，但是却没有办法为模板定义一 个新的名称。因为，模板不是类型。例如：

```c++
template<typename T, typename U>
class MagicType {
    public:
    T dark;
    U magic;
};
// 不合法
template<typename T>
typedef MagicType<std::vector<T>, std::string> FakeDarkMagic;
```

```c++
int add(A a,B b) {
	return a+b;
}
int main() {
	std::cout<<add<int,int>(1,2);
}
```

C++11 使用 using 引入了下面这种形式的写法，并且同时支持对传统 typedef 相同的功效：

通常我们使用 typedef 定义别名的语法是：typedef 原名称 新名称;，但是对函数指针等别 名的定义语法却不相同，这通常给直接阅读造成了一定程度的困难。

```c++
typedef int (*process)(void *);
using NewProcess = int(*)(void *);
template<typename T>
using TrueDarkMagic = MagicType<std::vector<T>, std::string>;
int main() {
    TrueDarkMagic<bool> you;
}
```

## 默认模板参数

我们可能定义了一个加法函数：

```c++
// c++11 version
template<typename T, typename U>
auto add(T x, U y) -> decltype(x+y) {
	return x+y;
}
// Call add function
auto ret = add<int, int>(1,3);
```

```c++
template <typename A,typename B>
auto add(A a,B b)->decltype(a+b){
	return a+b;
}
int main() {
	std::cout<<add<int,int>(1,2);
}
```

但在使用时发现，要使用 add，就必须每次都指定其模板参数的类型。 在 C++11 中提供了一种便利，可以指定模板的默认参数：

```c++
template<typename T = int, typename U = int>
auto add(T x, U y) -> decltype(x+y) {
	return x+y;
}
// Call add function
auto ret = add(1,3);
```

```c++
template <typename A=int,typename B=int>
auto add(A a,B b)->decltype(a+b){
	return a+b;
}
int main() {
	std::cout<<add(1,2);
}
```



## 变长参数模板

 C++11 加入了新的表示方 法，允许任意个数、任意类别的模板参数，同时也不需要在定义时将参数的个数固定。

```c++
template<typename... Ts> class Magic;
```

模板类 Magic 的对象，能够接受不受限制个数的 typename 作为模板的形式参数，例如下面的定义：

```c++
class Magic<int,
std::vector<int>,
std::map<std::string,
std::vector<int>>> darkMagic;
```

既然是任意形式，所以个数为 0 的模板参数也是可以的

如果不希望产生的模板参数个数为 0，可以手动的定义至少一个模板参数：

```c++
template<typename Require, typename... Args> class Magic;
```

如何对参数进行解包

使用 sizeof... 来计算参数的个数，：

```c++
template<typename... Ts>
void magic(Ts... args) {
	std::cout << sizeof...(args) << std::endl;
}
```

其次，对参数进行解包，到目前为止还没有一种简单的方法能够处理参数包，但有两种经典的处理 手法：

1:递归模板函数

```c++
#include <iostream>
template<typename T0>
void printf1(T0 value) {
	std::cout << value << std::endl;
}
template<typename T, typename... Ts>
void printf1(T value, Ts... args) {
    std::cout << value << std::endl;
    printf1(args...);
}
int main() {
    printf1(1, 2, "123", 1.1);
    return 0;
}
```

```c++
template <typename T0>
int add(T0 value){
	return value;
}
template <typename T, typename... Ts>
int add(T value,Ts ... args){
	return value+add(args...);
}
int main() {
	std::cout<<add(1,2,3,4,5,6);
}

```

变参模板展开

你应该感受到了这很繁琐，在 C++17 中增加了变参模板展开的支持，于是你可以在一个函数中完 成 printf 的编写：

```c++
template<typename T0, typename... T>
void printf2(T0 t0, T... t) {
    std::cout << t0 << std::endl;
    if constexpr (sizeof...(t) > 0) printf2(t...);
}
```

```c++
template <typename T0, typename... T>
int add(T0 t0,T... t){
	if constexpr (sizeof...(t) > 0) return t0+add(t...);
	return t0;
}
int main() {
	std::cout<<add(1,2,3,4,5,6);
}
```

事实上，有时候我们虽然使用了变参模板，却不一定需要对参数做逐个遍历，我们可以利用 std::bind 及完美转发等特性实现对函数和参数的绑定，从而达到成功调用的目的。

3. 初始化列表展开

递归模板函数是一种标准的做法，但缺点显而易见的在于必须定义一个终止递归的函数。 这里介绍一种使用初始化列表展开的黑魔法：

```c++
template<typename T, typename... Ts>
auto printf3(T value, Ts... args) {
    std::cout << value << std::endl;
    (void) std::initializer_list<T>{([&args] {
    std::cout << args << std::endl;
    }(), value)...};
}

```

```c++
auto add(T value,Ts... args){
	(void)std::initializer_list<T>{([&args,&value] {
		value+=args;
    }(), value)...};
    return value;
}
int main() {
	std::cout<<add(1,2,3,4,5,6);
}
```

折叠表达式

C++ 17 中将变长参数这种特性进一步带给了表达式，考虑下面这个例子：

```c++
#include <iostream>
template<typename ... T>
auto sum(T ... t) {
    return (t + ...);
}
int main() {
	std::cout << sum(1, 2, 3, 4, 5, 6, 7, 8, 9, 10) << std::endl;
}
```

非类型模板参数推导

前面我们主要提及的是模板参数的一种形式：类型模板参数。

```c++
template <typename T, typename U>
auto add(T t, U u) {
    return t+u;
}
```

其中模板的参数 T 和 U 为具体的类型。但还有一种常见模板参数形式可以让不同字面量成为模板参 数，即非类型模板参数：

```c++
template <typename T, int BufSize>
class buffer_t {
public:
    T& alloc();
    void free(T& item);
private:
    T data[BufSize];
}
buffer_t<int, 100> buf; // 100 作为模板参数
```

```c++
template<int A, int B>
class ADD{
public:
	auto add(){
		return A+B;
	}
};
int main() {
	ADD<2,2>a;
	std::cout<<a.add();
}
```

在这种模板参数形式下，我们可以将 100 作为模板的参数进行传递。在 C++11 引入了类型推导这 一特性后，我们会很自然的问，既然此处的模板参数以具体的字面量进行传递，能否让编译器辅助我们 进行类型推导，通过使用占位符 auto 从而不再需要明确指明类型？幸运的是，C++17 引入了这一特性， 我们的确可以 auto 关键字，让编译器辅助完成具体类型的推导，例如：

```c++
template <auto value> 
void foo() {
    std::cout << value << std::endl;
	return;
}
int main() {
	foo<10>(); // value 被推导为 int 类型
}
```

```c++
template<auto A, auto B>
auto add(){
	return A+B; 
}
int main() {
	std::cout<<add<2,2>();
}
```

## 2.6 面向对象

## 委托构造

C++11 引入了委托构造的概念，这使得构造函数可以在同一个类中一个构造函数调用另一个构造函 数，从而达到简化代码的目的：

```c++
#include <iostream>
class Base {
public:
    int value1;
    int value2;
    Base(int v) {
    	value1 = v;
    }
    Base(int value,int v) : Base(v) { // 委托 Base() 构造函数
    	value2 = value;
    }
};
int main() {
    Base b(2,9);
    std::cout << b.value1 << std::endl;
    std::cout << b.value2 << std::endl;
}
```

## 继承构造

在传统 C++ 中，构造函数如果需要继承是需要将参数一一传递的，这将导致效率低下。C++11 利 用关键字 using 引入了继承构造函数的概念：

```c++
#include <iostream>
class Base {
public:
    int value1;
    int value2;
    Base() {
    	value1 = 1;
    }
    Base(int value) : Base() { // 委托 Base() 构造函数
    	value2 = value;
    }
};
class Subclass : public Base {
public:
	using Base::Base; // 继承构造
};
int main() {
    Subclass s(3);
    std::cout << s.value1 << std::endl;
    std::cout << s.value2 << std::endl;
}
```

## 显式虚函数重载

在传统 C++ 中，经常容易发生意外重载虚函数的事情。例如：

```c++
struct Base {
	virtual void foo();
};
struct SubClass: Base {
	void foo();
};
```

SubClass::foo 可能并不是程序员尝试重载虚函数，只是恰好加入了一个具有相同名字的函数。另 一个可能的情形是，当基类的虚函数被删除后，子类拥有旧的函数就不再重载该虚拟函数并摇身一变成 为了一个普通的类方法，这将造成灾难性的后果。

C++11 引入了 override 和 final 这两个关键字来防止上述情形的发生。

## override 

当重载虚函数时，引入 override 关键字将显式的告知编译器进行重载，编译器将检查基函 数是否存在这样的虚函数，否则将无法通过编译：

```c++
struct Base {
	virtual void foo(int);
};
struct SubClass: Base {
    virtual void foo(int) override; // 合法
    virtual void foo(float) override; // 非法, 父类没有此虚函数
};
```

## final

则是为了防止类被继续继承以及终止虚函数继续重载引入的。

```c++
struct Base {
	virtual void foo() final;
};
struct SubClass1 final: Base {
}; // 合法
struct SubClass2 : SubClass1 {
}; // 非法, SubClass1 已 final
struct SubClass3: Base {
	void foo(); // 非法, foo 已 final
};
```

## 显式禁用默认函数

在传统 C++ 中，如果程序员没有提供，编译器会默认为对象生成默认构造函数、复制构造、赋值 算符以及析构函数。

另外，C++ 也为所有类定义了诸如 new delete 这样的运算符。当程序员有需要时， 可以重载这部分函数。

这就引发了一些需求：无法精确控制默认函数的生成行为。例如禁止类的拷贝时，必须将复制构造 函数与赋值算符声明为 private。

C++11 提供了上述需求的解决方案，允许显式的声明采用或拒绝编译器自带的函数。例如：

```c++
class Magic {
public:
    Magic() = default; // 显式声明使用编译器生成的构造
    Magic& operator=(const Magic&) = delete; // 显式声明拒绝编译器生成构造
    Magic(int magic_number);
}
```

## 强类型枚举

在传统 C++ 中，枚举类型并非类型安全，枚举类型会被视作整数，则会让两种完全不同的枚举类 型可以进行直接的比较（虽然编译器给出了检查，但并非所有），甚至同一个命名空间中的不同枚举类型 的枚举值名字不能相同，这通常不是我们希望看到的结果。

C++11 引入了枚举类（enumeration class），并使用 enum class 的语法进行声明：

```c++
enum class new_enum : unsigned int {
    value1,
    value2,
    value3 = 100,
    value4 = 100
};
```

这样定义的枚举实现了类型安全，首先他不能够被隐式的转换为整数，同时也不能够将其与整数数 字进行比较，更不可能对不同的枚举类型的枚举值进行比较。但相同枚举值之间如果指定的值相同，那 么可以进行比较：

```c++
if (new_enum::value3 == new_enum::value4) {
    // 会输出
    std::cout << "new_enum::value3 == new_enum::value4" << std::endl;
}
```

我们希望获得枚举值的值时，将必须显式的进行类型转换，不过我们可以通过重载 << 这个算符来 进行输出，可以收藏下面这个代码段：

```c++
#include <iostream>
template<typename T>
std::ostream& operator<<(typename std::enable_if<std::is_enum<T>::value, std::ostream>::type& stream, const T& e)
{
	return stream << static_cast<typename std::underlying_type<T>::type>(e);
}
```

这时，下面的代码将能够被编译：

```c++
std::cout << new_enum::value3 << std::endl
```

# 第 3 章语言运行期的强化

## Lambda 表达式

Lambda 表达式，实际上就是提供了一个类 似匿名函数的特性，而匿名函数则是在需要一个函数，但是又不想费力去命名一个函数的情况下去使用 的。

Lambda 表达式的基本语法如下：

```c++
[捕获列表](参数列表) mutable(可选) 异常属性 -> 返回类型 {
// 函数体
}
```

值捕获 与参数传值类似，值捕获的前提是变量可以拷贝，不同之处则在于，被捕获的变量在 Lambda 表达式被创建时拷贝，而非调用时才拷贝：

```c++
void lambda_value_capture() {
    int value = 1;
    auto copy_value = [value] {
    	return value;
    };
    value = 100;
    auto stored_value = copy_value();
    std::cout << "stored_value = " << stored_value << std::endl;
    // 这时, stored_value == 1, 而 value == 100.
    // 因为 copy_value 在创建时就保存了一份 value 的拷贝
}
```

引用捕获 与引用传参类似，引用捕获保存的是引用，值会发生变化。

```c++
void lambda_reference_capture() {
    int value = 1;
    auto copy_value = [&value] {
         return value;
    };
    value = 100;
    auto stored_value = copy_value();
    std::cout << "stored_value = " << stored_value << std::endl;
    // 这时, stored_value == 100, value == 100.
    // 因为 copy_value 保存的是引用
}
```

隐式捕获

手动书写捕获列表有时候是非常复杂的，这种机械性的工作可以交给编译器来处理，这时候可以在 捕获列表中写一个 & 或 = 向编译器声明采用引用捕获或者值捕获.

总结一下，捕获提供了 Lambda 表达式对外部值进行使用的功能，捕获列表的最常用的四种形式可 以是： 

• [] 空捕获列表 • [name1, name2, . . . ] 捕获一系列变量 

• [&] 引用捕获, 让编译器自行推导引用列表

 • [=] 值捕获, 让编译器自行推导值捕获列表

表达式捕获

上面提到的值捕获、引用捕获都是已经在外层作用域声明的变量，因此这些捕获方式捕获的均为左值，而不能捕获右值。

C++14 给与了我们方便，允许捕获的成员用任意的表达式进行初始化，这就允许了右值的捕获，被声明的捕获变量类型会根据表达式进行判断，判断方式与使用 auto 本质上是相同的：

```c++
#include <iostream>
#include <utility>
int main() {
    auto important = std::make_unique<int>(1);
    auto add = [v1 = 1, v2 = std::move(important)](int x, int y) -> int {
    	return x+y+v1+(*v2);
    };
    std::cout << add(3,4) << std::endl;
    return 0;
}
```

在上面的代码中，important 是一个独占指针，是不能够被 “=” 值捕获到，这时候我们可以将其转 移为右值，在表达式中初始化。

泛型 Lambda

上一节中我们提到了 auto 关键字不能够用在参数表里，这是因为这样的写法会与模板的功能产生 冲突。但是 Lambda 表达式并不是普通函数，所以 Lambda 表达式并不能够模板化。这就为我们造成了 一定程度上的麻烦：参数表不能够泛化，必须明确参数表类型。

幸运的是，这种麻烦只存在于 C++11 中，从 C++14 开始，Lambda 函数的形式参数可以使用 auto 关键字来产生意义上的泛型：

```c++
auto add = [](auto x, auto y) {
	return x+y;
};
add(1, 2);
add(1.1, 2.2);

```

## 3.2 函数对象包装器

## std::function

Lambda 表达式的本质是一个和函数对象类型相似的类类型（称为闭包类型）的对象（称为闭包对 象），当 Lambda 表达式的捕获列表为空时，闭包对象还能够转换为函数指针值进行传递，例如：

```c++
#include <iostream>
using foo = void(int); // 定义函数类型, using 的使用见上一节中的别名语法
void functional(foo f) { // 定义在参数列表中的函数类型 foo 被视为退化后的函数指针类型 foo*
	f(1); // 通过函数指针调用函数
}
int main() {
    auto f = [](int value) {
    	std::cout << value << std::endl;
    };
    functional(f); // 传递闭包对象，隐式转换为 foo* 类型的函数指针值
    f(1); // lambda 表达式调用
    return 0;
}
```

上面的代码给出了两种不同的调用形式，一种是将 Lambda 作为函数类型传递进行调用，而另一种 则是直接调用 Lambda 表达式，在 C++11 中，统一了这些概念，将能够被调用的对象的类型，统一称 之为可调用类型。而这种类型，便是通过 std::function 引入的。

C++11 std::function 是一种通用、多态的函数封装，它的实例可以对任何可以调用的目标实体进 行存储、复制和调用操作，它也是对 C++ 中现有的可调用实体的一种类型安全的包裹

（相对来说，函数 指针的调用不是类型安全的），

换句话说，就是函数的容器。当我们有了函数的容器之后便能够更加方便 的将函数、函数指针作为对象进行处理。例如：

```c++
#include <functional>
#include <iostream>
int foo(int para) {
	return para;
}
int main() {
    // std::function 包装了一个返回值为 int, 参数为 int 的函数
    std::function<int(int)> func = foo;
    int important = 10;
    std::function<int(int)> func2 = [&](int value) -> int {
    	return 1+value+important;
    };
    std::cout << func(10) << std::endl;
    std::cout << func2(10) << std::endl;
}

```

## std::bind 和 std::placeholder

std::bind 则是用来绑定函数调用的参数的，它解决的需求是我们有时候可能并不一定能够一次 性获得调用某个函数的全部参数，通过这个函数，我们可以将部分调用参数提前绑定到函数身上成为一 个新的对象，然后在参数齐全后，完成调用。例如：

```c++
int foo(int a, int b, int c) {
;
}
int main() {
	// 将参数 1,2 绑定到函数 foo 上，但是使用 std::placeholders::_1 来对第一个参数进行占位
	auto bindFoo = std::bind(foo, std::placeholders::_1, 1,2);
	// 这时调用 bindFoo 时，只需要提供第一个参数即可
	bindFoo(1);
}
```

## 3.3 右值引用

右值引用是 C++11 引入的与 Lambda 表达式齐名的重要特性之一。它的引入解决了 C++ 中大 量的历史遗留问题，消除了诸如 std::vector、std::string 之类的额外开销，也才使得函数对象容器 std::function 成为了可能。

## 左值、右值的纯右值、将亡值、右值

左值 (lvalue, left value)，顾名思义就是赋值符号左边的值。准确来说，左值是表达式（不一定是 赋值表达式）后依然存在的持久对象。

右值 (rvalue, right value)，右边的值，是指表达式结束后就不再存在的临时对象。

而 C++11 中为了引入强大的右值引用，将右值的概念进行了进一步的划分，分为：纯右值、将亡 值。

纯右值 (prvalue, pure rvalue)，纯粹的右值，要么是纯粹的字面量，例如 10, true；要么是求值 结果相当于字面量或匿名临时对象，例如 1+2。非引用返回的临时变量、运算表达式产生的临时变量、原 始字面量、Lambda 表达式都属于纯右值。

需要注意的是，字符串字面量只有在类中才是右值，当其位于普通函数中是左值。例如：

```c++
class Foo {
	const char*&& right = "this is a rvalue"; // 此处字符串字面量为右值

public:
    void bar() {
    	right = "still rvalue"; // 此处字符串字面量为右值
    }
};
int main() {
    const char* const &left = "this is an lvalue"; // 此处字符串字面量为左值
}
```

将亡值 (xvalue, expiring value)，是 C++11 为了引入右值引用而提出的概念（因此在传统 C++ 中，纯右值和右值是同一个概念），也就是即将被销毁、却能够被移动的值。

```c++
std::vector<int> foo() {
    std::vector<int> temp = {1, 2, 3, 4};
    return temp;
}
std::vector<int> v = foo();

```

在这样的代码中，就传统的理解而言，函数 foo 的返回值 temp 在内部创建然后被赋值给 v，然而 v 获得这个对象时，会将整个 temp 拷贝一份，然后把 temp 销毁，如果这个 temp 非常大，这将造成大量 额外的开销（这也就是传统 C++ 一直被诟病的问题）。

在最后一行中，v 是左值、foo() 返回的值就是 右值（也是纯右值）。

但是，v 可以被别的变量捕获到，而 foo() 产生的那个返回值作为一个临时值，一 旦被 v 复制后，将立即被销毁，无法获取、也不能修改。而将亡值就定义了这样一种行为：临时的值能 够被识别、同时又能够被移动。

在 C++11 之后，编译器为我们做了一些工作，此处的左值 temp 会被进行此隐式右值转换，等价于 

```
static_cast<std::vector<int>&&>(temp)
```

进而此处的 v 会将 foo 局部返回的值进行移动。也就是 后面我们将会提到的移动语义。

## 右值引用和左值引用

要拿到一个将亡值，就需要用到右值引用：T &&，其中 T 是类型。右值引用的声明让这个临时值的 生命周期得以延长、只要变量还活着，那么将亡值将继续存活。

C++11 提供了 std::move 这个方法将左值参数无条件的转换为右值，有了它我们就能够方便的获 得一个右值临时对象，例如：

```c++
#include <iostream>
#include <string>
void reference(std::string& str) {
	std::cout << " 左值" << std::endl;
}
void reference(std::string&& str) {
	std::cout << " 右值" << std::endl;
}
int main()
{
    std::string lv1 = "string,"; // lv1 是一个左值
    // std::string&& r1 = lv1; // 非法, 右值引用不能引用左值
    std::string&& rv1 = std::move(lv1); // 合法, std::move 可以将左值转移为右值
    
    std::cout << rv1 << std::endl; // string,
    const std::string& lv2 = lv1 + lv1; // 合法, 常量左值引用能够延长临时变量的生命周期
    // lv2 += "Test"; // 非法, 常量引用无法被修改
    
    std::cout << lv2 << std::endl; // string,string
    std::string&& rv2 = lv1 + lv2; // 合法, 右值引用延长临时对象生命周期
    rv2 += "Test"; // 合法, 非常量引用能够修改临时变量
    std::cout << rv2 << std::endl; // string,string,string,Test
    reference(rv2); // 输出左值
    return 0;
}
```

rv2 虽然引用了一个右值，但由于它是一个引用，所以 rv2 依然是一个左值。

注意，这里有一个很有趣的历史遗留问题，我们先看下面的代码：

```c++
#include <iostream>
int main() {
    // int &a = std::move(1); // 不合法，非常量左引用无法引用右值
    const int &b = std::move(1); // 合法, 常量左引用允许引用右值
    std::cout << a << b << std::endl;
}
```

第一个问题，为什么不允许非常量引用绑定到非左值？这是因为这种做法存在逻辑错误：

```c++
void increase(int & v) {
	v++;
}
void foo() {
	double s = 1;
	increase(s);
}
```

由于 int& 不能引用 double 类型的参数，因此必须产生一个临时值来保存 s 的值，从而当 increase() 修改这个临时值时，从而调用完成后 s 本身并没有被修改。

第二个问题，为什么常量引用允许绑定到非左值？原因很简单，因为 Fortran 需要。

## 移动语义

传统 C++ 通过拷贝构造函数和赋值操作符为类对象设计了拷贝/复制的概念，但为了实现对资源的 移动操作，调用者必须使用先复制、再析构的方式，否则就需要自己实现移动对象的接口。试想，搬家的 时候是把家里的东西直接搬到新家去，而不是将所有东西复制一份（重买）再放到新家、再把原来的东 西全部扔掉（销毁），这是非常反人类的一件事情。

传统的 C++ 没有区分『移动』和『拷贝』的概念，造成了大量的数据拷贝，浪费时间和空间。右值 引用的出现恰好就解决了这两个概念的混淆问题，例如：

```c++
#include <iostream>
class A {
public:
    int *pointer;
    A():pointer(new int(1)) {
    	std::cout << " 构造" << pointer << std::endl;
    }
    A(A& a):pointer(new int(*a.pointer)) {
    	std::cout << " 拷贝" << pointer << std::endl;
    } // 无意义的对象拷贝
    A(A&& a):pointer(a.pointer) {
        a.pointer = nullptr;
        std::cout << " 移动" << pointer << std::endl;
    }
    ~A(){
        std::cout << " 析构" << pointer << std::endl;
        delete pointer;
    }
};
// 防止编译器优化
A return_rvalue(bool test) {
    A a,b;
	if(test) return a; // 等价于 static_cast<A&&>(a);
	else return b; // 等价于 static_cast<A&&>(b);
}
int main() {
    A obj = return_rvalue(false);
    std::cout << "obj:" << std::endl;
    std::cout << obj.pointer << std::endl;
    std::cout << *obj.pointer << std::endl;
    return 0;
}
```

在上面的代码中：

首先会在 return_rvalue 内部构造两个 A 对象，于是获得两个构造函数的输出； 

函数返回后，产生一个将亡值，被 A (obj的)的移动构造（A(A&&)）引用，从而延长生命周期，并将这个右 值中的指针拿到，保存到了 obj 中，而将亡值的指针被设置为 nullptr，防止了这块内存区域被销 毁。

从而避免了无意义的拷贝构造，加强了性能。再来看看涉及标准库的例子：

```c++
#include <iostream> // std::cout
#include <utility> // std::move
#include <vector> // std::vector
#include <string> // std::string
int main() {
    std::string str = "Hello world.";
    std::vector<std::string> v;
    // 将使用 push_back(const T&), 即产生拷贝行为
    v.push_back(str);
    // 将输出 "str: Hello world."
    std::cout << "str: " << str << std::endl;
    // 将使用 push_back(const T&&), 不会出现拷贝行为
    // 而整个字符串会被移动到 vector 中，所以有时候 std::move 会用来减少拷贝出现的开销
    // 这步操作后, str 中的值会变为空
    v.push_back(std::move(str));
    // 将输出 "str: "
    std::cout << "str: " << str << std::endl;
    return 0;
}
```

前面我们提到了，一个声明的右值引用其实是一个左值。这就为我们进行参数转发（传递）造成了 问题：

```c++
void reference(int& v) {
	std::cout << " 左值" << std::endl;
}
void reference(int&& v) {
	std::cout << " 右值" << std::endl;
}
template <typename T>
void pass(T&& v) {
    std::cout << " 普通传参:";
    reference(v); // 始终调用 reference(int&)
}
int main() {
    std::cout << " 传递右值:" << std::endl;
    pass(1); // 1 是右值, 但输出是左值
    std::cout << " 传递左值:" << std::endl;
    int l = 1;
    pass(l); // l 是左值, 输出左值
    return 0;
}
```

对于 pass(1) 来说，虽然传递的是右值，但由于 v 是一个引用，所以同时也是左值。

因此 reference(v) 会调用 reference(int&)，输出『左值』。

而对于 pass(l) 而言，l 是一个左值，为什么 会成功传递给 pass(T&&) 呢？

这是基于引用坍缩规则的：在传统 C++ 中，我们不能够对一个引用类型继续进行引用，但 C++ 由 于右值引用的出现而放宽了这一做法，从而产生了引用坍缩规则，允许我们对引用进行引用，既能左引 用，又能右引用。

因此，模板函数中使用 T&& 不一定能进行右值引用，**当传入左值时，此函数的引用将被推导为左值。** 更准确的讲，无论模板参数是什么类型的引用，当且仅当实参类型为右引用时，模板参数才能被推导为 右引用类型。这才使得 v 作为左值的成功传递。 完美转发就是基于上述规律产生的。

所谓完美转发，就是**为了让我们在传递参数的时候，保持原来 的参数类型**（左引用保持左引用，右引用保持右引用）。为了解决这个问题，我们应该使用 std::forward 来进行参数的转发（传递）：

```c++
#include <iostream>
#include <utility>
void reference(int& v) {
	std::cout << " 左值引用" << std::endl;
}
void reference(int&& v) {
	std::cout << " 右值引用" << std::endl;
}
template <typename T>
void pass(T&& v) {
    std::cout << " 普通传参: ";
    reference(v);
    std::cout << " std::move 传参: ";
    reference(std::move(v));
    std::cout << " std::forward 传参: ";
    reference(std::forward<T>(v));
    std::cout << "static_cast<T&&> 传参: ";
    reference(static_cast<T&&>(v));
}
int main() {
    std::cout << " 传递右值:" << std::endl;
    pass(1);
    std::cout << " 传递左值:" << std::endl;
    int v = 1;
    pass(v);
    return 0;
}

```

输出结果为：

```
传递右值:
普通传参: 左值引用
std::move 传参: 右值引用
std::forward 传参: 右值引用
static_cast<T&&> 传参: 右值引用
传递左值:
普通传参: 左值引用
std::move 传参: 右值引用
std::forward 传参: 左值引用
static_cast<T&&> 传参: 左值引用

```

无论传递参数为左值还是右值，普通传参都会将参数作为左值进行转发，所以 std::move 总会接受 到一个左值，从而转发调用了 reference(int&&) 输出右值引用。

唯独 std::forward 即没有造成任何多余的拷贝，同时完美转发 (传递) 了函数的实参给了内部调用 的其他函数。

std::forward 和 std::move 一样，没有做任何事情，std::move 单纯的将左值转化为右值， std::forward 也只是单纯的将参数做了一个类型的转换，从现象上来看，std::forward(v) 和 static_cast(v) 是完全一样的。

读者可能会好奇，为何一条语句能够针对两种类型的返回对应的值，我们再简单看一看 std::forward 的具体实现机制，std::forward 包含两个重载

```c++
template<typename _Tp>
constexpr _Tp&& forward(typename std::remove_reference<_Tp>::type& __t) noexcept
{ return static_cast<_Tp&&>(__t); }

template<typename _Tp>
constexpr _Tp&& forward(typename std::remove_reference<_Tp>::type&& __t) noexcept
{
static_assert(!std::is_lvalue_reference<_Tp>::value, "template argument"
" substituting _Tp is an lvalue reference type");
    
return static_cast<_Tp&&>(__t);
}
```

在这份实现中，std::remove_reference 的功能是消除类型中的引用，而 std::is_lvalue_reference 用于检查类型推导是否正确,在 std::forward 的第二个实现中检查了接收到的值确实是一个左值，进 而体现了坍缩规则。

当 std::forward 接受左值时，_Tp 被推导为左值，而所以返回值为左值

而当其接受右值时，_Tp 被推导为右值引用，则基于坍缩规则，返回值便成为了 && + && 的右值。

这时我们能回答这样一个问题：为什么在使用循环语句的过程中，auto&& 是最安全的方式？因为当auto 被推导为不同的左右引用时，与 && 的坍缩组合是完美转发。

# 第 4 章容器

## std::array

与 std::vector 不同,，std::array 对象的大小是固定的

另外由于 std::vector 是自动扩容的，当存入大量的 数据后，并且对容器进行了删除操作，容器并不会自动归还被删除元素相应的内存，这时候就需要手动 运行 shrink_to_fit() 释放这部分内存。

```c++
std::vector<int> v;
std::cout << "size:" << v.size() << std::endl; // 输出 0
std::cout << "capacity:" << v.capacity() << std::endl; // 输出 0
// 如下可看出 std::vector 的存储是自动管理的，按需自动扩张
// 但是如果空间不足，需要重新分配更多内存，而重分配内存通常是性能上有开销的操作
v.push_back(1);
v.push_back(2);
v.push_back(3);
std::cout << "size:" << v.size() << std::endl; // 输出 3
std::cout << "capacity:" << v.capacity() << std::endl; // 输出 4
// 这里的自动扩张逻辑与 Golang 的 slice 很像
v.push_back(4);
v.push_back(5);
std::cout << "size:" << v.size() << std::endl; // 输出 5
std::cout << "capacity:" << v.capacity() << std::endl; // 输出 8
// 如下可看出容器虽然清空了元素，但是被清空元素的内存并没有归还
v.clear();
std::cout << "size:" << v.size() << std::endl; // 输出 0
std::cout << "capacity:" << v.capacity() << std::endl; // 输出 8
// 额外内存可通过 shrink_to_fit() 调用返回给系统
v.shrink_to_fit();
std::cout << "size:" << v.size() << std::endl; // 输出 0
std::cout << "capacity:" << v.capacity() << std::endl; // 输出 0
```

而第二个问题就更加简单，使用 std::array 能够让代码变得更加 ‘‘现代化’’，而且封装了一些操作函 数，比如获取数组大小以及检查是否非空，同时还能够友好的使用标准库中的容器算法，比如 std::sort。

使用 std::array 很简单，只需指定其类型和大小即可：

```c++
std::array<int, 4> arr = {1, 2, 3, 4};
arr.empty(); // 检查容器是否为空
arr.size(); // 返回容纳的元素数
// 迭代器支持
for (auto &i : arr)
{
// ...
}
// 用 lambda 表达式排序
std::sort(arr.begin(), arr.end(), [](int a, int b) {
return b < a;
});
// 数组大小参数必须是常量表达式
constexpr int len = 4;
std::array<int, len> arr = {1, 2, 3, 4};
// 非法, 不同于 C 风格数组，std::array 不会自动退化成 T*
// int *arr_p = arr;
```

当我们开始用上了 std::array 时，难免会遇到要将其兼容 C 风格的接口，这里有三种做法：

```c++
void foo(int *p, int len) {
	return;
}
std::array<int, 4> arr = {1,2,3,4};
// C 风格接口传参
// foo(arr, arr.size()); // 非法, 无法隐式转换
foo(&arr[0], arr.size());
foo(arr.data(), arr.size());
// 使用 ‘std::sort‘
std::sort(arr.begin(), arr.end())
```

## std::forward_list

std::forward_list 是一个列表容器，使用方法和 std::list 基本类似，因此我们就不花费篇幅进 行介绍了。

 需要知道的是，和 std::list 的双向链表的实现不同，std::forward_list 使用单向链表进行实现， 提供了 O(1) 复杂度的元素插入，不支持快速随机访问（这也是链表的特点），也是标准库容器中唯一一 个不提供 size() 方法的容器。当不需要双向迭代时，具有比 std::list 更高的空间利用率。

## 4.3 元组 tuple

关于元组的使用有三个核心的函数：

std::make_tuple: 构造元组

std::get: 获得元组某个位置的值 

std::tie: 元组拆包

```c++
#include <tuple>
#include <iostream>
auto get_student(int id)
{
    // 返回类型被推断为 std::tuple<double, char, std::string>
    if (id == 0)
    	return std::make_tuple(3.8, ’A’, " 张三");
    if (id == 1)
    	return std::make_tuple(2.9, ’C’, " 李四");
    if (id == 2)
    	return std::make_tuple(1.7, ’D’, " 王五");
    return std::make_tuple(0.0, ’D’, "null");
    // 如果只写 0 会出现推断错误, 编译失败
}
int main()
{
    auto student = get_student(0);
    std::cout << "ID: 0, "
    << "GPA: " << std::get<0>(student) << ", "
    << " 成绩: " << std::get<1>(student) << ", "
    << " 姓名: " << std::get<2>(student) << ’\n’;
    double gpa;
    char grade;
    std::string name;
    // 元组进行拆包
    std::tie(gpa, grade, name) = get_student(1);
    std::cout << "ID: 1, "
    << "GPA: " << gpa << ", "
    << " 成绩: " << grade << ", "
    << " 姓名: " << name << ’\n’;
}

```

std::get 除了使用常量获取元组对象外，C++14 增加了使用类型来获取元组中的对象：

```c++
std::tuple<std::string, double, double, int> t("123", 4.5, 6.7, 8);
std::cout << std::get<std::string>(t) << std::endl;
std::cout << std::get<double>(t) << std::endl; // 非法, 引发编译期错误
std::cout << std::get<3>(t) << std::endl;
```

## 运行期索引

如果你仔细思考一下可能就会发现上面代码的问题，std::get<> 依赖一个编译期的常量，所以下面 的方式是不合法的：

```c++
int index = 1;
std::get<index>(t);
```

那么要怎么处理？答案是，使用 std::variant<>（C++ 17 引入），提供给 variant<> 的类型模板 参数可以让一个 variant<> 从而容纳提供的几种类型的变量（在其他语言，例如 Python/JavaScript 等， 表现为动态类型）：

```c++
#include <variant>
template <size_t n, typename... T>
constexpr std::variant<T...> _tuple_index(const std::tuple<T...>& tpl, size_t i) {
    if constexpr (n >= sizeof...(T))
    	throw std::out_of_range(" 越界.");
    if (i == n)
    	return std::variant<T...>{ std::in_place_index<n>, std::get<n>(tpl) };
    return _tuple_index<(n < sizeof...(T)-1 ? n+1 : 0)>(tpl, i);
}
template <typename... T>
constexpr std::variant<T...> tuple_index(const std::tuple<T...>& tpl, size_t i) {
	return _tuple_index<0>(tpl, i);
}

template <typename T0, typename ... Ts>
std::ostream & operator<< (std::ostream & s, std::variant<T0, Ts...> const & v) {
    std::visit([&](auto && x){ s << x;}, v);
    return s;
}

int i = 1;
std::cout << tuple_index(t, i) << std::endl
```

## 元组合并与遍历

还有一个常见的需求就是合并两个元组，这可以通过 std::tuple_cat 来实现：

```c++
auto new_tuple = std::tuple_cat(get_student(1), std::move(t));
```

马上就能够发现，应该如何快速遍历一个元组？但是我们刚才介绍了如何在运行期通过非常数索引 一个 tuple 那么遍历就变得简单了，首先我们需要知道一个元组的长度，可以：

```c++
template <typename T>
auto tuple_len(T &tpl) {
	return std::tuple_size<T>::value;
}
```

这样就能够对元组进行迭代了：

```c++
// 迭代
for(int i = 0; i != tuple_len(new_tuple); ++i)
    // 运行期索引
    std::cout << tuple_index(new_tuple, i) << std::endl;
```

# 第 5 章智能指针与内存管理

## std::shared_ptr

它能够记录多少个 shared_ptr 共同指向一个对象，从而消除 显示的调用 delete，当引用计数变为零的时候就会将对象自动删除。

但还不够，因为使用 std::shared_ptr 仍然需要使用 new 来调用，这使得代码出现了某种程度上的 不对称。

std::make_shared 就能够用来消除显式的使用 new，所以 std::make_shared 会分配创建传入参 数中的对象，并返回这个对象类型的 std::shared_ptr 指针。例如：

```c++
#include <iostream>
#include <memory>
void foo(std::shared_ptr<int> i)
{
	(*i)++;
}
int main()
{
    // auto pointer = new int(10); // illegal, no direct assignment
    // Constructed a std::shared_ptr
    auto pointer = std::make_shared<int>(10);
    foo(pointer);
    std::cout << *pointer << std::endl; // 11
    // The shared_ptr will be destructed before leaving the scope
    return 0;
}

```

std::shared_ptr 可以通过 get() 方法来获取原始指针，通过 reset() 来减少一个引用计数，并 通过 use_count() 来查看一个对象的引用计数。例如：

```c++
auto pointer = std::make_shared<int>(10);
auto pointer2 = pointer; // 引用计数 +1
auto pointer3 = pointer; // 引用计数 +1
int *p = pointer.get(); // 这样不会增加引用计数
std::cout << "pointer.use_count() = " << pointer.use_count() << std::endl; // 3
std::cout << "pointer2.use_count() = " << pointer2.use_count() << std::endl; // 3
std::cout << "pointer3.use_count() = " << pointer3.use_count() << std::endl; // 3
pointer2.reset();
std::cout << "reset pointer2:" << std::endl;
std::cout << "pointer.use_count() = " << pointer.use_count() << std::endl; // 2
std::cout << "pointer2.use_count() = " << pointer2.use_count() << std::endl; // 0, pointer2 已 reset
std::cout << "pointer3.use_count() = " << pointer3.use_count() << std::endl; // 2
pointer3.reset();
std::cout << "reset pointer3:" << std::endl;
std::cout << "pointer.use_count() = " << pointer.use_count() << std::endl; // 1
std::cout << "pointer2.use_count() = " << pointer2.use_count() << std::endl; // 0
std::cout << "pointer3.use_count() = " << pointer3.use_count() << std::endl; // 0,pointer3 已 reset
```

## 5.3 std::unique_ptr

是一种独占的智能指针，它禁止其他智能指针与其共享同一个对象，从而保证代 码的安全：

```c++
std::unique_ptr<int> pointer = std::make_unique<int>(10); // make_unique 从 C++14 引入
std::unique_ptr<int> pointer2 = pointer; // 非法
```

make_unique 并不复杂，C++11 没有提供 std::make_unique，可以自行实现：

```c++
template<typename T, typename ...Args>
std::unique_ptr<T> make_unique( Args&& ...args ) {
	return std::unique_ptr<T>( new T( std::forward<Args>(args)... ) );
}
```

既然是独占，换句话说就是不可复制。但是，我们可以利用 std::move 将其转移给其他的 unique_ptr，例如：

```c++
#include <iostream>
#include <memory>
struct Foo {
    Foo() { std::cout << "Foo::Foo" << std::endl; }
    ~Foo() { std::cout << "Foo::~Foo" << std::endl; }
    void foo() { std::cout << "Foo::foo" << std::endl; }
};
void f(const Foo &) {
	std::cout << "f(const Foo&)" << std::endl;
}
int main() {
    std::unique_ptr<Foo> p1(std::make_unique<Foo>());
    // p1 不空, 输出
    if (p1) p1->foo();
    
    {
        std::unique_ptr<Foo> p2(std::move(p1));
        // p2 不空, 输出
        f(*p2);
        // p2 不空, 输出
        if(p2) p2->foo();
        // p1 为空, 无输出
        if(p1) p1->foo();
       	p1 = std::move(p2);
        // p2 为空, 无输出
        if(p2) p2->foo();
        std::cout << "p2 被销毁" << std::endl;
    }
    // p1 不空, 输出
    if (p1) p1->foo();
    // Foo 的实例会在离开作用域时被销毁
}
```

## 5.4 std::weak_ptr

如果你仔细思考 std::shared_ptr 就会发现依然存在着资源无法释放的问题。看下面这个例子：

```c++
struct A;
struct B;
struct A {
    std::shared_ptr<B> pointer;
    ~A() {
        std::cout << "A 被销毁" << std::endl;
        }
};
struct B {
    std::shared_ptr<A> pointer;
    ~B() {
    	std::cout << "B 被销毁" << std::endl;
	}
};
int main() {
    auto a = std::make_shared<A>();
    auto b = std::make_shared<B>();
    a->pointer = b;
    b->pointer = a;
}
```

运行结果是 A, B 都不会被销毁，这是因为 a,b 内部的 pointer 同时又引用了 a,b，这使得 a,b 的引 用计数均变为了 2，而离开作用域时，a,b 智能指针被析构，却只能造成这块区域的引用计数减一，这样 就导致了 a,b 对象指向的内存区域引用计数不为零，而外部已经没有办法找到这块区域了，也就造成了 内存泄露

解决这个问题的办法就是使用弱引用指针 std::weak_ptr，std::weak_ptr 是一种弱引用,弱引用不会引起引用计数增加

# 其他杂项

## noexcept 的修饰和操作

C++11 将异常的声明简化为以下两种情况： 1. 函数可能抛出任何异常 2. 函数不能抛出任何异常

并使用 noexcept 对这两种行为进行限制，例如：

```c++
void may_throw(); // 可能抛出异常
void no_throw() noexcept; // 不可能抛出异常
```

使用 noexcept 修饰过的函数如果抛出异常，编译器会使用 std::terminate() 来立即终止程序运 行。

noexcept 还能够做操作符，用于操作一个表达式，当表达式无异常时，返回 true，否则返回 false。

```c++
#include <iostream>
void may_throw() {
	throw true;
}
auto non_block_throw = []{
	may_throw();
};
void no_throw() noexcept {
	return;
}
auto block_throw = []() noexcept {
	no_throw();
};
int main()
{
    std::cout << std::boolalpha
    << "may_throw() noexcept? " << noexcept(may_throw()) << std::endl
    << "no_throw() noexcept? " << noexcept(no_throw()) << std::endl
    << "lmay_throw() noexcept? " << noexcept(non_block_throw()) << std::endl
    << "lno_throw() noexcept? " << noexcept(block_throw()) << std::endl;
    return 0;
}
```

noexcept 修饰完一个函数之后能够起到封锁异常扩散的功效，如果内部产生异常，外部也不会触发。

```c++
try {
	may_throw();
} catch (...) {
	std::cout << " 捕获异常, 来自 my_throw()" << std::endl;
}
try {
	non_block_throw();
} catch (...) {
	std::cout << " 捕获异常, 来自 non_block_throw()" << std::endl;
}
try {
	block_throw();
} catch (...) {
	std::cout << " 捕获异常, 来自 block_throw()" << std::endl;
}
最终输出为：
捕获异常, 来自 my_throw()
捕获异常, 来自 non_block_throw()
```

## 字面量

C++11 提供了原始字符串字面量的写法，可以在一个字符串前方使用 R 来修饰这个字符串，同时， 将原始字符串使用括号包裹，例如：

```c++
#include <iostream>
#include <string>
int main() {
    std::string str = R"(C:\File\To\Path)";
    std::cout << str << std::endl;
    return 0;
}
```

## 自定义字面量

C++11 引进了自定义字面量的能力，通过重载双引号后缀运算符实现：

```c++
std::string operator"" _wow1(const char *wow1, size_t len) {
	return std::string(wow1)+"woooooooooow, amazing";
}
std::string operator"" _wow2 (unsigned long long i) {
	return std::to_string(i)+"woooooooooow, amazing";
}
int main() {
    auto str = "abc"_wow1;
    auto num = 1_wow2;
    std::cout << str << std::endl;
    std::cout << num << std::endl;
    return 0;
}
```

自定义字面量支持四种字面量：

整型字面量：重载时必须使用 unsigned long long、const char *、模板字面量算符参数，在上 面的代码中使用的是前者；

浮点型字面量：重载时必须使用 long double、const char *、模板字面量算符； 

字符串字面量：必须使用 (const char *, size_t) 形式的参数表； 

字符字面量：参数只能是 char, wchar_t, char16_t, char32_t 这几种类型。

