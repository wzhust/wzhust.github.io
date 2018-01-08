# 类型推断

**我能想到的类型推断的3个地方：auto、decltype和模板类型参数**

## auto

看一段python的代码：

```python
	var = 'hello'
	print( var )
	var = 2333
	print( var )
```

python、javascript之类的所谓动态类型语言，变量在使用前并没有声明，而是直接拿来使用，像c++等静态类型语言，使用变量时，需要提前声明和定义

实际上，从技术角度来看，静态类型和动态类型的区别无非是类型检查的时间点不一样，前者是在编译阶段，后者是运行时期，但是这些都需要编译器或者解释器进行类型推断

编译代码时，c++的编译器掌握着所有的类型信息，所以其实我们可以让编译器帮我们推断类型，使用c++11编写上述代码：

```cpp
	auto var1 = "hello"; // 自动推断var1类型是const char*
	std::cout << var1 << std::endl;
	auto var2 = 2333; // 自动推断var2类型是int
	std::cout << var2 << std::endl;
```

在c++11中，auto原来的表示自动声明周期的语义废弃了，而被赋予了新的语义：根据初始化表达式推断变量类型，需要指出的是，auto本质上是一个占位符，它不是一种类型，所以`sizeof( auto )`或者`typeid( auto )`是非法的

auto用作自动类型推断，最大的优点是简化代码，尤其是模板类型，如`auto iter = myMap.begin()`

此外，auto还用于**返回值后置**的函数中：
```cpp
	auto GetX() -> X; // X是类型
	template<typename T1, typename T2>
	auto Sum( const T1& a, const T2& b ) -> decltype(a + b); // 前置返回值编译不通过
```

auto用于类型推断，缺点也很明显，除了阅读源码的时候要费心变量类型，有时候推断结果可能并不如你所想：

- 可以使用const、volatile、*、&和&&来修饰auto

```cpp
	auto n = 3;
	const auto cn = 3;
	auto p1 = new auto(n); // int*
	const auto p2 = new auto(n); // int* const
	const auto* p3 = new auto(n); // const int*
	auto* p4 = new auto(n); // int*
	auto p5 = new auto(&n); // int**
	auto* p6 = new auto(&n); // int**
	auto** p7 = new auto(&n); // int**
	auto p8 = new auto(&cn); // const int** 可以推断“底层”cv特性
	auto n1 = cn; // int 无法获取cv特性
	const auto n2 = cn; // const int 显式获取cv特性
	auto& n3 = cn; // const int& 可以获取cv特性
	const auto& n4 = n; // const int&
	auto&& n5 = cn; // const int&
	auto&& n6 = GetInt(); // int&&
	const int ar[3] = {};
	auto va1 = ar; // const int* 推断为指针
	auto& va2 = ar; // const int(&)[3] 推断为数组引用类型 
```

注意auto&&，编译器会推断出正确的引用类型，并不只是推断右值引用：左值、const左值和右值推断出左值引用、const左值引用和右值引用

## decltype

> c++11新增关键字`decltype`用于获取表达式的静态类型

相比c++另一个关键字`typeid`，用于获取运行期类型信息（编译器为每个类型定义了type_info结构），decltype是获取表达式的静态类型，主要用于**模板编程**（模板编程中类型是作为未知参数处理的）

decltype的一些应用：

- 用于模板中进行类型推断

```cpp
	template<typename T1, typename T2>
	auto Sum( T1 a, T2 b ) -> decltype( a + b ) {
		return a + b;
	}
```

- 与using/typedef配合定义类型别名

```cpp
	/* 用已知基本类型、常量和运算符定义类型别名 */
	using size_t = decltype( sizeof( 0 ) );
	using ptrdiff_t = decltype( (int*)0 - (int*)0 );
	using nullptr_t = decltype( nullptr );
	
	/* 简化代码，与auto有点类似 */
	using vecIter = decltype( vec.begin() );
	decltype( vec )::iterator iter = vec.begin();
	
	/* 为获取匿名类型的类型提供了语法支持 */
	struct {
		int m_n;
	} anon_struct;
	decltype(anon_struct) a;
	a.m_n = 3;
```

decltype推断传给它的表达式（并不计算表达式）的类型，推断规则与auto不同：