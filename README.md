# Introduction

C++ did not natively provide an equivalent to events or delegates in other languages until C++11, and even though the added `std::function` is nice and easy to use, it has some non-negligible performance overhead making it hard to replace the existing event notification systems, which are usually either hard-coded, or based on observer-pattern interfaces. Improvements or replacements to `std::function` have been hot topics every once a while, e.g. `function_view`. At any time, there are probably more than two dozens of active libraries about signals / events / delegates on Github. Each comes with a slightly different flavor, for instance, some want to be more generic and flexible, while some others want to be thread-safe. For the applications I've been working on, games and game engines, it is extremely important to keep everything fastest possible. I want to find a multicast-delegate that beats interface-based observers on performance. I want to have an alternative, equivalent in speed solution to hard-coded function calls. When it is fast enough, we can afford to use it everywhere to prevent tight couplings or design-pattern hell situations.

Long before C++11, this legendary article https://www.codeproject.com/Articles/7150/Member-Function-Pointers-and-the-Fastest-Possible pointed out that the usage of member function pointers can save overhead compared to virtual-based type-erasures. The topic eventually got refined in https://www.codeproject.com/articles/11015/the-impossibly-fast-c-delegates and further in https://www.codeproject.com/Articles/1170503/The-Impossibly-Fast-Cplusplus-Delegates-Fixed. The performance is close to perfect, but the syntax isn't. `auto dInstance = decltype(d)::create<Sample, &Sample::InstanceFunction>(&sample);` is far too mouthful, and contains a fundamental design flaw: If the delegate (the instance here) knows its target function upfront, there is no meaning to use a delegate at all - just use the bound function instead. In order to be useful, the delegate (= the sender) needs to be decoupled from the target (= receiver), which means the target needs to know the type of the sender, but not the other way around. Lastly, C++17 provides a new mechanism, `template<auto>`, which can eliminate the need of mentioning the type of value in template arguments, such as `<Sample, &Sample::InstanceFunction>`. 

In this project, I will demonstrate how to design a clean, foolproof event (aka multi-cast delegate) system that satisfies:
* Max performance - one pointer indirection
* RAII for connection and disconnection
* No inheritance in user code

Ideally, the user-code would look like this:
```cpp
struct Foo
{
	//defines the container for listeners
	Event<void(const Foo& foo)> update;
};

struct Bar
{
	//callback method
	void onFooUpdate(const Foo& foo);

	//connection object, can be default constructed (disconnected) 
	
	Listener<&Foo::update, &Bar::onFooUpdate> m_listenerFooUpdate;
	
	//or connected on construction
	//Listener<&Foo::update, &Bar::onFooUpdate> m_listenerFooUpdate { some_obj, this };

	//connect
	void someInitFunc(Foo* foo)
	{
		m_listenerFooUpdate.connect(foo, this);
	}
};
```


