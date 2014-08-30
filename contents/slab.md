#### slab分配机制 ####

----------------------------

##### slab分配机制是什么？#####

slab内存分配是一种用来高效管理内存的机制，可以避免频繁申请和释放内存带来的内存碎片问题，
同时提高内存分配效率。最早是在Solaris2.4的内核引入这个算法， 后面广泛应用在许多Unix和
类Unix(如FreeBSD)的操作系统上， 而在linux上直到2.6.23才成为默认的分配方式。
slab分配最简单的理解, 就是每次使用内存时，一次申请一片大内存，当需要使用内存时，
直接从slab分配，释放也是放回slab, 而不必从操作系统不断分配和释放。

为了可以重复分配和释放slab里面的item, 我们正常时把一个slab切分成长度相等的item, 比如slab是1M，每个
item是1024byte, 那么一个slab就是切分成1M/1024 = 1024个item. 

简单版的slab如下图所示:

![image](https://github.com/git-hulk/fatcache-note/blob/master/snapshot/slab_1.png)
<br />
<br />

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
<br />
<br />

##### slab的三种状态 #####

看过上面的内容之后，应该能知道slab大概的样子，以及Slabclass是干嘛的。 slabcloass每一级slab可以分配
的item数量是固定的， 所以slab可能会有三种状态: free slab(完全没有使用)， partial slab(部分使用),
full slab(全部使用).

最开始， 由于没开始使用，只会有free slab， 当使用一段时间后，就转换为partial slab, 如果这个slab已经被分配完，
则会变为full slab.

在fatcache中， 会维护这三种状态的slab, 其中free slab， full slab队列是slabclass所有级公用，partial slab是每一级都有一个
队列。

当某一级需要分配空间， 会经历下面的步骤：
```
  1. 如果这一级partial slab队列不为空， 先从这个队列里面取, 如果不为空，直接分配， 判断slab是满了，是？移到full slab队列。
  2. 如果这一级parital slab队列为空， 从free slab队列获取一个slab, 放到这一级slab，重新分配。
```
我们以内存的slab作为例子， 在`fc.c`里面, 我只列出关键部分:
```c

struct item *
slab_get_item(uint8_t cid) {
    
    ....
    
    if (!TAILQ_EMPTY(&c->partial_msinfoq)) {
        /* 如果不为空，则从partial slab队列里面取一个slab, _slab_get_item在下面定义*/
        return _slab_get_item(cid);
    }
    
    .....
    
    if (!TAILQ_EMPTY(&free_msinfoq)) {
        /* 如果parital 为空则从free slab队列里面取一个，放到partial slab队列*/
        sinfo = TAILQ_FIRST(&free_msinfoq);
        ASSERT(nfree_msinfoq > 0);
        nfree_msinfoq--;
        TAILQ_REMOVE(&free_msinfoq, sinfo, tqe);
        
        // 插入到partial slab队列
        TAILQ_INSERT_HEAD(&c->partial_msinfoq, sinfo, tqe);
        
        ...
        
        /* 仍然进入从partial slab队列获取的逻辑， 在下面函数 */
        return _slab_get_item(cid);
    }
}


static struct item * _slab_get_item(uint8_t cid) {
    ...
    // cid是class id, 指的是在cid这层分配slab.
    c = &ctable[cid];
    // 获取第一个partial slab info
    sinfo = TAILQ_FIRST(&c->partial_msinfoq);
    // 这个slab应该是不为满的状态
    ASSERT(!slab_full(sinfo));
    
    /* 拿到slab地址 */
    slab = slab_from_maddr(sinfo->addr, true);
    
    /* 从partial slab分配一个item */
    it = slab_to_item(slab, sinfo->nalloc, c->size, false);
    it->offset = (uint32_t)((uint8_t *)it - (uint8_t *)slab);
    it->sid = slab->sid;
    sinfo->nalloc++;
    
    /* 分配之后，判断slab是否满了， 如果满了，放到full slab队列中 */
    if (slab_full(sinfo)) {
        /* move memory slab from partial to full q */
        TAILQ_REMOVE(&c->partial_msinfoq, sinfo, tqe);
        nfull_msinfoq++;
       TAILQ_INSERT_TAIL(&full_msinfoq, sinfo, tqe);
    }
}


```
<br />
<br />

##### memory and disk slab #####

fatcache的slab来自两个地方，一个是内存，另一个是磁盘。在启动的时候，可以指定内存slab大小，
默认是64M, 我们也会指定分区，磁盘slab大小跟分区大小一致。然后再把内存和磁盘的这片空间，
切分为一块块slab, 再添加到free slab队列。

```
fatcache 写
1. 如果还有内存slab未写满，直接分配地址，返回。
2。 如果所有内存slab已经写满， 从内存full slab队列头部，剔除一个slab, 交换到磁盘slab，
空出内存slab, 重新分配。
```
我们可以看到，写的时候一定是写在内存里面，而读是，根据slab所在位置，直接读取，所以我们可以知道，
只有写才会让内存slab和磁盘slab数据进行交换， 读不会影响数据所在位置是在内存还是磁盘，换句话说，
只有更新才会影响fatcache的数据的热度。

看完这篇， 应该要理解的几个东西:
```
1. slab 和 slabclass
2. slab 三种状态，以及三种状态的转换
3. fatcache不仅有内存slab还有磁盘slab.
```

接下来讲一下，另外一个fatcache很核心的东西，[索引](./itemx.md)
