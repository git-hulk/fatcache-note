#### fatcache启动流程 ####

-------------------

到这里我们对fatcache的基础内容，主要是slab, 索引, 网络IO有一个大致的了解。
这一节来看看，fatcache是如何启动的，我们很自然的找到了`fc.c`里面的main函数。

我们先来看看里面主要做了哪些事情，然后再一一解释:

```c
int
main(int argc, char **argv)
{
    rstatus_t status;
    struct context ctx;

    /* 设置默认参数 */
    fc_set_default_options();

    /* 从启动命令行里面获取参数，并设置到settings对应的参数 */
    status = fc_get_options(argc, argv);
    
    ...
    
    /* 后台运行模式 */
    if (settings.daemonize) {
        status = fc_daemonize(false);
        if (status != FC_OK) {
            exit(1);
        }
    }

    status = fc_set_profile();
    if (status != FC_OK) {
        exit(1);
    }

    /*
     * 这里面会初始化一些公用的数据结构， 如mbuf, meesage, conn等队列。
     * 时间事件和slab的初始化也是在这个函数。
     */
    status = core_init();
    if (status != FC_OK) {
        exit(1);
    }

    fc_print();

    /* 初始化epoll, 并把server的监听事件添加到epoll中 */
    status = core_start(&ctx);
    if (status != FC_OK) {
        exit(1);
    }
    
    for (;;) {
        /* 开始处理网络事件，我们上面core_start已经把监听fd，添加到Epoll, 所以到这里会开始监听。 */
        status = core_loop(&ctx);
        if (status != FC_OK) {
            break;
        }
    }

    /* 什么也没做 */
    core_stop(&ctx);

    return 0;
}

```

我们先来看看Server如何启动监听:

首先在 `main` 函数里面调用了`core_start`
```c
rstatus_t
core_start(struct context *ctx)
{
    rstatus_t status;

    /* 创建ctx */
    ctx->ep = -1; 
    ctx->nevent = 1024;
    ctx->max_timeout = -1; 
    ctx->timeout = ctx->max_timeout;
    ctx->event = NULL;

    /* 创建epoll, 并放到ctx全局变量 */
    status = event_init(ctx, 1024);
    if (status != FC_OK) {
        return status;
    }   

    /* server_listen开始把监听fd添加到epoll, 然后开始监听连接事件 */
    status = server_listen(ctx);
    if (status != FC_OK) {
        return status;
    }   

    return FC_OK;
}
```
接下来看看server_listen, 我们只看一下，如何把server fd添加到epoll。
```c
rstatus_t
server_listen(struct context *ctx)
{
    ...
    struct conn *s;
    ...
    
    /* 我们可以看到server的fd会被封装成一个connection, 专门做监听的connection */
    s = conn_get(sd, false);
    ...
    /* 这里就是把监听的connection的fd添加到epoll */
    status = event_add_conn(ctx->ep, s);
    
    /* 删除发送事件监听 */
    status = event_del_out(ctx->ep, s);
}
```

我们可以看到监听的server fd先是被包成fd, 然后再通过`event_add_conn`添加到epoll, 后面还有一个`event_del_out`,
因为在`event_add_conn`里面设置了in和out监听事件, 但是作为专门接收的fd, 不会有out事件，所以这里关闭。

我们可以继续看看`conn_get`的实现:
```c
struct conn *
conn_get(int sd, bool client)
{
    struct conn *c; 

    /* 创建connection结构体 */
    c = _conn_get();
    if (c == NULL) {
        return NULL;
    }   
    c->sd = sd; 
    c->client = client ? 1 : 0;

    if (client) {
        /* client */
        c->recv = msg_recv;
        c->send = msg_send;
        c->close = client_close;
        c->active = client_active;
    } else {
        /* server accept */
        c->recv = server_recv;
        c->send = NULL;
        c->close = NULL;
        c->active = NULL;
    }

    log_debug(LOG_VVERB, "get conn %p c %d", c, c->sd);
    return c;
}
```

上面函数我们看到`conn_get`是通过第二个参数`bool client`来决定回调函数，我们这里说的是server监听连接，
所以应该是`false`, 然后server_recv作为回调函数。

接下去应该是无限循环调用`core_loop`, epoll的监听事件真正开始，我们来看一下里面的实现:
```c
rstatus_t
core_loop(struct context *ctx)
{
    int i, nsd;

    /* 等待网络事件触发或者无网络事件就会超时，超时时间为ctx->timeout */
    nsd = event_wait(ctx->ep, ctx->event, ctx->nevent, ctx->timeout);
    if (nsd < 0) {
        return nsd;
    }       

    /* 有事件到来， 则调用core_core执行相应的回调 */
    for (i = 0; i < nsd; i++) {
        struct epoll_event *ev = &ctx->event[i];
        /* core_start 初始化ev->data.ptr 为con */
        core_core(ctx, ev->data.ptr, ev->events);
    }   

    return FC_OK;
}
```
`core_core`里面实现比较简单，自己去看即可，他会根据到来的事件类型是in还是out, 还决定回调函数，
这个回调函数就是我们上面`conn_get`里面设置的回调。
