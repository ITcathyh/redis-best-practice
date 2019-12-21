缓存在我们日常开发中占据着举足轻重的地位，通过缓存组件可以让我们的系统有着多方位的提升空间。而Redis就一个代表性的缓存组件。正巧最近使用Redis比较频繁，所以打算通过文章记录一下在Redis开发中遇到的问题和一些开发规范。欢迎star和补充

# Key的设计
### 易于管理
即能通过名称大概知道所涉及业务。通常我们会以service:characteristics来进行命名，如pubg_chat:uid:room_id的形式，这样可以尽可能避免冲突（当然，不同业务使用不同的Redis是更好的）
### 尽量简洁
Redis本质上是一个内存数据库，而内存的大小是远小于硬盘的。如果Key过大的话，会导致Redis所能存储的内容变少。所以在日常中推荐Key能够尽可能的简介明了，用缩写来代替完整的单词
### 避免特殊字符
如逗号、换行、空格和引号等转义字符都是不应该使用的
### 设置生命周期
- Redis不应当成为一个永久存储的组件，为每一个Key都设置他的过期时间
- 如果确实需要Redis永久存储某类内容，那么因当采用异步“续命”的方式来进行，而不应该在一开始就为其设置永久的生命周期，避免后续需要变更时带来的维护灾难

# Value的使用
### 规避大Key！
- Redis单线程的，它会在执行完一个命令后才会执行其他命令
- 首先大Key在传输键值对时，会对网络造成压力（带宽问题），并且有的proxy会将大内容分片传输，进而再次增加了网络传输时间
- 其次，如list、hash这类结构，如果使用O(n)的指令或者使用del命令，那么会造成严重的阻塞
- 所以通常而言，string 类型控制在 10KB 以内，hash、list、set、zset 元素个数不要超过 5000
- ps：通常可采用hash的方式分割大key
### value的压缩
可考虑使用protobuf、MessagePack等方式进行序列化，这样一可以提升redis的利用率，二提高了序列化的效率，三是能提供value跨语言的能力

# 命令使用
### 避免频繁对string做append
可以考虑使用list进行替代
### 集合类操作
- O（n）指令应注意。对于set，zset，list，hash等集合类，应注意O（n）命令对于性能的影响。通常应该避免直接使用O（n）指令，可用HSCAN，SSCAN，ZSCAN进行渐进操作，防止命令的阻塞
- 渐进式删除。不应该直接使用del，而应该自己写脚本一点点的删除
### 禁用危险命令
keys、flushall、flushdb......这种不用多说，一来直接Redis就懵圈了，人也楞了
### 合理利用Pipeline模式
- 在mget大量数据时，proxy会拆包和解包，会导致proxy层的压力增加，而pipeline模式会直接转发。所以在批量获取的情况下，**pipeline的效率一般都会优于mget**
- 同时应该注意两点：
	- mget是原子操作，pipeline不是。所以业务上不可盲目采用
	- pipeline是可以发送不同命令的，当然使用lua也可以实现这一点
### 避免不必要的指令
如部分Redis Client会有TestOnBorrow之类的探测指令，在没有特殊要求的情况下应当避免此类指令，以减小redis负载和网络压力
### 对Lua应当做特殊要求
- 所有key都应该由KEYS数组来传递
- 所有value都应该由ARGS数组来传递
- 所有key，必须在1个slot上
### 性能查询指令
- slowlog get，查询慢命令
- info commandstats，查询执行过的命令信息，包含用时和次数等
- client list，查询引起阻塞的命令


# 客户端使用
### 避免混用实例
不同的业务线应该做到实例的拆分，避免混用导致的连锁问题：如key重合、命令阻塞等
### 使用连接池
每次使用都新建连接会有几个问题：
- 造成redis的负担增加
- 浪费网络资源
- 影响执行效率
- 难以维护
### 熔断
Redis本质也是一个“服务”，所以熔断机制不可少
### 鉴权
避免无关服务的滥用或导致数据出错
### 避免作为消息队列
Redis其实还能够支持消息队列的应用，但其读写效率是不及其他MQ如Kafka、RabbitMQ等。且由于其结构的设置，不太能够支撑MQ的一些主要特性，所以应当避免使用。

# 其他
### 淘汰策略
根据⾃⾝业务类型，选好maxmemory-policy(最⼤内存淘汰策略)，设置好过期时间
默认策略是volatile-lru，即超过最⼤内存后，在过期键中使⽤lru算法进⾏key的剔除，保证不过期数据不被 删除，但是可能会出现OOM问题
- allkeys-lru：根据LRU算法删除键，不管数据有没有设置超时属性，直到腾出⾜够空间为⽌
- allkeys-random：随机删除所有键，直到腾出⾜够空间为⽌。
- volatile-random:随机删除过期键，直到腾出⾜够空间为⽌
- volatile-ttl：根据键值对象的ttl属性，删除最近将要过期数据。如果没有，回退到noeviction策略
- noeviction：不会剔除任何数据，拒绝所有写⼊操作并返回客⼾端错误信息"(error) OOM command not allowed when used memory"，此时Redis只响应读操作
### 不滥用Redis事务
Redis事务不像DB的事务这么“安全”，也不支持回滚，所以不应当过多的使用。因为这块我没怎么使用过，详见[官方文档](https://redis.io/topics/transactions)

# 参考资料
- [Redis官方文档](https://redis.io/documentation)
- [阿里云Redis开发规范](https://www.infoq.cn/article/K7dB5AFKI9mr5Ugbs_px)
