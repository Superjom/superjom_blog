CPP Currency in Action note(1)  Hello world example
=====================================================
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

*2014-12-12*

最近参数服务器项目需要C++11的多线程编程，在网上找了一本《Cpp Currency in Action》，学习一下，顺便做一个笔记。

一个简答的Hello world的例子：

.. code-block:: C++
    :linenos:

    #include <iostream>
    #include <thread>
    
    void hello() 
    {
        std::cout << "Hello Concurrent World\n";
    }

    int main()
    {
        std::thread t(hello);
        t.join();
    }


这段简单的代码里，与传统的代码相比，有几个需要注意的地方。
首先是 `#include <thread>` ，这个会引入C++11里多线程库。

然后是相比于传统的单线程方式，这段代码启用了一个线程来运行hello()，而不是直接调用hello()函数。

在新的hello线程开始运行之后，main线程其实也在同步运行。 如果不进行限制，那么main线程会直接运行 `return 0;` 退出程序，这个时候，也许hello线程还没运行完。 所以，这就是后面 `t.join()` 的必要性了。

`join()` 操作会使得调用线程(main)等待被调用的线程(hello)运行完后再继续运行后面的程序。 
所以，在这个例子里，main线程会等待hello线程运行完之后再继续运行后面 `return 0;` ，这样就合理很多了。


References
-----------
CPP Currency in Action 第一章


.. raw:: html

    <!-- 多说评论框 start -->
    <div class="ds-thread" data-thread-key="cpp-concurrency.rst" data-title="CPP Currency in Action note(1)  Hello world example" data-url="http://superjom.duapp.com/program-language/cpp-concurrency.html"></div>
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
