#### 网络框架 ####

------------------------

##### epoll #####

fatcache使用Epoll作为网络IO事件通知机制，也只支持Epoll, 这个有点奇怪。从代码上来说，
fatcache和twitter的另外一个项目twemproxy代码相似度非常高，像message, mbuf, queue等基础数据结构，
都是直接使用，但twemproxy后面支持了其他的网络IO模型，而fatcache没有支持，也许是twemproxy在比较后面才支持。

这里稍微介绍一下epoll和其他IO模型如Poll, Select的一些区别:

```
Epoll: 和FreeBSD的kqueue类似，epoll监听多个fd的时候，使用的是红黑树，查找操作是O(1), 而Select, Poll是轮询，所以都是O(N), 就是说当监听10万个fd, Select, Poll需要从用户态拷贝大量fd到内核态，轮询一遍，这很费时。

Select: select对于监听的fd数目是有限制的，最大是FD_SETSIZE, 这个可以手动调整，当fd数目超过FD_SETSIZE时，
就会产生错误。

Poll： poll跟select基本一样，除了不对fd数目做限制之外。
```

需要注意的是，Epoll有两种触发模式： LT(level trigger)和ET(edge trigger). 两种方式不同在于:
假设现在有5k数据到来，每次只能读取4k.

`LT`: 在这种模式下，读取一次之后，还有1k数据未读，后续继续调用epoll_wait, 还是会立即返回fd, 继续读取剩余数据。

`ET`: 而ET比较有个性，如果这次只读了4k, 没有继续读剩余的1k, 下次epoll_wait不会返回这个fd,除非有写了新数据。

所以，使用ET模式，可能会不小心落掉一些数据，对开发者要求更高。

更加详细的使用方法，建议查看: [How to use epoll](https://banu.com/blog/2/how-to-use-epoll-a-complete-example-in-c/)这篇文章或者直接查看man手册。
<br />
<br />

##### fatcache Epoll #####

我们先对`fc_event.c`实现的函数都做一遍简单说明:

我们先来看看Server如何启动监听:

1. `main` 函数里面调用了`core_start`
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


