#### (四) slab分配机制 ####
----------------------------

为什么要有slab?

我们前面说过，fatcache对于写磁盘做了优化，就是通过slab管理做到。slab管理简单来看，就是每次申请一块大内存(fatcache默认1M)，
不必每次去申请一小块内存，而是直接从已经申请的slab中拿出一小块来写，当不使用时，不用释放，而是放回slab，
这样，就可以大大减少申请和释放的频率。

fatcache把磁盘空间切割成一个个的slab, 每次写满一个slab之后，才会写回磁盘，也就是说每次写入磁盘的数据都是一个slab长度（1M）,
这样就可以不必每次写入一个kv都要写一次磁盘。

好处:

>1. 避免SSD的写放大，SSD每次写磁盘的最小单位是page(一般是4k),也就是即使是写入1byte，SSD内部也是相当于写4k,
这就是我们说的写放大，我们每次只会写入1个slab, 不会有写放大。

>2. 减少大量的小块的IO随机写， 同时把大量的随机写转化为顺序写，可以延长SSD的写寿命。

<br />
<br />

##### 4.1 slab分配机制？#####

slab内存分配是一种用来高效管理内存的机制，可以避免频繁申请和释放内存带来的内存碎片问题，
同时提高内存分配效率。最早是在Solaris2.4的内核引入这个算法， 后面广泛应用在许多Unix和
类Unix(如FreeBSD)的操作系统上， 而在linux上直到2.6.23才成为默认的分配方式。
slab分配最简单的理解, 就是每次使用内存时，一次申请一片大内存，当需要使用内存时，
直接从slab分配，释放也是放回slab, 而不必从操作系统不断分配和释放。

为了可以重复分配和释放slab里面的item, 我们正常时把一个slab切分成长度相等的item, 比如slab是1M，每个
item是1024byte, 那么一个slab就是切分成1M/1024 = 1024个item. 

简单版的slab如下图所示:

![image](https://github.com/git-hulk/fatcache-note/blob/master/snapshot/slab_1.png)
每次我们添加kv到fatcache的时候，直接从slab中拿到一个item，然后写入。

<br />
<br />

-----------------------

##### 4.2 slab class #####

由于并不是所有的kv的长度都是一样的，这就需要不同item长度的slab, 而slabclass就是这些
不同长度item的slab的集合。在一般设计slabclass的时候，可以把slabclass看作一个item长度递增
的数组，并把slab放到对应的队列中。当在添加数据的时候，找到能容纳kv长度的slabclass,然后从未使用完或者空闲的slab,
分配一个item空间。

如slabclass有n级， 假设每个slab是1M, 第一级item长度是100byte, 每级增长因子是1.2。

第一级item的长是100bytes, item count = 1M/100

第二级item的长度是100byte＊1.2 = 120byte, item count = 1M/120

第三级item的长度是120*1.2 = 144bytes, item count = 1M/144

以此类推.

当需要申请80byte的内存时候， 从第一级开始找，直到遇到第一个比它大的slab, 这里就是第一级的slab。
如果要申请130bytes的内存， 就是找到第三级.

回到fatcache, 在[配置参数](./configure.md)里面-f 选项就是配置增长因子， -I 配置slab大小。
<br />
<br />

##### 4.3 memory and disk slab #####

fatcache的slab来自两个地方，一个是内存，另一个是磁盘。在启动的时候，可以指定内存slab大小，
默认是64M, 我们也会指定磁盘分区。然后再把内存和磁盘的空间，切分为一块块slab, 再添加到free slab队列。
fatcache的内存slab, 充当的角色是写缓存，每次写的时候，一定是写在内存slab, 当写入时，发现没有内存slab,
就将最老的slab刷入磁盘。

```
fatcache 写
1. 如果还有内存slab未写满，直接分配地址，返回。
2. 如果所有内存slab已经写满， 从内存full slab队列头部，剔除一个slab, 交换到磁盘slab，
空出内存slab, 重新分配。
```
我们可以看到，写的时候一定是写在内存里面，而读是，根据slab所在位置，直接读取，所以我们可以知道，
只有写才会让内存slab和磁盘slab数据进行交换， 读不会影响数据所在位置是在内存还是磁盘，换句话说，
只有更新才会影响fatcache的数据的热度。
<br />
<br />

--------------------------

##### 4.4 slab在fatcache的使用 #####

看过上面的内容之后，应该能知道slab大概的样子，以及Slabclass是干嘛的。 slabcloass每一级slab可以分配
的item长度和数量是固定的， 所以slab可能会有三种状态: free slab(完全没有使用)， partial slab(部分使用),
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
我们以内存的slab作为例子， 在`fc_slab.c`里面, 我只列出关键部分:
```c

struct item *
slab_get_item(uint8_t cid) {
    
    ....
    /* item 索引使用耗尽，应该做一些剔除 */
    if (itemx_empty()) {
        status = slab_evict();
        if (status != FC_OK) {
            return NULL;
        }   
    }  
    
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

------------------------

##### 4.5 slab空洞问题 #####

我们细心看一下 `_slab_get_item`函数，不难发现每次都是从Slab，末端开始分配。也就是说，如果item删除，
我们并不会重新利用，相当于这个item只有在内存slab耗尽时，刷到磁盘slab才会重新利用，这样这个带有空洞的slab
被刷到磁盘，也就是磁盘的数据也会有空洞， 不仅仅是内存刷到磁盘的slab造成空洞，直接删除磁盘slab里的item,也会
有空洞，造成空洞的过程类似下面:

![image](https://github.com/git-hulk/fatcache-note/blob/master/snapshot/slab_hole.png)

对于内存空洞，解决方案可以，对每个slab增加一个队列，记录空洞地址，使用时，优先从空洞队列分配，无空洞时，
从slab后面分配。同时放回slab时，如果是最后一个分配，直接放回slab, 否则放到空洞队列。

***NOTE:*** 上面说的是针对内存空洞，不包含磁盘slab，因为fatcache主要是使用磁盘slab, 如果对每个磁盘slab，
增加空洞队列，可能会耗掉不少内存，需要提前评估。

----------------------------

##### the end #####

看完这篇， 应该要理解的几个东西:
```
1. slab 和 slabclass
2. slab 三种状态，以及三种状态的转换
3. fatcache不仅有内存slab还有磁盘slab.
```

接下来讲一下，另外一个fatcache很核心的东西，[索引](./itemx.md)
