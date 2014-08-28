#### slab分配机制 ####

----------------------------

##### slab分配机制是什么？#####

slab内存分配是一种用来高效管理内存的机制，可以避免频繁申请和释放内存带来的内存碎片问题，
同时提高内存分配效率。最早是在Solaris2.4的内核引入这个算法， 后面广泛应用在许多Unix和
类Unix(如FreeBSD)的操作系统上， 而在linux上直到2.6.23才成为默认的分配方式。
slab分配最简单的理解, 就是每次使用内存时，一次申请一片大内存，当需要使用内存时，
直接从slab分配，释放也是放回slab, 而不必从操作系统不断分配和释放。

为了可以重复分配和释放slab, 我们正常时把一个slab切分成长度相等的item, 比如slab是1M，每个
item是1024byte, 那么一个slab就是切分成1M/1024 = 1024个item. 

简单版的slab如下图所示:

![image](https://github.com/git-hulk/fatcache-note/blob/master/snapshot/slab_1.png)

##### slab class #####

我们申请内存的时候，需要不同长度的内存，这就需要不同item长度的slab, 而slabclass就是这些
不同长度item的slab的集合。在一般设计slabclass的时候，可以把slabclass看作一个item长度递增
的数组，并把slab放到对应的队列中。

如slabclass有3级， 假设每个slab是1M, 第一级item长度是100byte, 每级增长因子是1.2。

第一级item的长是100bytes, item count = 1M/100

第二级item的长度是100byte＊1.2 = 120byte, item count = !M/120

第三级item的长度是120*1.2 = 144bytes, item count = 1M/144

以此类推.

当需要申请80byte的内存时候， 从第一级开始找，直到遇到第一个比它大的slab, 这里就是第一级的slab。
如果要申请130bytes的内存， 就是找到第三级.

回到fatcache, 在[配置参数](./configure.md)里面-f 选项就是配置增长因子， -I 配置slab大小。

##### slab的三种状态 #####

看过上面的内容之后，应该能知道slab大概的样子，以及Slabclass是干嘛的。 slabcloass每一级slab可以分配
的item数量是固定的， 所以slab可能会有三种状态: free slab(完全没有使用)， partial slab(部分使用),
full slab(全部使用).




