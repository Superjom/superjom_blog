.. highlight:: C
   :linenothreshold: 5
======================================
strcmp memset memcpy memmove memset 
======================================
之前面试，被问过这类实现题。 不会做，现在做个整理。 

strcpy
---------
字符串匹配

.. code-block:: c
    :emphasize-lines: 2,5
    :linenos:
    
    char *strcpy(char *to, const char* from) {
        assert( to && from);
        char *adr = to;
        // 赋值操作会返回左值
        while((*to++ = *from++) != '\0') { }
        return adr;
    }

memset
----------
以字节为单位，将数组设置为指定值。

.. code-block:: c
    :emphasize-lines: 3
    :linenos:

    void *memset(void *start, int cont, size_t size) {
        // 从大类型向小类型转换，必须强制转化
        unsigned char* p = (unsigned char*) start;

        while (size > 0) {
            *p ++ = (unsigned char) cont;
            -- size;
        }
        return start;
    }


memcpy
--------
拷贝不重叠的内存块

.. code-block:: c
    :emphasize-lines: 6
    :linenos:

    void *memcpy(void *to, void *from, size_t size) {
        assert(to && from);
        void *pto = (unsigned char*) to;
        void *pfrom = (unsigned char*) from;
        // 判断是否重叠
        assert((pto >= pfrom + size) || (pfrom >= pto + size));
        while (size-- > 0) {
            *pto++ = *pfrom++;
        }
        return to;
    }


memmove
-----------
拷贝内存块，并考虑重叠部分

.. code-block:: c
    :emphasize-lines: 12-17
    :linenos:

    void *memmove(void *to, const void *from, size_t size) {
        if(!(to && from && size>0) return NULL;
        char *pfrom = (char *) from;
        char *pto = (char *)to;
        //判断重叠
        if(to <= pfrom || pto >= pfrom + size) {
            for(int i=0; i<size; i++) {
                *pto++ = *pfrom++;   
            } // for
            
        // 避免覆写， 反向copy
        } else {
            pfrom += size;
            pto += size;
            for (int i=0; i<size; i++) {
                *pto -- = *pfrom --;
            }
        }
        return to;
    }

查了下网，说是 memmove 与 memcpy 的区别为重叠情况进行了特殊考虑

下面演示下原理:

比如如下重叠部分::

    from:   |--------------------------|
    to:             |+++++++++++++++++++++++++++|

可以看到， from 和 to 的内存块的中间部分有重叠。 

如果直接从前往后赋值，会直接改写重叠部分的内容（from部分的也会改变），直接影响后续的copy::

    from:   |-------**********--------|
    to:             **********++++++++++++++++++|
    pointer:        **********

可以看到，当pointer复制到中间重叠部分时，原始from内存块的内容发生了变换， 而且 **改写了下一步需要复制的内容** 。

而如果反向复制，能够避免改写将要复制的内容::

    from:   |--------------------*******
    to:             |++++++++++++*******++++++++|
    pointer:                     *******              

而from下面将要复制的内容都不会发生变化。
