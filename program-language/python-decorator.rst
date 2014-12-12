.. highlight:: python
   :linenothreshold: 5
===================
python 装饰器
===================
python decorator 被描述为一种语法糖，提供了一种更加优美的编程方式。
如果将一些代码转化为 decorator 方式，代码的结构会非常清楚。

以下首先从基础的函数来介绍：

不定参数的函数
----------------
在python，list, tuple, dict等容器贯穿了基本库始终。

比如一个函数的表示包括两个部分：

1. 一段程序
2. 参数

其中参数部分，由list 或者 dict来表示。

函数的参数可以是显式的::

    def func():
        pass

    def func(a):    # 参数被传为list: [a,]
        pass

    def func(name='superjom'):  #参数被传为dict: {'name': name}
        pass


参数也可以是不定的::

    def func(*args):
        pass

    func(1, 2, 3)   # 参数： args=[1,2,3]

    def func(*args, **argv):
        pass
    
    func(1,2, name='superjom')  # 参数： args=[1,2], argv={name:'superjom'}


修饰函数的函数
------------------
写一个debug函数，可以输出传输给一个函数的参数

.. code-block:: python
    :emphasize-lines: 9,18
    :linenos:

    def debug1(func, *args, **argv):
        print "passed parameters:", args, argv
        return func(*args, **argv)

    def afunc(name):
        pass

    # debug afunc
    debug1(afunc, name)


    def debug2(func):
        def _debug(*args, *kw):
            print "parameters:", args, kw
            func(*args, **kw)
        return _debug

    debug2(func)(name)

如上面的示例，debug函数有两种方式： debug1, debug2。
debug2 比 debug1 的好处是，debug2会返回一个函数，方便连接起来一系列操作：

.. code-block:: python
    :linenos:

    process2(debug2(func)) (name)

而 debug1 必须采用如下方式：

.. code-block:: python
    :linenos:

    process1(debug1(func, name))

上文说到装饰器是一种语法糖，采用debug2的定义方式更加简洁。

相应的模板是：

.. code-block:: python
    :linenos:

    def mydecorator(func):
        def _mydecorator(*args, **kw):
            # do something
            res = func(*args, **kw):
            return res
        return _mydecorator




decorator 的语法糖
----------------------

作为语法糖，装饰器有一个很漂亮的语法： `@dec`

上面定义的 `debug2` 可以如下方式使用：

.. code-block:: python
    :linenos:

    @debug2
    def func(name): # equal to call debug2(func) (name)
        pass

等价于
            
.. code-block:: python
    :linenos:

    process2(debug2(func)) (name)

同样，也可以用装饰器将一系列操作连接起来：

.. code-block:: python
    :linenos:

    @process
    @debug2
    def func(name): # equal to call process(debug2(func)) (name)
        pass

.. note::
    
    装饰器调用的顺序，从里往外。

带参数的装饰器
----------------
装饰器就是一个修饰函数的函数，默认将被修饰的函数作为第一个参数传递。

如果需要传递一些参数给装饰器，可以采用类似的方式，在装饰器的外面添加一层函数： 

.. code-block:: python
    :linenos:

    # pass parameter to decorator
    def mydecorator(arg1, arg2):
        def _mydecorator(func):
            # args, kw are parameters passed to func
            def __mydecorator(*args, **kw): 
                # do some operations
                res = func(*args, **kw)
                return res
            return __mydecorator
        return _mydecorator

一个实例，通过给装饰器判断是输出args 还是 kw：

.. code-block:: python
    :linenos:

    def debug(show_args=False, show_kw=False):
        def _debug(func):
            def __debug(*args, **kw):
                if show_args:
                    print 'args', args
                if show_kw:
                    print 'kw', kw
            return __debug
        return _debug

    @debug(True, True)
    def func(*args, **kw):
        pass


装饰器的示例
--------------
给函数添加一个缓存：

.. code-block:: python
    :linenos:

    cache = {}
    cache_names = []

    # define a decorator
    def add2memory(size=1000):
        def _add2memory(func):
            def __add2memory(name):
                if name in cache:
                    return cache[name]

                cache_names.append(name)
                if len(cache) == size:
                    key_to_rm = cache_names[0]
                    del cache_names[0]
                    del cache[key_to_rm]
                    res = func(name)
                    cache[name] = res
                    return res
            return __add2memory
        return _add2memory            

    # call the decorator
    @add2memory(200)
    def func(name):
        pass
        






