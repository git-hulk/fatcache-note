#### 访问模型 ####

------------------

现在我们应该知道slab是什么， 索引是干什么的，网络监听，接下去我们可以来说说，fatcacahe是如果处理一次
请求，这里我们以get/set为例来说明。

下图是fatcache大体的架构图:

![image "fatcache view"](https://github.com/git-hulk/fatcache-note/blob/master/snapshot/fatcache_view.png)

#####  接收Client的数据 #####

我们上一节讲到，fatcache开始监听， 如果有连接到来，就会会通过`conn_get`创建一个connection, 加到监听列表中，
并设置回调函数。

其中接收函数为:`fc_message.c`源码中的`msg_recv`，`msg_recv`调用 `msg_recv_chain`, 接着就会调用`msg_parse`进行解析。
```
static rstatus_t
msg_parse(struct context *ctx, struct conn *conn, struct msg *msg)
{
      ...
      
    msg->parser(msg);

    switch (msg->result) {
    case MSG_PARSE_OK:
        status = msg_parsed(ctx, conn, msg);
        break;

    case MSG_PARSE_FRAGMENT:
        status = msg_fragment(ctx, conn, msg);
        break;

    case MSG_PARSE_REPAIR:
        status = msg_repair(ctx, conn, msg);
        break;

    case MSG_PARSE_AGAIN:
        status = FC_OK;
        break;

    ...
}
```

上面`msg->parser`这个函数调用就是`fc_memcache.c`中的`memcache_parse_req`, 这个函数里面就是一个mc协议解析的状态机，

对发送过来的数据，根据[MC协议](./protocol.md)各个命令格式，一条条命令进行解析。

解析会有下面四种状态:


a) `MSG_PARSE_OK`: 解析到一条完整的协议，解析去就是处理这条协议。

b) `MSG_PARSE_FRAGMENT`: 接收到get/gets这中多个key的协议，因为message结构就是对应一个key, 
所以有多个key的时候，需要拆分成多个message。

c) `MSG_PARSE_REPAIR`: 因为fatcache接收数据是使用mbuf(大约8k), 所以有可能一条请求的数据的同一个
字段被断开在两个mbuf, 比如key1是一个完整的key，被拆分到两个mbuf.
即Mbuf1 => | SET ke| Mbuf2 =>|y1 0 0 3 \r\n abc\r\n|。 
所以我们就需要通过repair, 把ke这两个字节移到到Mbuf2, 继续解析才能构成完整的字段。

d) `MSG_PARSE_AGAIN`: 这个比较简单，就是当前mbuf已经解析完，然后还没构成完成的一条命令，需要更多数据。

<br />
<br />


##### Get #####

上面对于fatcache协议解析的状态简单介绍，帮助初学者理解整个协议解析机制。

我们在[索引机制](./itemx.md) 那节应该说明， fatcache是通过索引来找到对应的item, 再读取item里面的value.

![image](https://github.com/git-hulk/fatcache-note/blob/master/snapshot/get_fatcache.png)

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

```c
struct item *
slab_read_item(uint32_t sid, uint32_t addr)
{
    ...
    sinfo = &stable[sid];
    c = &ctable[sinfo->cid];
    size = settings.slab_size;
    it = NULL;

    /* 如果item在memory slab, 直接拷贝到readbuf */
    if (sinfo->mem) {
        off = (off_t)sinfo->addr * settings.slab_size + addr;
        fc_memcpy(readbuf, mstart + off, c->size);
        it = (struct item *)readbuf;
        goto done;
    }

    /* 如果item在disk slab */
    off = slab_to_daddr(sinfo) + addr;
    /* 由于我们设置slab文件路径是设备，需要对齐读取的地址，为512的倍数 */
    aligned_off = ROUND_DOWN(off, 512);
    aligned_size = ROUND_UP((c->size + (off - aligned_off)), 512);

    n = pread(fd, readbuf, aligned_size, aligned_off);
        it = (struct item *)(readbuf + (off - aligned_off));

done:
    ASSERT(it->magic == ITEM_MAGIC);
    ASSERT(it->cid == sinfo->cid);
    ASSERT(it->sid == sinfo->sid);

    return it;
}
```
