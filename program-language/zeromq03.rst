ZeroMQ Documentation 03 -- Sockets and Patterns
===================================================
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

*2014-12-12*

Multipart Messages
---------------------
我们能够使用ZeroMQ建立multipart message.

当你使用multipart message时，每个部分都是zmq_msg对象。 当你发送一个包含五个part的message，你看必须construct, send, 和销毁五个zmq_msg对象。 

这里是send multipart message的一个例子

.. code-block:: C++
    :lineno:

    zmq_msg_send(&message, socket, ZMQ_SENDMORE);
    ...
    zmq_msg_send(&message, socket, ZMQ_SENDMORE);
    ...
    zmq_msg_send(&message, socket, 0);

下面是你如何接收和处理message中所有的部分:

.. code-block:: C++
    :lineno:

    while(true) {
        zmq_msg_t message;
        zmq_msg_init (&message);
        zmq_msg_recv (&message, socket, 0);
        // process the message frame
        ...
        zmq_msg_close (&message);
        if(!zmq_msg_more (&message)) 
            break;  // last message frame
    }

对于multipart message，你需要知道的一些信息：

* 当你发送multipart message时，由于异步性，信息的第一个part实际的网络传输只有你send了最后一个part之后才可能开始
* 当你使用 zmq_pool()，当你接收到了message的第一个部分时，所有其他的part实际上也到达了
* multipart message的所有部分都会同时被receive，你可能收到了所有part，或者一个也没有收到
* multipart message的所有part会在你send最后一个部分后才会发出，信息的所有part在此之前都会存储在内存中，所以，在使用multipart message时，确认你它的所有部分加起来不会爆内存
* 除了关闭socket之外，没有办法取消一个已经部分发送的message

动态发现问题
------------
当你设计更大的分布式架构时，碰到的一个问题就是动态发现问题。

也就是，每个组件如何发现对方？这个在组件不断加入和退出时尤其地麻烦，所以我们称之为"动态发现问题"』

这个问题有很多解决方法，其中最简单的就是硬编码，人工更新网络的变化。 
也就是，当你添加了一个新的组件，那么重新配置这个网络。

一个小型的 Pub-Sub 网络:

.. image:: https://github.com/imatix/zguide/raw/master/images/fig12.png

现实中，这会导致网络架构原来越笨重和脆弱。 

假设你有一个publisher和一百个subscribers. 你通过配置一个publisher的端点来连接每个subscriber到这个publisher.

这很简单，subscriber动态增长也不会出现问题。 

现在如果你添加更多的publisher. 突然，任务变得麻烦起来。 如果你要把已有的subscriber连上这些publisher，就比较麻烦了。

如果publisher也有很高的动态性，那么动态发现的成本就会变得越来越高。

一个带代理的Pub-Sub网络:

.. image:: https://github.com/imatix/zguide/raw/master/images/fig13.png

这个麻烦是有很多方式解决的，最简单的就是增加一个中介。 
也就是，在网络中添加一个静态的端点，让其他的节点都连接它。

ZeroMQ并没有内置消息代理，但是用它实现中介很容易。

你也许想知道，如果所有的网络最终都会变得很大以致需要中介，那为什么不为所有的应用添加一个消息代理呢？
对于初学者，这是个公平的妥协。只是每次都使用一个星型拓扑，不纠结性能的话，往往没有问题。
然而，消息代理是一种贪婪的东西，将它们作为中央的代理，有点过于复杂，依赖状态，最终会出现问题。

一个更好的想法是，把中间作为简单的无状态message开关。
一个好的例子就是HTTP代理； 它客观存在，而没有任何特殊的角色。
添加一个pub-sub代理能够解决我们之前说的动态发现问题。
我们将代理设在我们网络的中间，这个代理打开一个XSUB socket，一个XPUB socket，而且把这些socket连到可知的IP地址和端口上。 然后，所有其他的进程连上这个代理，而不是直接互联。 

扩展的Pub-Sub

.. image:: https://github.com/imatix/zguide/raw/master/images/fig14.png

我们需要XPUB和XSUB socket，因为ZeroMQ从subscriber到publisher实现订阅。
XSUB和XPUB就像SUB和PUB，只是它们把订阅作为特殊的message.
代理通过从XSUB socket读取，再将它们写入到XPUB socket，来将这些订阅message从subscriber的一边传给publisher的一边。

Shared Queue (DEALER and ROUTER socket)
-----------------------------------------
在Hello World client/server应用中，一个client与一个服务就行交互。 
然而在实际的应用中，我们需要往往需要实现多个client和多个服务的交互。
这需要我们扩大service的能力(比如使用多线程，多进程，多节点).
唯一的约束是，serverce必须是无状态的，所有的状态保存在request，或者在某些如数据库的共享存储空间中。

Request 分发:

.. image:: https://github.com/imatix/zguide/raw/master/images/fig15.png

有两种方式来将多个client连接到多个service. 其中野蛮的方式是，将每个client连接到多个service端点上。 一个client socket能够连接到多个service端点， REQ socket之后会在service进行分发。
比如，你将一个client连接到三个service端点; A, B和C。 
这个client进行了四个request R1, R2, R3, R4. 其中R1和R4面向service A， R2面向B，R3面向service C。

这样的设计使得你能够很方便地加入client。 你也能添加更多的service，但是要麻烦很多， 但是每个client都必须知道service的拓扑结构。 你需要遍历所有的client，告知它们新的service。

这肯定不是我们希望在ZeroMQ中实现的。 所以我们将要写一个小型的message队列代理来提供灵活性。 
这个代理绑定到两个端点，从client端点到service端点。 

当你使用REQ来与REP交流时，你得到一个同步的request-reply会话。
Client发送一个request，service读取request，然后回复。 
Client再读取回复。 
如果client或者service尝试做点其他的事情(比如，不等回复，连续发送两个request)，会出现一个错误。

但是我们的代理需要是非阻断的。 
很明显，我们能够使用zmq_poll()来等待任何一个socket上的活动，但是我们不能使用REP和REQ。

扩展的Request和reply

.. image::  https://github.com/imatix/zguide/raw/master/images/fig16.png

幸运的是，有两个叫做DEALER和ROUTER的socket来帮助你做非阻塞的request-response. 
你可以查看第三章高级的Request-Reply模式来查看DEALER和ROUTER socket如何帮你处理异步的request-reply流。
现在，我们将要看看DEALER和ROUTER如何通过中介来帮助我们扩展REQ-REP，也就是我们一个小的代理。

在这样一个小的扩展的request-reply模式中，REQ与ROUTER交流，DEALER与REP交流。 
在DEALER和ROUTERK之间，我们必须有代码来将message从一个socket发送到另外一个socket。

这个request-reply代理绑定两个端点，一个是client，另外一个是worker。 
为了测试这个代理，你需要修改你的workder以判定它连接到了worker上。

这里有一个人client的例子：

.. code-block:: C++
    :lineno:

    #include "zhelper.hpp"

    int main(int argc, char * argv[]) 
    {
        zmq::context_t context(1);

        zmq::socket_t requester(context, ZMQ_REQ);
        requester.connect("tcp://localhost:5559);

        for(int request = 0; request < 10; request++) {
            s_send (request, "hello");
            std::string s = s_recv(requester);
            std::cout << "Received reply" << request
                << " [" << s < "]" << std::endl;
        }
        return 0;
    }

下面是worker的例子：

.. code-block:: c++
    :lineno:

    #incude "zhelpers.hpp"

    int main(int argc, char * argv[]) {
        zmq::context_t context(1);

        zmq::socket_t responder(context, ZMQ_REP);
        responder.connect("tcp://localhost:5560");

        while(true) {
            std::string s = s_recv(responder); 
            std:cout << "Received request: " << s << std::endl;

            // do some 'work'
            Sleep(1);
            // send reply back to client
            s_send(responder, "World");
        }
        return 0;
    }

下面是request-reply代理的例子：

.. code-block:: c++
    :lineno:

    int main(int argc, char * argv[]) {
        // prepare context and sockets
        zmq::context_t context(1);
        zmq::socket_t frontend(context, ZMQ_ROUTER);
        zmq::socket_t backend (context, ZMQ_DEALER);

        frontend.bind("tcp://*:5559"); //*
        backend.bind("tcp://*:5560"); //*

        // initialize poll set
        zmq:pollitem_t items [] = {
            { frontend, 0, ZMQ_POLLIN, 0},
            { backend,  0, ZMQ_POLLIN, 0}
        };

        while(true) {
            zmq::message_t message;
            int64_t more;   // multipart detection

            zmq::poll (&items[0], 2, -1);

            if(items[0].revents & ZMQ_POLLIN) {
                while(true) {
                    // process all parts of the message
                    frontend.recv(&message);
                    size_t more_size = sizeof(more);
                    frontend.getsockopt(ZMQ_RCVMORE, &more, &more_size);
                    backend.send(message, more? ZMQ_SENDMORE, 0);

                    if (!more) break;
                }
            }
            if (items[1].revents & ZMQ_POLLIN) {
                while(true) {
                    // process all parts of the message
                    backend.recv(&message);
                    size_t more_size = sizeof(more);
                    backend.getsockopt(ZMQ_RCVMORE, &more, &more_size);
                    frontend.send(message, more? ZMQ_SENDMORE, 0);
                    if(!more) break;
                }
            }
        }
        return 0;
    }

Request-Reply代理:

.. image:: https://github.com/imatix/zguide/raw/master/images/fig17.png

使用一个request-reply代理能够使你的client/server架构更加容易扩展，因为client看不到worker，worker也看不到clients，唯一静态的节点就是中间的代理。

ZeroMQ内置的代理函数
----------------------
在上一章的rrbroker是非常有效的可重用的。

它使我们能够高效地创建pub-sub和共享队列。
ZeroMQ用函数zmq_proxy()内置了这个方法::

    zmq_proxy(frontend, backend, capture);

当我们调用了zmq_proxy，就如同开始了rrbroker的循环。 

下面实现一个message queue的例子：

.. code-block:: c++
    :lineno:

    //
    //  Simple message queuing broker in C++
    //  Same as request-reply broker but using QUEUE device
    //
    // Olivier Chamoux <olivier.chamoux@fr.thalesgroup.com>

    #include "zhelpers.hpp"

    int main (int argc, char * argv[])
    {
        zmq::context_t context(1);

        //  Socket facing clients
        zmq::socket_t frontend (context, ZMQ_ROUTER);
        frontend.bind("tcp://*:5559"); //*

        //  Socket facing services
        zmq::socket_t backend (context, ZMQ_DEALER);
        zmq_bind (backend, "tcp://*:5560"); //*

        //  Start built-in device
        zmq_device (ZMQ_QUEUE, frontend, backend);
        return 0;
    }
    

    
