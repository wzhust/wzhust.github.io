# [语言核心](http://en.cppreference.com/w/cpp/language)

---

## [lambda表达式](http://en.cppreference.com/w/cpp/language/lambda)

> **[capture] (parameters) mutable -> return-type { function body }**

参数列表、mutable关键字、尾置返回值是可选的，所以最简单的lambda表达式是`[]{}`

### 捕获规则

- 捕获列表只能捕获上下文环境中的**自动生命周期**的变量（包括this），也就是说，块作用域外的lambda捕获列表必须为空

- 值捕获和引用捕获的语义不可冲突或者重复，比如非法捕获式`[=, &a, b]`

- 尽量不要使用[=]或者[&]的默认捕获方式（容易产生空悬引用）

- 值捕获的变量不可更改，除非指定mutable

```cpp
	// 这段代码展示值捕获与引用捕获的区别
	void f( int p ) {
	int a = 1, b = 1;
		auto f1 = [p, a, &b] () mutable {
			auto f2 = [p, &a, b] () mutable {
				a = 2; b = 2;
			};
			a = 3; b = 3;
			f2();
		};
		f1();
	}
```

### lambda与函数对象

> c++0x利用函数对象（重载函数调用操作符的类）也能实现lambda的同等功效，但是c++11的lambda表达式更加直观和方便，其匿名特性也避免了对命名空间的污染

如果从函数对象的角度看lambda表达式（假设编译器使用了函数对象实现lambda），很多细节就清楚了：

- 捕获的变量实际上保存为匿名函数对象的成员，包括普通成员和引用成员，分别对应值捕获和引用捕获

```cpp
	// f1和f2是等效的
	std::string str( "mollyzz" );
	struct t_anonymousFunctor {
		t_anonymousFunctor( std::string& p ) : m_ref( p ) {}
		void operator()() const { std::cout << m_ref << std::endl; }
		std::string& m_ref;
	};
	t_anonymousFunctor f1( str );
	auto f2 = [&str] { std::cout << str << std::endl; };
```

- lambda默认是const的，类比函数对象的调用符重载函数也是const的，在该函数内改变成员变量是不允许的

- lambda作为编译器实现的一种闭包类型，可以初始化函数指针，前提是lambda没有捕获任何值，另外，lambda可以初始化模板std::function

```cpp
	using t_fptr = void(*)();
	t_fptr pf1 = [] {};
	//t_fptr pf2 = [&str] {}; // error
	std::function<void()> pf3 = [&str] {};
```

### lambda与函数式编程

> lambda让c++编写具有函数式风格的代码更加容易

- 函数是一等公民

lambda作为一种匿名函数对象，可以存放在容器，可以在模块间传递（典型的如STL中算法），可以进行组合运算（组合出功能复杂的函数），其地位跟普通类型和类类型是一样的

```cpp
	// 函数传递给异步模块
	auto task = []( auto callback ) { /* do something */ callback(); };
	auto callback = [] {/* do something when task finished */ };
	auto future = std::async( task, callback );
```

- 高阶函数:参数或者返回值是函数的函数

```cpp
	// std::bind参数是函数对象，返回值也是函数对象
	std::function<int(int)> unaryFunc = std::bind( []( int a, int b ) { return a + b; }, _1, 3 );
```

- 其他特性，如延迟计算

```cpp
	// 模仿python的生成器，FibonacciGenerator可以代表无限的斐波那契数列（不考虑整型溢出）
	std::function<int()> FibonacciGenerator = []() {
		int a1 = 0; int a2 = 0;
		return [a1, a2]() mutable {
			int nFibonacci = 0;
			if( a1 == 0 && a2 == 0 ) 
				nFibonacci = 1;
			else 
				nFibonacci = a1 + a2;
			a1 = a2;
			a2 = nFibonacci;
			return nFibonacci;
		};
	}();

	for( int i = 0; i < 10; ++i )
		std::cout << FibonacciGenerator() << std::endl;
```

### lambda总结

1. 匿名对象，务须费心为类型取名，也避免污染命名空间
2. 用于局部作用域代码逻辑的复用，简明直观
3. 方便地在不同模块间传递一些代码逻辑和上下文环境变量

## [右值引用](http://en.cppreference.com/w/cpp/language/reference)