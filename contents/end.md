#### 结束篇 ####

-----------------------------

最后对fatcache的特点做一下总结:

1) 单线程， 实现更加简单，但性能没多线程的好.

2) 无随机写，通过slab管理，转化为顺序写，减少小块IO写，无写放大

3) 随机读， 读性能没有写性能好

4) 索引管理， 快速判断数据是否存在，同时可以快速定位数据的位置，最多只有一次IO

5) slab分为内存和磁盘两种， 读写磁盘是direct io，不会使用pagecache, 内存slab更多是写缓冲的角色。

fatcache很适合用在那些对于数据响应时间并不是要求太高，最好是介于全内存和DB之间，同时数据量比较大的场景。

该笔记中还有很多没有提到的内容，如果有兴趣，欢迎一起讨论。

##### Contact #####

```Sina Weibo```: [@改名hulk](http://www.weibo.com/tianyi4)

```Gmail```: [hulk.website@gmail.com](mailto:hulk.website@gmail.com)

```Blog```: [www.hulkdev.com](http://hulkdev.sinaapp.com)
