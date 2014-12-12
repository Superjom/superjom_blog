排序 = 匹配
==============
.. sectionauthor:: Superjom <yanchunwei@outlook.com>

|today|

如果遇到类似这样的问题：

存在一个文本文件，每行一段字符，求出其中重复度大于n的行并输出。

解决这个问题就需要一种匹配的方法，具体匹配，可以从Hashmap或排序的角度去思考。

Hashmap方法
---------
从前往后扫描数据同时建立hashmap记录命中次数，最终扫描Hashmap输出命中数值>n的记录

复杂度： O(N)    空间占用：O(N)

实现::

    awk '{
        dic[$0] ++;
    } END {
        for (line in dic) {
            if (dic[line] > n) {
                print line;
            }
        }	
    }' file

排序方法
----------
首先按行排序，然后前后相同的行构成的一段记录作为一个记录段，记录段长度>n的输出其中首行。 

复杂度：O(NlogN) 空间占用：O(N)

实现::

    sort file | awk '{
        line = $1;
        if(last_line == line) {
            last_count ++;
        } else {
            last_line = line;
            if (last_count > n) print last_line, last_count;
            last_count = 0;
        }
    } END {
        if( last_count > n) print last_line, last_count;
    }'

如此，至少在单机情况下，Hashmap方法要比排序方法要经济一些。 

但是类似这类匹配问题，排序的方法似乎更加常用一点，在hadoop处理里面，更是sort的天下，个人认为，应该有以下两个原因：

一、排序的方法更加灵活
考虑这样一个匹配问题，两个文本文件，一个文件里每行一个url，比如"http://write.blog.csdn.net/postedit"，另外一个文件，每行一个domain，如"csdn.net"，现在需要将两个文件的记录匹配起来，即找到domain对应的url。

数据示例:

url.list.txt::

    http://zhongyi.sina.com/news/myys/20143/194296.shtml
    http://esf.sina.com.cn/
    http://write.blog.csdn.net/postedit
    http://news.sina.com.cn/c/t/20140402/1145177.shtml
    http://view.news.qq.com/original/intouchtoday/n2751.html

domain.list.txt::

    qq.com
    sina.com.cn


如果用Hashmap的方法，必须先对每个url取出domain吧，这个规则就复杂一点了，二级域名，以及domain具体从后往前算到第几个key都是问题。 

 但是排序能够很巧妙地绕过确定domain的这个问题，首先从url取出主战地址是比较简单的，比如 "http://write.blog.csdn.net/postedit" -> "http://write.blog.csdn.net" ，
只要从前扫描到"/","?"类似的特殊符号截止便可。 
将url.list.txt中每个url提取出来的主站地址加上domain.list.txt 中的记录从尾向前排序便
可。

最终自然会得到：

sina.com.cn::

    http://zhongyi.sina.com/news/myys/20143/194296.shtml
    http://esf.sina.com.cn/
    http://news.sina.com.cn/c/t/20140402/1145177.shtml

qq.com::

    http://view.news.qq.com/original/intouchtoday/n2751.html
    http://write.blog.csdn.net/postedit

当然，结尾那个csdn的url未命中可以直接舍去。
如此排序方法更加简便，而且能够将二级域名的url也会排列在一起，方便进一步的处理。

二、排序方法的输出方便后续处理

例如在hadoop的环境里，数据需要进行多步的处理。 
那么中间数据格式采用文本的方式是比较稳妥的。 
Hashmap方法作为中间步骤的输出必须进行转化，
比如编程"key \t value"的方式，而且这个过程容易丢失原始数据的信息。 而排序方法的则可以直接输出。

比如这样一个案例： 
将上面提到的两个文件的数据量均扩大到必须用集群才能处理的程度，那么如何去解决这样一个问题呢？ 

**利用Hashmap:**

首先限于内存，全局Hashmap的方法肯定是用不了的了，但是可以利用局部Hashmap，比如将两个数据混合起来，
map阶段产生domain，reduce阶段按照domain分桶（还是扯到产生domain的问题上），之后在局部利用Hashmap输出，最终用一轮或半轮mapreduce汇合结果。

**利用排序：**

可以考虑排序就是一个将行全局反向排序的问题，分桶，最终输出全局排序的结果，再另外处理一下结果。
例外，对于大数据，排序出来的结果有这样的优势：
命中的domain后面肯定会接着其附属的所有链接（包括子域名下的），
而所有子域名后面必然接着其子链接，这个结果是全局有序的，如果后期需要对子域名方面的信息统计，直接扫描之前的结果就可以了，而Hashmap方法的输出信息没有这么全面。



