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

上面我们说了，fatcache只支持epoll, 也就是在其它类unix的操作系统，如FreeBSD，就傻逼了，除非自己支持。

我们下面来看看fatcache如何使用Epoll, 我们从`fc.c`的`main`方法调用`core_start`
