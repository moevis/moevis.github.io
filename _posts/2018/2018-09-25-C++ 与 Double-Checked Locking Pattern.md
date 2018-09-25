---
published: true
layout: post
categories: cheatsheet
---
随手翻译了一下，原文来自：https://www.aristeia.com/Papers/DDJ_Jul_Aug_2004_revised.pdf

## 介绍

你应该在很多地方用过单例模式，在很多语言中，像 js，python 这些可以利用模块本身的特性实现单例。但是到了 C++ 这边，就需要考虑额外的东西：单例模式需要考虑线程安全。

当然，最典型一个解决方案是 Double-Checked Locking Pattern（DCLP），这个方法可以对一些共享的变量提供线程安全的处理（比如我们提到的单例）。但是，这个方法仍有问题：不可靠。或者说，没有一种绝对可靠并且可移植的方法来实现线程安全的单例。甚至 DCLP 可能会因为多种原因在单核和多核机器上崩溃。

## 单例模式和多线程

我们先来看一个单例模式的最简单的实现，并看看它到底有什么问题：

```c++
// from the header file
class Singleton {
public:
static Singleton* instance();
	...
private:
	static Singleton* pInstance;
};

// from the implementation file
Singleton* Singleton::pInstance = 0;

Singleton* Singleton::instance() {
if (pInstance == 0) {
}
return pInstance;
```

在单线程环境中，这段代码可以很好地工作，除了中断处理会有麻烦。如果你在 Singleton::instance() 中接收到一个中断，并在中断处理中调用 Singleton::instance 时，会出问题。不过除了中断外，这段代码的确可以很好地工作。

然而这段代码在多线程环境中会出现问题，假如线程 A 进入了 instance 函数，并在`if (pInstance == 0) { ` 运行后休眠，这时候 pInstance 还是 nullptr；这时候线程 B 也进入 instance 函数并运行到同样位置，它看到 pInstance 还是 nullptr，于是也开始初始化函数。再过一会儿线程 A 继续运行，它也开始初始化 pInstance，问题就这么发生了。 pInstance 被初始化了两次。

当然，这个解决这个问题的方法也很简单，加入一个锁即可：

```c++
Singleton* Singleton::instance() {
	Lock lock; // acquire lock (params omitted for simplicity)
	if (pInstance == 0) {
		pInstance = new Singleton;
	return pInstance;
} // release lock (via Lock destructor)
```

然而这个操作的代价很昂贵，每一次获取单例元素都会请求一次加锁，然而两次初始化的问题仅仅是在第一次获取 pInstance 的情况下才可能发生。假如我们访问 pInstance n 次，那么除了第一次加锁是有用的，接下来的 n - 1 次都是毫无必要的。所以这时候 DCLP 被设计出来解决这个问题。

## The Double-Checked Locking Pattern

DCLP 的关键工作是在获取 pInstance 的时候，首先判断一次 pInstance 是否存在，然后再加锁：

```c++
Singleton* Singleton::instance() {
    if (pInstance == 0) { // 1st test
    	Lock lock;
        if (pInstance == 0) { // 2nd test
        	pInstance = new Singleton;
        }
    }
    return pInstance;
}
```

在加锁完毕后有可能在另外的线程以及初始化完毕了 pInstance，所以在加锁后第二次检查 pInstance，由于这时候已经在锁中，所以其他线程都无法再初始化 pInstance。

然而这个 DCLP 的实现依然是有问题的，比如实际上经过编译器优化后的代码的执行顺序并不是我们所想象的那样。

## DCLP 和执行顺序

再思考一下初始化 pInstance 的代码：

```c++
pInstance = new Singleton;
```

这里实际上是做了三个工作：

- Step 1：为 Singleton 对象申请内存
- Step 2：构造 Singleton 对象
- Step 3：让 pInstance 指向刚刚申请的地址

一个严重的问题就是，编译器可能并不是严格这样顺序执行命令的。尤其在第二和第三步执行顺序可能调换。我们来看一遍出现交换后的代码：

```c++
Singleton* Singleton::instance() {
    if (pInstance == 0) {
        Lock lock;
        if (pInstance == 0) {
            pInstance = // Step 3
            	operator new(sizeof(Singleton)); // Step 1
            new (pInstance) Singleton; // Step 2
        }
    }
    return pInstance;
}
```

现在我们设想这一个场景：

- 线程 A 进入 instance，发现 pInstance 未初始化，于是请求了 lock，并完成了 Step1 和 Step 3，然后进入休眠状态。
- 线程 B 进入 instance，发现 pInstance 已经初始化了，于是将 pInstance 指针返回。然而在外接调用 pInstance 的方法很可能就崩溃了，因为 pInstance 实际上并未初始化完毕。

DCLP 在这时候失效了，因为编译器打乱了语句执行顺序。这就是使用 DCLP 最危险的地方，我们希望运算的先后顺序能够保证，但是我们的 C++ 并不提供这样的约束语法。

对于 C++ 编译器的这些行为，我们可以再来一个例子：

```c++
void Foo() {
    int x = 0, y = 0; // Statement 1
    x = 5; // Statement 2
    y = 10; // Statement 3
    printf("%d,%d", x, y); // Statement 4
}
```

这是一段很蠢的代码，在 C 和 C++ 中，我们可以保证上面的代码执行结果是输出 "5, 10"，但是这也是我们知道的仅有的一点线索，我们甚至不知道 1-3 是否会执行，事实上一个优秀的编译器应该将它们优化掉，因为我们要的仅仅是输出一个字符串而已。如果 1-3 执行了，我们知道 1 应该在 2-4 之前执行，而 4 应该在 1-3 之后。但是我们不能保证 2 和 3 的执行顺序。编译器可以选择让 2 先执行，或者让 3 先执行，甚至是 2 和 3 并行。

所以作为程序员，也许你开始寻找其他方法来避免这种情况：

```c++
Singleton* Singleton::instance() {
    if (pInstance == 0) {
        Lock lock;
        if (pInstance == 0) {
            Singleton* temp = new Singleton; // initialize to temp
            pInstance = temp; // assign temp to pInstance
        }
    }
    return pInstance;
}
```

你打算定义一个临时变量，这样在初始化 pInstance 时就不会让 pInstance 指向一个未初始化的区域。然而你还是想错了，编译器会发现 temp 变量是不必要的，很可能将这一步优化掉。于是你拥有了和之前一样的代码。

## volatile 关键字

你也许也知道 volatile 关键字，根据 C++ 标准可以知道，当获取一个 volatile 的左值时，C++ 会保证之前所有的 I/O，函数调用，修改变量等操作都完成，对后续没有副作用。所以我们可以这样写 DCLP：

```c++
class Singleton {
public:
static Singleton* instance();
    ...
private:
	static Singleton* pInstance; // volatile added
	int x;
	Singleton() : x(5) {}
};
```

实现：

```c++
Singleton* Singleton::pInstance = 0;
Singleton* Singleton::instance() {
    if (pInstance == 0) {
        Lock lock;
        if (pInstance == 0) {
        	Singleton* temp = new Singleton; // volatile added
       		pInstance = temp;
        }
    }
    return pInstance;
}
```

经过编译器 inline 后，代码是这样的：

```c++
if (pInstance == 0) {
    Lock lock;
    if (pInstance == 0) {
    	Singleton* volatile temp =
        static_cast<Singleton*>(operator new(sizeof(Singleton)));
        temp->x = 5; // inlined Singleton constructor
        pInstance = temp;
    }
}
```

虽然 temp 是 volatile 的，但是 *temp 不是，这意味着 temp->x 也不是。现在我们知道 non-volatile 的变量有时候会被乱序执行，很容易就看出来编译器可以将 temp->x 的赋值提前，pInstance 就可以在他指向的数据初始化 ( temp->x = 5) 之前被分配内存，这样的写法依然是有问题的。另一种写法是：

```c++
class Singleton {
public:
	static volatile Singleton* volatile instance();
	...
private:
	// one more volatile added
	static Singleton* volatile pInstance;
};

// from the implementation file
volatile Singleton* volatile Singleton::pInstance = 0;
volatile Singleton* volatile Singleton::instance() {
    if (pInstance == 0) {
        Lock lock;
        if (pInstance == 0) {
            // one more volatile added
            Singleton* volatile temp =
            	new Singleton;
            pInstance = temp;
        }
    }
    return pInstance;
}
```

到这一步，有些人可能会问，为什么 Lock 变量没有加 volatile 约束呢。毕竟我们已经把 pInstance 和 temp 都加了这个约束了。道理很简单， Lock 是来自一个线程库，所以我们可以假定 ——Lock 已经被线程库做了约束了（这个假设对于目前我们所知道的所有线程库都是正确的）。

有些人认为 volatile 约束可以让代码在多线程环境中也能正确工作，不过其实它还会在两种情况下出错：

- 第一，虽然 C++ 标准中编译器对于线程内变量的读写可以做到禁止重排序，但是他对于不同线程之间变量的读写依然没有限制。所以，实际上很多编译器依然会产生费线程安全的机器码，如果你的多线程代码加上 volatile 可以正确运行，那么可能你的编译器正确地在线程间的读写顺序上做了约束（可能性不大），或者是你太幸运了（很可能），并且，就算你的代码能正确运行，也不能保证可移植性。
- 第二，使用 const 来约束的对象在构造函数运行完毕前，依然不会成为常量。在这句话中：

```c++
Singleton* volatile temp = new Singleton;
```

正在被构造的对象在下面语句运行完毕前依然不会受到 volatile 约束：

```c++
new Singleton;
```

你可以想象我们的代码在构造函数中依然会被打乱执行顺序，所以我们这意味着我们又回到了之前的场景。当然，我们可以用很糟糕的方法来解决他们——给所有成员变量加上 volatile 约束。比如这样（为了简化代码，我们假定 Singleton 类中只有一个成员变量 x）：

```c++
Singleton()
{
	static_cast<volatile int&>(x) = 5; // note cast to volatile
}
```

在经过编译器 inline 操作后，代码会变成这样：

```c++
class Singleton {
public:
static Singleton* instance();
	...
private:
    static Singleton* volatile pInstance;
	int x;
	...
};

Singleton* Singleton::instance()
{
	if (pInstance == 0) {
        Lock lock;
        if (pInstance == 0) {
            Singleton* volatile temp =
            static_cast<Singleton*>(operator new(sizeof(Singleton)));
            static_cast<volatile int&>(temp->x) = 5;
            pInstance = temp;
        }
    }
}
```

现在对 x 的赋值必须在给 pInstance 赋值之前，因为两者都加上了 volatile 约束。

不幸的是，C++ 运行抽象是一个单线程的机器，C++ 编译器依然可能选择生成线程不安全的机器码。然而你还要考虑这样的写法使你的代码失去被编译器做优化的机会，导致运行效率降低——好吧，经过本节讨论，似乎我们回到了原点，而且似乎更糟糕了。等等，还有多处理器没有讲呢！

## DCLP 遇上多处理器

假设你及其有多核处理器，每个核有独立的缓存，同时也共享同一片内存空间。这样的结构需要明确定义当单核的独立缓存发生更改的时候，如何让其他核心知道。这就是著名的缓存一致性问题

假设 A 处理器修改了共享变量 x，之后又修改了共享变量 y。这些新的值必须要同步进主存，以便其他核心能够看到他们。而以递增的顺序更新主存中的变量效率会更高，所以如果 y 的地址在 x 的前边，那么其他处理器将看到 y 先于 x 发生变化。这样会对 DCLP 的算法造成严重问题。正确的单例模式要求单例对象先被初始化，再赋值给 pInstance。然而多处理器可能会看到相反的初始化顺序，造成严重问题。一般的解决方案是使用内存屏障（memory barriers），下面是伪代码的表示，我们没有明确写出具体的内存屏障的代码，因为这些实际的代码往往和你的架构相关：

```c++
Singleton* Singleton::instance () {
    Singleton* tmp = pInstance;
    ... // insert memory barrier
    if (tmp == 0) {
        Lock lock;
        tmp = pInstance;
        if (tmp == 0) {
            tmp = new Singleton;
            ... // insert memory barrier
            pInstance = tmp;
        }
    }
    return tmp;
}
```

Arch Robison 指出这是矫枉过正：

> 理论上来说，你不需要任何双向的屏障，第一个屏障是为了防止单例对象被其他线程获取到，是一个下行屏障（读取）。第二个屏障是为了防止单例对象被多次初始化，是一个上行屏障（写入）。这两个操作被称为：acquire 和 release。也许这样子可以比硬件平台上实现的双向屏障的性能更好。

不过，以上的这些代码，依然依靠你的机器上有实现可靠的内存屏障。有趣的是，内存屏障的做法在单处理器上也可以很好运行，因为内存屏障可以看做一个硬性同步点，使代码正确性不会因为编译器的乱序执行而受影响。

## 结论

对于单例模式，我们不仅仅要学 DCLP 的思想，还有很多值得注意的点：

- 第一，记得在单核上基于时间片轮转的并行并不是真正的并行，所以一些在单核架构中线程安全的方法在多核中不会正确运行，即使你指定了相同的编译器。
- 第二，虽然单例问题可以用 DCLP 来解决，但是本身 DCLP 和单例没有必然的联系，我们在使用单例时也倾向优化多线程的访问。如果你的代码对于多线程加锁性能很敏感，那么我建议用一个缓存变量来避免频繁上锁。

比如，你要避免下面的写法：

```c++
Singleton::instance()->transmogrify();
Singleton::instance()->metamorphose();
Singleton::instance()->transmute();
```

而应该这么写：

```c++
Singleton* const instance =
	Singleton::instance(); // cache instance pointer
instance->transmogrify();
instance->metamorphose();
instance->transmute();
```

这样即使你使用了锁，依然可以保证在一个线程访问中只加了一次锁。

- 第三，尽量避免使用懒加载（Lazily-initialized）的单例，除非你真的需要。你可以选择在进程的一开始初始化资源——因为所有的多线程程序都是以单线程启动的。这样你的代码在单次运行时可以加载的避免不确定性。
- 第四，可以考虑用 monostate 模式来取代单例。
- 第五，另一种可以替代全局单例的思想是为每一个线程提供一个单例，使用线程的本地变量进行存储，这样你可以避免很多线程问题，不过这也意味着你要在多线程程序中管理多个单例。
- 最后，DCLP  及其在 C++ 中可能出现的问题说明了用一种没有原生线程语法的语言来写线程安全的代码是非常困难的。多线程问题对代码生成过程（codegen）影像无处不在。正如 Peter Buhr 指出的，希望将多线程拒绝于语言之外，并用库来包装，最终会导致 (1) 对应的多线程库生成大量约束编译器行为的代码（比如 pthread 所做的那样）或者是 (2) 拒绝一切有关的优化，不管他是不是跑在单核上。
