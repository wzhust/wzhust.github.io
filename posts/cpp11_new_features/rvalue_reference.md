# 右值引用

> **c++11引入rvalue reference语法，主要解决两个问题：资源移动（move semantics）和参数转发（perfect forward）**

## 概念：左值、右值和右值引用

> **左值和右值的概念很难定义，一般通过其特征来描述，一个容易实施的判断方法是，能取地址、有名字的变量是左值，反之为右值**

- 左值和右值

```cpp
	int a = 1, b = 2; // a和b是左值
	int c = a + b; // 表达式a + b是右值
	auto d = static_cast<double>(c); // 表达式static_cast<double>(c)是右值
	auto f = [] { return std::string( "wzhust.github.io" ); };
	auto e = f(); // 表达式f()是右值
	// 除了字符串字面常量，其他字面常量都是右值
	// &[] { return 2333; }; // error 匿名对象lambda也是右值
	f().clear(); // 右值是可以修改的
```

c++11之前只有左值引用，一般用来绑定左值（别名），不过const左值引用可以绑定任意左值和右值（只读），比如
```cpp
	const std::string& strCfRef = std::string("hello ") + "c++11";
	// do something with strCfRef
```
值得注意的是，**const左值引用延长了被绑定右值的生命周期（变得跟该引用变量的生命周期一样长）**，上例中，若非`strCfRef`指向了该右值，表达式结束后，临时`std::string`对象就销毁了

- 右值引用

c++11引入新的引用语法T&&，表示右值引用，用来绑定右值
```cpp
	std::string&& strRRef = std::string("hello ") + "c++11";
	// do something with strRRef
```