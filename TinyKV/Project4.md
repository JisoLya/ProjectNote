# Project4 Transactions

考虑如下一个情况，如果两个client同时写一个key会发生什么事情呢？如果另一个client写入紧接着有读取这个key的数据，他们应该读到自己所写入的值吗？在Project4中，你将会通过建立一个事务系统解决这一问题。

事务保证了 *snapshot isolation* 保证事务的读操作将看到一个一致的数据库的版本快照（实际上读取比该事务早的最后一次提交值）。该事务的写操作成功提交，仅当基于该快照的任何并发修改与该事务的修改没有冲突（即写-写冲突）。

TinyKV的事务设计遵循**percolator**协议，一种两阶段提交协议。

事务是一系列的读写集合，一个事务拥有一个开始时间戳以及一个提交时间戳(这个结束时间戳需要大于开始时间戳)。一个事务读取在开始时间戳时有效的一个key的版本。任何一个被当前事务写入的key不能被其他事务写入，在开始时间戳与提交时间戳之间，否则，这个事务需要取消(这个称为写冲突)。

协议开始的时候需要先从TinyScheduler获取一个开始时间戳，接着建立一个本地事务，从数据库中读取数据(使用*包含开始时间戳*的版本`kvGet`和`kvScan`)，但是只在内存写入。一旦事务被创建，client会将一个key作为primary key，client会给TinyKV发送一个`KvPrewrite`信息，信息中包含事务中所有的写请求。TinyKV会尝试对信息中事务要求的所有key进行加锁，如果任何一个上锁失败了，TinyKV会给client响应事务失败，client可以在之后重新尝试事务；如果所有的key都被上锁了，那么prewirte成功，每一个锁都存储了primary key和事物的TTL。

实际上，由于一个事务中的key可能在多个region中，因此他们被存储在不同的Raft Groups中，client会发送多个`KvPrewrite`请求，每一个都发给对应的region leader，每一个prewrite只包含对region的修改。
如果所有的prewirte都成功了，客户端将向包含primary key的region发送提交commit请求。commit请求将会包含一个commit时间戳(同样是从TinyScheduler获取的)，这个时间戳代表着事务commit然后该事务的所有写入在这个时间之后都会对其他事务可见。

如果任何一个prewrite失败了，事务会被client回滚，通过发送一个`KvBatchRollback`请求给所有的region。

在TinyKV中，TTL检查并不是自发进行的，为了触发超时检查，客户端会通过`KvCheckTxnStatus`请求向TinyKV发送当前时间，这个请求通过Start Timestamp和primary key来唯一的标识事务。此时的锁可能已经不存在或者事务已经提交了；如果锁仍旧存在，那么TinyKV将锁的TTL与请求中的时间戳进行比对。如果锁已经超时，TinyKV会将事务回滚。无论结果如何，TinyKV都会返回锁的当前状态，客户端通过这个状态来发送`KvResolveLock`请求执行后续操作(例如强制提交或者回滚)。客户端通常在因其他事务的锁冲突导致prewrite失败时触发此类检查。

e.g.
```text
场景示例：
1. 事务 A 尝试写入键 K1，但发现 事务 B 已持有 K1 的锁。
2. 事务 A 发送 KvCheckTxnStatus 请求，附带当前时间 T1 和事务 
    B 的主键及起始时间戳。
3. TinyKV 检查事务 B 的锁：
    若事务 B 的锁 TTL 已过期（当前时间 T1 > 锁的 start_ts + TTL），则回滚该锁。
    返回锁状态为“已超时”。
4.事务 A 收到响应后，发送 KvResolveLock 请求清理残留锁，随后重试写入。
```

若primary key提交成功，客户端将向其他所有region提交该事务涉及的其他key。此类提交请求应始终成功，因为服务器在响应prewrite请求时已做出承诺：只要收到该事务的提交请求，就必须确保提交成功。一旦客户端收到所有预写操作的确认响应，事务唯一可能失败的情况是超时（此时主键提交也将失败）。而一旦主键提交成功，其他键的锁将永久有效，不再受超时机制影响。

若主键提交失败，客户端将通过`KvBatchRollback`请求回滚整个事务，清理所有预写阶段遗留的锁。

## PartA
在早期项目中实现的 原始API（Raw API） 直接将用户的键（Key）和值（Value）映射到底层存储引擎（Badger）中的键值对。由于 Badger 本身不感知分布式事务层，因此需要在 TinyKV 中处理事务逻辑，并将用户的键值编码后存储到底层存储。这一机制通过多版本并发控制（MVCC） 实现。在本项目中，你将在 TinyKV 中实现 MVCC 层。

实现MVCC意味着利用transactional API来代替原有的API。区别于一个键对应一个值，TinyKV存储一个key所对应值的所有版本。例如，如果一个key对应值10，接着将他修改为20，TinyKV将会存储10和20这两个数据以及他们对应的时间戳，在他们有效的时间内。

TinyKV使用三种列簇，`default`来保存用户的值，`lock`保存锁，`write`来记录修改。列簇`lock`可以利用user key来访问；他存储了序列化的`Lock`数据结构(定义在lock.go中)。`default`列簇通过user key和事务的start timestamp来访问，只存储了用户的数据。`write`列簇通过user key和事务的commit timestamp来访问，存储了`Write`数据结构(定义在write.go)

user key和timestamp被拼接成为一个encoded key。这样的编码可以保证首先根据user key升序排列，接着根据time stamp升序排列。这样就保证了在迭代encoded key的时候，获取的都是最新的版本。编码和解码的函数都定义在transaction.go中。

这一阶段需要实现一个简单的结构体`MvccTxn`,在partB和partC中，你需要使用`MvccTxn API`来实现一个事务API， `MvccTxn`提供了读写操作，这写操作基于user key和锁、写入、和值的逻辑表示。修改都被`MvccTxn`手机，一旦某条命令的所有修改都被收集了，他们会被立刻写入到下层的数据库。这保证了命令的成功或失败都是原子性的。注意到一个MVCC事务和TinySQL的事务是不同的。一个MVCC事务只包含了*一条命令的所有操作*而不是*一系列的命令*。

