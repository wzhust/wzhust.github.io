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

在c++11中，auto原来的表示自动声明周期的语义废弃了，而被赋予了新的语义：根据初始化表达式推断变量类型，需要指出的是，auto本质上是一个占位符，它不是类型，`sizeof( auto )`是非法的

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
	const auto p2 = new auto(n); //
	const auto* p3 = new auto(n);
	auto* p4 = new auto(n);
	auto p5 = new auto(&n);
	auto* p6 = new auto(&n);
	auto** p7 = new auto(&n);
	auto p8 = new auto(&cn);
	auto n1 = cn;
	const auto n2 = cn;
	auto& n3 = cn;
	const auto& n4 = n;
	auto&& n5 = cn;
	auto&& n6 = GetInt();
```

## 基本概念：左值、右值和右值引用

> **左值和右值的概念很难定义，一般通过其特征来描述，一个容易实施的判断方法是，能取地址、有名字的变量是左值，反之为右值**

### 左值和右值

```cpp
	int a = 1, b = 2; // a和b是左值
	int c = a + b; // 表达式a + b是右值
	auto d = static_cast<double>(c); // 表达式static_cast<double>(c)是右值
	static_cast<double&&>(d); // 表达式static_cast<double&&>(d)是右值（xvalue/expired value）
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

### 右值引用

c++11引入新的引用语法T&&，表示右值引用，用来绑定右值
```cpp
	std::string&& strRRef = std::string("hello ") + "c++11";
	// do something with strRRef
```
strRRef是右值引用，同样能给右值“续命”，而且能修改右值，比如调用`strRRef.clear()`，这点是const左值引用办不到的；另外，为了保持语义上的完整，我们可以定义const右值引用，但却是鸡肋，毕竟如果不能改变右值，直接用const左值引用绑定就可以了

<table cellspacing="0"><caption>c++11引用类型与可引用的值类型</caption><tr><th></th><th>左值</th><th>const左值</th><th>右值</th><th>const右值</th><th></th></tr><tr><th>左值引用</th><th>Y</th><th>N</th><th>N</th><th>N</th><th></th></tr><tr><th>const左值引用</th><th>Y</th><th>Y</th><th>Y</th><th>Y</th><th>全能类型，可用于拷贝语义</th></tr><tr><th>右值引用</th><th>N</th><th>N</th><th>Y</th><th>N</th><th>用于移动语义</th></tr><tr><th>const右值引用</th><th>N</th><th>N</th><th>Y</th><th>Y</th><th>暂无用途</th></tr></table>

## 移动语义

> **如何将大象从A冰箱转移到B冰箱：c++11标准以前的做法是，复制一个大象，把复制品放入B冰箱，然后销毁A冰箱的大象**

c++11定义右值引用的目的之一，是能够正确识别右值，因为右值所持有的资源能够安全地被“窃取”（毕竟右值在表达式结束后是要销毁的）；另外，在某些情况下，“窃取”左值的资源也是合理的（程序本来的意图），可以使用`std::move`将左值转换成右值以便传递给恰当的函数

```cpp
template<typename T>
typename std::remove_reference<T>::type&& move( T&& t ) noexcept;
```

```cpp
	class t_cand
	{
	public:
		t_cand( const t_cand& p ) { /* allocate memory & copy */ }
		t_cand( t_cand&& p ) { m_lstrWord = p.m_lstrWord; p.m_lstrWord = nullptr;/* move(steal) */ }
	private:
		unsigned char *m_lstrWord;
	};
	t_cand GetCand();
	t_cand a( GetCand() ); // 接收右值（临时对象），调用参数为右值引用的构造函数
	t_cand b( a ); // 接收左值，调用参数为const左值引用的构造函数
	t_cand c( std::move( b ) ); // 接受右值，执行移动构造，构造完成后b不再持有资源（b.m_lstrWord == nullptr）
```

下面举几个例子说明下c++11移动语义带来的程序效率的改进和编程思想的改变：

- **交换两个对象的值**

```cpp
	template<typename T>
	void Swap( T& a, T& b ) {
		T temp( std::move( a ) );
		a = std::move( b );
		b = std::move( temp );
	}
```

如果`T`是持有资源的类型，且支持移动语义（比如具有移动拷贝构造函数和移动赋值操作符），这个交换过程就非常高效，只需要交换指针或者句柄；如果`T`不支持移动语义，这个交换过程依然会按照拷贝语义正常运行（const左值引用形参可以绑定任意类型实参）

c++11规定，只要你实现了拷贝构造和赋值运算的其中一个版本，另一个版本是默认删除的（=delete），除非两个版本都定义；只定义拷贝语义版本，程序不会出任何问题，运行结果也是正确的，但很多高效的操作就不存在了

- **`std::vector`的扩容**

如果考虑c++98标准下的动态数组扩容，大概过程是，先开辟更大的空间，然后将旧元素拷贝到新空间，最后销毁旧空间的元素，听起来简直跟转移大象的思路一模一样的，但是我们都知道，复制大象和销毁大象这俩过程，实在是效率低下

如果有c++11移动语义的支持，且数组中的元素支持移动拷贝和构造，那这个过程就高效多了，拷贝的只是指针或者句柄，销毁过程也会因为旧元素已经被“窃取”而变得很轻量

```cpp
	new ( &new_space[i] ) t_cand ( std::move( old_space[i] ) ); // 无论t_cand是否实现了移动语义，该操作总能正确进行
```

除了扩容操作，新标准的`std::vector`中很多地方都会因为移动语义变得很高效，比如`push_back`操作，实现了两个版本：const左值引用形参和右值引用形参；如果你的类实现了移动语义，在使用动态数组的时候，将自动享用这些高效操作

- **一些不可复制的对象可以放入容器并应用一些算法操作**

c++11标准库实现的智能指针`std::unique_ptr`是不可复制（但支持移动）的对象，实现了对原生指针的轻量级封装，在c++11标准之前的容器是不能存放这样的对象的，但新标准的容器因为支持移动语义，可以存放该类对象（同样的还有`std::thread`等）；甚至因为新标准的`std::sort`等算法支持移动语义，对容器中的这类元素进行某些算法操作，也十分高效（旧标准算法甚至都不支持）

```cpp
	// 如果使用了c++11之前的vector和sort 这段代码是无法编译通过的
	std::vector<std::unique_ptr<t_cand>> vecPtr;
	for( int i = 0; i < 100; ++i ) {
		vecPtr.push_back( std::make_unique<t_cand>() );
	}
	std::sort( vecPtr.begin(), vecPtr.end(), []( auto& lhs, auto& rhs ) { return true/* or false */; } );
```

- **对象的值传递**

比较分割字符串的两种写法：
```cpp
	std::vector<std::string> StrSplit( const std::string& p_str ) {
		std::vector<std::string> res;
		// ...
		return res; // v是左值，但优先移动，不支持移动时仍可复制
	} // 版本1 通过返回值返回结果
	void StrSplit( const std::string& p_str, std::vector<std::string>& p_res ) {
		// ...
	} // 版本2 通过参数返回结果
```

如果是c++98标准，我们会毫不犹豫地选择版本2，因为它高效，一直以来，对于“大”的对象，传值总是要三思后行（当然，考虑传值的效率总是对的）；不过如果使用新标准，只要传递的对象支持移动语义，版本1更加直观，同时也不会损失效率

**总体上看，如果你编写的类持有资源，或者在处理资源类的函数中，实现移动语义（借助右值引用语法和std::move）总是有益的，无论是使用STL容器和算法，还是处理临时对象等，效率自然而然地得到提升**

**移动语义是C++11中最重要的新特性之一，它解决了C++中大量的历史遗留问题，使C++标准库的实现在多种场景下消除了不必要的额外开销，也使得另外一些标准库（如std::unique_ptr）成为可能。即使你并不直接使用右值引用，也可以通过标准库，间接从这一新特性中受益。**

## 完美转发

> **完美转发（perfect forward）是相对于某个转发函数而言的，它能够将传递给它的参数保持原本属性不变地传递给目标函数**

看一个转发的例子：

```cpp
	void f( std::string&& ) {}
	void f( std::string& ) {}
	void f( const std::string& ) {}
	template<typename T>
	void Forward1( T p_str ) {
		f( p_str );
	}
	template<typename T>
	void Forward2( T& p_str ) {
		f( p_str );
	}
	const std::string str;
	Forward1( str ); // const对象，希望调用const左值引用版本，但是无法办到
	Forward1( std::string() ); // 临时对象，希望调用右值引用版本，但是无法办到
	//Forward2( std::string() ); // 编译错误，无法绑定右值
	Forward2( str ); // 能够正确转发
```

上述通过值传递参数并不是完美转发，不仅无法转发左右值属性，还把CV特性给丢了；即使采用引用传参，也无法“完美”地将参数传递给`f`，实际上，不管是`T p_str`还是`T& p_str`，`p_str`都是左值（**引用类型的变量都是左值**），所以右值引用版本无论如何是调不到的

为了解决转发问题，c++11引入了两个新语法概念：引用折叠（reference collapsing）和右值转换规则：

### 引用折叠和右值转换

**一般来讲，我们不能直接定义一个引用的引用，但是通过类型别名或者模板类型参数可以间接定义，引用折叠规则发生在这类情况下：**

- X& &、X&& &和X& &&都被折叠成X&
- X&& &&被折叠成X&&

```cpp
	using t1 = int&&;
	using t2 = t1&; // int&
	using t3 = t1&&; // int&&
	using t4 = t2&&; // int&
```

右值转换规则发生在**类型推断语境**下，主要在两个地方体现：变量自动推断auto&&和模板类型推断T&&，这里主要说的是模板类型推断语境下的右值引用：

```cpp
	template<typename T>
	void Forward( T&& p ) {}
	const std::string constStr;
	std::string str;
	Forward( constStr ); // T是const std::string& p的类型是const std::string&
	Forward( str ); // T是std::string& p的类型是std::string&
	Forward( std::string() ); // T是std::string p的类型是std::string&&
```

当函数模板采用右值引用作为形参时，我们可以向它传递任意类型的实参：左值、const左值、右值和const右值，类型推断会得到合适的参数类型：左值引用、const左值引用、右值引用和const右值引用，不仅左值右值属性保留，CV特性也保留了

目前来看，还差最后一步，将转发函数的参数传递给目标函数，因为引用类型变量都是左值，所以我们需要进行一些强制类型转换，以便明确告知目标函数该引用参数所绑定对象的真实左值右值属性，利用标准库的`std::forward`可以达到目的

```cpp
	template<typename T>
	constexpr T&& forward( typename std::remove_reference<T>::type& t ) noexcept; // 重载版本1
	template<typename T>
	constexpr T&& forward( typename std::remove_reference<T>::type&& t ) noexcept; // 重载版本2
```

标准库函数`std::forward`返回`static_cast<T&&>(t)`，也不过是做了强制类型转换

下面看一下最终实现的完美转发例子：

```cpp
	template<typename T>
	void Forward( T&& p ) {
		f( std::forward<T>( p ) );
	}
```

1. 如果传递给`Forward`的是左值`std::string`，则T为`std::string&`，p类型是`std::string&`，调用重载版本1的`std::forward`，进行强制类型转换`static_cast<T&&>(p)`，然后结合引用折叠，转换结果类型是`std::string&`，调用接收左值引用参数的`f`，转发正确
2. 如果传递给`Forward`的是const左值`const std::string`，则T为`const std::string&`，p类型是`const std::string&`，调用重载版本1的`std::forward`，进行强制类型转换`static_cast<T&&>(p)`，然后结合引用折叠，转换结果类型是`const std::string&`，调用接收const左值引用参数的`f`，转发正确
3. 如果传递给`Forward`的是右值`std::string`，则T为`std::string`，p类型是`std::string&&`，调用重载版本2的`std::forward`，进行强制类型转换`static_cast<T&&>(p)`，转换结果类型是`std::string&&`，调用接收右值引用参数的`f`，转发正确

c++11语法层面支持完美转发之后，很多问题的实现就非常优雅和方便了，举一个实际的例子：

-新标准的`std::vector`有个成员函数`emplace_back`

```cpp
	template<typename... Args>
	void emplace_back( Args&&... args );
```

该成员实现了一种类似`push_back`的操作，不过它的形参与容器中元素的某个构造函数形参一致，以实现“就地构造”，所以要求`emplace_back`能够完美地将参数转发给容器元素的构造函数（比如支持移动语义的构造函数就应该能够识别传给`emplace_back`的临时变量）

```cpp
	class t_elem {
	public:
		t_elem();
		t_elem( std::string&&, int );
		t_elem( std::string&, int );
		t_elem( const std::string&, int );
		// ...
	}
	std::vector<t_elem> vec;
	std::string str( "lvalue" );
	const std::string constStr( "const lvalue" );
	vec.emplace_back( str, 1 );
	vec.emplace_back( constStr, 2 );
	vec.emplace_back( std::string( "rvalue" ), 3 );
```