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
