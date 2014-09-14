#### Message and Mbuf ####

---------------------------

我们上一节[fatcache启动流程](./main.md)已经说到，一个请求到来，会把连接的fd封装成connection,
添加到epoll, 开始接受客户端请求。

接下去client可以开始向fatcache发送请求，fatcache会先接收请求-> 解析数据 -> 执行相应命令->返回。


首先介绍一下message，如果你看过conn结构体的成员，会看到两个message变量，message是干什么用的呢?
```c
struct conn {
    ...
    struct msg         *rmsg;          /* current request being rcvd */
    struct msg         *smsg;          /* current response being sent */
    ...
}
```
从注释上来看，应该是rmsg应该是对应接收到的客户端发送的请求数据，而smsg是发送给客户端的响应数据。
所以我们可以知道，message就是来放接收的数据，实际上, 真正接收的数据是存放在message的一个mbuf队列里面。
而message更应该是理解成是一个请求，比如说一次接收了两个的get key的请求，fatcache会把他们分裂成两个message,
所以说message应该理解成一次的request。

我们来看一下`msg_recv`函数, 这个是`conn_get`里面设置的client连接回调函数。

实现的大概流程如下:

![image](https://github.com/git-hulk/fatcache-note/blob/master/snapshot/message.png)

我们再来细看'req_recv_next':
```c
struct msg *
req_recv_next(struct context *ctx, struct conn *conn, bool alloc)
{
    struct msg *msg;

    /* 链接关闭的标记eof, 这里调用说明是client主动关闭 */
    if (conn->eof) {
        msg = conn->rmsg;

        /* client sent eof before sending the entire request */
        if (msg != NULL) {
            conn->rmsg = NULL;

            ASSERT(msg->peer == NULL);
            ASSERT(msg->request && !msg->done);

            log_error("eof c %d discarding incomplete req %"PRIu64" len "
                      "%"PRIu32"", conn->sd, msg->id, msg->mlen);

            /* 把msg放回message的空闲队列中，可重复使用 */
            req_put(msg);
        }
                return NULL;
    }

    msg = conn->rmsg;
    /* 如果是conn->rmsg不为空，说明还有未处理完的数据，直接返回继续处理 */
    if (msg != NULL) {
        ASSERT(msg->request);
        return msg;
    }

    /* 是否分配新message, 如果是检查message的buf是否还有未处理数据，应该设置为fase */
    if (!alloc) {
        return NULL;
    }

    /* 创建新的message */
    msg = req_get(conn);
    if (msg != NULL) {
        conn->rmsg = msg;
    }

    return msg;
}
```
从上面我们可以看到，conn->rmsg不为空，说明还有为处理完的数据，直接返回继续处理。
接下来看`msg_recv_chain`
```c

```
