ZeroMQ Documentation 01 -- 简介
================================
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

*2014-12-10*

最近项目会用到ZeroMQ，之前很少写过网络程序，这次专门学习一下，顺便写个稍微详细的笔记吧。

简介
-----
ZeroMQ(也被称作 OMQ, 或者zmq)看起来像一个嵌入式的网络库，但是用起来像一个并发库。

ZeroMQ能够提供携带元信息的sockets，通过进程内，进程间，TCP，多播等方式传播。

此外，它还提供了上层的一些高级功能，比如，你可以采用N-to-N的模式来连接sockets，比如fan-out, pub-sub, task distribution和request-reply模式等。

通过异步的I/O模型，你可以很方便地开发多核应用。


Ask and Ye Shall Receive
---------------------------
首先由一个简单的小例子开始。 

我们建立一个 Hello World服务器，它绑定到5555端口，读取request，并且返回"world"。

.. code-block:: c
    :linenos:

    // Hello World server
    #include <zmq.hpp>
    #include <string>
    #include <iostream>
    #include <unistd.h>

    int main()
    {
        zmq::context_t context(1);
        zmq::socket_t socket(context, ZMQ_REP);
        socket.bind("tcp://*:5555"); //*

        while(true) {
            zmq:message_t request;
            // wait for next request from client
            socket.recv(&request);
            std::cout << "Received Hello" << std::endl;
            // do some work
            Sleep(1);
            // send reply back to client
            zmq::message_t reply(5);
            memcpy((void*) reply.data(), "World", 5);
            socket.send(reply);
        }
        return 0;
    }


传输的模式演示是

.. image:: https://github.com/imatix/zguide/raw/master/images/fig2.png

REQ-REP socket对是被锁定的，客户端先 zmq_send() ，然后 zmq_recv()，如此循环。

对应的客户端程序：

.. code-block:: c
    :linenos:

    // Hello World client in C++
    // Connects REQ socket to tcp://localhost:5555
    // Sends "Hello" to server, expects "World" back
    //
    #include <zmq.hpp>
    #include <string>
    #include <iostream>

    int main()
    {
        zmq::context_t context(1);
        zmq::socket_t (context, ZMQ_REQ);
        std::cout << "Connecting to server ..." << std::endl;
        socket.connect("tcp://localhost:5555");

        for(int request_nbr = 0; request_nbr != 10; request_nbr++) {
            zmq::message_t request(6);
            memcpy((void*) request.data(), "Hello", 5);
            std::cout << "Sending Hello " << request_nbr << "..." << std::endl;
            socket.send(request);

            zmq::message_t reply;
            socket.recv(&reply);
            std::cout << "Received World " << request_nbr << std::endl;
        }
        return 0;
    }

Getting the Message Out
---------------------------
第二个经典的模式是one-way数据分发，服务器为一个客户端集合推送更新。

我们看一个天气预报更新的例子：

服务器

.. code-block:: c
    :linenos:

    // Weather update server in C++
    // Binds PUB socket to tcp://*:5556      /* 
    // Publishes random weather updates
    //
    #include <zmq.hpp>
    #include <stdio.h>
    #include <stdlib.h>
    #include <time.h>
    
    #define within(num) (int) ((float) num * random()  (RAND_MAX + 1.0))

    int main()
    {
        zmq::context_t context(1);
        zmq::socket_t publisher(context, ZMQ_PUB);
        publisher.bind("tcp://*:5556"); //*
        publisher.bind("ipc://weather.ipc");

        // Initialize random number generator
        srandom ((unsigned) time (NULL));
        while(true) {
            int zipcode, temperature, relhumidity;

            zipcode = within(1000000);
            temperature = within(215) - 80;
            relhumidity = within(50) + 10;

            zmq::message_t message(20);
            snprintf((char*) message.data(), 20, "05d %d %d", zipcode, temperature, relhumidity);
            publisher.send(message);
        }
        retunrn 0;
    }

客户端

.. code-block:: c
    :linenos:

    // Connects SUB socket to tcp://localhost:5556
    // Collects weather updates and finds avg temp in zipcode

    #include <zmq.hpp>
    #include <iostream>
    #include <sstream>
    
    int main()
    {
        zmq::context_t context(1);
        std::cout << "Collecting updates from weather server...\n" << std::endl;
        zmq::socket_t subscriber(context, ZMQ_SUB);
        subscriber.connect("tcp://localhost:5556");

        // Subscribe to zipcode, default is NYC, 10001
        const char *filter = (argc > 1) ? argv[1] : "10001 ";   //*
        subscriber.setsockopt(ZMQ_SUBSCRIBE, filter, strlen(filter));

        // process 100 updates
        int update_nbr;
        long total_temp = 0;
        for (update_nbr = 0; update_nbr < 100; update_nbr++) {
            zmq::message_t update;
            int zipcode, temperature, relhumidity;
            subscriber.recv(&update);

            std::istringstream iss(static_cast<char*>(update.data()));
            iss >> zipcode >> temperature >> relhumidity;
            total_temp += temperature;
        }

        std::cout << "Average temperature for zipcode " << filter
            << " was " << (int) (total_temp / update_nbr) << "F" 
            << std::endl;

        return 0;
    }

模式的示意图是

.. image:: https://github.com/imatix/zguide/raw/master/images/fig4.png

注意，当使用SUB socket时，必须通过 zmq_setsockopt() 指定一个subscription源.

当然，如果没有subscribe任何一个源，那么自然不会收到任何信息。

也可以subscribe多个源，那么收到的信息会合并起来推送至subscriber.

PUB-SUB socket pair是异步的。 客户端一直进行 zmq_recv(), 而源一直进行 zmq_send().


Getting the Context Right
---------------------------
ZeroMQ应用尝尝开始于建立一个context，然后用它建立sockets.

在C语言中，在开头调用 zmq_ctx_new()，在结尾调用 zmq_ctx_destroy()

如果使用fork()系统调用，则每个进程需要一个自己的context.

Making a Clean Exit
--------------------
ZeroMQ对于如何退出有特殊的要求。

如果你遗留了任何sockets没有关闭，那么 zmq_ctx_destroy() 函数会永远挂着。

即使关闭了所有的sockets，只要有遗留的connects或者sends，那么zmq_ctx_destroy() 依旧会挂着，除非你在关闭sockets前，将那些sockets的LINGER 设置为0.

对于ZeroMQ对象，我们需要担心的是 messages, sockets, 以及 contexts. 幸运的是，只需要几个做法：

* 尽可能使用 zmq_send() 和 zmq_recv()，这样不用担心 zmq_msg_t 对象。
* 如果你使用了 zmq_msg_recv()，尽可能早地调用 zmq_msg_close() 销毁收到的message。
* 如果你打开和关闭很多的sockets，那么说明你需要重构你的应用了。 在某些场景下，socket handles直到context被销毁了之后才会被注销掉。
* 当退出应用时，关闭sockets并且调用 zmq_ctx_destroy(). 这个会销毁context。

这是C语言中需要注意的。 在一个有自动对象析构功能的开发语言中，sockets和context会在你离开scope时自动销毁。

Why We Needed ZeroMQ
-----------------------
ZeroMQ有很多优点：

* 它利用后台线程异步处理I/O。 应用间的线程采用无锁设计，所以ZeroMQ应用不使用锁、semaphores, 或者其他wait states.
* 系统内的成员可以动态加入或者退出，ZeroMQ会自动重连。 这意味着你可以以任何方式启动成员。 你可以建立 "service-oriented architectures" (SOAs)，services能够在如何时候加入或者退出网络。
* 如果后台I/O queue满了，ZeroMQ会自动block发送方，或者将后续的message扔掉。 这取决于具体的模式。
* 具体的传播方式透明化：TCP, multicast, in-process, inter-process。 采用不同的传播方式，不需要修改自己的代码。
* 你可以有多种方式route message，比如 request-reply 和 pub-sub模式。
* 它可以帮助你建立代理来只用一个单独的调用就能queue, forward, 或者 capture message. 代理能够简化网络的结构。
* 它对具体的message的格式没有要求。 大小从0到GB的大小。 你可以自由定义message的格式。
* 对于网络错误，它会自动重传。
* 它比较节能高效。


References
-----------

http://zguide.zeromq.org/page:all


.. raw:: html

    <!-- 多说评论框 start -->
    <div class="ds-thread" data-thread-key="zeromq01.rst" data-title="ZeroMQ Documentation 01 -- 简介" data-url="http://superjom.duapp.com/program-language/zeromq01.html"></div>
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
