### redis的持久化机制
redis提供了两种持久化的机制RDB，AOF
RDB
开启关闭：默认开启
同步机制：在指定的时间内发生了一个（or多个）写操作就同步一次
储存的内容：具体的值
优点：

* 储存的文件是经过压缩，体积小
* 因为储存的是具体的值所以恢复快
* 适用于备份
缺点：
* 每次同步的时候都是会同步所有的数据，每次隔5分钟同步一次，如果服务器挂掉了，就有可能造成数据的丢失
* 数据在保存到rdb的时候，redis会fork出一个子进程来同步，数据比较大的时候，会比较耗时间

AOF
开启关闭：修改redis.conf文件，appendonly
同步机制：每秒同步或者每次操作后同步
储存的内容：储存的是命令，不会压缩
优点：

* 每秒保存一次数据，即使服务器发生故障也不会造成数据的丢失
* redis命令会直接追加到aof文件后面，因此每次备份的时候的时候只要追加数据就可以了
* aof文件大了，redis会进行重写，只保留最小的命令集合
缺点：
* aof文件没有被压缩，因此aof文件的体积比较大
* aof每秒中都要写入备份，因此如果并发量较大，就会造成效率慢的问题
* aof文件储存的是命令，灾难恢复的速度不及rdb



### rehash的解释：
在创建hashMAP的时候可以设置来个参数，一般默认
初始化容量：创建hash表时桶的数量
负载因子：负载因子=map的size/初始化容量
当hash表中的负载因子达到负载极限的时候，hash表会自动成倍的增加容量（桶的数量），并将原有的对象
重新的分配并加入新的桶内，这称为rehash。这个过程是十分耗性能的，一般不要
一般建议设置比较大的初始化容量，防止rehash，但是也不能设置过大，初始化容量过大 浪费空间



### redis文件处理器的四个组成部分：
套接字 >> IO多路复用程序 >> 文件分派器  >> 事件处理器
redis中io多路复用器模块是单线程执行，事件处理器也是单线程执行，两个线程不一样，依靠队列保证顺序。这样的好处是io多路复用线程接受和响应 和事件处理之间不会来回切换上下文进行处理。



### redis的淘汰机制
noeviction: 不删除策略, 达到最大内存限制时, 如果需要更多内存, 直接返回错误信息。 大多数写命令都会导致占用更多的内存(有极少数会例外, 如 DEL )。

allkeys-lru: 所有key通用; 优先删除最近最少使用(less recently used ,LRU) 的 key。

volatile-lru: 只限于设置了 expire 的部分; 优先删除最近最少使用(less recently used ,LRU) 的 key。

allkeys-random: 所有key通用; 随机删除一部分 key。

volatile-random: 只限于设置了 expire 的部分; 随机删除一部分 key。

volatile-ttl: 只限于设置了 expire 的部分; 优先删除剩余时间(time to live,TTL) 短的key。

volatile-lfu：从所有配置了过期时间的键中驱逐使用频率最少的键

allkeys-lfu：从所有键中驱逐使用频率最少的键



### redis冷热数据

redis冷热数据的交换
热数据：将所有的Key都认为是热数据
冷数据：对于Value部分，在内存不足的情况下，实例本身会根据最近访问时间，访问频度，Value大小等维度选取出部分value作为冷数据后台异步存储到磁盘上直到内存小于制定阈值为止。

认为key都是热数据的原因
1，Key的访问频度比Value要高很多。
2，作为KV数据库，通常的访问请求都需要先查找Key确认Key是否存在，而要确认一个key不存在，就需要以某种形式检查所有Key的集合。在内存中保留所有Key，可以保证key的查找速度与纯内存版完全一致。
3，Key的大小占比很低。一般的value都比key的值要大

Redis混合存储实例的适用场景
1，数据访问不均匀，存在热点数据；
2，内存不足以放下所有数据，且Value较大(相对于Key而言)

冷热数据的识别
当内存不足时的情况下，实例会按照最近访问时间，访问频度，value大小等维度计算出value的权重，将权重最低的value存储到磁盘上并从内存中删除。
冷热数据识别时采用和Redis类似的近似计算方法,支持多种策略, 通过随机采样小部分数据来降低CPU和内存消耗，通过eviction pool利用采样历史信息来辅助提高准确率。

冷热数据转换过程：
冷数据 --> 热数据
异步：
1，主线程在执行前，判断value是否在内存中
2，如果不在，则生成数据加载任务并挂起当前线程，去执行其他线程
3，后台线程执行完成数据加载任务后更新访问的key的value值，并且通知主线程
4，主线程继续执行上次未完成的线程
同步：
如果发现key不在内存中就，直接启动数据加载任务，然后更新key的值返回给当前线程

热数据 --> 冷数据
异步：
1，主线程在内存接近最大值时，生成一系列数据换出任务；
2，后台线程执行这些数据换出任务，执行完毕之后通知主线程；
3，主线程更新释放内存中的value，更新内存中数据字典中的value为一个简单的元信息；
同步：
如果写入流量过大，异步方式来不及换出数据，导致内存超出最大规格内存。主线程将直接执行数据换出任务，达到变相限流的目的。



### redis底层数据结构

#### str

SDS（simple dynamic string）作为 Redis 的默认字符串表示

3.2之前

```c
struct sdshdr{
     //记录buf数组中已使用字节的数量
     int len; //等于 SDS 保存字符串的长度 4byte
     int free; //记录 buf 数组中未使用字节的数量 4byte
     char buf[]; //字节数组，用于保存字符串 字节\0结尾的字符串占用了1byte
}
```

** SDS 的buf数组会以'\0'结尾，这样可以重用 C 语言库<string.h> 中的一部分函数，避免了不必要的代码重复



3.2之后进行了优化，根据字符串的大小分别选择相应的储存结构

```
struct __attribute__ ((__packed__)) sdshdr5 {
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */ //使用长度
    uint8_t alloc; /* excluding the header and null terminator */ //最大容量
    unsigned char flags; /* 3 lsb of type, 5 unused bits */ // 标志位
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
...
```



redis的对象是存在一个redisObject中 一共占用16个字节

```c
typedef struct redisObject {
    unsigned type:4;  // 类型 4bits
    unsigned encoding:4; // 编码方式 4bits
    unsigned lru:22; // LRU 时间（相对于 server.lruclock） 24bits
    int refcount; // 引用计数 Redis里面的数据可以通过引用计数进行共享 32bits
    void *ptr; // 指向对象的值 64-bit
} robj;// 16bytes
```



内存读取数据到高速缓存区一次读 64个字节

64 - 16 (redisObject) - 8（sds3.2之前) = 39

64 - 16 (redisObject) - 4（sds3.2之后）= 44

String 类型对象三种实现方式，int，embstr，raw
字符串内容可转为 long，采用 int 类型，否则长度<39（3.2版本前39,3.2版本后分界线44） 用 embstr，其他用 raw

#### list

ziplist紧凑的数据结构 quikilist 

#### hash

字典，数据小的话用ziplist



### redis数据一致性原理

**主从一致性原理**

从节点第一次进行连接时，主节点会生成 RDB 文件进行全量复制，同时将新写入的命令存储进缓冲区，发送给从节点，从而保证数据一致性；

为了减少数据同步给主节点带来的压力，可以通过从节点级联的方式进行同步。

**网络开小差了**

网络断连重新连接后，主从节点通过分别维护的偏移量来同步写命令。



### redis 分片策略

为了使得集群能够水平扩展，首要解决的问题就是如何将整个数据集按照一定的规则分配到多个节点上，常用的数据分片的方法有：范围分片，哈希分片，一致性哈希算法，哈希槽等。

Redis Cluster 采用虚拟哈希槽分区，所有的键根据哈希函数映射到 0 ~ 16383 整数槽内，计算公式：slot = CRC16(key) & 16383。每一个节点负责维护一部分槽以及槽所映射的键值数据。

Redis 虚拟槽分区的特点：

- 解耦数据和节点之间的关系，简化了节点扩容和收缩难度。
- 节点自身维护槽的映射关系，不需要客户端或者代理服务维护 槽分区元数据
- 支持节点、槽和键之间的映射查询，用于数据路由，在线集群伸缩等场景。

客户端命令执行流程如下所示：

- 客户端根据本地 slot 缓存发送命令到源节点，如果存在键对应则直接执行并返回结果给客户端。
- 如果节点返回 MOVED 错误，更新本地的 slot 到 Redis 节点的映射关系，然后重新发起请求。
- 如果数据正在迁移中，节点会回复 ASK 重定向异常。格式如下: ( error ) ASK { slot } { targetIP } : {targetPort}
- 客户端从 ASK 重定向异常提取出目标节点信息，发送 asking 命令到目标节点打开客户端连接标识，再执行键命令。

***当持有槽的主节点下线时，从故障发现到自动完成转移期间整个集群是不可用状态，对于大多数业务无法忍受这情况，因此建议将参数 cluster-require-full-coverage 配置为 no ，当主节点故障时只影响它负责槽的相关命令执行，不会影响其他主节点的可用性\***。

#### 数据倾斜问题解决

hot key

将hotkey打散，取随机值（范围：redis 实例数*2的整数倍），变成random_hotkey(要把random放在前面，因为Twemproxy 的分片算法在计算过程中，越靠前的字符权重越大，考后的字符权重则越小。也就是说对于key名，前面的字符差异越大，算出来的分片值差异也越大，更有可能分配到不同的实例)

过程先获取一个随机值，得到一个temhotkey，获取数据的时候先从temhotkey获取，如果没有数据则从hotkey获取，然后将hotkey的值存入temhotkey，temhotkey的过期时间要大于hotkey，避免缓存雪崩

big key

将数据拆分 mset mget

### redis 脑裂

redis的集群脑裂是指因为网络问题，导致redis master节点跟redis slave节点和sentinel集群处于不同的网络分区，此时因为sentinel集群无法感知到master的存在，所以将slave节点提升为master节点。此时存在两个不同的master节点，就像一个大脑分裂成了两个。
集群脑裂问题中，如果客户端还在基于原来的master节点继续写入数据，那么新的master节点将无法同步这些数据，当网络问题解决之后，sentinel集群将原先的master节点降为slave节点，此时再从新的master中同步数据，将会造成大量的数据丢失。

#### 脑裂数据不同步原因

主从切换后，从库一旦升级为新主库，哨兵就会让原主库执行 slave of 命令，和新主库重新进行全量同步。而在全量同步执行的最后阶段，原主库需要清空本地的数据，加载新主库发送的 RDB 文件，这样一来，原主库在主从切换期间保存的新写数据就丢失了。

#### 解决办法

- min-slaves-to-write：这个配置项设置了主库能进行数据同步的最少从库数量；
- min-slaves-max-lag：这个配置项设置了主从库间进行数据复制时，从库给主库发送 ACK 消息的最大延迟（以秒为单位）

即使原主库是假故障，它在假故障期间也无法响应哨兵心跳，也不能和从库进行同步，自然也就无法和从库进行 ACK 确认了。这样一来，min-slaves-to-write 和 min-slaves-max-lag 的组合要求就无法得到满足，原主库就会被限制接收客户端请求，客户端也就不能在原主库中写入新数据了。



### redis事务

1，事务开始

multi命令的执行

2，命令入队

会检查客户输入的命令是否有语法错误，错误直接结束

3，事务执行

按照先进先出的原则，redis不支持回滚机制，但是他会检查命令是否错误，继续执行其他的命令

A ：原子性 （最终执行的是后是原子性）不进行cpu调度，要么都执行要么都不执行

C：一致性  （崩溃后恢复）

I： 持久性 （持久化机制rdb aof） 

D：隔离性 （单线程）

