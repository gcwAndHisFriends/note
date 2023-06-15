# std::async要接收值

`std::async`生成`std::future`，它的析构将会阻塞直至`async`完成。由于没有赋值，因此两次产生的`std::future`对象均是临时变量，将会在表达式完成后析构，因此第一个`async`的阻塞将会使两次调用有序，最终结果为`z`。

```c++
#include <iostream>
#include <string>
#include <future>

int main() {
  std::string x = "x";

  std::async(std::launch::async, [&x]() {
    x = "y";
  });
  std::async(std::launch::async, [&x]() {
    x = "z";
  });

  std::cout << x;
}
```



只有`b.bar()`会调用`B`的虚函数，而在`A`的构造和析构时都不会，因为那些时候都没有**基类对象**，而是纯粹的操作父类对象`A`。

```c++
#include <iostream>
using namespace std;
struct A {
  A() { cout<<"1"<<endl;foo(); }
  virtual ~A() { cout<<"2"<<endl;foo(); }
  virtual void foo() { std::cout << "3"<<endl; }
  void bar() {cout<<"4"<<endl; foo(); }
};

struct B : public A {
  virtual void foo() { std::cout << "5"<<endl; }
};

int main() {
  B b;
  b.bar();
}
```

```c++
1
3
4
5
2
3

```

`continue`**会跳转到循环末尾而非开始**

```c++
#include <iostream>

int main() {
    int i=1;
    do {
        std::cout << i;
        i++;
        if(i < 3) continue;
    } while(false);
    return 0;
}
```

```c++
1
```

数组对象构造**有序**，析构反序。

```c++
#include <iostream>

class show_id
{
public:
    ~show_id() { std::cout << id; }
    int id;
};

int main()
{
    delete[] new show_id[3]{ {0}, {1}, {2} };
}
```

```c++
210
```

螺旋修饰

```c++
int const* const t[10]
```

1:一个有十个元素的数组

2:这个数组中的元素是const的

3:这个数组里存储的是指针

4:这个指针指向对象是const的

5:指针指向的是int类型

# ADL查找

`f(x)`会触发ADL查找，因此实参所在的`local`空间会被搜索，其中的`f(A)`会被加入候选函数，由于它比`template f(T)`更为特殊，因此成为最佳匹配函数。

而`::f(x)`是有限定查找，限定查找的含义是必须在所限定的空间中进行查找，因此ADL的过程被跳过了。而空限定名指示的是当前可见的空间，因此只找到了`template f(T)`这一匹配。