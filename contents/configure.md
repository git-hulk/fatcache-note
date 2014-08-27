#### 配置参数 ####

--------------------

```
Usage: fatcache [-?hVdS] [-o output file] [-v verbosity level]
           [-p port] [-a addr] [-e hash power]
           [-f factor] [-n min item chunk size] [-I slab size]
           [-i max index memory[ [-m max slab memory]
           [-z slab profile] [-D ssd device] [-s server id]

Options:
  -h, --help                  : this help
  -V, --version               : show version and exit
  -d, --daemonize             : run as a daemon
  -S, --show-sizes            : print slab, item and index sizes and exit
  -o, --output=S              : set the logging file (default: stderr)
  -v, --verbosity=N           : set the logging level (default: 6, min: 0, max: 11)
  -p, --port=N                : set the port to listen on (default: 11211)
  -a, --addr=S                : set the address to listen on (default: 0.0.0.0)
  -e, --hash-power=N          : set the item index hash table size as a power of two (default: 20)
  -f, --factor=D              : set the growth factor of slab item sizes (default: 1.25)
  -n, --min-item-chunk-size=N : set the minimum item chunk size in bytes (default: 84 bytes)
  -I, --slab-size=N           : set slab size in bytes (default: 1048576 bytes)
  -i, --max-index-memory=N    : set the maximum memory to use for item indexes in MB (default: 64 MB)
  -m, --max-slab-memory=N     : set the maximum memory to use for slabs in MB (default: 64 MB)
  -z, --slab-profile=S        : set the profile of slab item chunk sizes (default: n/a)
  -D, --ssd-device=S          : set the path to the ssd device file (default: n/a)
  -s, --server-id=I/N         : set fatcache instance to be I out of total N instances (default: 0/1)
  ```

上面是fatcache全部的用户可配置参数, 接下来看一下除了-h显示帮助, -V显示版本这两个配置，其他配置选项的作用。

```
-d, --dammonize 

fatcache server是否后台运行，一般部署的时候，会后台执行，这个比较简单，略过。

```

```
-S, --show-sizes

打印slab, item和index的长度
```

```
-o, --output=S 

日志输出路径， 默认是输出到终端
```

```
-v, --verbosity=N

日志级别，0~11, 默认是6
```

```
 -p, --port=N 
 
 监听端口，默认是11211. 这个也是没什么好说
```
```
-a, --addr=S

监听地址，如果有多块网卡的时候，可以设置监听的ip，默认是0.0.0.0
```
```
-e, --hash-power=N 

这个参数决定item index的hash表的大小，hash的bucket数= 2 ** N, 这个数值如果设置太小，
可能会导致冲突变多，设置太大，会浪费空间， 应该根据实际的使用量来确定.
```
```
 -f, --factor=D
 
 每一层slab之间item长度的增长因子,默认1.25，比如slab calss有3级（实际应用中会有远大于3），
 如果第一级item长度是20byte, 第二级的就是 20 * 1.25 = 25bytes, 第三级就是 25 *1.25, 以此类推...
 设置这个参数，比较需要技术含量，如果算不准，就不要乱改了。
```
```
-n, --min-item-chunk-size=N

item最小长度，也就是slabclass第一层item的大小, 默认84bytes, 设置大小不能小于84.
```
```
-I, --slab-size=N

slab大小，默认是1M. 这个是fatcache写入磁盘的单位，所以如果有修改，需要对比一下性能.
```
```
-i, --max-index-memory=N

索引占用最大内存，索引数量 = max-index-memory/sizeof(itemx), sizeof当前在64bit操作系统上是44bytes.
索引的数量也就是能容纳k-v数，如果太小，可能会有大量k-v剔除。这个值要根据k-v大概数量先算好。
```
```
-m, --max-slab-memory=N

内存slab的最大内存，由于fatcache操作ssd是direct io, 这个相当于ssd的cache。
这个设置越大，读写速度会越快.
```
```
-z, --slab-profile=S

slab class优化配置，可以配置Slab每一层的item大小，不用factor作为增长因子。
```
```
-D, --ssd-device=S

数据存放的磁盘文件路径，如果是分区直接使用，如果是普通文件，需要先撑大文件。
比如设置1G, 需要把文件lseek到1G, 不然fatcache读出来的size = 0, 会产生错误.
```
```
-s, --server-id=I/N

当部署多个实例，需要设置这个参数，N表示实例个数，I表示当前实例编号。
多实例能更好利用ssd并发读写，比如ssd总共是8G可用，那么当前实例读写开始地址 = (8G/8) *I。
```

fatcache 的配置就是上面这些，阅读可以先大概知道，后续真正使用再回来看。
接下去看一下 [fatcache的访问模型](./view_model.md)
