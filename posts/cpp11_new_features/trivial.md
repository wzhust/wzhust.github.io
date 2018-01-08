# 一些语法新特性

## nullptr

c++11定义了空指针类型`std::nullptr_t`，同时也定义了一个该类型的常量`nullptr`，实际上你也可以自己定义一个该类型的对象，用于表示空指针

空指针类型不是指针类型，它是一种编译器实现的特殊类型，能够隐式地转换为指针和成员指针

使用c++11之前的标准也可以轻易实现一个空指针类型，且语义与`std::nullptr_t`相同：

```cpp
	class t_nullptr {
	public:
		template<typename T>
		operator T*() const { return 0; } // 隐式转换普通指针
		template<typename C, typename T>
		operator T C::*() const { return 0; } // 隐式转换成员指针
	private:
		void operator&() const; // 禁止取地址
		intptr_t dummy = 0;
	};
	const t_nullptr NullPtr; // 定义一个常量对象
```

空指针类型比0或者NULL的优势体现在：

- 清晰，当看到函数调用`f( 0, 0, 0 )`和`f( 0, nullptr, 0 )`的时候，后者确实携带了更多信息：第二个参数是指针
- 避免重载函数的混淆，`f( 0 )`并不能调到`f( int* )`版本，如果有`f( int )`版本的话；在一些复杂的重载函数调用中，使用0意图传递空指针，不一定奏效
- 模板类型推断，和重载调用相似，传递0意图推断空指针行不通

## 基于范围的for

这个语法有点借鉴一些动态语言的意思，当我们的集合类实现了有效的begin和end，就可以使用该语法，而不用去自己迭代

```cpp
	class t_vec {
	public:
		int* begin() { return nullptr; }
		int* end() { return nullptr; }
	};
	t_vec myVec;
	// e为副本，存在拷贝
	for( auto e : myVec ) {
	}
	// e为引用
	for( auto&& e : myVec ) { // auto&或者const auto&
	}
```

## 静态断言static_assert

我们知道，c++有个assert函数用于运行时的断言，有时候编译器的断言也很有用，尤其是模板编程

```cpp
	static_assert(sizeof( uintptr_t ) == 4, "not 32-bit platform");
	static_assert(std::is_same<std::string, decltype(var)>::value, "var is not std::string type");
	template<typename T1, typename T2>
	void BitCopy( T1& src, T2& dest ) {
		static_assert(sizeof(src) == sizeof(dest), "not same size");
		memcpy(&dest, &src, sizeof(src));
	}
```

static_assert可以放在代码的任何位置，它并不是个函数

## 类内非static成员初始化

c++11支持**非static成员**的“就地”初始化，而且这种初始化的优先级低于构造函数初始化，所以当构造函数的初始化列表有某个成员的初始化，则采用构造函数的初始化

```cpp
	class t_myClass
	{
	public:
		t_myClass( int p ) : m_a( p ) {}
		t_myClass( const std::string& p ) : m_d( p ) {}
	private:
		int m_a = 3;
		float m_b{ 2.3f };
		//double m_c( 0.3 ); // 不能使用小括号初始化
		std::string m_d{ "hello" };
};
```

当一个类中的一些成员需要公共的初始值时，这种就地初始化的方式非常方便

## 强类型枚举

c++11之前的枚举能够隐式转换成整数，且枚举常量的作用域跟枚举类型一样，同时枚举的字节宽度也是不确定的，所以c++11定义了强类型枚举

```cpp
	enum class e_color : short { ecRed, ecBlue, ecGreen };
	e_color eColor = e_color::ecRed;
	
	// enum class { ecA, ecB } eLetter; // error 强类型枚举类型必须有名字
```

- 强类型枚举常量的作用域不会延伸到该枚举类型的作用域，必须通过枚举类型引用
- 强类型枚举没有与整数的隐式转换
- 可以定义字节宽度

## override和final

这两个关键字增强了继承的可控性，同时也避免了一些错误的发生

```cpp
	class t_object {
	public:
		virtual void f( unsigned int p ) = 0;
	};

	class t_base : public t_object {
	public:
		void f( int p ) override; // error not override
		virtual void g( int p ) final;
	};

	class t_derived : public t_base {
	public:
		void g( int p ); // error g is final in t_base
	};
```

另外，c++标准里，如果重写了父类方法，不需要显式加virtual，这也是造成一些问题的点

## 模板别名

c++11扩充了`using`关键字的语义，可以用来定义类型别名，与typedef功能相似，不过更加强大

```cpp
	using myString = std::string; // typedef std::string myString;
	using myFunc = int(*)(int, int);
	std::vector<int> vec;
	using myIter = decltype(vec)::iterator;
	// 模板别名，typedef无法做到
	template<typename T>
	using myMap = std::map<T, T>;
	myMap<int> oMap;
	// 使用typedef实现模板别名
	template<typename T>
	struct t_myMap {
		typedef std::map<T, T> myMap;
	};
	t_myMap<int>::myMap oMap2;
```

**使用using代替typedef的理由有两个：using比typedef更直观；using可以定义模板别名，这点typedef要迂回才能做到，而且有些情况下，引用模板别名还要加typename，十分不方便**

## 原生字符串

在原生字符串中，不发生转义，一般用于正则表达式的模板串

```cpp
	const char* sz1 = R"(abc\n)"; // 包含5个字符
	const char* sz2 = "abc\n"; // 包含4个字符
```

## delete和default

## 初始化列表

## 常量表达式

## 可变参模板