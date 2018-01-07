# 右值引用

> **c++11引入rvalue reference语法，主要解决两个问题：资源移动（move semantics）和参数转发（perfect forward）**

## 基本概念：左值、右值和右值引用

> **左值和右值的概念很难定义，一般通过其特征来描述，一个容易实施的判断方法是，能取地址、有名字的变量是左值，反之为右值**

### 左值和右值

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

### 右值引用

c++11引入新的引用语法T&&，表示右值引用，用来绑定右值
```cpp
	std::string&& strRRef = std::string("hello ") + "c++11";
	// do something with strRRef
```
strRRef是右值引用，同样能给右值“续命”，而且能修改右值，比如调用`strRRef.clear()`，这点是const左值引用办不到的；另外，为了保持语义上的完整，我们可以定义const右值引用，但却是鸡肋，毕竟如果不能改变右值，直接用const左值引用绑定就可以了

<table><caption>c++11引用类型与可引用的值类型</caption><tr><th></th><th>左值</th><th>const左值</th><th>右值</th><th>const右值</th><th></th></tr><tr><th>左值引用</th><th>Y</th><th>N</th><th>N</th><th>N</th><th></th></tr><tr><th>const左值引用</th><th>Y</th><th>Y</th><th>Y</th><th>Y</th><th>全能类型，可用于拷贝语义</th></tr><tr><th>右值引用</th><th>N</th><th>N</th><th>Y</th><th>N</th><th>用于移动语义</th></tr><tr><th>const右值引用</th><th>N</th><th>N</th><th>Y</th><th>Y</th><th>暂无用途</th></tr></table>

## 移动语义

> **如何将大象从A冰箱转移到B冰箱：c++11以前的做法是，复制一个大象，把复制品放入B冰箱，然后销毁A冰箱的大象**

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

如果`T`是持有资源的类型，且支持移动语义（比如具有移动拷贝构造函数和移动赋值操作符），这个交换过程就非常高效，只需要交换指针或者句柄；如果`T`不支持移动语义，这个交换过程依然会按照拷贝语义正常运行（毕竟const左值引用形参可以绑定任意类型实参）

c++11规定，只要你实现了拷贝构造和赋值运算的其中一个版本，另一个版本是默认删除的（=delete），除非两个版本都定义；只定义拷贝语义版本，程序不会出任何问题，运行结果也是正确的，但很多高效的操作就不存在了，比如高效的交换以及后面提到的很多场景

- **`std::vector`的扩容**

如果考虑c++98标准下的动态数组扩容，大概过程是，先开辟更大的空间，然后将旧元素拷贝到新空间，最后销毁旧空间的元素，听起来简直跟转移大象的思路一模一样的，但是我们都知道，复制大象和销毁大象这俩过程，实在是效率低下

如果有c++11移动语义的支持，且数组中的元素支持移动拷贝和构造，那这个过程就高效多了，拷贝的只是指针或者句柄，销毁过程也会因为旧元素已经被“窃取”而变得很轻量

```cpp
	new_space[i] = std::move( old_space[i] ); // 调用合适的赋值操作
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
	std::sort( vecPtr.begin(), vecPtr.end(), []( auto& lhs, auto& rhs ) { return true; } );
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

> **完美转发（perfect forward）是相对于某个转发函数而言的，它能够将传递给它的参数保持原本属性不变地传递给其他函数**

看一个转发的例子：

```cpp
	void f( std::string&& ) {}
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
	Forward1( str );
	Forward1( std::string() ); // 临时对象，希望调用右值引用版本，但是无法办到
	//Forward2(std::string() ); // 编译错误，无法绑定右值
```

上述通过值传递参数并不是完美转发，不仅无法转发左右值属性，还把CV特性给丢掉了；即使采用引用传参，也无法“完美”地将参数传递给`f`，实际上，不管是`T p_str`还是`T& p_str`，`p_str`都是左值（**引用类型的变量都是左值**），所以右值引用版本无论如何是调不到的

