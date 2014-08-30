#### 索引机制 ####

-------------------------

索引是fatcache是一个很核心的机制，fatcache是通过查找索引来找到数据所在位置。删除数据的时候，
直接删除索引，而不是真正取删除数据。

--------------------------

我们可以先来看一下索引(itemx)的定义，在`fc_itemx.h`里面
```c
struct itemx {
    // 队列指针
    STAILQ_ENTRY(itemx) tqe;    /* link in index / free q */
    // key的sha1加密值，如果sha1一样，认为是同一个key.
    uint8_t             md[20]; /* sha1 message digest */
    // 这个key对应value的slab id.
    uint32_t            sid;    /* owner slab id */
    // 这个ke对应value在slab里面的偏移位置
    uint32_t            offset; /* item offset from owner slab base */
    // 我们协议看到的cas unique值。
    uint64_t            cas;    /* cas */
} __attribute__ ((__packed__));
```

itemx在hashtable里面的结构如下:

![image](https://github.com/git-hulk/fatcache-note/blob/master/snapshot/hashtable.png)

tqe: 我们看到，如果key hash冲突到同一个bucket, 就通过tqe指针，把这些冲突key,组成一个队列。

md: 这是一个长度为20的字符串，这个是key进行sha1后得到的，如果不同key通过sha1得到的相同字符串，
那么认为key相同，不过这个概率很小，基本可以忽略，所以可知，查找是通过对比md来对比，而不是真正的key,
这样可以减少key在itemx的存储空间， 又可以提高比较效率.

sid: 记录这个索引的key, 对应value所在的slab编号.

offset: 记录这个索引的key，对应value在slab中的偏移。

cas: 每个key都会记录一个unique数值，可以看作版本号，在check and set的时候使用.

通过上面描述，我们可以大概知道，如果查找一个key，应该是先对key做hash, 找到落在哪个bucket, 再对key做sha1,
得到md, 最后遍历这个itemx链表，找到对应itemx, 我们可以看一下代码, 在`fc_itemx.c`
```c
1 struct itemx *
2 itemx_getx(uint32_t hash, uint8_t *md)
3 {
4    struct itemx_tqh *bucket;
5    struct itemx *itx;
6
7    bucket = itemx_bucket(hash);
8
9    STAILQ_FOREACH(itx, bucket, tqe) {
10        if (memcmp(itx->md, md, sizeof(itx->md)) == 0) {
11            break;
12        }   
13    }   
14
15    return itx;
16 }
```

其中参数`hash`是外部计算好的hash值, md也是根据key做sha1得到的。可以看到第7代码是先获取到bucket链表，然后遍历bucket，
查找md一致的itemx， 这段代码比较简单，这里不再罗嗦.
<br />
<br />
##### 根据itemx读数据 #####

上述的过程是根据key找到了对应的itemx(索引)， 而我们真正的数据是存储在slab中，接下来看看如何通过itemx找到对应数据。

我们来看看，fatcache里面处理一次get的过程，代码来自'fc_request.c',
```c
static void
req_process_get(struct context *ctx, struct conn *conn, struct msg *msg)
{
    struct itemx *itx;
    struct item *it;

    /* 上面解释过的函数， 先找到索引 */
    itx = itemx_getx(msg->hash, msg->md);
    
    /* 如果找不到索引，说明key不存在，直接返回. */
    if (itx == NULL) {
        ...
        rsp_send_status(ctx, conn, msg, type);
        return;
    }
    
    /* 如果存在就读取到对于的item，我们下面细看一下 */
    it = slab_read_item(itx->sid, itx->offset);
    /* 如果读不到item是server异常，返回错误 */
    if (it == NULL) {
        rsp_send_error(ctx, conn, msg, MSG_RSP_SERVER_ERROR, errno);
        return;
    }
    /* 判断是否过期 */
    if (item_expired(it)) {
        rsp_send_status(ctx, conn, msg, MSG_RSP_NOT_FOUND);
        return;
    }

    /* 数据存在而且没有过期， 就返回 */
    rsp_send_value(ctx, conn, msg, it, itx->cas);
}
```
上面如果itemx存在，就会调用`slab_read_item`来获取item, 下面看一下这个函数:
```c
struct item *
slab_read_item(uint32_t sid, uint32_t addr)
{
    
    struct slabclass *c;    /* slab class */
    struct item *it;        /* item */
    struct slabinfo *sinfo; /* slab info */
    int n;                  /* bytes read */
    off_t off;              /* offset to read from */
    size_t size;            /* size to read */
    off_t aligned_off;      /* aligned offset to read from */
    size_t aligned_size;    /* aligned size to read */
    
    ASSERT(sid < nstable);
    ASSERT(addr < settings.slab_size);
    
    /* 根据sid找到slab info, sinfo是一个大数组，存放memory slab和disk slab的所有slab info. */
    sinfo = &stable[sid];
    /* 根据slab info里面记录slab info所在的slab class id, 找到对应slabclass */
    c = &ctable[sinfo->cid];
    size = settings.slab_size;
    it = NULL;
    
    /* slab 在内存里面 */
    if (sinfo->mem) {
        /* 计算slab在所在的位置+ key所在slab的偏移，计算item的位置 */
        off = (off_t)sinfo->addr * settings.slab_size + addr;
        /* 把item拷贝到readbuf */
        fc_memcpy(readbuf, mstart + off, c->size);
        it = (struct item *)readbuf;
        goto done;
    }

    /* 如果item在磁盘，注意，读磁盘可能是设备，需要对齐读的地址 */
    off = slab_to_daddr(sinfo) + addr;
    /* item地址其实地址对齐到512的整数倍 */
    aligned_off = ROUND_DOWN(off, 512);
    /* item的长度对齐到512的整数倍 */
    aligned_size = ROUND_UP((c->size + (off - aligned_off)), 512);

    /* 读取item的数据 */
    n = pread(fd, readbuf, aligned_size, aligned_off);
    if (n < aligned_size) {
        log_error("pread fd %d %zu bytes at offset %"PRIu64" failed: %s", fd,
                  aligned_size, (uint64_t)aligned_off, strerror(errno));
        return NULL;
    }
    it = (struct item *)(readbuf + (off - aligned_off));

done:
    /* 数据正确性校验 */
    ASSERT(it->magic == ITEM_MAGIC);
    ASSERT(it->cid == sinfo->cid);
    ASSERT(it->sid == sinfo->sid);

    return it;
}
```

我们可以看到读取item,经历了下面的几个过程:
```
-    先根据itemx里面的sid, 作为下标，在stable, 也就是slab info table,里面找到slab info.
-    根据slab info 里面的mem字段判断，item是在内存还是disk。
-    然后根据sid所在slab class id, 计算item长度， 计算item的长度，从内存或者disk读取到数据，返回.
```

总结上面， 需要找到key，需要找索引，然后找到slab info, 然后根据slab info找到slab, 再读取数据.

##### itemx 耗尽 #####

----------------------

[参数配置](./configure.md) 这一节里面`-i, --max-index-memory=N`, 就是最大索引的使用内存，默认64M，在64bit操作系统，
每个索引占用44bytes, 如果当key的数量超过索引的最大数目时，就会产生剔除。
过程大概如下:
```c
1. itemx队列是否是空? 
2. 如果不是拿到索引，返回.
3. 如果为空， 就把磁盘最旧的slab剔除，回收索引，放到itemx的空闲队列。
4. 返回1，重复.
```
看一下fatcache里面的实现, 看一下`fc_slab.c`,可以看过，要分配item之前，会先判断索引是否已经用完，
如果用完的话，会把最旧的slab剔除，回收索引.
```c
struct item *
slab_get_item(uint8_t cid)
{
    ...
   
    if (itemx_empty()) {
        /* item 索引使用耗尽，应该做一些剔除 */
        status = slab_evict();
        if (status != FC_OK) {
            return NULL;
        }   
    } 
    
    ...
}

```
<br />
<br />
##### The End #####

这一节对itemx（索引）进行比较详细的介绍，包括在索引的作用，用法等。 还有一个注意的点，剔除slab的时候，
只是回收索引，然后将slab放到空闲slab队列，并没有真正擦除slab的数据，这样下次写数据就会覆盖之前的数据，
减少擦除slab的时间，这个设计非常赞..

我们知道了slab机制，索引机制，相当于知道数据如何存储，如何查找数据。接下来我们说一下[网络框架](./network.md)，
之后再说一下，协议解析过程, 这样就可以把 从client发送数据->接收->解析->返回 串起来了.

下一节 [网络框架](./network.md)
