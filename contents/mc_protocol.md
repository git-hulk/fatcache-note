#### MC 协议 ####

-----------------

MC指的是memcached, 它支持文本和二进制协议，fatcache只使用文本协议。 我们这里不对二进制协议做
解释，想要的可以看 [mc二进制协议](https://code.google.com/p/memcached/wiki/BinaryProtocolRevamped)

我们常用的协议可以分为以下几类:

*   Storage Commands, 写相关命令
*   Retrieval Commands, 读相关命令
*   Delete Comands 删除命令
*   Delete Comands 修改过期时间命令
