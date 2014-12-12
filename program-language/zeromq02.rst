ZeroMQ Documentation 02 -- Sockets and Patterns
==================================================
.. sectionauthor:: Superjom <yanchunwei {AT} outlook.com>

*2014-12-11*

PS： 发现已经有ZeroMQ的中文翻译文档了。 看了看，感觉翻译的质量还可以。

正好，我也只是做一个笔记，这样就没有翻译的压力了，尽可能的保留专业名词，记录几个自己感兴趣的重点就可以了。 


在上一章中介绍了ZeroMQ的基础，和一些模式的例子： request-reply, pub-sub.

在这一章中，我们会学习如何在实际的应用中使用这些工具。

The Socket API
---------------
在网络编程中，socket是事实上的标准应用程序的接口。

ZeroMQ也基于socket，并且把基于socket的开发变得有趣。

ZeroMQ的socket很容易掌握，一个生命周期由四部分组成：

* 创建和销毁套接字， zmq_socket(), zmq_close()
* 配置socket，给它们设置选项 zmq_setsockopt(), zmq_getsockopt()
* 创建ZeroMQ与网络的连接，将ZeroMQ socket接入网络， zmq_bind(), zmq_connect()
* 通过在socket上写入和接收消息来传递数据，zmq_send(), zmq_recv()

在ZeroMQ中，socket都是void的指针，而message是一种数据结构。

把socket接入网络拓扑中
----------------------
要在两个节点间建立连接，你可以在其中一个节点使用zmq_bind()，在另外一个节点中使用zmq_connect().

依照常规，使用zmq_bind()的是一个server，它有一个公共的地址。 而使用zmq_connect()的节点是一个client，它的地址没有要求。

ZeroMQ的连接和经典的TCP的连接有些不同：

* 它们对具体的传输协议透明，inproc, ipc, tcp, epgm
* client的zmq_connect()和server的zmq_bind()的前后顺序没有要求。并不需要connect前已经有server bind一个地址啥的。
* ZeroMQ的message队列是异步的。
* 一个socket可能有很多的输入输出连接。
* 网络连接是在后台进行，而且如果出现问题，ZeroMQ会自动重连。
* 应用程序无法直接调用这些连接，它们是被封装在socket下的。

假如我们先启动client，之后再启动server。在传统的网络中会出现问题(client不能连接一个不存在的server)。

但是，ZeroMQ允许我们随意启动和停止各部件。

一个server可以bind到很多多样的端点，而只需要一个socket。 这里的多样表示的是多种协议，多个地址的自由组合。

如::

    zmq_bind(socket, "tcp://*5555");
    zmq_bind(socket, "tcp://*:9999");
    zmq_bind(socket, "ipc://myserver.ipc");

但你不能重复绑定到同样的端点，这将导致异常。

同样地，client也能够通过zmq_connect()连接到任意数量的终端。


发送和接收Message
--------------------
你通过zmq_send()和zmq_recv()方法来发送和接收message。 

ZeroMQ的输入/输出模式和TCP的输入和输出模式有很大不同。

我们来看TCP socket与ZeroMQ socket在携带数据时的主要区别：

* 类似UDP，ZeroMQ socket携带的是message，而不是TCP的字节流。 Message是特定长度的二进制数据块。
* ZeroMQ socket在后台线程中处理输入/输出。 也就是，message被接收到本地的input queue，或者被本地的Output queue发出。 具体的message的行为与当前应用的状态无关（透明）
* ZeroMQ socket内置1-to-N的传播。

还是ZeroMQ的异步性，本地的input queue和output queue将会缓存message。 所以，当调用了zmq_send()并返回，也不代表message已经被发送出去了。

输入/输出线程
-------------
前面有提过ZeroMQ采用input queue和output queue来缓存message并实现异步传输和接收。 那么中间维护queue的就是具体的线程了。

可以在创建context时，指定输入/输出线程的个数:

.. code-block:: c
    :linenos:

    void* context;
    context = zmq_init(1);  // one input/output thread

也可以在后面再修改线程个数：

.. code-block:: c
    :linenos:

    int io_threads = 4;
    void * context = zmq_ctx_new ();
    zmq_ctx_set (context, ZMQ_IO_THREADS, io_threads);
    assert (zmq_ctx_get (context, ZMQ_IO_THREADS) == io_threads);

每个线程可以处理每秒1G的数据，可以通过这个指标确定线程个数。

通过异步性，ZeroMQ应用程序的一个socket可以同时处理成千上万个连接。

如果你只是进程间通信，而不涉及到远程的IO，那么将线程个数设为1会好一点。

消息模式
----------
ZeroMQ向节点快速高效地传输整块的数据。

你可以将线程、进程映射为节点。 ZeroMQ为你的应用程序提供了一个单一的socket API，这个API对于具体的传播协议(in-process, inter-process, TCP, multicast)是是透明的。

ZeroMQ能够自动重连节点。

它在发送端和接收端将message缓存到queue中，并且会小心地管理queue以防止内存爆掉。 它采用无锁技术，将所有的I/O操作放入后台。

ZeroMQ感觉特定的模式来queue和route message. 就是这些模式，融合了开发者的经验和智慧。

ZeroMQ的模式是通过成对的匹配类型的socket实现的。
要理解ZeroMQ的模式，你许哟啊理解socket的类型，以及它们如何配合。

内置的ZeroMQ核心模式有：

* Request-reply, 把一组client连接到一组service. 这是一种的过程调用，以及任务分发模式。
* Pub-sub, 将一组publisher连接到一组subscriber. 这是一种数据分发模式。
* Pipeline, 以fan-out/fan-in模式来连接节点，可能会有多个步骤和循环。 这是一个并行任务分发和收集的模式。
* Exclusive pair: 两个socket专享连接的模式。 这是同一个进程中两个线程连接的模式。 不要与正常socket对混淆。

对于connect-bind pair，有一些可行的socket的组合：

* PUB and SUB
* REQ and REP
* REQ and ROUTER
* DEALER and REP
* DEALER and ROUTER
* DEALER and DEALER
* ROUTER and ROUTER
* PUSH and PULL
* PAIR and PAIR

任何其他的组合都会发生错误。

使用Message
----------------
libzmq核心库实际上有两个API来发送和接收信息，zmq_send() 和 zmq_recv().

但是zmq_recv()在处理变长的消息长度时，会有问题：它会根据你提供的buffer的长度截断消息。所以，存在另外一个API来更灵活地控制zmq_msg_t结构：

* 初始化message: zmq_msg_init(),  zmq_msg_init_size(), zmq_msg_init_data()
* 为了读取一个message，你可以使用zmq_msg_init()来建立一个空的message，然后传给zmq_msg_recv().
* 从一个个新的数据来创建mesage，你可以使用zmq_msg_init_size()来创建一个message，同事分配一定大小的数据块。 然后使用memcpy来填充数据，最后将message传给zmq_msg_send().
* 要释放一个message，可以调用zmq_msg_close()。 这会终止引用，并且最终ZeroMQ会销毁这个message。
* 要取得message的内容，你使用zmq_msg_data(). 要知道内容的大小，使用zmq_smg_size()
* 要取得message的内容，你使用zmq_msg_data(). 要知道内容的大小，使用zmq_msg_size()
* 在你传递一个message给zmq_msg_send()之后，ZeroMQ会清空message，将size设置为0. 你不能重复send同一个message，也不能在send后取message的内容。
* 这些规则不适用于zmq_send()和zmq_recv()，在那些情况下，你直接传递字节数组，而不需要依靠message。

如果你想要重复发送同样的message内容，那可以创建第二个message，使用zmq_msg_init()来初始化它，然后使用zmq_msg_copy()来创建第一个message的副本。 **这个不会复制内存，而只是增加引用计数** . 同样的方法，你复制多次message，就可以send多次。 message内容的引用计数会在send后自动减少，在最后一个message被sent或者close之后，message的内容内存会被销毁。

ZeroMQ也支持multipart message，这使得你可以发送和接收一系列的frame.
这在实际应用中非常广泛，在后续也会重点介绍到。

Frames(或者叫message parts)，是ZeroMQ message的基本格式。 
一个Frame是长度指定的数据块。 

最初，一个ZeroMQ message就是一个frame。在之后，我们用multipart message扩展了它。变化只是添加了一个代表"more"的位。 
ZeroMQ API现在可以让你写message的同时设置"more"标记，从而可以检查这个标记以确定一个message后续是否还有更多的frame。

一些参照：

* 一个message可以有一个或多个part
* 这些part被称为"frame"
* 每个part都是一个zmq_msg_t对象
* 在低层次的API中，你单独地发送或者接收每个part
* 在高层次的API中，提供了发送multipart message的封装

其他一些有用的建议：

* 你可以发送0长度的message，比如，作为进程间的一种信号
* ZeroMQ能够保证整体性地传送了一个message的所有part，全部发送，或者一个也不发送
* 由于异步性，multipart message并不会立刻传送，而是过一个不确定的时间之后。 所以应该保证其大小不会爆内存
* 一个message(single or multipart)不能爆内存。 如果你要发送不定长的文件，即使使用multipart message，将file拆成多块来传送，也不会降低整体的内存消耗。
* 对于没有提供自动对象析构的语言，你必须调用zmq_msg_close().

References
-----------
http://zguide.zeromq.org/page:all

baidu: ZeroMQ-Guard 翻译(中文版)   *翻译有一点怪怪的*


.. raw:: html

    <!-- 多说评论框 start -->
    <div class="ds-thread" data-thread-key="zeromq02.rst" data-title="ZeroMQ Documentation 02 -- Sockets and Patterns" data-url="http://superjom.duapp.com/program-language/zeromq02.html"></div>
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
