#### 命令处理 ####

-----------------------------

我们前面已经说到，当fatcache接收到一次完整请求之后，就会到`fc_request.c`里面的`req_process`函数，开始处理请求。

我们开始讲解一些各个命令的在fatcache的处理:

1） GET/GETS

GET/GETS 都是由`req_process_get`同一处理，只是返回的时候，GETS会多返回cas字段。
```c
static void
req_process_get(struct context *ctx, struct conn *conn, struct msg *msg)
{
    struct itemx *itx;
    struct item *it;
    
    /* 根据md获取索引 */
    itx = itemx_getx(msg->hash, msg->md);
    
    /* 索引为空，说明这个key不存在 */
    if (itx == NULL) {
        msg_type_t type;

        /*  
         * get/gets多个key, frag_id不为空。
         * 如果是单个key或者是多个key的最后一个key, 直接返回 "END\r\n"
         * 如果是多个key, 返回EMPTY.
         */
        if (msg->frag_id == 0 || msg->last_fragment) {
            type = MSG_RSP_END;
        } else {
            type = MSG_EMPTY;
        }   

        /* 返回 */
        rsp_send_status(ctx, conn, msg, type);
        return;
    } 
    
    /* 从memory或者disk读取value */
    it = slab_read_item(itx->sid, itx->offset);
    if (it == NULL) {
        rsp_send_error(ctx, conn, msg, MSG_RSP_SERVER_ERROR, errno);
        return;
    }
    /* item是否过期? */
    if (item_expired(it)) {
        rsp_send_status(ctx, conn, msg, MSG_RSP_NOT_FOUND);
        return;
    }

    /* 发送value */
    rsp_send_value(ctx, conn, msg, it, itx->cas);
}
```
1. `itemx_getx` 会先通过hash, md获取到索引，md是一个20byte的sha1值。

2. 如果没有过期，会通过`slab_read_item` 读取item。

3. 之后会通过`item_expired`判断是否过期.

4. 最后通过`rsp_send_value` 根据协议拼装发送回client.

Note: `slab_read_item` 通过索引里面的mem判断数据在内存还是在磁盘，如果在内存，直接拷贝，如果在磁盘，需要读磁盘，由于读磁盘是direct io, 所以需要对读地址进行对齐。

下面是GET/GETS的示意图
![image](https://github.com/git-hulk/fatcache-note/blob/master/snapshot/get_fatcache.png)

2) DELETE

fatcache的删除非常简单，直接把索引删除即可，没有索引就找不到数据，不用真正去擦除内存或者磁盘里面的数据，而这些空间会在内存或者磁盘不够的时候，重新被写。这个设计很巧妙，不用擦除，下次直接覆盖。下面是delete的实现:
```c
static void
req_process_delete(struct context *ctx, struct conn *conn, struct msg *msg)
{   
    bool found;

    // 删除索引
    found = itemx_removex(msg->hash, msg->md);
    if (!found) {
        rsp_send_status(ctx, conn, msg, MSG_RSP_NOT_FOUND);
        return;
    }

    rsp_send_status(ctx, conn, msg, MSG_RSP_DELETED);
}  
```

我们看到，只有简单通过`itemx_removex` 从索引表里面删除索引，直接返回。

3) SET

添加数据的时候比较特殊， 我们之前说过(Ps: 之前说了那么多，鬼记得你说什么)，写入数据时，一定是写在内存。 
所以我们写一段时间后，会发现内存slab耗尽，这时候会把最老的slab交换到磁盘，空出一个内存slab, 让新数据写入。

所以添加新数据可能会经历下面几步。

1) 如果之前key已经存在? 删除索引

2) 是否还有索引? 有, 直接返回。否则，通过`slab_evict`将磁盘最老的slab剔除，回收索引。 

3）是否还有内存slab item可用? 有，返回item。 否则, 通过`slab_drain`把最老的内存slab交换到磁盘.

4) 交换磁盘过程中，如果没有空闲的磁盘slab, 同样通过`slab_evict`, 把最老的磁盘slab的数据剔除。

由于这块代码比较多，这里就不列出代码，把大概流程画出来，有兴趣自己去看代码。
![image](https://github.com/git-hulk/fatcache-note/blob/master/snapshot/set_process.jpg)

还有其他一些如： Add, Replace, Append..等等命令，比较简单，大家看懂上面的处理，再去看其他命令处理都不是问题。
这里为了限制篇幅，我们也不完全解释所有命令。

##### The End #####

这一节主要说了get、set、delete的具体处理方式。我们前面已经讲了, slab, 索引(itemx), 网络, 数据接收以及协议解析， 
(除了具体解析之外，由于主要根据mc协议格式，逐个解析字段，没什么好说), 我们这里说了具体的处理，整个完整的fatcache也就这么多内容了。 
