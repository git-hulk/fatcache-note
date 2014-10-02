#### 用户数据接收和解析 ####

-------------------

我们在上一节[fatcache启动流程](./main.md), 可以看到，fatcache开始监听之后，如果有新client到来，就会通过`event_add_conn`, 
把一个连接添加到epoll的监听列表，开始监听client发送过来的请求数据, 并设置接收和发送回调函数分别为`msg_recv`和`msg_send`。

<br />

##### Message #####

fatcache所有接收数据或者发送数据都会放在`strcut msg`的`mbu`f链表中，一个msg代表一个request或者response。

下面是`fc_message.h`中 `struct msg`的主要成员:
```c
struct msg {
    TAILQ_ENTRY(msg)     c_tqe; /* 指向下一个处理完的请求 */
    TAILQ_ENTRY(msg)     m_tqe; /* 指向下一个发送的msg */
    uint64_t             id;    /* 当前msg 对应的id */
    struct msg           *peer; /* 对应的msg, 如果是当前msg是request，peer指向即为response */
    struct conn          *owner;/* msg所属的connection, 每一个connection可能会处理多个request */
    struct mhdr          mhdr;  /* 接收或者发送数据存放的链表 */
    uint32_t             mlen;  /* 接收或者发送数据的长度 */
    int                  state; /* 当前解析到的状态 */
    uint8_t              *pos;  /* 当前解析到的buf位置 */
    uint8_t              *token; /* 存放当前解析的token */
    msg_parse_t          parser; /* 解析msg的函数，在_msg_get函数里面设置 */
    msg_parse_result_t   result; /* 解析的结果 */
    msg_type_t           type;  /* 解析到的命令类型，get/gets/del.. */
    uint8_t              *key_start;      /* key存放的开始地址 */
    uint8_t              *key_end;        /* key存放的结束地址 */

    uint32_t             hash;            /* key的hash值 */
    uint8_t              md[20];          /* key进行sha1加密的结果 */

    /* 协议数据 */
    uint32_t             flags;           /* flags */
    uint32_t             expiry;          /* expiry */
    uint32_t             vlen;            /* value length */
    uint32_t             rvlen;           /* running vlen used by parsing fsa */
    uint8_t              *value;          /* value marker */
    uint64_t             cas;             /* cas */
    uint64_t             num;             /* number */

    /* 如果协议切分成多个fragments是用到，如get/gets多个key */
    struct msg           *frag_owner;     /* owner of fragment message */
    uint32_t             nfrag;           /* # fragment */
    uint64_t             frag_id;         /* id of fragmented message */

    err_t                err;             /* 遇到错误? */
    unsigned             error:1;         /* error? */
    unsigned             request:1;       /* request? or response? */
    unsigned             quit:1;          /* 是否为quit命令 */
    unsigned             noreply:1;       /* 是否需要把结果返回给客户端 */
    unsigned             done:1;          /* 处理是否结束? */
    unsigned             first_fragment:1;/* 第一个处理的片段? 比如get/gets需要在开始输出VALUE常量，需要用这个标志 */
    unsigned             last_fragment:1; /* 最后一个处理的片段? 返回结束时，需要用到这个标志 */
    unsigned             swallow:1;       /* swallow response? */
}
```

> 1) 如果msg->request == 1,表示这个msg是request， 对应这个mbuf链表存放就是接收的数据.

> 2) 如果msg->requeset == 0, 表示这个msg是response, 对应mbuf链表存放的是发送数据。
<br />

##### request和response如何对应? #####

`struct msg`中有个peer字段， request msg 的peer就是response msg, 相反response msg的peer为 request msg.
每个msg一次只能存储一个key, 如果是 `get|gets key1, key2, ..., keyn` 就会被切分成多个msg.
这些请求处理后会放到输出队列，触发写事件，开始调用写回调数据来返回数据到客户端.

<br />

##### 接收client数据 #####

我们来看一下，`fc_message.c`里面`msg_recv`的实现：
```c
rstatus_t
msg_recv(struct context *ctx, struct conn *conn)
{
    ...
    conn->recv_ready = 1;
    do { 
        // 创建一个msg
        msg = req_recv_next(ctx, conn, true);
        if (msg == NULL) {
            return FC_OK;
        }   

        // 接收并开始处理请求数据
        status = msg_recv_chain(ctx, conn, msg);
        if (status != FC_OK) {
            return status;
        }   
    } while (conn->recv_ready);

    return FC_OK;
}
```
上面代码, `req_recv_next` 创建一个request msg, 进入`msg_recv_chain`开始接收数据并处理请求。
下面看一下`msg_recv_chain`的详细实现.
<br />

##### 1) 接收数据 #####
```c
// 判断最后mbuf是否还有剩余空间， 如果没有创建一个新的buf,放到队列 
mbuf = STAILQ_LAST(&msg->mhdr, mbuf, next);
if (mbuf == NULL || mbuf_full(mbuf)) {
    mbuf = mbuf_get();
    if (mbuf == NULL) {
        return FC_ENOMEM;
    }
    mbuf_insert(&msg->mhdr, mbuf);
    msg->pos = mbuf->pos;
}
ASSERT(mbuf->end - mbuf->last > 0);

msize = mbuf_size(mbuf);
// 开始接收数据
n = conn_recv(conn, mbuf->last, msize);

```
<br />
##### 2) 处理请求数据 #####

接收完数据之后，进入`msg_parse` 开始处理请求数据:

```c
static rstatus_t
msg_parse(struct context *ctx, struct conn *conn, struct msg *msg)
{
    rstatus_t status;

    if (msg_empty(msg)) {
        /* no data to parse */
        req_recv_done(ctx, conn, msg, NULL);
        return FC_OK;
    }
    //调用fc_memcache.c里面的memcache_parse_req函数开始根据协议处理数据
    msg->parser(msg);

    // 协议解析结果
    switch (msg->result) {
        // 解析到完整的命令，开始处理
        case MSG_PARSE_OK:
            status = msg_parsed(ctx, conn, msg);
            break;

        // 需要解析成多个fragment, 如get/gets多个key,需要被分成多个msg来处理
        case MSG_PARSE_FRAGMENT:
            status = msg_fragment(ctx, conn, msg);
            break;

        // 不完整的字段，需要把一部分接收数据放到下一个mbuf的开始，重新解析
        case MSG_PARSE_REPAIR:
            status = msg_repair(ctx, conn, msg);
            break;

        // 数据不完整，需要重新接收数据
        case MSG_PARSE_AGAIN:
            status = FC_OK;
            break;

        default:
            status = FC_ERROR;
            conn->err = errno;
            break;
    }

    return conn->err != 0 ? FC_ERROR : status;
}
```

我们看到协议解析会有下面几种状态:

> 1) `MSG_PARSE_OK`: 根据协议解析到一条完整的请求，接下去可以处理请求。

> 2) `MSG_PARSE_FRAGMENT`: 需要分成多个碎片处理， 我们前面说到，每个request msg只能存储一个key, 所以像get/gets带有多个key的协议，
需要被切分成多个msg, 这时候状态就是fragment.

> 3) `MSG_PARSE_REPAIR`: 除了VALUE之外的其他字段，比如exprietime是123456，现在接收到123, 456在下一个mbuf, 就需要通过repair, 把123移动到下一个mbuf,
并重新解析， value是允许跨mbuf, 只会返回again状态，不是repair状态。

> 4)  `MSG_PARSE_AGAIN`： 当协议数据不完整，返回again状态.

> 5)  `default` : 不是上面四种解析状态，就是出错，直接关闭连接，因为mc协议是通过空格来切分数据，前面数据出错，后续数据会受影响，直接关闭连接。

<br />
上面已经介绍了，接收和解析完数据之后，开始处理请求。

`msg_parsed`会调用`req_recv_done`, 接着`req_recv_done` 调用`req_process` 开始处理请求，并返回数据。

具体每种类型命令如何处理，我们在下一节[命令处理](./compelete_process.md) 详细介绍。
