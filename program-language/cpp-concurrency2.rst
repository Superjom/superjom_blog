CPP Currency in Action note(2)  Managing threads
=====================================================
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

*2014-12-12*

这个章节会包括：

* 开启一个线程，以及指定让线程运行指定代码的多种方式
* 等待线程运行完，或者让线程持续运行下去
* 唯一标识线程

发射一个线程
-------------
在上一个章节里，一个hello world的例子里，`hello()` 函数会输出一个hello。 

thread能够使用如何可调用的类型，所以也可以传入一个可调用类的对象。

.. code-block:: C++
    :linenos:

    class background_task
    {
    public:
        void operator() () const 
        {
            do_something();
            do_somethin_else();
        }
    };

    background_task f;
    std::thread my_thread(f);

有可以用一个lambda函数来实现相同的功能：

.. code-block:: C++
    :linenos:

    std::thread my_thread([]() {
        do_something();
        do_something_else();
    });

一旦你启动了线程，那么一个需要考虑的问题是，是否需要等待此线程运行完，或者让它自己运行。
如果在线程被销毁前，你没有决定好，那你的程序就终止了。
所以，你必须决定好你的线程当前是被 `join` 还是 `detach` .
记住，你必须在thread被destroy前决定好，否则，当你join或者detach的时候，可能线程已经退出好长时间了。
或者，如果你detach这个线程，那这个线程可能在thread被destroy后还会继续运行好长时间。

如果你不等待线程退出，那么你需要保证线程在退出之前，它所接触到的数据是一直有效的。
这不是一个新的问题，即使在单线程的程序上，也可能出现这个问题。 只是在多线程的环境中，更容易出现类似的问题。

出现这样的问题的一个场景是，给线程传入了一个指向局部变量的指针或者引用，而且线程在退出函数前还没有终止。
下面是一个例子。

.. code-block:: C++
    :linenos:

    struct func
    {
        int& i;
        func(int& i_) : i(i_) {}

        void operator() () 
        {
            for(unsigned j = 0; j < 1000000000; ++j) 
            {
                do_something(i);
            }
        }
    };

    void oops()
    {
        int some_local_state = 0;
        func my_func(some_local_state);
        std::thread my_thread(my_func);
        my_thread.detach();
    }

在上面的例子里，my_thread指向了一个局部变量 `some_local_state` 的引用，并且在 `oops` 函数退出后， `my_thread` 可能还在运行，那么它会访问一个呗销毁的 `some_local_state` 。
这在多线程程序中就会出现问题。

解决这个问题的一个常规的方法就是，让thread的函数自包含，并且将所需要的参数复制到线程中，而不是共享数据。
那么，在线程的运行期，外界的变量销毁与否都不会有任何影响了。

Waiting for a thread to complete
-----------------------------------
如果你想要让主线程等待子线程执行完再继续运行，那你可以调用 `join()` 。
这就能保证局部变量的在子线程运行期的有效性。

join() 操作同时会删除线程的存储空间。 所以，对于一个线程， `join()` 只能调用一次。 
一旦调用了 `join()` ，那么 `std::thread` 对象永远不能 `joinable` ，而且 `joinable()` 会返回 `false` 。

Waiting in exceptional circumstances
------------------------------------------
为了防止线程出现异常，而无法就行 `join` ，下面有一个解决的小例子：

.. code-block:: c++
    :linenos:

    struct func;

    void f() 
    {
        int some_local_state = 0;
        func my_func(some_local_state);
        std:: thread t(my_func);

        try
        {
            do_somethin_in_current_thread();
        }
        catch(...) 
        {
            t.join();
            throw;
        }
        t.join();
    }

一种渐变方法是，提供一个类，并且在类的析构函数中调用 `join()` 

.. code-block:: c++
    :linenos:

    class thread_guard;
    {
        std::thread& t;

    public:
        explicit thread_guard(std::thread& t_): 
            t(t_)
        {}
        ~thread_guard()
        {
            if(t.joinable())
            {
                t.join();
            }
        }
        thread_guard(thread_guard const&) = delete;
        thread_guard& operator=(thread_guard const&) = delete;

    };

    struct func;

    void f()
    {
        int some_local_state = 0;
        func my_func(some_local_state);
        std::thread t(my_func);
        thread_guard g(t);

        do_somethin_in_current_thread();
    }

Running threads in the background
---------------------------------------
在 `std::thread` 对象上调用 `detach()` 会让线程在后台运行，而不能有如何直接的交流的方式。
也不能等待线程执行完毕； 如果一个线程被 `datech` ，那么不可能再获得此线程的引用，也不能在此线程上执行 `join` 操作。
被 `detach` 的线程在后台运行，所有权被移交给了 C++ Runtime Library，以保证线程退出时，其资源会被正确回收。

被 `detach` 的线程在UNIX系统里被称为 `daemon threads` 。
它们在后台运行，没有直接的方式来与它们交互。
这类线程一般是长时间运行的，默默执行一些操作，比如监测文件系统，或者自动清除一些缓存。 

`join()` 和 `detach()` 互相间是互斥的操作，所以，下面的操作是合法的

.. code-block:: c++
    :linenos:

    std::thread t(do_background_work);
    t.detach();
    assert(!t.joinable());

假如有下面一个场景，你设计了一个gui的文件编辑器，当用户在当前窗口开启了一个另外一个窗口，那么另外一个窗口的生命周期与当前窗口是无关的。 
那么下面一段代码演示了这样一个过程：

.. code-block:: c++
    :linenos:

    void edit_document(std::string const& filename) 
    {
        open_document_and_display_gui(filename);
        while(!done_editing())
        {
            user_command cmd = get_user_input();
            if(cmd.type == open_new_document)
            {
                std::string const new_name = get_filename_from_user();
                std::thread t(edit_document, new_name);
                t.detach();
            } else {
                process_user_input(cmd);
            }
        }
    }

Passing arguments to a thread function
---------------------------------------------
可以通过对向 `std::thread` 的构造函数传入额外参数的方式来对线程的函数传递参数。
但是，需要注意的是， **默认，这些参数都是被拷贝到线程空间的** 。

下面是一个简单的例子:

.. code-block:: c++
    :linenos:

    void f(int i, std::string const& s);
    std::thread t(f, 3, "hello");

上面的 `std::string` 对象会被复制，这样线程执行是安全的，但是如果传入指针的话，就可能出现问题了。

比如：

.. code-block:: c++
    :linenos:

    void f(int i, std::string const&s);

    void oops(int some_param)
    {
        char buffer[1024];
        sprintf(buffer, "%i", some_param);
        std::thread t(f, 3, buffer);
        t.detach();
    }

上面以指针的方式传入字符串，thread复制了指针，但是指向的数据有可能在线程的生命期内过期。

可以用下面的代码强制复制字符串：

.. code-block:: c++
    :linenos:

    std::thread t(f, 3, std::string(buffer));

当然，也可能有一个相反的场景： 你希望以引用的方式传入数据，这样可以在线程中随时修改外界变量。
可以通过 `std::ref` 显式定义引用:

.. code-block:: c++
    :linenos:

    std::thread t(update_data_for_widget, w, std::ref(data));

如果你对 `std::bind` 的行为比较熟悉，那么 `std::thread` 的传参方式是一样的。

.. code-block:: c++
    :linenos:

    class X
    {
        public:
            void do_lengthy_work();
    };
    X my_x;
    std::thread t(&X::do_lengthy_work, &my_x);

Transferring owership of a thread
------------------------------------
`std::thread` 只能被 `std::move` ，而不能被拷贝。
所以，想要传递 thread的所有权，只有 `std::move` 了。

下面是一个例子：

.. code-block:: c++
    :linenos:

    void some_function();
    void some_other_function();
    std::thread t1(some_function);
    std::thread t2 = std::move(t1);
    t1 = std::thread(some_other_function);
    std::thread t3;
    t3 = std::move(t2);
    t1 = std:move(t3);

`std::thread` 支持 `std::move` 操作，也就以为着，其所有权可以传递到函数外：

.. code-block:: c++
    :linenos:

    std::thread f()
    {
        void some_function();
        return std::thread(some_function);
    }

    std::thread g()
    {
        void some_other_function(int);
        std::thread t(some_other_function);
        return t;
    }

类似地，如果线程的所有权需要传递给函数，它只能接收安值传递的 `std::thread` 的一个对象作为参数，比如

.. code-block:: c++
    :linenos:

    void f(std::thread t);
    void g()
    {
        void some_function();
        f(std::thread(some_function));   // 按值传入
        std::thread t(some_function);
        f(std::move(t));    // move
    }

`std::thread` 的move操作支持的好处是，你可以实现 `thread_guard` 类来实际地控制住线程的所有权。
比如，使得任何其他代码无法join 或者detach `thread_guard` 接管的线程。
相关的实现如下：

.. code-block:: c++
    :linenos:

    class scoped_thread
    {
        std::thread t;

    public:
        explicit scoped_thread(std::thread t_):
            t(std::move(t_))
        {
            if(!t.joinable())
                throw std::logic_error("No thread");
        }
        ~scoped_thread()
        {
            t.join();
        }
        scoped_thread(scoped_thread const &) = delete;
        scoped_thread& operator=(scoped_thread const &) = delete;
    };

    struct func;

    void f()
    {
        int some_local_state;
        scoped_thread t(std::thread(func(some_local_state)));
        do_something_in_current_thread();
    }

这个例子与上面的 `thread_guard` 有几点不同：

1. 当调用线程到达函数 `f` 的结尾时， `scoped_thread` 对象会被析构，同时 `join` 管理的线程
2. 在 `scoped_thread` 中，线程直接被按值传入，而不需要另外建立一个临时变量。
3. 在 `scoped_thread` 中，直接 `join` 线程，而不需要像 `thread_guard` 中那样，先判定线程是否joinable(因为其完全拥有线程，不允许另外的地方执行join操作).

`std::thread` 的 `std::move` 操作支持，也使得其能够被支持 `move` 操作的容器存储。
这代表着你可以批量生成很多的线程，并且批量join。

.. code-block:: c++
    :linenos:

    void do_work(unsigned id);

    void f()
    {
        std::vector<std::thread> threads;
        for(unsigned i = 0; i < 20; ++i) 
        {
            threads.push_back(std::thread(do_work, i));
        }
        std::for_each(threads.begin(), thread.end(),
                        std::mem_fn(&std::thread::join));
    }

Choosing the number of threads at runtime
---------------------------------------------
C++ 标准库里有一个函数 `std::thread::hardware_concurrency()` 。

这个函数能够返回你的应用程序实际上能够同时运行的线程数目。
在一个多核系统中，这个值可能是CPU核的个数。

开启过多的线程，可能会对系统性能带来一些副作用。

Identifying threads
--------------------
线程的标识是类型 `std::thread::id` ，它可以有两种方式得到：
1. 通过调用线程对象的 `get_id()` 操作
2. 也可以调用 `std::this_thread::get_id()`

`std::thread::id` 的对象可以被任意对比或者复制。 
如果 `std::thread` 对象咋执行，那么其id会得到具体的标识，如果没有执行，那么id会返回一个默认的id类型的对象，标记着 "not any thread"

线程的id可以用来标记不同的线程，从而进行不同的操作，比如

.. code-block:: c++
    :linenos:

    std::thread::id master_thread;
    void some_core_part_of_algorithm()
    {
        if(std::this_thread::get_id() == master_thread)
        {
            do_master_thread_work();
        }
        do_common_work();
    }

References
-----------
CPP Currency in Action 第二章


.. raw:: html

    <!-- 多说评论框 start -->
    <div class="ds-thread" data-thread-key="cpp-concurrency2.rst" data-title="CPP Currency in Action note(2)  Managing threads" data-url="http://superjom.duapp.com/program-language/cpp-concurrency2.html"></div>
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
