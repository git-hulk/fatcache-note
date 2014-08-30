#### 索引机制 ####

-------------------------

索引是fatcache是一个很核心的机制，fatcache是通过查找索引来找到数据所在位置。删除数据的时候，
直接删除索引，而不是真正取删除数据。


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
查找md一致的itemx.
