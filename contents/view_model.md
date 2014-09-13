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




