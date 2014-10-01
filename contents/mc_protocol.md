#### MC 协议 ####

-----------------

MC指的是memcached, 它支持文本和二进制协议，fatcache只使用文本协议。 我们这里不对二进制协议做
解释，想要的可以看 [mc二进制协议](https://code.google.com/p/memcached/wiki/BinaryProtocolRevamped)，同时下面罗列的也只是常用的协议。

我们常用的协议可以分为以下几类:

*   Storage Commands, 写相关命令
*   Retrieval Commands, 读相关命令
*   Delete Comands 删除命令
*   Touch Comands 修改过期时间命令
*   其他stats命令，fatcache不支持.

<br />
<br />

##### Storage 命令 #####

----------------------------

***请求行协议格式:***
```
<command name> <key> <flags> <exptime> <bytes> [noreply]\r\n

cas <key> <flags> <exptime> <bytes> <cas unique> [noreply]\r\n

/** 字段注解 **/
<command name>: 对于写命令有set/add/replace/append/prepend这几种。

<key>: 查找value对应的关键字.

<flags>: 16bit数值，对server透明，返回时带回client,可用来标识当前value数据类型，1.2.1后为32bit.

<exptime>: 过期时间, 如果是0, 表示用不过期。

<bytes>: 后续发送data块(value)的字节数, 长度不包括(\r\n).

<noreply>: server不用回复client, 如果出错，server可能还会返回，client没有读取，就会错乱.

<cas unique>: 唯一的64bits数字，表示value的版本, 在做cas命令使用.
```
storage 命令由请求行和数据行两部分组成, 上面的协议格式是请求行， 接下来是数据行：
`<data block>\r\n`, 这个一般就是value.

上面我们可以看到cas命令的请求行格式跟其他命令不太一样，后面有个cas unqiue, 这个是由gets命令返回的
unique数值。 cas命令使用数值跟当前key的unique对比，如果不一样，表示当前key已经被更新过，返回失败。
还有对于`prepend`和`append` 两个命令，flags和exptime是不起作用的。


***返回格式***
```
- "STORED\r\n" 表示执行成功
- "NOT_STORED\r\n" 表示执行失败, 一般用来表示add或者replace条件不满足.
- "EXISTS\r\n" 表示cas命令的那个key已经被修改过.
- "NOT_FOUND\r\n" 表示cas的key不存在
```
<br />
<br />

##### Retrieval 命令 #####

--------------------------

***请求行协议***
```
get <key>*\r\n
gets <key>*\r\n

/** 字段注解 **/
<key>* : 多个用空格分割的key
```

***返回格式***
```
get和gets都是支持多个key, 也就是返回可能会有多个value,最后以END\r\n表示结束.
格式类似:
value1
value2
...
END\r\n

每个value格式如下:
VALUE <key> <flags> <bytes> [cas unique]\r\n
<data block>\r\n

key flags bytes和Storage命令里面的意义一样，不再重复, [cas unique]只有gets命令才会返回, 
datab block是返回的value。
```
<br />
<br />

##### Delete 命令 #####

-----------------------

***请求行协议***
```
delete <key> [noreply]\r\n

/** 字段注解 **/
<key> 查找关键字
[noreply] client要求不回复, 同Storage命令. 
```
***返回格式***
```
- "DELETED\r\n" 表示正常删除成功
- "NOT_FOUND" 表示key不存在
```

注: 删除还有一个flushall命令，删除所有key, 这里不解释.
<br />
<br />

##### Increment/Decrement 命令#####

-------------------------------

***请求行协议***
```
incr|decr <key> <value> [noreply] \r\n

<key>: 查找关键字

<value>: 10进制数字，对key对应的value增加或者减少的大小.
```

***返回格式***
```
- "value\r\n" value是更新之后的value
- "NOT_FOUND\r\n" 表示key对应的value不存在
```
注: 如果decr的之后的value<0, 结果还是返回0. 对应incr溢出，大于64bit的最大值，则是截断.
<br />
<br />
##### Touch 命令 #####
***请求行协议***
```
touch <key> <exptime> [noreply]\r\n

字段意义不变
```
***返回格式***
```
- “TOUCHED\r\n” 表示修改成功
- “NOT_FOUND” 表示修改的key不存在
```

#### The End ####

还有其他很多命令如Version, Quit, Stats, Slab 这些状态或者统计信息的命令我们这里没有细说，
想要了解可以自己去深入了解。

下一节是fatcache里面很重要的概念 [slab机制](./slab.md)
