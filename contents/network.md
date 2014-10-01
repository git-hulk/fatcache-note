#### 网络框架 ####

------------------------

##### epoll #####

fatcache使用Epoll作为网络IO事件通知机制，也只支持Epoll。从代码上来说，
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

`event_init` 创建epoll, 这个函数在fatcache启动的时候调用。每个fatcache实例只会有一个epoll对象。
后续有需要监听的fd, 都是添加到到这个epoll对象。
```c
int
event_init(struct context *ctx, int size)
{
    int status, ep; 
    struct epoll_event *event;

    ASSERT(ctx->ep < 0); 
    ASSERT(ctx->nevent != 0); 
    ASSERT(ctx->event == NULL);

    /* 创建epoll对象 */
    ep = epoll_create(size);
    if (ep < 0) {
        log_error("epoll create of size %d failed: %s", size, strerror(errno));
        return -1; 
    }   

    /* 提前为Epoll的事件申请空间 */
    event = fc_calloc(ctx->nevent, sizeof(*ctx->event));
    if (event == NULL) {
        status = close(ep);
        if (status < 0) {
            log_error("close e %d failed, ignored: %s", ep, strerror(errno));
        }   
        return -1; 
    }   
    
    /* 把epoll相关数据结构都是放在ctx里面 */
    ctx->ep = ep; 
    ctx->event = event;
    
    return 0;   
}
```

`event_deinit` 销毁epoll对象，这个只有在Fatcache关闭的时候会使用，只是关闭对应fd,比较简单, 自己去看。

`event_add_conn`: 我们注意到添加epoll的fd, 是包装成connection的形式传递进来。然后添加的监听参数都是
`EPOLLIN | EPOLLOUT | EPOLLET`, 并把connection放到event.data.ptr， 可以作为事件到来时，回调函数的参数。
```c
int
event_add_conn(int ep, struct conn *c)
{
    int status;
    struct epoll_event event;

    ASSERT(ep > 0);
    ASSERT(c != NULL);
    ASSERT(c->sd > 0);

    /* 监听事件类型是in和out */
    event.events = (uint32_t)(EPOLLIN | EPOLLOUT | EPOLLET);
    /* connection作为event的data */
    event.data.ptr = c;

    /* connection的fd添加到epoll */
    status = epoll_ctl(ep, EPOLL_CTL_ADD, c->sd, &event);
    if (status < 0) {
        log_error("epoll ctl on e %d sd %d failed: %s", ep, c->sd,
                  strerror(errno));
    } else {
        c->send_active = 1;
        c->recv_active = 1;
    }

    return status;
}
```

`event_del_conn`是跟`event_add_conn`刚好相反，把一个链接的fd从epoll移除，也就是关闭链接。

```c
int
event_del_conn(int ep, struct conn *c)
{
    int status;

    ASSERT(ep > 0);
    ASSERT(c != NULL);
    ASSERT(c->sd > 0);

    /* 把connection fd从epoll中移除，即不再监听 */
    status = epoll_ctl(ep, EPOLL_CTL_DEL, c->sd, NULL);
    if (status < 0) {
        log_error("epoll ctl on e %d sd %d failed: %s", ep, c->sd,
                  strerror(errno));
    } else {
        c->recv_active = 0;
        c->send_active = 0;
    }

    return status;
}
```

另外两个函数`int event_add_out(int ep, struct conn *c);` 和 `int event_del_out(int ep, struct conn *c);` 的作用是，
添加和删除out事件，实现和event_add[del]_conn类似。

最后一个函数`event_wait`， 之前的实现是初始化和为epoll添加或者删除监听的fd，epoll真正的监听是从event_wait开始。
```c
int event_wait(int ep, struct epoll_event *event, int nevent, int timeout)
{
    ...
    for (;;) {
        /* epoll等待事件或者超时 */
        nsd = epoll_wait(ep, event, nevent, timeout);
        if (nsd > 0) {
            /* 有文件事件到来 */
            return nsd;
        }

        if (nsd == 0) {
            /* 超时 */
            if (timeout == -1) {
               log_error("epoll wait on e %d with %d events and %d timeout "
                         "returned no events", ep, nevent, timeout);
                return -1;
            }

            return 0;
        }

        if (errno == EINTR) {
            continue;
        }

        log_error("epoll wait on e %d with %d events failed: %s", ep, nevent,
                  strerror(errno));

        return -1;
    }
    
    ...
}
```
##### the end #####

从上面的代码来看， fatcache在接收到一个Client链接，会生成一个conn结构，添加到epoll里面来，
并开始监听这个连接。

epoll在fatcache里面的实现比较简单，自己看代码应该问题不大。
