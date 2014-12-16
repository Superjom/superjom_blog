CPP Currency in Action note(3)  Sharing data between threads
===============================================================
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

*2014-12-13*

这一章节的内容会包括：

* 在线程间共享数据会出现的问题
* 使用互斥锁来保护数据
* 保护共享数据可行的措施

想象这样一个场景，如果你跟一个朋友共享一个公寓，这个公寓里只有一个浴室和一个厨房。 
当然，即使你跟你朋友关系非常好，也往往不会同时洗澡。

当你想要洗澡的时候，你的朋友在浴室里待了特别长的时间，这无疑让人沮丧。

相同的情况也会出现在多线程程序上，如果你想要在多线程上共享数据，那么你需要建立规则来确定那些线程能够接触那些数据，
并且对应的数据更新如何被传递给其他线程。 

在多线程中共享数据可能并不光是一种便利，如果不给正确使用的话，也可能出现bug。 

这个章节的就是关于C++多线程间安全共享数据的。
也就是最大化收益，同时避免隐含的问题。

Problems with sharing data between threads
--------------------------------------------
多线程共享数据的安全问题，完全是因为多线程修改共享数据出现的问题。
因为，如果所有线程都是只读数据，那么任何线程的存在都不会影响其他线程对数据的读取。
但是，当多线程共享数据，并且有线程开始修改共享数据，那么隐患就开始了。 
在这种情况下，你需要特别注意避免出现问题。

容易出问题的一个地方就是变量，比如用一个变量size来标记一个列表的长度。 
这样，在每次插入或者删除一个元素时，需要同时修改size变量。 
这就会出现问题，比如一个线程删除了一个元素，而另外一个线程恰好正在读取size变量。

共享数据出现最简单的问题就是类似上面的broken invariance.

Race conditions
*************************************
想象这样一个场景，你在电影院买电影票。 
大家在多个柜台排队，当然，每个座位只有一张票，那么谁能够买到最后一张票呢。 
这很像一个比赛。 

在协同中， 一个race condition就是任何最终结果依赖于线程运行操作顺序的场景。 
一般情况下，race并不不会出现问题，比如多线程同时向一个队列中插入数据，谁先谁后并没有太多考究的地方。 
但是，如果出现上面一节里面的broken invariance，那么谁先修改元素，谁后读取size这个就有考究了。 

如果操作占用的CPU时间比较少，那么race condition的问题一般比较难出现，而且难以重现。
但是如果操作的次数足够多，那么race condition就会表现出来。

如果你正在写多线程的程序，那么race conditions会成为你生涯的一个硬骨头。
写软件的很大的复杂之处就在于避免有问题的race conditions.

Avoiding problemic race condiitions
*************************************
有多种方式解决有问题的race conditions。 
最简单的方法是，在共享数据外面添加一层保护的程序，以保证在任何一个线程眼里，对数据的修改或许还没开始，或者已经完毕（类似元操作）。
C++ 标准库里提供了类似的机制，后面将会谈到。

另外一个方法是修改数据格式的设计，使得对共享数据的修改是不可分割的操作，也就是避免前面size变量那种broken invariance。
这个方法也称为无锁编程。

C++标准库提供的保护共享数据最基本的机制是mutex，下面将会介绍。

Protecting shared data with mutexes
*************************************
当你有一个类似之前那样有size变量的列表作为共享数据时，你可以用 `mutex` 将对应的操作包裹起来。 
在进入共享数据之前，你 `lock` 共享数据的 `mutex` ，当你结束操作之后，再 `unlock` 锁。 
线程库能够保证，当有一个线程成功 `lock` 之后，所有其他试图 `lock` 相同的 `mutex` 线程都会等待，直到这个 `mutex` 被 `unlock` 之后，
这就代表了，相同的时间内，只能有一个线程进行这段操作，从而也保证了其他线程永远不可能看到 `broken invariance`

`mutex` 是C++中最常规的共享数据的保护机制。
但它不是万能的； 
你需要确保自己保护到了该保护的数据，而且能够避免 `race conditions` 问题。 
`Mutex` 自身也有问题，比如死锁(这个在后面会谈到)，或者保护了过多或者过好的数据。


Using mutexes in C++
**********************
在 C++中，你通过 `std::mutex` 来创建一个 `mutex` ， 通过 `lock` 和 `unlock` 分别锁定和解锁 `std::mutex` 。 
当然，你需要记住，每一个 `lock` 必须对应一个 `unlock` 。
不推荐直接调用这两个操作，因为就像 `new` 对应 `delete` 一样，你可能忘记 `delete` 的扫尾工作，又或者中间出现异常，导致在扫尾工作之前提前退出，这些都是非常脆弱的。

C++标准库提供了一个类模板 `std::lock_guard` 。 
它在构造函数的时候 `lock` 特定的 `mutex` ，在析构函数的时候自动 `unlock` 特定的 `mutex` 。
所以它的作用域在一个scope内。

`std::mutex` 和 `std::lock_guard` 都声明在 `<mutex>` 头文件中。

.. code-block:: c++
    :linenos:

    #include <list>
    #include <mutex>
    #include <algorithm>

    std::list<int> some_list;
    std::mutex some_mutex;
    
    void add_to_list(int new_value)
    {
        std::lock_guard<std::mutex> guard(some_mutex);
        some_list.push_back(new_value);
    }

    bool list_contains(int value_to_find)
    {
        std::lock_guard<std::mutex> guard(some_mutex);
        return std::find(some_list.begin(), some_list.end(),
                value_to_find) != some_list.end();
    }

使用了 `lock_guard` 之后，线程就会互斥执行相关的代码。

但是，如果某个成员函数返回了被保护数据的指针或者引用，
那么即使在成员函数里使用了 `mutex` 就行了周密的保护也没有意义。 
因为你把被保护的共享数据又暴露到了危险之下。 
**任何代码，只要得到了共享数据的指针或者引用，都能够无保护地任意修改共享数据** 。

确保共享数据被有效保护，除了被约束的操作外没有其他访问共享数据的后门。

Structuring code for protecting shared data
************************************************
正如你所看到的，保护共享数据并不是仅仅将所有成员函数都加上 `std::lock_guard` 这么简单。 
只要传出特定的指针或者引用，那么之前的保护都形同虚设。 
在某种程度上，检测指针和引用是很容易的，只需要检查是否有成员函数返回了被保护数据的指针或者引用。

如果你再挖深一点，问题并没有那么简单。
你不光要检查，成员函数没有向它的调用者返回指针或者引用；
同时还要检查，成员函数没有将指针和引用传递给它们调用的函数。
这只是一个隐患：那些函数可能将指针或引用存储下来以便多次重用。
只要指针或者引用外泄，那么保护机制就不复存在。

比如下面这个例子

.. code-block:: c++
    :linenos:

    class some_data {
        int a;
        std::string b;
    public:
        void do_something();
    };

    class data_wrapper
    {
    private:
        some_data data;
        std::mutex m;

    public:
        template<typename Function>
        void process_data(Function func) 
        {
            // 将被保护的数据传递给用户提供的函数
            // 这是个隐患
            std::lock_guard<std::mutex> l(m); 
            func(data);
        }
    };

    some_data * unprotectected;

    void malicious_function(some_data &protected_data) {
        unprotected = &protected_data;
    }

    data_wrapper x;
    void foo()
    {
        // data_wrapper的成员函数中调用了 malicious_function
        x.process_data(malicious_function);
        unprocted->do_something();
    }

所以，上面的问题是难以避免的，C++标准库也无能为力，只能自己遵守一些原则:
**不要将被保护数据的指针和引用传递到锁所保护的scope之外，
包括不要从函数中返回指针引用，或者将它们存储到外部可见的内存中，将它们作为user-supplied函数的参数也不可取**

在下面一节，你会看到，即使用 `std::mutex` 保护了，也依旧有可能会出现 race conditions。

Spotting race conditions inherent in interfaces
**************************************************
有时候看起来，单个操作是安全的，但是也许全局看起来是有问题的。 
比如一个链表，如果只是单纯对单个节点用锁保护，但是链表的操作涉及到前后两个节点，那么尽管互斥操作，但是还是会出现race conditions.
这时候需要的是一个全局的锁对整个数据结构进行保护。

因为接管链表的单个操作是安全的，但是多个操作之间也可能出现race conditions.

比如，用 `std::stack` 作为例子，它只提供了5个接口：

* push() 添加元素
* pop() 从尾部去掉一个元素
* top() 从头部读取一个元素
* empty() 来返回stack是否为空
* size() 返回stack的元素个数

具体的接口如下：

.. code-block:: c++
    :linenos:

    template<typename T, typename Container=std::dequeue<T> >
    class stack
    {
    public:
        explicit stack(const Container&);
        explicit stack(Container&& = Container());
        ...

        bool empty() const;
        size_t size() const;
        T& top();
        T const& top() const;
        void push(T const &);
        void push(T&&);
        void pop();
        void swap(stack&&);
    };

这里的问题是， `empty()` 和 `size()` 的结果是不可依赖的。
也许在单线程的时候，这些结果并不会出现问题，但是，在多线程的时候，也许单个调用是安全的，
但是，由于多个操作之间的相对顺序，依旧会出现race conditions.
比如，当stack中只有一个元素时，一个线程A在调用 `empty()` 的时候，另外一个线程B恰好从stack中pop出了一个元素。 
此时A得到的信息是stack非空，于是也试图pop出一个元素，于是就发生了一个异常。

下面是这段操作的代码，AB线程都在运行这段代码
.. code-block:: c++
    :linenos:

    stack<int> s;
    if(!s.empty())
    {
        int const value = s.top();
        s.pop();
        do_something(value);
    }

对应的race condition的运行顺序如下:

.. code-block:: c++
    :linenos:

    // Thread A                                 Thread B
    if(!s.empty())
                                                if(!s.empty())
        int const value = s.top();
                                                    int const value = s.top();
        s.pop();
        do_something(value);                        s.pop();    // 如果stack中只有1个元素，异常!
                                                    do_something(value);

可以看到，标准库里将 `pop()` 和 `top()` 拆分开来，而不是 `pop` 直接得到元素的设计是导致race conditions最根本的问题。
当然，拆分开始是有一些道理的，比如先 `top()` ，在复制操作出现异常，那么元素在 `pop()` 之前还存在于 `stack` 中。
当然，对于接口的设计问题，有一些可行的方法来避免出现上面的问题。
当然，是需要一些妥协的。

选项1 传入一个参数
++++++++++++++++++++++++++++++++
第一种方法是用引用的方式传入一个参数，然后通过修改该参数的方式传回获取的元素。

.. code-block:: c++
    :linenos:

    std::vector<int> result;
    some_stack.pop(result);

这个方法是可行的，但是会有一些弊端。

* 它需要在pop之前，先创建一个目标类型的临时对象，这个在时间和空间上代价比较高。
* 对于一些类型，构建一个空的对象可能不被构造函数支持，比如一些需要传参的构造函数。
* 赋值操作可能不被支持，比如很多自定义函数都没有赋值操作或者拷贝构造函数

选项2 要求一个不产生异常的拷贝构造函数或者move构造函数
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++

选项3 返回一个呗pop元素的指针
+++++++++++++++++++++++++++++
返回指针，可以避免发生异常。 
而且复制指针的代价比复制比较大的类型的代价会小很多。

但是一些弊端就是：

* 指针，也就意味着需要额外的内存管理的机制
* 对于一些基本类型，比如int，复制指针以及访问内存管理的代价比直接赋值的代价要高很多

对于任何一种使用指针作为接口的场景， `std::shared_ptr` 都会是一个理想的选择。
它的好处是

* 通过引用计数，自动管理内存，防止出现内存泄露。 
* 对内存操作完整的支持，完全不必调用 `new` 和 `delete` ，后者会出现各种问题

选项4 同时提供选项1 和 选项2或3中的一个
++++++++++++++++++++++++++++++++++++++++
如果你选择了选项2或者3，那么选项1也比较容易了。 
对于通用的代码，多个选项可以有更多的灵活性。

一个线程安全的stack
+++++++++++++++++++++
下面展示了一个利用选项1和3实现的无 race conditions问题的stack：

.. code-block:: c++
    :linenos:

    #include <exception>
    #include <memory>

    struct empty_stack: std::exception
    {
        const char* what() const throw();
    }

    template<typename T>
    class threadsafe_stack
    {
    public:
        threadsafe_stack();
        threadsafe_stack(const threadsafe_stack&);
        threadsafe_stack& operator= (const threadsafe_stack&) = delete;

        void push(T new_value);
        std::shared_ptr<T> pop();
        void pop(T& value);
        bool empty() const;
    };

可以看到，现在接口只剩下四个， `swap` 操作不是必须的。 
同时，即使不用 `empty()` ， `pop` 出现问题也会抛出 `empty_stack` 异常。
所以，最终的接口可以缩减到三个。

下面是具体的实现：

.. code-block:: c++
    :linenos:

    #include <exception>
    #include <memory>
    #include <mutex>
    #include <stack>

    struct empty_stack : std::exception
    {
        const char* what() const throw();
    }

    template<typename T>
    class threadsafe_stack
    {
    private:
        std::stack<T> data;
        mutable std::mutex m;

    public:
        threadsafe_stack() { }
        threadsafe_stack(const threadsafe_stack& other) 
        {
            std::lock_guard<std::mutex> lock(other.m);
            data = other.data;
        }
        threadsafe_stack& operator= (const threadsafe_stack&) = delete;

        void push(T new_value) 
        {
            std::lock_guard<std::mutex> lock(m);
            data.push(new_value);
        }
        std::shared_ptr<T> pop()
        {
            std::lock_guard<std::mutex> lock(m);
            if(data.empty()) throw empty_stack();
            std::shared_ptr<T> const res(std::make_shared<T>(data.top()));
            data.pop();
            return res;
        }
        void pop(T& value) 
        {
            std::lock_guard<std::mutex> lock(m);
            if (data.empty()) throw empty_stack();
            value = data.top();
            data.pop();
        }
        bool empty() const 
        {
            std::lock_guard<std::mutex> lock(m);
            return data.empty();
        }
    };

请注意上面的拷贝构造函数，其中在复制时，调用了被拷贝方的mutex进行保护。 
从上面讨论的stack的top和pop两个函数的例子来看，race conditions问题出现的原因是，操作太细化了，
没有考虑一个比较大的操作全局的保护。
当然，另外一个极端：将很大一块代码用同一个锁来保护，比如一个系统中有很多的共享数据，
将这些共享数据用同一个锁来保护，这个会带来严重的性能问题，因为尽管是多线程系统，
但是锁强制它们以单线程的方式互斥运行，这个根本失去了多线程并行的效果。

第一个版本的Linux采用了全局的内核锁，所以，尽管能够支持多线程，但是多核CPU中，一次只能允许一个进程运行。
也就是，多核CPU不会比单核CPU的性能好，由于切换的代价，甚至更差。
后面版本改进了内核锁，使得能够利用多核CPU。

锁机制带来的一个问题是，有的时候，在同一个操作中，你可能需要用到多个锁。
就像之前说的那样，扩大锁保护代码的粒度是有效的，这样只需要一个锁就行了。
但是，有时必须多个锁，比如当锁需要保护类的多个实例的时候。

当你用多个锁来保护一个操作的时候，另外一个问题有显现了出来： `deadlock` .

死锁大概是race conditions的反面：两个线程间不是比赛谁先，而是互相等待对方，这样两者都停止运行了。

Deadlock: the problem and a solution
++++++++++++++++++++++++++++++++++++++++
说到死锁，假设你有一个玩具，由两个部分构成：一个小熊，一个小鼓——就是一个普通的小熊打鼓的玩具。
要想玩这个玩具，需要凑齐两个部分才行。
恰好你有两个一样大的小孩（女孩），两个人各拿着其中一部分，都等着对方交出另外一部分。
于是，两个人僵持着，谁都玩不了这个玩具。 这个就是死锁的一种现象了。

孩子争玩具，而多线程争锁：
每个线程需要锁定多个锁之后才能运行，每个都占用了其中一个锁，在等待其他线程解除另外的锁。
于是，多个线程在僵持中，都不能继续运行。

对于避免死锁，一个通常的建议是，以一定的顺序锁定mutex。 
比如，先锁定锁A，然后是B，那就不可能出现死锁了。 
这个听起来很简单，但是在一些情况中，这并不容易。

比如这样一个例子，对一个类的两个实例进行 `swap` ，每个实例采用自己的mutex就行保护。
为了能够同时保护两个实例，因此 `swap` 函数需要同时锁定两个实例的mutex。
现在如果有两个线程，一个执行 `swap(A, B)` ，另外一个执行 `swap(B, A)` ，那么两个mutex被锁定的顺序都不同了。
如果两个线程恰好执行到同一个语句，那就很可能出现死锁了。

谢天谢地，C++标准库对这种情况提供了解决方法。 
`std::lock` 这个函数能够同时锁定多个mutex，这样不需要纠结顺序，也不会出现死锁了。

.. code-block:: c++
    :linenos:

    class some_big_object;
    void swap(some_big_object& a, some_big_object& b);

    class X
    {
    private:
        some_big_object some_detail;
        std::mutex m;
    public:
        X(some_big_object const& sd): some_detail(sd) { }

        friend void swap(X& a, X& b) 
        {
            if(&a == &b) return;
            std::lock(a.m, b.m);    // 同时锁定两者的mutex
            // std::adopt_lock 是告诉lock_guard，mutex已经被锁定 
            // 只需要做后面的unlock的工作
            std::lock_guard<std::mutex> lock_a(a.m, std::adopt_lock);
            std::lock_guard<std::mutex> lock_b(b.m, std::adopt_lock);
            swap(a.some_detail, b.some_detail);
        }
    };

代码将a和b的mutex同时锁定，并在之后，用两个 `std::lock_guard` 来自动 unlock a和b的mutex。

尽管 `std::lock` 能够帮助你避免deadlock，但是，它需要多个锁被同时锁定。
但是，如果mutex需要被分开锁定，那么它就无能为力了。

Further guidelines for avoiding deadlock
++++++++++++++++++++++++++++++++++++++++++++++
死锁的存在并不一定由于mutex，也可能是其他的一些情况。

死锁的最终表现都是互相等待。

Avoid Nested Locks
``````````````````````
当你持有了一个lock，然后调用了另外一个线程，并等待它结束。
如果这个线程在等待你已有的lock呢？那么你和子线程就陷入了死锁当中，这就是嵌套锁的隐患。

如果你需要锁定多个mutex，就使用 `std::lock` 来处理多个lock的死锁问题。

Void calling user-supplied code while holding a lock
```````````````````````````````````````````````````````
对于用户自定义的函数，你无法知道它会干什么。
也许它会嵌套调用锁，从而违反第一个原则。

Acquire locks in a fixed order
``````````````````````````````````
如果你必须要多个锁，而且这些锁不能用 `std::lock` 来同时锁定。
那么，最好的就是，在每个线程里面锁定锁的顺序完全一致。
这个其实做起来其实是比较简单的。

比如，要保护一个链表，那么你需要为每个节点准备一个mutex，
当修改某个节点时，获取对应的mutex便可。
一些比较复杂的操作，比如删除操作，那么你可能需要三个节点的mutex：
需要被删除的目标节点以及其左右两个节点。
这个时候，就需要保持顺序的一致了。 
如果三个节点ABC，要删除中间B节点。 
如果你以顺序 BAC的顺序获取node，那就有一些问题了。
比如，在删除下个节点C的时候，顺序是CBD，那么BC的锁定顺序不同就显现出来了。
此时如果有两个线程同时运行，分别删除B和C节点，那么死锁就可能出现了。

Use a lock hierarchy
``````````````````````
尽管这是一个规定锁顺序的特殊的例子，但是一个层次锁能够提供运行时锁顺序的检测。
方法是，将你的应用分成层次，然后规定每一层中可以锁定的mutex。 
当代码试图去锁定一个mutex时，如果它已经有了更低层的mutex，那么它就会被禁止继续锁定当前锁。
你可以通过为每个mutex指定一个层次号，然后记录每个线程锁定的mutex来动态监测层次锁被使用的情况。

下面是一个例子：

.. code-block:: c++
    :linenos:

    hierarchical_mutex high_level_mutex(10000);
    hierarchical_mutex low_level_mutex(5000);

    int do_low_level_stuff();
    
    int low_level_func()
    {
        std::lock_guard<hierarchical_mutex> lk(low_level_mutex);
        return do_low_level_stuff(low_level_func());
    }

    void high_level_stuff(int some_param);

    void high_level_func()
    {
        std::lock_guard<hierarchical_mutex> lk(high_level_mutex);
        high_level_stuff(low_level_func());
    }

    void thread_a()
    {
        high_level_func();
    }

    hierarchical_mutex other_mutex(100);
    void do_other_stuff();

    void other_stuff()
    {
        high_level_func();
        do_other_stuff();
    }

    void thread_b()
    {
        // 锁定了一个层次号为100的mutex
        std::lock_gurad<hierarchical_mutex> lk(other_mutex);
        other_stuff();  // wrong: 需要锁定10000的mutex，但是已经有低层次的mutex，不被允许！
    }

这个例子还展示了 `std::lock_guard` 对用户自定义锁的支持， hierarchical_mutex是用户自定义的，但是也能很好地支持，因为hierarchical_mutex中实现了 `std::lock_guard` 需要的三个函数： `lock()` `unlock()` `try_lock()` ，下面是一个hierarchical_mutex具体的实现：

.. code-block:: c++
    :linenos:

    class hierarchical_mutex
    {
        std::mutex internal_mutex;
        unsigned long const hierarchy_value;
        unsigned long previous_hierarchy_value;
        static thread_local unsigned long this_thread_hierarchy_value;

        void check_for_hierarchy_voilation()
        {
            if(this_thread_hierarchy_value <= hierarchy_value)
            {
                throw std::logic_error("mutex hierarchy violated");
            }
        }
        void update_hierarchy_value()
        {
            previous_hierarchy_value = this_thread_hierarchy_value;
            this_thread_hierarchy_value = hierarchy_value;
        }
    public:
        explicit hierarchical_mutex(unsigned long value):
            hierarchy_value(value),
            previous_hierarchy_value(0)
        {}
        void lock()
        {
            check_for_hierarchy_voilation();
            internal_mutex.lock();
            update_hierarchy_value();
        }
        void unlock()
        {
            this_thread_hierarchy_value = previous_hierarchy_value;
            internal_mutex.unlock();
        }
        bool try_lock()
        {
            check_for_hierarchy_voilation();
            if(!internal_mutex.try_lock())
                return false;
            update_hierarchy_value();
            return true;
        }
    };
    thread_lock unsigned long
        hierarchical_mutex::this_thread_hierarchy_value(ULONG_MAX);

其中的核心就是用 `thread_local` 的值来表示当前线程的层次值：
`this_thread_hierarchy_value` 被初始化为最大值，这样，任何mutex都能够被允许锁定。
因为这个值被声明为 `thread_local` ，每个线程都会有一个自己的拷贝，所以每个线程中这个值的状态是与其他线程完全无关的。
此时，如果一个线程第一次锁定一个mutex，那么此mutex的层次号将会赋值到this_thread_hierarchy_value，
如果此线程继续尝试锁定第二个mutex，那么此时会有检测：this_thread_hierarchy_value是否比此mutex的层次号大，大则可以锁定，并且将this_thread_hierarchy_value赋值给previous_hierarchy_value，将当前锁定的mutex的层次号赋值给this_thread_hierarchy_value。
否则，抛出异常。

`try_lock()` 的工作机制和 `lock()` 相似，只是如果不被允许锁定一个锁，只会默默返回，而不会抛出异常。

尽管是一个运行时的检测，但是花的时间是很少的。 而且，你可以用它来避免死锁。
这个设计模式比较有效，如果你采用这种方式对应用分层，那么从最初的设计阶段就能够避免很多死锁了。 
即使最终没有落实成代码就行运行时检测，也是有帮助的。

Extending these guidelines beyond locks
``````````````````````````````````````````
就像最开始的时候说的那样，死锁的产生并不一定只是因为mutex，只要出现循环等待的情况就是死锁。
所以，有必要将上面基于mutex的原则推广开来。
比如，就像你应该避免嵌套锁一样，你应该避免在锁定了一个mutex后，去等待一个线程退出，因为那个线程也许正在等待锁定你之前锁定的mutex。
类似地，如果你在等待一个线程执行完，那么将线程层次化，比如，一个线程只等待比它层次低的线程退出。
一个简单的方法就是，如果你在一个函数中启动了多个线程，那么保证这些线程在这个函数中join。

一旦你设计了代码来避免死锁， `std::lock()` 和 `std::lock_guard` 能够解决大部分简单的锁定工作，但是有些时候需要更加灵活的方法。
`std::unque_lock` 提供了比 `std::lock_guard` 更丰富的控制。

Flexible locking with std::unique_lock
```````````````````````````````````````````
`std::unique_lock` 将mutex变成一个可以被 `std::move` 的变量，使得mutex可以被传递。
因此其提供了比 `std::lock_guard` 更大的灵活性。
比如，不同于 `std::lock_guard` 实际占有某个mutex，当mutex被作为 `std::unique_lock` 进行传递时，当前拥有 `std::unique_lock` 实例的代码并不一定实际拥有某个mutex。 

当然，这些信息需要被保存在 `std::unique_lock` 实例中，这个会占用一些空间和时间。

之前用 `std::lock_guard` 实现的代码也完全可以用 `std::unique_lock` 实现，而且代码行数差不多：

.. code-block:: c++
    :linenos:

    class some_big_object;
    void swap(some_big_object& lhs, some_big_object& rhs);
    class X
    {
    private:
        some_big_object some_detail;
        std::mutex m;
    public:
        X(some_big_object const& sd): some_detail(sd) { }

        friend void swap(X& lhs, X& rhs)
        {
            if(&lhs == &rhs)
                return;
            // 和 lock_guard一样，传入 std::defer_lock 表示在构造函数中不要锁定
            // 留到后面锁定
            std::unique_lock<std::mutex> lock_a(lhs.m, std::defer_lock);
            std::unique_lock<std::mutex> lock_b(rhs.m, std::defer_lock);
            // 使用 std::lock同时锁定多个mutex
            // 避免顺序造成的死锁问题
            std::lock(lock_a, lock_b);
            swap(lhs.some_detail, rhs.some_detail);
        }
    };

`std::unique_lock` 能够被传递给 `std::lock()` ，因为它也支持必须的三个操作： `lock(), try_lock(), unlock()` 。

前面讲到， `std::unique_lock` 实例并不一定实际拥有一个mutex，如果不拥有mutex，那么这个 `std::unique_lock` 需要有一个标记来告知析构函数不要去 `unlock` 指向的mutex，否则，需要在析构函数中 `unlock` 拥有的mutex。
由于这个标记的存在，使得 `std::unique_lock` 会比 `std::lock_guard` 占用更多的空间和时间。 
但是 `std::unique_lock` 的在实际当中的应用还是很有意义的，因为它能够提供更大的自由度。

另外的情况下，比如你需要传递mutex的所有权， `std::unique_lock` 也会比较实用。

Transferring mutex ownership between scopes
```````````````````````````````````````````````
`std::unique_lock` 可以通过被 `std::move` 来传递mutex的所有权。

一种可能的用途是允许一个函数锁定一个mutex，然后将mutex的所有权传给调用方，使得调用方能够在同一个lock的保护下，执行额外的操作。
下面的代码演示了这样的场景， `get_lock()` 锁定了mutex，然后在返回mutex前准备数据。

.. code-block:: c++
    :linenos:

    std::unque_lock<std::mutex> get_lock()
    {
        extern std::mutex some_mutex;
        std::unique_lock<std::mutex> lk(some_mutex);
        prepare_data();
        return lk;
    }

    void process_data()
    {
        std::unique_lock<std::mutex> lk(get_lock());
        do_something();
    }

因为lk是函数中生成的自动变量，因此会被直接返回；编译器会自动执行move构造函数，而不需要显式调用 `std::move` 。
`get_lock` 中锁定了mutex，进行 `prepare_data` ，之后将拥有mutex的所有权直接传递给 `process_data` ，后者将mutex用 `std::unique_lock` 进行管理，并继续在mutex的保护下，完成 `do_something` 操作。
在这两个操作期间，是没有其他线程能够干涉其运行的。

这个模式适用于，mutex需要依赖当前程序运行的状态决定是否锁定，
或者，当成一个参数传入一个返回 `std::unique_lock` 对象的函数。
这样的一个使用实例是，程序并不会直接返回锁，而是返回一个协议类对象，
来保证对某些共享数据的加锁保护。
在这种情况下，任何需要访问共享数据的程序都必须通过这个协议类的对象。
当你结束访问了，则销毁这个对象，那么锁被打开，其他线程就能够访问共享数据了。
这样一个协议类的对象最好是支持move操作的（这样，它能够被函数返回），
同时类中的锁成员元素也需要支持move.

`std::unique_lock` 的灵活性也体现在，允许实例在它们被destroy之前，通过 `unlock` 操作放弃他们的锁。
`std::unique_lock` 支持 `lock, unlock, try_lock` 等基本操作，也使得它能够支持 `std::lock` 。
在 `std::unque_lock` 对象销毁前unlock一个锁意味着，你能够在锁不需要的时候自己手工解锁，这个会很大地提高性能。

Locking at an appropriate granularity
````````````````````````````````````````````
锁的粒度就是mutex保护的数据的大小。

一个粒度适当的锁会保护少量的数据，而一个粒度粗略的锁会保护大量数据。
不光选择锁的粒度来保护需要保护的数据很重要，而且需要确保只在需要保护的操作上用锁。

比如，在超市购物时，选购商品是可以并行的，而结账是需要排队的。
如果在选购商品上用一个全局锁，让大家排队购物，那么这个规定真心不爽。

如果多个线程在等待操作同一个资源，那么任何一个线程，如果持有锁期间做了一些不必要的操作，那么总体上多线程花费的时间会增长。

所以，只在真正需要锁mutex的时候锁定；尝试将任何数据处理的工作在锁外进行。
尤其需要注意的是，当持有锁的时候，不要做耗时较多的操作，比如I/O。
I/O操作要比内存中读写要慢至少上百倍，同样的时间应该允许其他线程做更多计算的工作。
所以，除非锁真的是用于保护文件的操作，否则不要在I/O期间持有锁，并尽量将其他类似耗时的操作移出锁保护的范围，留给多线程并发计算尽可能多的机会。

`std::unique_lock` 能够很好地处理上面的情况，因为当你不再需要锁的时候，可以手动 `unlock()` ，当你在后面的代码中又需要锁定了，那么 `lock()` 之，非常灵活：

.. code-block:: c++
    :linenos:

    void get_and_process_data()
    {
        std::unque_lock<std::mutex> my_lock(the_mutex);
        some_class data_to_process = get_next_data_chunk();
        my_lock.unlock();   // 下面处理数据阶段不需要锁定
        result_type result = process(data_to_process);
        my_lock.lock();     // 下面需要锁定来写入结果
        write_result(data_to_process, result);
    }

你不需要在整个 `process()` 期间都锁定mutex，这个跟 `lock_guard` 比较起来就有优势了。

如果你可以只用一个mutex来保护整个数据结构，
那么不光对于锁的竞争会增多，而且，压缩被锁定的时间也变得很难。
更多的操作需要等待同一个锁，所以锁被锁定的相对时间会增长，这个也更加凸现了锁粒度选择的重要性。

就像例子里展现的，所谓的适当的锁粒度，表示的不光是被锁保护的数据的大小，而且锁定期间所就行操作的耗时大小也是重要的考量。
** 总的来说，一个锁在就行具体操作时，需要尽可能压缩被锁定的时间** 。
这也就是说，除了必须，耗时的工作，比如I/O操作，或者获取另外一个锁（即使不会死锁）也不要在锁定期间就行。

之前有一个swap的例子，需要同时锁定两个数据结构的mutex。
然后再进行比较。 
如果此时需要比较的数据仅仅是两个int呢？ 复制int的代价是很低的。
你可以在持有两个锁的时候，简单地复制下两个值，然后释放锁，进行后续的比较操作。
这也就意味着，你缩减了持有锁的时间。

下面是一个简单的实现：

**Listing 3.10 Locking one mutex at a time in a comparison operator**

.. code-block:: c++
    :linenos:

    class Y
    {
    private:
        int some_detail;
        mutable std::mutex m;

        int get_detail() const
        {
            std::lock_guard<std::mutex> lock_a(m);
            return some_detail;
        }
    public:
        Y(int sd) : some_detail(sd) { }

        friend bool operator== (Y const& lhs, Y const& rhs)
        {
            if(&lhs == &rhs)
                return true;
            int const lhs_value = lhs.get_detail();
            int const rhs_value = rhs.get_detail();
            return lhs_value == rhs_value;
        }
    };

上面的代码首先分别持有锁复制了两个值（两个锁不需要同时锁定，有更好的异步性），然后在无锁的状态下就行比较然后返回结果。
这个改变在类型复制代价小的情况下能够缩减锁定时间，但是弊端是，它彻底改变了comparison的语义。

现在的两个锁是先后锁定，而且锁定操作并没有持续到整个比较操作的整个过程。
这样结果仅仅是两个值在不同时间的比较，也许在复制操作后，原始值被修改，但是后续的操作依旧采用旧的值就行比较。 整个操作的意义与原始的方法相比，发生了彻底的改变。

所以，当就行这些优化措施时，需要确保不修改原始操作的语义。

**如果你不在操作的整个过程锁定，那么持续就会暴露到race conditions的危险当中**

Alternative facilities for protecting shared data
++++++++++++++++++++++++++++++++++++++++++++++++++++
尽管mutex是最通用的机制，但对于保护共享数据，它并不是唯一的选择。

对于具体的情况，需要具体分析。

Protecting shared data during initialization
```````````````````````````````````````````````
假设，你有一个共享数据，但是它在初始化的时候耗时巨多，也许是一个数据库连接，或者一些内存分配。
`Lazy initialization` 就像在单线程代码中普遍使用的——每个访问此资源的操作都会检测下，资源是否已经被初始化了。

.. code-block:: c++
    :linenos:

    std::shared_ptr<some_resource> resource_ptr;
    void foo()
    {
        if(!resource_ptr)
        {
            resource_ptr.reset(new some_resource);
        }
        resource_ptr->do_something();
    }

如果共享数据的并行访问是安全的，那么剩下需要多线程保护的部分就是初始化了。
将上面的代码直接转化成多线程，会使得并行的访问被互斥化，整体的访问就变成序列化而不是并行访问了。
这是因为，每个线程在检测资源是否已经被初始化时，需要锁定一个mutex。

**Listing 3.11 Thread-safe lazy initialization using a mutex**

.. code-block:: c++
    :linenos:

    std::shared_ptr<some_resource> resource_ptr;
    std::mutex resource_mutex;
    void foo()
    {
        std::unique_lock<std::mutex> lk(resource_mutex);
        if(!resource_ptr)
        {
            resource_ptr.reset(new some_resource);
        }
        lk.unlock();
        resource_ptr->do_something();
    }

这个代码够简单，但是因为互斥检测，存在严重的性能问题。
为此，后来人提出了声名狼藉的 `Double-Checked Locking` 模式：
像下面的代码里一样，先无锁读取指针，只当指针是 `NULL` 的时候锁定。
指针在锁定lock之后，还会继续就行一次检测(这也就是所谓Double-Checked)以防止另外的线程在第一次检测和锁定期间已经就行了初始化。

.. code-block:: c++
    :linenos:

    void undefined_behaviour_with_double_checked_locking()
    {
        if(!resource_ptr)
        {
            std::lock_guard<std::mutex> lk(resource_mutex);
            if(!resource_ptr)
            {
                resource_ptr.reset(new some_resource);
            }
        }
        resource_ptr->do_something();
    }

不幸的是，这个模式是万恶的，因为它有潜在的race conditions问题。
因为，一个线程在锁外的读操作和另外一个线程在锁内的写操作(初始化)并没有方法同步。
这也就在指针及其指向对象之间产生了一个race conditions.

即使一个线程看到了 `resource_ptr` 被其他线程初始化（过去式还是进行时是个问题），此线程会直接进行 `resource_ptr->do_something()` ，而不管初始化的工作是否已经完毕。
再啰嗦一下，
如果这个过程中，初始化工作正在进行，那么其余的线程会以为初始化工作已经完成（ `!resource_ptr` 为 true），从而读取了未完全初始化的共享数据。
这是个未定义的行为。

C++标准库也认为这是个重要的场景，所以提供了 `std::once_flag` 和 `std::call_once` 来处理这个场景。
不同于锁定一个锁，然后检测指针，每个线程都可以调用 `std::call_once` ，并且会安全地知道目标指针是否已经被其他线程成功初始化了。
所以，应该在实际中类似的场景使用。

下面是对之前的代码用 `std::call_once` 的重写：

.. code-block:: c++
    :linenos:

    std::shared_ptr<some_resource> resource_ptr;
    std::once_flag resource_flag;

    void init_resource()
    {
        resource_ptr.reset(new some_resource);
    }

    void foo()
    {
        std::call_once(resource_flag, init_resource);   // init_resource 只会被运行一次
        resource_ptr->do_something();
    }

在这个例子里， `std::once_flag` 和需要被初始化的共享数据都是一段范围内的命名对象。 
但是 `std::call_once()` 也可以被用于类成员，如下所示：

**Listing 3.12 Thread-safe lazy initialization of a class member using std::call_once**

.. code-block:: c++
    :linenos:

    class X
    {
    private:
        connection_info connection_details;
        connection_handle connection;
        std::once_flag connection_init_flag;

        void open_connection()
        {
            connection  = connection_manager.open(connection_details);
        }
    public:
        X(connection_info const& connection_details_):
            connection_details(connection_details_)
        {}
        void send_data(data_packet const& data)
        {
            // 类似 std::bind的构造函数，this指针需要传入
            std::call_once(connection_init_lag, &X::open_connection, this);
            connection.send_data(data);
        }
        data_packet receive_data()
        {
            std::call_once(connection_init_flag, &X::open_connection, this);
            return connection.receive_data();
        }
    };

值得注意的是，和 `std::mutex` 类似， `std::once_flag` 是不能被复制和move的。

有潜在race conditions问题的初始化操作的场景如下，
有一个被声明为 `static` 的局部变量。 
对这个变量的初始化被定义在它首次声明出现的地方；
对于多线程调用这个函数，这也意味着，有race conditions出现的隐患。
在C++11之前的C++编译器在实际中都是有问题的，因为有可能不止一个线程认为它自己是最早运行这段代码，而且有义务进行初始化。
在C++11中，这个问题被解决了： 
初始化过程只能在一个线程中进行，而且其他线程只能在初始化工作完毕后才能继续往后执行。
这样，就没有race conditions隐患了。

在实际应用中，当需要一个全局的实例时，  `static` 模式可以作为 `std::call_once` 的替代。

下面的代码是没有问题的：

.. code-block:: c++
    :linenos:

    class my_class;
    my_class& get_my_class_instance()
    {
        static my_class instance;   // 这里static对象的初始化是线程安全的
        return instance;
    }

Protecting rarely updated data structures
``````````````````````````````````````````````
考虑一个DNS表，表中的记录可能好几年都不会有变化。
但是，为了保持记录的有效性，记录还是要定期检查的，
比较，偶尔还是可能有一些变化。

尽管少有更新，但是仍旧有可能发生。
而且，如果多个线程访问缓存，那么还是需要数据保护来防止线程访问到过期的数据。

没有专用的数据结构来支撑这个需求，
更新的时候需要就行更新操作的线程对数据结构独占操作。
一旦修改完毕，多线程又可以安全地并行地读取数据结构了。
使用一个 `std::mutex` 来保护数据结构是有一些问题的，因为它会消除数据结构读取时的并行性。
这里需要一种新的mutex，称作 `reader-writer` mutex，因为它允许两种不同的使用：
writer间互斥，reader间并行。

C++标准库并没有直接提供这样一个mutex。 但你可以使用 `boost::shared_mutex` 。 
对于更新操作， `std::lock_guard<boost::shared_mutex>` 和 `std::unique_lock<boost::shared_mutex>` 都可以被用作锁。
那些不需要更新数据结构的线程可以使用 `boost::shared_lock<boost::shared_mutex>` 来获取共享的访问。
这个和 `std::unque_lock` 的使用方法相似，只是同事可以有多个线程共享锁定同一个 `boost::shared_mutex` 。 
唯一的约束就是，如果任何一个线程有一个共享的锁，任何尝试获取独占锁的线程都会被阻塞，直到所有其他线程都解锁了，类似地，如果任何一个线程有一个独占锁，任何其他的线程均无法获取共享锁或者独占锁，直到那个线程释放了锁。

下面的例子给定了DNS缓存的实现，采用 `std::map` 来保存缓存，用 `boost::shared_mutex` 来保护共享数据。

**Listing 3.13 Protecting a data structure with a boost::shared_mutex**

.. code-block:: c++
    :linenos:

    #include <map>
    #include <string>
    #include <mutex>
    #include <boost/thread/shared_mutex.hpp>

    class dns_entry;

    class dns_cache
    {
        std::map<std::string, dns_entry> entries;
        mutable boost::shared_mutex entry_mutex;
    public:
        dns_entry find_entry(std::string const& domain) const
        {
            // 共享锁
            // 多个reader可以并行运行
            boost::shared_lock<boost::shared_mutex> lk(entry_mutex);
            std::map<std::string,dns_entry>::const_iterator const it =
                entries.find(domain);
            return (it == entries.end()) ? dns_entry() : it->second;
        }
        void update_or_add_entry(std::string const& domain,
                    dns_entry const& dns_details)
        {
            // 互斥/独占锁 只能互斥运行
            std::lock_guard<boost::shared_mutex> lk(entry_mutex);
            entries[domain] = dns_details;
        }
    };


References
-----------
CPP Currency in Action 第三章


.. raw:: html

    <!-- 多说评论框 start -->
    <div class="ds-thread" data-thread-key="cpp-concurrency3.rst" data-title="CPP Currency in Action note(3)  Sharing data between threads" data-url="http://superjom.duapp.com/program-language/cpp-concurrency3.html"></div>
    <!-- 多说评论框 end -->
    <!-- 多说公共JS代码 start (一个网页只需插入一次) -->
    <script type="text/javascript">
    var duoshuoQuery = {short_name:"superjom"};
    (function() {
            var ds = document.createElement('script');
                    ds.type = 'text/javascript';ds.async = true;
                            ds.src = (document.location.protocol == 'https:' ? 'https:' : 'http:') + '//static.duoshuo.com/embed.unstable.js';
                                    ds.charset = 'UTF-8';
                                            (document.getElementsByTagName('head')[0] 
                                                     || document.getElementsByTagName('body')[0]).appendChild(ds);
                                                })();
    </script>
    <!-- 多说公共JS代码 end -->

