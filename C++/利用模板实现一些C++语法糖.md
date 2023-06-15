# 利用模板实现一些C++语法糖

多个数相加

```c++
#include <iostream>
#include <memory>
using namespace std;
template <typename T>
auto sum(T x){
	return x;
}
template <typename T1, typename T2,typename... T>
auto sum(T1 x, T2 y,T... args){
	return sum(x + y, args...);
}

int main(){
	cout<<sum(1,2,3,4,5)<<endl;
}

```

多个数取max

```c++
#include <iostream>
#include <memory>
using namespace std;
template <typename T>
auto MAX(T x){
	return x;
}
template <typename T1,typename... T>
auto MAX(T1 x,T... args){
	return max(x,MAX(args...));
}

int main(){
	cout<<MAX(1,2,3,4,5)<<endl;
}

```

取最小值

```c++
#include <iostream>
#include <memory>
using namespace std;
template <typename T>
auto MIN(T x){
	return x;
}
template <typename T1,typename... T>
auto MIN(T1 x,T... args){
	return min(x,MIN(args...));
}

int main(){
	cout<<MIN(1,2,3,4,5)<<endl;
}
```



map

```c++
#include <iostream>
#include <memory>
#include <vector>
using namespace std;

template <typename T1,typename T2>
void Map(T1& a,T2 func){
	for(auto &i:a){
		auto tmp=i;
		i=func(tmp);
	}
}

int main(){
	vector<int>vec{1,2,3,4};
	Map(vec,[](auto x){return x+2;});
	for(auto i:vec){
		cout<<i<<endl;
	}
}

```

另一种map

```c++
#include <iostream>
#include <memory>
#include <vector>
using namespace std;

template <typename T1,typename T2>
void Map(T1& a,T2 func){
	for(auto &i:a){
		func(i);
	}
}

int main(){
	vector<int>vec{1,2,3,4};
	Map(vec,[](auto& x){x+=2;});
	for(auto i:vec){
		cout<<i<<endl;
	}
}

```

reduce

```c++
#include <iostream>
#include <memory>
using namespace std;

template <typename T1,typename T2>
auto reduce(T1 func,T2 x){
	return x;
}
template <typename T1, typename T2,typename... Targ>
auto reduce(T1 func,T2 x,Targ... args){
	return func(x,reduce(func,args...));
}

int main(){
	cout<<reduce([](int x,int y){return x*y;},1,2,3,4,5)<<endl;
}

```

以空格分割,打印多种数据，然后换行

```c++
#include <iostream>
#include <memory>
using namespace std;
template <typename T>
void OUT(T x){
	cout<<x<<endl;
}
template <typename T1,typename... T>
void OUT(T1 x,T... args){
	cout<<x<<" ";
	OUT(args...);
}
int main(){
	OUT(1,2,3,4,5);
}
```

将多个数据拼接成字符串

```c++
#include <iostream>
#include <memory>
using namespace std;

template <typename T>
std::string to_str(T x){
	if constexpr (std::is_same<T, std::string>::value)
		return x;
	else if constexpr (std::is_same<T, const char *>::value)
		return std::string(x);
	else
		return std::to_string(x);
}

template <typename T>
std::string str(T x){
	return to_str(x);
}
template <typename T1,typename... T>
std::string str(T1 x,T... args){
	return to_str(x)+str(args...);
}


int main(){
	string tmp="pkp";
	cout<<str(tmp,2,"xiasui",3,4,"jojo",114514);
}
```

