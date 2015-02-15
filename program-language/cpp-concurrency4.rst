CPP Currency in Action note(4) Synchronizing concurrnet operation
********************************************************************
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

*2014-12-17*

这个章节包括内容：

* 等待一个事件
* 等待未来的一次性事件
* 加时间约束的等待
* 使用同步操作来简化代码

在上一章，我们看到了很多保护共享数据的方法。 但是，有时你不光需要保护数据，而且要协同不同线程中的操作。
比如，一个线程需要等待其他线程先完成一个任务之后再运行，或者需要等待一个事件或者条件。
C++标准库中提供了线程协同的工具，比如 `condition variables` 和 `futures` 。

在这个章节，我们将会讨论如何利用 `condition variables` 和 `futures` 来等待事件，或者如何使用他们来简化协同操作。

Waiting for an event or other condition
=============================================
想象你夜里坐一辆火车，不想坐过站。
一种方式是整夜醒着，并且留意火车到哪儿了，但是你会很累。
当然，你也可以查看下列车时刻表，查下列车到站时间，定下闹钟。
这样，你就不会错过站了，但是如果火车晚点了，那你就会在到站前提前醒了。
又或者，你的闹钟没电了，然后你会睡过头。 
最理想的方式就是，你睡觉就是，快到站的时候列车员来叫你（国内卧铺已经提供这种先进的服务了）。

那么，类比到多线程持续，如果要让一个线程在等待另外一个线程完成某个任务，同样有几种选项。
第一种方法是，让执行任务的那个线程设定一个标识，当它完成那个任务时，就修改标记，而另外一个线程不断查看那个标记。
这种方法是非常耗费资源的，持续检测标记会耗费CPU时间，此外，当mutex被等待线程中的一个锁定时，其他等候的线程都不能锁定。
这种方法就像前面的，整夜醒着等待火车到站。

第二种方法是让等待的线程每次检测完标记，用 `std::this_thread::sleep_for()` 睡眠一段时间。

.. code-block:: c++
    :linenos:

    void wait_for_flag()
    {
        std::unique_lock<std::mutex> lk(m);
        while(!flag)
        {
            lk.unlock();
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
            lk.lock();
        }
    }

这是个巨大的进步，因为等候线程在sleep时，不会耗费CPU计算资源。
但是，设定一个合理的sleep时间是很难的；
太短的话，等候线程会检测标记的频率会比较高，耗费CPU资源；
太长的话，任务完成时，等候线程可能还睡着，导致延迟。

第三种同时也是很推荐的方法是，使用C++标准库中提供的机制来等待事件。
用来等候另外一个线程促发一个事件的最基础的机制是 `condition variable` 。
一个条件变量可以被关联到一些事件或者其他条件，一个或多个线程都在等待那个条件被满足。
当一些线程觉得条件满足了，就可以 `notify` 一个或多个等待此条件的线程，来唤醒它们并且允许其继续运行。

Waiting for a condition with condition variables
---------------------------------------------------
C++标准库提供了条件变量的两种实现： `std::condition_variable` 和 `std::condition_variable_any` 。
它们都被声明在 `<condition_variable>` 头文件中。
两者都需要一个mutex来实现功能，
不同的是，前者必须使用 `std::mutex` ，
然而，后者可以使用任何支持最基本mutex机制的类型，所以有一个 `_any` 后缀。
因为 `std::condition_variable_any` 比 `std::condition_variable` 有更大的灵活性，所以额外的消耗也是难以避免的。 
所以，推荐使用 `std::condition_variable` ，除非必须灵活性，才考虑使用 `std::condition_variable_any` 。

下面是一个具体应用的例子：

**Listing 4.1 Waiting for data to process with a std::condition_variable**

.. code-block:: c++
    :linenos:

    std::mutex mut;
    std::queue<data_chunk> data_queue;
    std::condition_variable data_cond;

    void data_preparation_thread()
    {
        while(more_data_to_prepare())
        {
            data_chunk const data = prepare_data();
            std::lock_guard<std::mutex> lk(mut);
            data_queue.push(data);
            data_cond.notify_one();
        }
    }

    void data_processing_thread()
    {
        while(true)
        {
            std::unique_lock<std::mutex> lk(mut);
            data_cond.wait(
                    lk, []{return !data_queue.empty();});
            data_chunk data = data_queue.front();
            data_queue.pop();
            lk.unlock();
            process(data);
            if(is_last_chunk(data))
                break;
        }
    }

首先，你用一个队列来在两个线程间传递数据。
当数据准备完毕，准备数据的线程会加锁，然后将数据压入队列中。
之后，它会调用 `notify_one()` 来唤醒一个 `std::condition_variable` 等候列表里的线程（如果存在的话）。

在另外一方面，你有个处理线程。
这个线程首先加锁，这个时候使用的是 `std::unique_lock` 而不是 `std::lock_guard` 。
这个线程之后在条件变量 `data_cond` 上调用了 `wait()` ，并传入了一个lambda函数 `[]{ return !data_queue.empty()` 来检查是否唤醒的条件满足。
如果条件不满足，那么 `wait()` 会解锁，然后将这个线程阻塞在等候队列中。
当条件变量被其他线程调用了 `notify_one()` ，在休眠的等候线程就会被唤醒，同时检查唤醒条件(前面的lambda函数)，如果条件满足，那么加锁运行；否则解锁，继续等待。
这就是为什么中间使用了 `std::unique_lock` ，因为线程可能还需要自己 `unlock` 锁。 

在调用 `wait()` 的过程中，一个条件变量可以无限次检查条件。
在检查条件时，它会加锁，wait()只会在条件满足时立刻返回，否则会一直阻塞线程向下执行。

使用队列来实现线程间的数据传递是一个非常常用的场景。 
如果做的足够好，同步操作只会限制在队列本身，这个会减少同步操作的数目以及race conditions的可能。
下面我们会实现一个通用的线程安全队列。

Building a thread-safe queue with condition variables
--------------------------------------------------------
当你尝试去设计一个通用的队列时，首先有必要花几分钟思考下需要准备那些操作。
下面让我们参考下C++标准库中的设计。

**Listing 4.2 std::queue interface**

.. code-block:: c++
    :linenos:

    template <class T, class Container = std::deque<T> >
    class queue {
        public:
            explicit queue(const Container&);
            ...

            void swap(queue  &q);
            bool empty() const;
            size_type size() const;

            T& front();
            const T& front() const;
            T& back();
            const T& back() const;

            void push(const T& x);
            void push(T&& x);
            void pop();
            template <class... Args> void emplace(Args&&... args);

    };

如果你忽略了构造函数，赋值操作和swap操作，那么剩下的接口就可以分为三类：

1. 查询队列状态，比如 `empty()` 和 `size()`
2. 查询队列元素，比如 `front()` 和 `back()`
3. 修改队列，比如 `push()` , `pop()` , `emplace()` 

这些就和之前的stack例子一样。
类似的，你也需要将 `front()` 和 `pop()` 结合起来作为一个操作，就像在stack中，你也需要结合 `top()` 和 `pop()` 。
Listing 4.1 中的代码有一些细微的差别，当使用一个队列来实现线程间传递数据时，
接收数据的线程常常需要等待数据。

我们为 `pop()` 添加两个变种：

1. `try_pop()` : 尝试从队列中pop一个数据，并且不管成功与否，都会立刻返回
2. `wait_and_pop()` : 等待，直到有数据为止

相关的接口如下

.. code-block:: c++
    :linenos:
    
    #include <memory>

    template<typename T>
    class threadsafe_queue
    {
    public:
        threadsafe_queue();
        threadsafe_queue(const threadsafe_queue&);
        threadsafe_queue& operator=(
                const threadsafe_queue&) = delete;

        void push(T new_value);

        bool try_pop(T& value);
        std::shared_ptr<T> try_pop();

        void wait_and_pop(T& value);
        std::shared_ptr<T> wait_and_pop();

        bool empty() const;
    };

类似于前面stack的实现，对于 `try_pop` 可以有两个重载：

1. 使用引用获得结果，并且可以将pop的状态用一个bool返回，如果获得数据，可以返回true，否则返回false。
2. 直接用指针返回数据，有数据，则直接返回指针，否则返回NULL.

Listing 4.3中的具体接口如下：

.. code-block:: c++
    :linenos:

    #include <queue>
    #include <mutex>
    #include <condition_variable>

    template<typename T>
    class threadsafe_queue
    {
    private:
        std::mutex mut;
        std::queue<T> data_queue;
        std::condition_variable data_cond;
    public:
        void push(T new_value)
        {
            std::lock_guard<std::mutex> lk(mut);
            data_queue.push(new_value);
            data_cond.notify_one();
        }

        void wait_and_pop(T& value)
        {
            std::unique_lock<std::mutex> lk(mut);
            data_cond.wait(lk, [this] {return !data_queue.empty();});
            value = data_queue.front();
            data_queue.pop();
        }

        threadsafe_queue<data_chunk> data_queue;

        void data_preparation_thread()
        {
            while(more_data_to_prepare())
            {
                data_chunk const data = prepare_data();
                data_queue.push(data);
            }
        }

        void data_processing_thread()
        {
            while(true)
            {
                data_chunk data;
                data_queue.wait_and_pop(data);
                process(data);
                if(is_last_chunk(data)) break;  // 线程退出条件
            }
        }
    };

现在需要的互斥量和条件变量都被threadsafe_queue内置了，所以，使用threadsafe_queue不需要例外的协同设置。

**Listing 4.5 Full class definition for a thread-safe queue using condition variables**

.. code-block:: c++
    :linenos:

    #include <queue>
    #include <memory>
    #include <mutex>
    #include <condition_variable>

    template<typename T>
    class threadsafe_queue
    {
    private:
        mutable std::mutex mut; // mutable 允许在const函数中修改类状态
        std::queue<T> data_queue;
        std::condition_variable data_cond;
    public:
        threadsafe_queue() {}
        threadsafe_queue(threadsafe_queue const& other)
        {
            std::lock_guard<std::mutex> lk(other.mut);
            data_queue = other.data_queue;
        }
        
        void push(T new_value)
        {
            std::lock_guard<std::mutex> lk(mut);
            data_queue.push(new_value);
            data_cond.notify_one();
        }

        void wait_and_pop(T& value)
        {
            std::unique_lock<std::mutex> lk(mut);
            data_cond.wait(lk, [this]{return !data_queue.empty();});
            value = data_queue.front();
            data_queue.pop();
        }

        std::shared_ptr<T> wait_and_pop()
        {
            std::unique_lock<std::mutex> lk(mut);
            data_cond.wait(lk, [this]{return !data_queue.empty();});
            std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
            data_queue.pop();
            return res;
        }

        bool try_pop(T& value)
        {
            std::lock_guard<std::mutex> lk(mut);
            if(data_queue.empty())
                return false;
            value = data_queue.front();
            data_queue.pop();
            return true;
        }

        std::shared_ptr<T> try_pop()
        {
            std::lock_guard<std::mutex> lk(mut);
            if(data_queue.empty())
                return std::shared_ptr<T>();
            std::shared_ptr<T> res(std::make_shared<T>(data_queue.front()));
            data_queue.pop();
            return res;
        }

        bool empty() const  // 这里会用到mutable 的mutex
        {
            std::lock_guard<std::mutex> lk(mut);
            return data_queue.empty();
        }
    };

这里注意 `empty()` 函数，被标记为 `const` ，但是因为 `mut` 是 `mutable` 的，所以可以在构造函数和 `empty()` 中可以被修改。

条件变量在有多个线程在等待同一个事件的场景下也有用。
如果多个线程来协同完成同一个任务。
当一个新数据准备完毕，就调用 `notify_one()` 就会通知一个等候的线程来判断wait条件。
具体哪个线程被通知并不能确定，也可能所有的线程都正在处理数据，并没有线程在wait。

另外一种可能的情况是，多个线程都在等待同一个事件，而且它们全部都需要回复。
比如共享数据被初始化完毕，然后所有处理线程都需要被唤醒(尽管前面有更好的解决方式)。
或者多个线程需要等待共享数据的更新，比如一个定期的初始化。
在这些情况下，准备数据的线程可以调用 `notify_all()` 来唤醒所有的线程来检查自己在准备的条件是否已经成熟。

如果等候线程只等一次，当条件是 `true` ，那么它就不会再等待这个条件变量了，
那么一个条件变量也许不是最好的选择了。
在这个场景下， 一个 `future` 也许更加适合。

Waiting for one-off events with futures
===========================================
假设你坐飞机去度假。 
一旦你来到机场，在完成了各种手续之后，你坐在座位上等待登机的通知。
你也许有很多方式来打发等待的时间，比如读书看报玩手机啥的，
但是最重要的是，你正在等待一件事：登机的通知。
这个通知是未来一次性的-- 下次你在来机场的时候，你会等待另外一架飞机。

C++ 标准库用一个称作 `future` 的来建模这类一次性的事件。
如果一个线程需要等待一个一次性事件，它以某种方式取得一个代表这个事件的 `future` 。
这个线程可以在未来的执行其他任务时会周期性地短时间等待来查看事件是否已经发生，
或者，它正在做另外一件事情，直到需要一个事件发生之后，它才继续进行。
一个 `future` 可以关联到数据。一旦其关联的事件发生了，就不能重新设置(reset)了。

在C++标准库中有两类 `feature` ，都被定义在 `<future>` 头文件中：
 
1. `unique feature` (std::future<>)
2. `shared futures` (std::shared_future<>)

`std::future` 的一个实例会独占一个事件，而 `std::shared_future` 可以将多个feature联系到同一个事件。

一次性事件中，最基础的就是在后台运行的计算返回结果。

Returning values from background tasks
-----------------------------------------
假设你有个长时间的计算任务，但是当前不需要结果。
你可以启动一个线程在后台执行计算任务，并且在前台线程需要结果的时候返回结果。
在这种场景下， `std::async` 函数模板就能发挥作用了。

你可以使用 `std::async` 来启动一个异步的任务，这个任务的结果你目前暂时不需要，但是在将来的某个时刻需要。
不同于给你返回一个 `std::thread` 对象，然后让你等待， `std::async` 会返回一个 `std::future` 对象，它会持有函数的返回结果。 

当你需要结果的时候，可以调用 `get()` ，那么当前线程会被阻塞，直到 `future` 的结果执行完毕了。

下面是一个例子：

**Listing 4.6 Using std::future to get the return value of an asynchronous task**

.. code-block:: c++
    :linenos:
    
    #include <future>
    #include <iostream>

    int find_the_answer_to_ltuae(); // 一个长时的计算任务 结果暂时不需要
    void do_other_stuff();

    int main()
    {
        std::future<int> the_answer<std::async(find_the_answer_to_ltuae);   // 开启异步线程计算
        do_other_stuff();
        std::cout<<"The answer is " << the_answer.get() << std::endl;   // 现在申请结果 
        return 0;
    }

和 `std::thread` 一样， `std::async` 允许你通过添加额外参数的方式为函数传递参数。
如果第一个参数是一个成员函数的指针，第二个参数提供了对象（直接提供，或者指针，或者 `std::ref` 封装）
，那么后续的参数将传递给对应的成员函数。
否则，第二个及后续的参数都会被传递给对应的函数。
就像 `std::thread` ，如果参数是rvalue，那么将会采用move操作传入。
这使得只支持move的参数的传递成为可能。

**Listing 4.7 Passing arguments to a function with std::async**

.. code-block:: c++
    :linenos:

    #include <string>
    #include <future>

    struct X
    {
        void foo(int, std::string const&);
        std::string bar(std::string const&);
    };

    X x;
    auto f1 = std::async(&X::foo, &x, 42, "hello"); // 调用 x.foo("hello")
    auto f2 = std::async(&X::bar, x, "goodbay");    // tmpx 是x复制而来
    struct Y
    {
        double operator() (double);
    };
    Y y;
    auto f3 = std::async(Y(), 3.141);   // tmpy 是Y() 通过move操作复制而来
    auto f4 = std::async(std::ref(y), 2.718);   // 直接调用 y(2.718)
    class move_only
    {
    public:
        move_only();
        move_only(move_only&&);
        move_only(move_only const&) = delete;
        move_only& operator=(move_only&&);
        move_only& operator=(move_only const&) = delete;
        void operator() ();
    }
    auto f5 = std::async(move_only());  // move_only的 std::move复制

默认情况下， `std::async` 是否会启动一个新线程取决于具体实现，或者当future在被等待时，任务是否是异步运行的。
在很多情况下，这就是你所需要的，但是你可以通过给 `std::async` 传递额外的参数来定制其运行的方式。

* `std::launch::deferred` 来指定函数调用延迟到调用 `wait()` 或者 `get()` 时
* `std::launch::async` 来指定函数以自己的线程异步运行
* `std::launch::deferred | std::launch::async` 来指定按默认方式运行

比如下面的例子：

.. code-block:: c++
    :linenos:

    auto f6 = std::async(std::launch::async, Y(), 1.2); // 开启新线程异步运行
    auto f7 = std::async(std::launch::deferred, baz, std::ref(x));  // 延迟云溪功能
    auto f8 = std::async(
            std::launch::deferred | std::launch::async,
            baz, std::ref(x));  // 默认方式运行
    auto f9 = std::async(baz, std::ref(x));
    f7.wait();  // 现在调用并执行被延迟的函数

Associating a task with a future
----------------------------------
`std::packaged_task<>` 将一个future绑定到一个函数或者可执行对象上。
当 `std::packaged_task<>` 对象被调用了，它会调用对应的函数或者可执行对象，然后将future设置为true，将结果存储起来。

`std::packaged_task<>` 的模板参数是一个函数签名，就像 `void()` 对应一个无传参无返回的函数，
或者 `int(std::string&, double*)` 对应一个两个传参返回int的函数。
当你构造了一个 `std::packaged_task` ，你必须传入一个函数或者可调用对象来接受具体类型的参数，但返回值的类型可以通过隐变化，比如int到float型。

特定函数签名的返回类型指定了 `std::future<>` 从 `get_future()` 成员函数返回的类型。
比如，一个 `std::packaged_task<std::string(std::vector<char>*, int)>` 局部的类的定义如下：

**Listing 4.8 Partial class definition for a specialization of std::packaged_task**

.. code-block:: c++
    :linenos:

    template<>
    class packaged_task<std::string(std::vector<char>*, int)>
    {
    public:
        template<typename Callable>
        explicit packaged_task(Callable&& f);
        std::future<std::string> get_future();
        void operator() (std::vector<char>*, int);
    };

`std::packaged_task` 对象可调用的对象，
而且可以被 `std::function` 作为可调用对象就行封装，或者被直接调用。
当 `std::packaged_task` 被作为函数对象调用了，那参数就会被传递给其包含的函数中，然后返回值被存储为 `std::future` 对象中的异步结果，future对象可以通过 `get_futre()` 取得。
你因此可以将一个任务封装到 `std::packaged_task` 对象中。
当你需要其结果时，你可以等待future准备完毕。 
下面的是实际的例子：

Parsing tasks between threads
******************************
许多GUI框架都需要从某个特定的线程来更新GUI，
所以，如果另外一个线程需要更新GUI，它必须向正确的线程发送一个消息。
`std::packaged_task` 提供了一种方式来避免为每个线程或者每种GUI操作都需要一种自定义的消息。

**Listing4.9 Running code on a GUI thread using std::packaged_task**

.. code-block:: c++
    :linenos:

    #include <deque>
    #include <mutex>
    #include <futurea>
    #include <thread>
    #include <utility>

    std::mutex m;
    std::deque<std::packaged_task<void()> > tasks;

    bool gui_shutdown_message_received();
    void get_and_process_gui_message();

    void gui_thread()
    {
        while(!gui_shutdown_message_received())
        {
            get_and_process_gui_message();
            std::packaged_task<void()> task;
            {
                std::lock_guard<std::mutex> lk(m);
                if(tasks.empty()) continue;
                task = std::move(tasks.front());
                tasks.pop_front();
            }
            task();
        }
    }

    std::thread gui_bg_thread(gui_thread);

    // 以任务为单位提交 而不是以消息为单位
    // 这样省去了接收message然后调用对应操作的麻烦
    template<typename Func>
    std::future<void> post_task_for_gui_thread(Func f)
    {
        std::packaged_task<void()> task(f);
        std::future<void> res = task.get_future();
        std::lock_guard<std::mutex> lk(m);
        tasks.push_back(std::move(task));
        return res;
    }

这个例子使用 `std::packaged_task<void()>` 来封装任务，具体的操作可以是不接收参数和返回数据的函数或可调用对象。

Making (std::)promises
-------------------------------
当你有一个需要处理大量网络连接的应用，
那么用单独的线程来处理每个连接比较简单，因为这样可以使网络交互更加容易实现。
这个模式在连接比较少的时候比较适用，
但是当连接数目足够巨大时，为每个连接而分配的一个线程会导致总线程数巨大，而极大地耗费系统资源。
因此，常规的方法就是，分配固定数目的线程（也可能只有一个），而每个线程都需要处理多个连接。

想象这多个线程中的一个线程的工作机制。
数据包会从多个连接传过来，然后以一种随机的顺序被处理，
类似地，数据包在发送时，也会在队列中排列起来。
在很多情况中，应用的其他部分都会等待一个特定的网络连接成功发出或者接收数据。

`std::promise<T>` 提供了多种设置值的方法，这个值可以通过 `std::future<T>` 对象读取。
一个 `std::promise/std::future` 对可以提供这种机制；
等待线程可能在这个future上被阻塞，而提供数据的线程可以用promise来设置对应的数据，并且将future状态设置为完成。

就像 `std::packaged_task` ，你可以通过在 `std::promise` 上调用 `get_future()` 得到绑定的 `std::futrue` 对象.
当promise的值被设置了，那么对应的future的状态就会被设为完成，并且可以返回值。
如果你销毁一个没有设置值的 `std::promise` ，那么会得到一个异常。

Listing 4.10 展示了线程处理连接的例子。
在这个例子中，你使用一个 `std::promise<bool>/std::future<bool>` 对来标记数据的传输成功与否的状态；
绑定到future的数据就是简单的成功/失败标记。

.. code-block:: c++
    :linenos:

    #include <future>
    
    void process_connections(connection_set& connections)
    {
        while(!done(connections))   // 循环 直到处理完毕
        {
            for(connection_iterator
                    connection = connections.begin(), 
                    end = connections.end();
                connection != end;
                ++condition)    // 循环处理所有的连接
            {
                if(connection->has_incomming_data())  // 处理接收的数据
                {
                    data_packet data = connection->incoming();
                    std::promise<payload_type>& p
                        = connection->get_promise(data.id);
                    p.set_value(data.payload);
                }
                if(connection->has_outgoing_data())  // 处理要发出的数据
                {
                    outgoing_packet data = 
                        connection->top_of_outgoing_queue();
                    connection->send(data.payload);
                    data.promise.set_value(true);   // 成功发出后 更新标记
                }
            }
        }
    }

Saving an exception for the future
------------------------------------
考虑下面的一小段代码。 
如果你给 `square_root()` 传递-1，那么就会抛出一个异常：

.. code-block:: c++
    :linenos:

    double square_root(double x)
    {
        if(x < 0) 
        {
            throw std::out_of_range("x<0");
        }
        return sqrt(x);
    }

现在想象你不是从当前的线程触发异常：

.. code-block:: c++
    :linenos:

    double y = square_root(-1);

你采用异步调用：

.. code-block:: c++
    :linenos:

    std::future<double> f = std::async(square_root, -1);
    double y = f.get();

如果异步执行和单线程执行的行为相同，那是比较合理的。
就像 y 得到了函数调用的结果，如果调用 `f.get()` 的线程也能看到异常，那么就非常好了。

这就是具体的工作方式： 
如果异步的函数调用抛出了异常，那么这个异常就会被存储到future的数值空间中，然后future的状态被设置为完毕，调用 `get()` 会抛出存储的异常。

如果你用 `std::packaged_task` 封装函数，那么异常的行为也是一致的。当函数抛出异常，那么future对象会将异常存储到对应的数据空间，并且将future的状态设置为完毕。后面调用 `get()` 的线程会抛出异常。

自然， `std::promise` 也提供了相似的机制。
如果你希望存储一个异常而不是一个值，你调用 `set_exception()` 成员函数，而不是 `set_value()` 。 
这个可以在 `catch` 语句段中使用：

.. code-block:: c++
    :linenos:

    extern std::promise<double> some_promise;
    
    try
    {
        some_promise.set_value(calculate_value());
    }
    catch(...)
    {
        some_promise.set_exception(std::current_exception());
    }

这段代码使用 `std::current_exception()` 来取得抛出的异常；
也可以使用 `std::copy_exception()` 来不抛出异常而直接存储一个新的异常。

.. code-block:: c++

    some_promise.set_exception(std::copy_exception(std::logic_error("foo ")));

这样就比使用一对 `try/catch` 语句段要清爽很多了，这种方式应该有限采用，因为它能够使得编译器有机会提前优化代码。

然而， `std::future` 也有它的限制，就是只有一个线程能够等待结果。
如果你需要等待多个线程的同一个事件，你需要使用 `std::shared_future` 来代替。

Waiting from multiple threads
--------------------------------
尽管 `std::future` 能够很好地应付两个线程间的数据传递。
但是 `std::future` 不能用于向多个线程传递数据，因为其存储的结果数据理论上只能使用一次，而且并没有多线程协同的功能。
因为 `std::future` 是只能move的，所以，其结果只能被一个线程获得。

如果你需要让多个线程等待同一个事件，那么可以使用 `std::shared_future` 。
`std::shared_future` 是可以复制的，所以你可以有多个对象来获取同一个状态。

现在，有了 `std::shared_future` ， 在一个单独对象上的成员函数依旧是为非同步的，所以为了避免多线程访问一个对象带来的data races，你必须用一个锁就行保护。
推荐的方法是，单独复制一个对象，然后每个线程操作自己的那个副本。

`std::shared_future` 的一个潜在的使用就是对一个电子表格的并行处理；
每个单元里都有一个最终值，这个值可能被其他单元里的公式用到。
在相互依赖的单元中，可以使用一个 `std::shared_future` 来引用第一个单元。
如果不同的单元中的公式并行运行，那些不需要依赖或者依赖的值可访问的话，任务会并行起来。 否则，对于那些依赖的其他值没有完毕的，会暂时阻塞。

引用某些同步状态的 `std::shared_future` 的实例是从 引用那些状态的 `std::future` 构造而来。
因为 `std::future` 对象不能将状态的所有权共享给其他对象，因此所有权必须被move给 `std::shared_future` ，之后， `std::future` 的状态便被清空。

.. code-block:: c++
    :linenos:

    std::promise<int> p;
    std::future<int> f(p.get_future());
    assert(f.valid());
    std::shared_future<int> sf(std::move(f));   // 所有权被move传递
    assert(!f.valid());
    assert(sf.valid());

就像对其他可以move的对象，所有权的传递对rvalue是透明的，
所以，你可以直接从 `std::promise` 对象的 `get_future()` 的返回值中直接构建一个 `std::shared_future` 对象。

比如：

.. code-block:: c++
    :linenos:

    std::promise<std::string> p;
    std::shared_future<std::string> sf(p.get_future()); // rvalue的透明move

这里，所有权的传递是隐含的。 

`std::shared_future<>` 是从 `std::future<std::string>` 返回的rvalue直接构造出来的。

`std::future` 有一个 `share()` 成员函数来创建一个新的 `std::shared_future` 对象，而且将所有权直接move给它。
这个比较方便：

.. code-block:: c++
    :linenos:

    std::promise< std::map< SomeIndexType, SomeDataType, 
            SomeComparator, SomeAllocator>::iterator> p;
    auto sf = p.get_future().share();

在这个例子里， `sf` 的类型被推断为 `std::promise< std::map< SomeIndexType, SomeDataType, SomeComparator, SomeAllocator>::iterator>` 。
如果map的comparator或者allocator被修改了，你只需要修改promise的类型，而future的类型会自动更新。

Waiting with a time limit
==============================
在之前介绍的future里，只要future的状态没有完毕，那么等待的时间都可能变成无限。
但很多情况下，我们都需要有一个时间观念，比如，在长时间的等待中，定期发送一个 `I's still alive` 的状态，或者限定一个最长时间后自动退出。

有两类你可以指定的时间：

1. 一个 `duration-based` 超时， 当你在一个固定长度的时间内等待（比如30s）
2. 一个 `absolute` 超时，你可以等到一个特定的时间点 （比如，2月15号17点30分）

绝大多数的wait函数都对应着提供了两类变种，用不同的后缀来表示，一个有 `_for` 后缀，另外一个是 `_until` 后缀。

比如， `std::condition_variable` 有两个wait扩展： `wait_for` 和 `wait_until` ，在固定的时间内被唤醒且等待的条件成立，又或者超时了都会返回。

在我们具体查看这两个函数实现的细节之前，首先来看下C++标准库中的时间定义。 
首先是 clock。

Clocks
-------
根据C++标准库，clock就是一个时间的信息源。
具体来说，clock是一个类，它提供了四类截然不同的信息：

* 时间 `now`
* 表示时间的类型
* 时间周期
* clock是否以标准的时间周期运行，是否是一个稳定的时钟

clock现在的时间可以通过调用clock类的静态成员函数 `now()` 获得；
比如， `std::chrono::system_clock::now()` 会返回系统时钟的当前时间。
特定clock的时刻类型被typedef为 `time_point` ，所以， `some_clock::now()` 的返回类型是 `some_clock::time_point` 。

时钟的单位周期被设置为1秒的分数比例。 
如果一个时钟一秒走25下，那么它就有一个 `std::ratio<1, 25>` 的单位周期，
而一个2.5秒走一下的单位周期就是 `std::ratio<5, 2>` .
如果一个时钟的单位周期需要在运行时确定，或者后面需要被调节，
那么其周期应该设置为一个平均值，或者最小单位。

如果一个时钟以标准周期运行，并且不能被调整，
那么这个时钟就被称为一个稳定的时钟。
对于稳定时钟，静态成员数据 `is_steady` 会返回 true， 否则返回false。
特定地， `std::chrono::system_clock` 不会是steady，因为它的时钟周期是可以被调节的。 
这样的调节可能会使现在调用的 `now()` 的结果比先前调用 `now()` 得到的结果还要早，这样无疑是违背客观规律的。
时钟的稳定性在计算超时是很重要的，所以C++标准库提供了一个稳定的时钟： `std::chrono::steady_clock` 。

C++标准库提供的其它时钟包括：

* `std::chrono::system_clock` ， 代表了系统的真实时间，并且提供了 `time_t` 值与时刻之间互相转换的函数
* `std::chrono::high_resolution_clock` ，提供了最小的时刻周期

这些时钟都被定义在 `<chrono>` 头文件中。 

Durations
----------
持续时间是时间支持的最简单的部分；
它们被类模板 `std::chrono::duration<>` 来定义。
第一个模板参数是表示类型（比如 int, long, double），
第二个模板参数表示时钟周期。
比如，一个用short表示分钟数的持续时间表示为： `std::chrono::duration<short, std::ratio<60,1>>` ，
因为一分钟有60秒。
在另一方面，一个用double来表示毫秒数的持续时间表示为 `std::chrono::duration<double, std::ratio<1, 1000>>` ，
因为每一毫秒是 1/1000 秒。

标准库在命名空间 `std::chrono` 中提供了一些现成的类型， 

* nanosechonds
* microseconds
* milliseconds
* seconds
* minutes
* hours

持续时间之间的转换是隐含的。 
显式的转换可以用 `std::chrono::duration_cast<>` 来来实现。

.. code-block:: c++
    :linenos:

    std::chrono::milliseconds ms(54802);
    std::chrono::seconds s = 
        std::chrono::duration_cast<std::chrono::seconds> (ms);

结果会被截断，而不是四舍五入，所以在这个例子里，s最终为54.

时间间隔支持数学运算，
所以你可以在时间间隔间加减，或者乘除有一个常数。
因此， `5 * seconds(1)` 就和 `seconds(5)` 或者 `minites(1) - seconds(55)` 相同。
时间单位的个数可以通过 `count()` 操作获得。 因此， `std::chrono::milliseconds(1234).count()` 的值是 1234.

基于时间间隔的wait可以使用 `std::chrono::duration<>` 来实现。
比如，你可以等待35毫秒来使future的状态变成完毕：

.. code-block:: c++
    :linenos:

    std::future<int> f = std::async(some_task);
    if(f.wait_for(std::chrono::milliseconds(35)) == std::future_status::ready)
        do_something_with(f.get());

wait函数会返回一个状态，来标明或者等待超时，或者等待的事件发生。
在这种情况下，你在等待一个future，所以如果超时了，函数会返回 `std::future_status::timeout` ，
如果处理完毕，会返回 `std:future_status::ready` ，
如果future的任务被推迟了，会返回 `std::future_status:deferred` 。
等待的超时使用内部的一个稳定的时钟来计算的，所以35毫秒表示过去35毫秒，即使时钟周期被改变了，时间间隔的长度也不会发生变化。
当然，由于系统调度或者系统时钟的精确度问题，时间的时间可能会比35毫秒长。

Time points
---------------
时刻是由 `std::chrono::time_point<>` 类模板的对象来表示的，
它的第一个模板参数表示使用那种时钟，
第二个木板参数是具体的时间单位。
时刻的数值是从某个称为时钟原点的特定的时刻开始的时间的长度，
时钟原点是一个基本的性质，而C++标准库中不能直接查询或者设定。
典型的时钟原点代表1970年1月1号0时0刻。
不同的时钟间可以共享时钟原点，或者设置独立的时钟原点。
尽管你不能直接查询时钟原点的时间，但是，你可以通过 `time_since_epoch()` 来得到时间原点到某个指定的 `time_point` 的时间长度。

比如，你指定了一个时刻 `std::chrono::time_point<std::chrono::system_clock, std::chrono::minutes>` 。
这个将会得到一个相对于系统时钟的原点，并用分钟来表示的时钟（时刻）。

你可以增加或者减少时间间隔，所以 `std::chrono::high_resolution_clock::now() + std::chrono::nanoseconds(500)` 将会给你一个未来500纳秒后的时刻。

你可以通过两个相同时钟类型的时刻之间相减来得到之间的间隔。 
这个有助于计算一段代码运行的时间：

.. code-block:: c++
    :linenos:

    auto start = std::chrono::high_resolution_clock::now();
    do_something();
    auto stop = std::chrono::high_resolution_clock::now();
    std::cout << "do_something() took "
            << std::chrono::duration<double, std::chrono:seconds> (stop - start).count()
            << " seconds" << std::endl;

**Listing 4.11 Waiting for a condition variable with a timeout**

.. code-block:: c++
    :linenos:

    #include <condition_variable>
    #include <mutex>
    #include <chrono>

    std::condition_variable cv;
    bool done;
    std::mutex m;

    bool wait_loop()
    {
        auto const timeout = std::chrono::steady_clock::now() + 
            std::chrono::milliseconds(500);
        std::unique_lock<std::mutex> lk(m);
        while(!done)
        {
            if(cv.wait_until(lk.timeout) == std::cv_status::timeout)
                break;
        }
        return done;
    }


**此章节未完待续...**
References
-----------
CPP Currency in Action 第四章

.. raw:: html

    <script>window._bd_share_config={"common":{"bdSnsKey":{},"bdText":"","bdMini":"2","bdMiniList":["qzone","tsina","weixin","renren","tqq","sqq","hi","youdao"],"bdPic":"","bdStyle":"0","bdSize":"16"},"slide":{"type":"slide","bdImg":"5","bdPos":"left","bdTop":"159"}};with(document)0[(getElementsByTagName('head')[0]||body).appendChild(createElement('script')).src='http://bdimg.share.baidu.com/static/api/js/share.js?v=89860593.js?cdnversion='+~(-new Date()/36e5)];</script>

.. raw:: html

    <!-- 多说评论框 start -->
    <div class="ds-thread" data-thread-key="cpp-concurrency4.rst" data-title="CPP Currency in Action note(4) Synchronizing concurrnet operation" data-url="http://superjom.duapp.com/program-language/cpp-concurrency4.html"></div>
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
