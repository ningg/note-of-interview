## 常见问题汇总

### Redis vs. Memcached

几个疑问：

1. Redis 是单线程对外服务的，具体是如何对外服务的？
2. Memcached 缓存，是什么模式，对外服务的？
3. 2 种方式，有什么差异和不同：
	1. 单线程 + IO 多路复用
	2. 多线程

简单说明：

> Redis 是 C 语言实现的，Memcached 也是 C 语言实现的。
> 
> 1. 支持的数据类型：
>     1. Redis 支持丰富的数据类型：List、Set、ZSet、Hash、String
>     2. Memcached 支持：String
> 2. 持久化和主从结构：
>     1. Redis，支持 2 类持久化 RDB 和 AOF，也支持 Master-Slave 结构，异步复制
>     2. Memcached 不支持持久化，断电数据丢失
> 3. 虚拟内存：
>     1. Redis 2.4 之前，支持 Virtual Memory，但因为性能问题，不再支持
>     2. Memcached 也不支持 Virtual Memory，因此，约束存储空间为内存大小
> 4. 事务：
>     1. Redis 支持简单事务：MULTI，多个操作绑定在一起执行
>     2. Memcached 不支持事务
> 
> 一般考虑：
> 
> 1. 需要数据持久化，选用 Redis
> 2. 需要丰富的数据类型，选用 Redis（例如消息队列、有序集合）
> 3. 只是简单的分布式缓存，选用 Memcached （无法做数据库，因为无法持久化）

Redis vs. Memcached 对比如下：

1. 数据类型：
	1. Memcached 只支持简单的数据类型：字符串；
	2. Redis 数据类型更丰富：List、Set、ZSet、Hash、字符串；
2. 持久化：
	1. Memcached 不支持持久化，断电重启后，数据丢失；
	2. Redis 支持持久化：AOF、RDB；
3. 集群方案：
	1. Memcached 需要借助 proxy 进行一致性 hash；
	2. Redis 有 redis Cluster 方案；
4. 主从结构：
	1. Redis：自带 Master-Slave 结构
	2. Memcached：需要依赖 Proxy 自建 Master-Slave 结构，而且，可能导致 MS 上数据不一致，细节参考：[分布式缓存的一起问题--Memcached](https://timyang.net/data/cache-failure/)
5. 网络 IO 模型：都是使用了 IO 多路复用
	1. Memcached：多线程，master、worker 线程，master 监听主线程和 worker 工作线程；多线程，充分发挥 CPU 的多核优势，但需要加锁解决多线程并发问题，引入性能损耗；(CAS 命令，乐观锁)
	2. Redis：单线程，因为没有锁竞争问题，但，单线程在 CPU 计算过程中，IO 是完全阻塞的


#### 单线程 + IO 多路复用

单线程 + IO 多路复用，特点：

1. 没有并发，不需要加锁
2. 没有线程上下文切换
3. 只能使用 CPU 的一个核心：可以在多核服务器上，启动多个 Redis 实例，来改善
4. 因为只能使用一个 CPU 核心，不适合「计算密集型」的操作，因此，大部分都是「读」和「写」
5. 单线程，所有的命令串行执行，每个命令的 内存 IO、CPU 执行过程，是串行的

适用场景：

1. key-value：小数据量，避免内存 IO，导致的 CPU 长时间空闲
2. 长连接：适用于长连接的情况（多线程模式长连接容易造成线程过多，造成频繁调度）



#### 多线程

多线程，对外提供服务，特点：

1. CPU 利用率：充分利用多核 CPU；
2. 大量请求时，产生大量线程，上下文切换开销大
3. 多线程并发时，共享资源加锁，锁竞争，引入开销

适用场景：

1. Key-value：大数据量（单个 key-value 100KB），能够充分利用 CPU 资源


#### 参考来源

* [对比 Redis 与 Memcached](http://blog.huangz.me/diary/2015/comparison-of-redis-and-memcached.html)
* [Clarifications about Redis and Memcached-From antirez](http://antirez.com/news/94)
* [Memcached](https://github.com/memcached/memcached)
* [Redis](https://github.com/antirez/redis)
* [Redis和Memcache对比及选择](http://www.cnblogs.com/EE-NovRain/p/3268476.html)
* [memcached与redis实现的对比](https://www.qcloud.com/community/article/129)
* [Redis与Memcached的区别](http://gnucto.blog.51cto.com/3391516/998509)
* [Memcached和Redis比较](http://www.bbsmax.com/A/kvJ3ExwdgM/)
* [分布式缓存的一起问题--Memcached](https://timyang.net/data/cache-failure/)

### JAVA BIO vs. NIO





### TCP 状态

http://www.cnblogs.com/sunxucool/p/3449068.html



### 生产者-消费者模式



### 














#### 参考来源

* [Java NIO梳理](http://ningg.top/java-nio/)















