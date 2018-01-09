# 智能指针

> **浅层次来讲，智能指针就是利用RAII机制封装指针，同时提供类似指针的操作的类，摈弃原生指针能够避免很多资源泄漏、空悬指针和异常安全问题；更进一步，智能指针其实是把c++的值语义变成了引用语义，类似Java中的变量**

c++11引入的智能指针有3个：
std::unique_ptr：独占资源所有权
std::shared_ptr：共享资源所有权，我们平时说的智能指针大概是指这个了
std::weak_ptr：无资源所有权，可以观察资源使用情况

## std::unique_ptr

## std::shared_ptr

## std::weak_ptr

std::weak_ptr一般由std::shared_ptr构造或赋值而来，它对共享资源没有所有权，也不支持*或者->操作，一般用来观察资源的使用情况，调用lock成员可以返回一个指向共享资源的shared_ptr，进而访问对象

```cpp
	auto sp = std::make_shared<std::string>(); // 引用计数为1
	std::weak_ptr<std::string> wp( sp ); // 构造weak_ptr观察共享资源，不改变引用计数
	//sp = nullptr; // sp.reset();
	auto nUseCount = wp.use_count(); // 返回引用计数值
	bool bExpired = wp.expired(); // 等级wp.use_count() == 0
	auto sp1 = wp.lock(); // 获取指向共享资源的一个shared_ptr，如果资源已经释放，返回空的shared_ptr，否则引用计数加1
	nUseCount = wp.use_count();
```

举几个std::weak_ptr的用途：

- 解除std::shared_ptr的循环引用问题

假设两个共享对象A和B，A对象拥有一个指向B对象的共享指针，便于访问B，同时B也有访问A的需求，于是B需要持有指向A的一个指针，那么有三种选择：原生指针、std::unique_ptr、std::shared_ptr和std::weak_ptr

使用原生指针和std::unique_ptr，当A对象释放之后，它们可能指向无效的资源；使用std::shared_ptr虽然不会出现空悬指针的问题，但是A和B形成了相互引用，它们都无法被释放；使用std::weak_ptr正好能够很好的解决这个问题，它不增加A的引用计数值，同时在A有效期间，能够访问A

**弱指针std::weak_ptr提供了便利的手段解决一些循环（环状）引用问题，采用引用计数技术一个难题就是环状引用，当我们在引用环上加上弱引用，这个环就很容易打破了**

- 观察者模式

假设有一个被观察者，它有一群观察者，当被观察者的状态发生改变的时候，需要通知观察者们，但是被观察者根本不care观察者的生命周期，也就是说，如果某个或某些观察者被销毁了，被观察者就不需要通知它们了

上述也是std::weak_ptr使用的场景，被观察者持有观察者们的弱引用，当期状态改变时，通过弱引用（可以判断观察者是否已经被销毁）通知观察者

- 共享资源缓存池

```cpp
	std::shared_ptr<t_resource> FastGetResource( t_resId id ) {
		static std::unordered_map<t_resId, std::weak_ptr<t_resource>> cache;
		auto resPtr = cache[id].lock();
		if( !resPtr ) { // 缓存池没有
			resPtr = GetResource( id ); // 调用真正获取资源的接口（可能很耗时）
			cache[id] = resPtr; // 缓存资源弱引用
		}
		return resPtr;
	}
```

**当共享资源提供方提供了资源时，保留一份弱指针指向该资源，当再次申请该资源时，可以通过弱指针查看之前的共享资源是否释放，如果没有释放，直接返回一个共享指针，如果已经释放，重新获取资源**