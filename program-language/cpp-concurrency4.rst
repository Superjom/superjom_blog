CPP Currency in Action note(4) Synchronizing concurrnet operation
===================================================================
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
---------------------------------------------
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


