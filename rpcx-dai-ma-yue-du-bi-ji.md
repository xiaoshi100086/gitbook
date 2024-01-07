# rpcx代码阅读笔记

### 连接池

#### 作用

tcp的连接建立需要额外的工作，并且是运输层的概念，与应用层无关。所以client侧发起rpc请求是可以复用连接，减少不必要的流程。

#### 使用场景

* 频繁进行tcp读写的场景适合使用连接池
* 不频繁的tcp读写不适合连接池。一方面防止过度设计，一方面也是减少额外的工作（连接池中，需要应用层和tcp的心跳为维护连接的状态 ）

#### 设计要点

减少连接池的读写竞争

可以梳理连接的所属关系如下：

| 第1层 | 第1.5层 | 第2层 | 第3层 |
| --- | ----- | --- | --- |
| 服务  | 配置    | 节点  | 连接  |

其中1.5层的配置是指连接的配置，比如tcp保活的各种参数。

其中服务和配置是服务初始化就确认且保持不表，节点会因为被调服务的扩缩容而发生增删，连接会因为网络波动和读写异常发生关闭。所以我们可以为每个服务和配置维护一个连接池，这样减少了不同服务和配置使用连接池的读写竞争。

当然我们还可以使用区间锁来减少读写竞争，对于一个连接池，不同的节点地址放到不同的区间。只给区间加读写锁，减少不同节点的读写竞争。

连接池的大小

todo 看了http的连接池再来写

### 单连接

单连接是连接复用的一种特殊的方式，即，同时间多个请求使用一个连接完成读写。为了区分报文是哪个连接的，服务框架会为每个请求分配一个请求id。client发起请求会将该请求id和调用的一些参数保持起来，收到响应后再删除该请求id。

特别的，关闭连接后也是需要将还在使用该连接的请求进行失败响应。

## 连接的关闭

当没有使用连接池，每次请求和响应都需要关闭连接。如果使用正常的tcp关闭会有timewait问题（频繁的建立和关闭tcp连接会有大量的临时端口位于timewait状态，进而导致无端口可用的问题），可以使用如下方法解决：

* 让client主动关闭。这样server端不会有该问题，因为server会处理大量请求会频繁进行tcpd连接和关闭，client通常不会。但client如果也是这样，也不能解决问题
* 调小timewait。不建议这样做，因为调整是对全系统的连接生效，并且timewait是为了保证正常退出的，调小timewait可能会导致异常
* 使用长连接，即连接池。这样可以避免关闭连接
* 使用异常退出，使能linger并设置退出时间为0。推荐。这样退出就直接发送RESET包异常关闭连接。但异常关闭会导致socket的写缓存的数据直接丢失，在网络游荡的报文可能发到下一个新连接。但是这些问题可以通过应用层来进行保证。rpc每次结束请求都不会有缓存数据没有发送，请求都分配一个seq自增字段来防止游荡的报文发到下一个新连接。
