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


