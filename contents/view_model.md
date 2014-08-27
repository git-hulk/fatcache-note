#### 访问模型 ####

------------------

这一节主要介绍一下fatcache的访问模型， 我们简单以get/set为例，大概过一下整体的访问模型，
在后面的章节中，能对fatcache的访问流程有个整体认识， 完整的一次请求过程， 在后续的章节详细介绍。

下图是fatcache大体的架构图:

![image "fatcache view"](https://github.com/git-hulk/fatcache-note/blob/master/snapshot/fatcache_view.png)

##### get #####

> 1. 根据key，在hashtable找到itemx, 也就是索引， 如上图找到itemx3.

> 2. itemx里面有sid，也就是slab id, 根据slab id,再找到slabinfo.

> 3. slabinfo 里面有一个mem，如果为1表示在内存的slab, 否则在磁盘中.

> 4. 根据slabinfo所在的位置和索引的偏移，读取到数据，返回.

<br />
<br />
##### set #####


> 1. set操作先从itemx空闲队列，获取一个itemx.

> 2. 从内存slab获取item空间? 没有内存slab item可用, 把最老的内存slab交换到磁盘， 留出内存slab

> 3. 写入数据，返回.

<br />

上面介绍了最简单的一个读写流程，后面再来一一细化每个过程。

下一节介绍 [网络框架](./network.md)
