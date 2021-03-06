# ch3 共享对象

## 可见性

可见性是微妙的,这是因为可能发生错误的事情总是与直觉大相径庭. 在一个单线程环境里, 如果向一个变量先写入值, 然后再没有写干涉的情况下读取这个变量, 你希望能得到相同的返回值. 这看起来是很自然的. 但是当读和写发生在不同的线程中, 情况却根本不是这样 ---> 通常不能保证读线程及时地读取其他线程写入的值, 甚至可以说根本不可能. 为了确保跨线程写入的内存可见性, 你必须使用同步机制

### 锁和可见性

内置锁可以用来确保一个线程以某种可预见的方式看到另一个线程的影响

当一个域声明为volatile类型后, 编译器与运行时环境会监视这个变量: 它是共享的, 而且对它的操作不会与其他的内存操作一起被重排序. volatile变量不会被缓存到寄存器或者缓存在对其他处理器隐藏的地方. 所以, 读一个volatile类型的变量时, 总会返回由某一个线程所写入的最新值.

> 只有当volatile变量能够简化实现和同步策略的验证时,才使用它们. 当验证正确性必须推断可见性问题时, 应该避免使用volatile变量. 正确使用volatile变量的方法包括: 用于确保他们所引用的对象状态的可见性, 或者用于标识重要的生命周期事件(比如初始化 或 关闭) 的发生

> 加锁可以保证可见性和原子性; volatile 变量只能保证可见性

只有满足了下面所有便准后,才能使用volatile变量

* 写入变量时并不依赖变量的当前值; 或者能够确保只有单一的线程修改变量的值
* 变量不需要与其他的状态变量共同参与不变约束
* 访问变量时,没有其他的原因去要加锁

## 发布和逸出

**发布(publishing)**一个对象的意思是使它等该被当前范围外的代码所使用. 

一个对象再尚未准备好时就将它发布,这种请款称为为**逸出(escape)**

### 安全构建的实践

this引用再构造时逸出. 发布的内部EventListener实例是一个封装的ThisEscape中的实例. 但是对象只有通过构造函数返回后, 才处于可预言的,稳定的状态, 所以从构造函数内部发布的对象, 只是一个未完成构造的对象. 甚至即使在构造函数的最后一行发布的引用也是如此. 如果this引用在构造过程中逸出,这样的对象会被观察为 **没有正确构建的(not properly constructed)**于是新的线程在所属对象完成构造前就看见它. 在构造函数中创建线程并没有错, 但是最好不要立即启动它. 取而代之的是, 发布一个start 或 initialize方法来启动对象拥有的线程. 在构造函数中调用一个可覆盖的(那些既不是private 也不是final的)实例方法同样会导致this引用在构造期间逸出.

如果想在构造函数中注册监听器或启动线程, 可以使用一个私有的构造函数和一个公共的工厂方法,这样避免了不正确的创建

> 不要让this引用在构造期间逸出

一个导致this引用在构造期间逸出的常见错误, 是在构造函数中启动一个线程. 当对象在构造函数中创建了一个线程时, 无论时显式的(通过将它传给构造函数) 还是隐式的(因为Thread 或 Runnable是所属对象的内部类), this引用几乎总是被新线程共享

> 更明确的说法是, 只要 this引用在构造函数完成前不会从线程中逸出. 只要构造函数结束前 没有其他线程使用this引用, this引用就可以通过构造函数存储到某处

## 线程封闭

访问共享的,可变的数据要求使用同步. 一个可以避免同步的方式就是不共享数据. 如果数据仅在单线程中被访问, 就不需要任何同步. 线程封闭(Thread confinement)技术是实现线程安全的最简单方法之一. 当对象封闭在一个线程中时, 这种做法会自动称为线程安全的, 及时被封闭的对象本身不是.

### Ad-hoc线程限制

**Ad-hoc线程限制** 是指维护线程限制性的任务全部落在实现上的这种情况. 因为没有可见性修饰符与本地变量等语言特性协助将对象限制在目标线程上, 所以这种方法非常容易出错. 

线程限制的一种特例是将它用于volatile变量. 只要你确保只通过但一县城写入共享的volatile变量,那么在这些volatile变量上执行"读-改-写"操作就是安全地. 这种情况下, 你就将修改操作限制在单一的线程中,从而组织了竞争条件. 并且, 可见性保证volatile变量能够确保其他线程能看到最新的值

### 栈限制

### ThreadLocal

一种维护线程限制的更加规范的方式是使用ThreadLocal,它允许你将每个线程与持有数据的对象关联起来.ThreadLocal提供了get/set 访问器, 为每个使用它的线程维护一份单独的拷贝. 所以get总是返回由 **当前执行线程**  通过set设置的最新值

ThreadLocal 变量通常用于防止在基于可变的单体(Singleton) 或全局变量的设计中, 出现(不正确的)共享.

线程首次调用`ThreadLocal.get()`方法时, 会请求`initialValue`提供一个初始值. 概念上 可以将ThreadLocal<T> 看作map<Thread,T> 它存储了与线程相关的值, 不过事实上它并非这样实现的. 玉仙城相关的值存储在线程对象自身中, 线程终止后, 这些值会被垃圾回收

## 不可变性

### Final域

final关键字来源于C++的const机制, 不过受到了更多的限制. 它对不可变性对象的创建提供了支持. final域值不能修改的(尽管如果final域指向的对象是可变的, 这个对象仍然可以被修改), 然而它在Java存储模型中还有着特殊的语义. final 域是的确保初始化安全性(initialization safety)成为可能,初始化安全性让不可变性对象不需要同步就能自由地被访问和共享.

### 不可变对象与初始化安全性

处于不可变对象的重要性, Java存储模型为共享不可变对象提供了特殊的初始化安全性的保证. 对象的引用对其他线程可见, 并不意味着对象的状态一定对消费线程可见. 为了保证对象状态有一个一致性视图, 我们需要同步

另一方面, 及时发布对象引用时没有使用同步, 不可变对象仍然可以被安全的访问. 为了获取这种初始化安全性的保证, 应该满足所有不可变性的条件: 不可修改的状态, 所有域都是final类型, 正确的构造.

### 安全发布的模式

如果一个对象不是不可变的, 他就必须被安全地发布, 通常发布线程与消费线程都必须同步化. 如何确保消费线程能够看到处于发布当时地对象状态; 我们要解决对象发布后对其修改地可见性

> 为了安全地发布对象,对象的引用以及对象地状态必须同时对其他线程可见. 一个正确创建的对象可以通过下列条件安全地发布:
>
> * 通过静态初始化器初始化对象地引用
> * 将它的引用存储到volatile域或AtomicReference
> * 将它的引用存储到正确创建地对象地final域中
> * 或者将它地引用存储到由锁正确保护地域中

线程安全的库中的容器提供了如下的线程安全保证:

* 置入 Hashtable,synchronizedMap,ConcurrentMap 中的主键以及键值, 会安全地发布到可以从Map获取他们的任意线程中,无论是直接获取还是通过迭代器(iterator)获取
* 置入Vector , CopyOnWriteArrayList , CopyOnWriteArraySet, SynchronizedList,SynchronizedSet 中的元素, 会安全的发布到可以从容器中获得他的任意线程
* 置入BlockingQueue 或者 ConcurrentLinkedQueue的元素, 会安全地发布到可以从队列中获得他的任意线程中



静态初始化器由JVM在类的初始阶段执行, 由于JVM内在的同步, 该机制保证了这种方式初始化的对象可以被安全的发布

### 高效不可变对象(Effectively immutable objects)

有些对象在发布后就不会被修改, 其他线程想要在内有额外同步的情况下安全地访问他们, 此时安全地发布至关重要. 所有的安全发布机制都能保证, 只要一个对象 "发布当时(as-published)"的状态对所有访问线程可见, 那么到它的引用也都是可见的. 如果"发布当时"的状态不会再改变, 那么确保任意访问是安全的,就变得重要.

> 任何线程都可以在没有额外的同步下安全地使用一个安全发布的高效不可变对象

### 可变对象

如果对象在创建后被修改, 那么安全发布仅仅可以保证 "发布当时"状态的可见性. 对于可变状态, 同步不仅仅由于对象发布, 而且还用于在每次对象被访问后, 保证后续变化的可见性. 为了保证安全地共享可变对象, 可变对象必须被i安全发布, 同时必须是线程安全的 或者是被锁保护的.

> 发布对象的必要条件依赖于对象的可变性
>
> * 不可变对象可以通过任意机制发布
> * 高效不可变对象必须要安全发布
> * 可变对象必须要安全发布, 同时必须要线程安全或被锁保护

### 安全地共享对象

