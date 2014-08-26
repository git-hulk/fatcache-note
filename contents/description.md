#### 简介 ####

------------------

fatcache是由twitter，TA是基于SSD, 同时使用memcaced的协议，但只支持文本协议(memcached 支持二进制协议).

对于SSD开发来说, 有两种不同的理解:
> 1. SSD是更快的磁盘.
> 2. SSD是内存的扩展，TA的角色就是可以认为以更低价的存储来扩展内存容量.

fatcache作为cache, 所以对TA来说，SSD应该是内存扩展, 以廉价的方式来代替内存.

--------------------------------

##### 为什么选择SSD而不是直接使用内存? #####

>1. 内存虽然比SSD快了很多, 但是同等密度的容量，价格也高出很多.
2. 虽然时间推移, 数据会越来越多, 需要不断对内存不断横向扩容.
3. 对于数据来说, 只有很少的一部分数据会被经常访问, 我暂时叫热数据, 当冷数据(访问比较少)比例大大高于热数据时, 这种存储方式就很不划算.



##### 需要提前理解的东西? ######
> 1. [memcache 协议](https://github.com/memcached/memcached/blob/master/doc/protocol.txt), fatcache是基于mc的文本协议来开发.
> 2. [slab机制](http://en.wikipedia.org/wiki/Slab_allocation). 
  

##### fatcache Vs Memcached #####
  + fatcache的实际存储数据可能在内存或者SSD, Memcached全部内存.
  + fatcache多了一层索引。 索引除了快速判断key是否存在，同时用来记录数据所在位置，快速读取value.
  + MC 哈希表存储真实数据, fatcache哈希表只存储索引信息.
  + fatcache不支持二进制协议.
