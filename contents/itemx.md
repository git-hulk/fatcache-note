#### 索引机制 ####

-------------------------

索引是fatcache是一个很核心的机制，fatcache是通过查找索引来找到数据所在位置。删除数据的时候，
直接删除索引，而不是真正取删除数据。

我们可以先来看一下索引(itemx)的定义，在`fc_itemx.h`里面
```c
struct itemx {
    // itemx如果不用，通过这个指针放回空闲队列，可以重新利用
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

