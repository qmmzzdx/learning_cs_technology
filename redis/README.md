# Redis 相关知识总结

## 1. Redis是什么？

**Redis（Remote Dictionary Server）** 是一个开源的、基于内存的、支持多种数据结构的 NoSQL 键值数据库。

**核心特性**：
- **内存存储**：数据主要存储在内存中，读写性能极高
- **数据结构丰富**：支持字符串、哈希、列表、集合、有序集合等
- **持久化**：支持 RDB 快照和 AOF 日志两种持久化方式
- **高可用**：支持主从复制、哨兵模式、集群模式
- **原子操作**：所有操作都是原子性的
- **发布订阅**：支持消息的发布订阅模式

**应用场景**：
- 缓存系统
- 会话存储（Session Storage）
- 实时排行榜
- 消息队列
- 分布式锁
- 计数器
- 社交网络功能

## 2. Redis底层的数据结构？

### 2.1 基本数据类型

**String（字符串）**：
- 最大存储 512MB
- 可存储文本、数字、二进制数据
- 底层实现：SDS（简单动态字符串）

**Hash（哈希）**：
- 字段值对集合
- 适合存储对象
- 底层实现：ziplist 或 hashtable

**List（列表）**：
- 有序字符串列表
- 支持左右插入弹出
- 底层实现：ziplist 或 linkedlist

**Set（集合）**：
- 无序唯一字符串集合
- 支持交集、并集、差集运算
- 底层实现：intset 或 hashtable

**ZSet（有序集合）**：
- 有序唯一字符串集合
- 每个元素关联一个分数用于排序
- 底层实现：ziplist 或 skiplist + hashtable

### 2.2 高级数据类型

**BitMap（位图）**：
```redis
SETBIT online:20231017 1001 1        # 用户1001上线
GETBIT online:20231017 1001          # 检查用户是否上线
BITCOUNT online:20231017             # 统计在线用户数
```

**HyperLogLog**：
```redis
PFADD uv:20231017 "user1" "user2" "user3"
PFCOUNT uv:20231017                  # 估算UV，误差约0.81%
```

**GEO（地理位置）**：
```redis
GEOADD drivers 116.3974 39.9093 "driver1"
GEORADIUS drivers 116.3974 39.9093 10 km  # 查找10公里内司机
```

**Stream（流）**：
```redis
XADD orders * user_id 1001 product "iPhone"  # 添加消息
XREAD COUNT 10 STREAMS orders 0              # 读取消息
```

### 2.3 应用场景详解

**String 应用场景**：
```redis
# 缓存用户信息
SET user:1001 '{"name":"John","age":30}'
GET user:1001

# 分布式锁
SET lock:order 1 NX PX 30000

# 计数器
INCR article:1001:views
INCRBY user:1001:points 10
```

**List 应用场景**：
```redis
# 消息队列
LPUSH message:queue "task1"
RPOP message:queue

# 最新消息列表
LPUSH news "latest news"
LTRIM news 0 99  # 保持最新100条
```

**Hash 应用场景**：
```redis
# 存储用户对象
HSET user:1001 name "John" age 30 email "john@example.com"
HGET user:1001 name
HGETALL user:1001

# 购物车
HSET cart:1001 product1 2 product2 1
HINCRBY cart:1001 product1 1
```

**Set 应用场景**：
```redis
# 标签系统
SADD article:1001:tags "tech" "programming"
SINTER article:1001:tags article:1002:tags  # 共同标签

# 抽奖系统
SADD lottery:20231017 "user1" "user2" "user3"
SRANDMEMBER lottery:20231017 1  # 随机抽取1个
```

**ZSet 应用场景**：
```redis
# 排行榜
ZADD leaderboard 100 "player1" 200 "player2"
ZREVRANGE leaderboard 0 9 WITHSCORES  # 前10名

# 延迟队列
ZADD delay:queue 1697517045 "task1"  # 时间戳作为分数
ZRANGEBYSCORE delay:queue 0 1697517044  # 获取到期任务
```

## 3. ZSet底层实现？

### 3.1 存储结构选择

**使用 ziplist 的条件**：
- 元素数量 < 128
- 每个元素值 < 64 字节

**使用 skiplist + dict 的条件**：
- 不满足上述条件时使用跳表+字典组合

### 3.2 跳表结构详解

**跳表节点结构**：
```c
typedef struct zskiplistNode {
    sds ele;                    // 元素值
    double score;               // 分数
    struct zskiplistNode *backward;  // 后向指针
    struct zskiplistLevel {
        struct zskiplistNode *forward;  // 前向指针
        unsigned long span;             // 跨度
    } level[];                  // 层级数组
} zskiplistNode;
```

**跳表结构**：
```c
typedef struct zskiplist {
    struct zskiplistNode *header, *tail;
    unsigned long length;       // 节点数量
    int level;                  // 最大层数
} zskiplist;
```

**有序集合结构**：
```c
typedef struct zset {
    dict *dict;                 // 字典，用于O(1)查找
    zskiplist *zsl;             // 跳表，用于范围查询
} zset;
```

### 3.3 跳表操作复杂度

- **插入**：O(logN)
- **删除**：O(logN)  
- **查找**：O(logN)
- **范围查询**：O(logN + M)，M为返回元素数量

## 4. 跳表实现原理？

### 4.1 跳表层级机制

**层高生成算法**：
```c
int zslRandomLevel(void) {
    int level = 1;
    while ((random() & 0xFFFF) < (0.25 * 0xFFFF)) {
        level += 1;
    }
    return (level < 64) ? level : 64;
}
```

**跳表示例**：
```
Level 3: 1 --------------------------------> 9
Level 2: 1 ------------> 5 ------------> 9
Level 1: 1 ----> 3 ----> 5 ----> 7 ----> 9
Level 0: 1 -> 2 -> 3 -> 4 -> 5 -> 6 -> 7 -> 8 -> 9
```

### 4.2 跳表操作详解

**查找过程**：
1. 从最高层开始遍历
2. 如果下一个节点分数小于目标分数，继续前进
3. 否则下降到下一层继续查找
4. 直到找到目标节点或到达底层

**插入过程**：
1. 随机生成节点层高
2. 记录每层的前驱节点
3. 创建新节点并插入到各层链表中
4. 更新跨度信息

## 5. Redis为什么用跳表不用B+树？

### 5.1 性能对比

**内存访问模式**：
- **跳表**：指针跳转，缓存友好
- **B+树**：节点结构复杂，缓存不友好

**实现复杂度**：
- **跳表**：约200行代码，无复杂平衡操作
- **B+树**：需要节点分裂合并，代码复杂

**写入性能**：
- **跳表**：插入删除只需调整局部指针
- **B+树**：可能触发递归节点分裂，成本高

### 5.2 实际测试数据

| 操作类型 | 跳表 | B+树 |
|---------|------|------|
| 插入性能 | 100% | 85% |
| 查询性能 | 100% | 95% |
| 内存占用 | 100% | 120% |
| 代码复杂度 | 简单 | 复杂 |

## 6. 压缩列表实现？

### 6.1 压缩列表结构

**整体布局**：
```
+--------+--------+--------+------------+--------+--------+--------+
| zlbytes | zltail | zllen  | entry1     | entry2 | ...   | zlend  |
+--------+--------+--------+------------+--------+--------+--------+
```

**节点结构**：
```
+----------+-----------+-------+
| prevlen  | encoding  | data  |
+----------+-----------+-------+
```

**prevlen 编码**：
- 前驱节点长度 < 254：1字节
- 前驱节点长度 >= 254：5字节（0xFE + 4字节长度）

### 6.2 连锁更新问题

**场景**：
1. 在多个250字节节点中间插入一个254字节节点
2. 后续节点的prevlen需要从1字节扩展为5字节
3. 节点扩展可能触发后续节点的连锁扩展

**解决方案**：Redis 3.2 引入 quicklist，5.0 引入 listpack

## 7. listpack介绍？

### 7.1 listpack结构

**整体布局**：
```
+--------+--------+--------+------------+--------+--------+
| lpbytes | numele | entry1 | entry2     | ...   | lpEnd  |
+--------+--------+--------+------------+--------+--------+
```

**节点结构**：
```
+-----------+-------+---------+
| encoding  | data  | len     |
+-----------+-------+---------+
```

**优势**：
- 无prevlen字段，避免连锁更新
- 更紧凑的内存布局
- 支持多种编码节省内存

## 8. 哈希表扩容机制？

### 8.1 渐进式rehash过程

**触发条件**：
- 负载因子 > 1 且未进行BGSAVE/BGREWRITEAOF
- 负载因子 > 5 强制扩容

**rehash步骤**：
1. 分配ht[1]，大小为第一个大于等于ht[0].used*2的2^n
2. 设置rehashidx=0，开始rehash
3. 每次增删改查时，迁移ht[0]中rehashidx位置的桶
4. rehashidx++，直到完成所有桶迁移
5. 释放ht[0]，将ht[1]设置为ht[0]

### 8.2 rehash期间的读写操作

**查找操作**：
1. 在ht[0]中查找
2. 如果未找到且在rehash中，在ht[1]中查找

**插入操作**：
- 直接插入到ht[1]中

**删除/更新操作**：
- 先在ht[0]中操作，如果未找到且在rehash中，在ht[1]中操作

## 9. String的存储结构？

### 9.1 SDS结构

**SDS定义**：
```c
struct sdshdr {
    int len;        // 已使用长度
    int alloc;      // 总分配长度
    char flags;     // 类型标识
    char buf[];     // 字符数组
};
```

**SDS类型**：
- sdshdr5：长度 < 32
- sdshdr8：长度 < 256  
- sdshdr16：长度 < 65536
- sdshdr32：长度 < 2^32
- sdshdr64：超大字符串

### 9.2 SDS优势

**O(1)获取长度**：
```c
static inline size_t sdslen(const sds s) {
    unsigned char flags = s[-1];
    switch(flags & 7) {
        case 0: return ((struct sdshdr5 *)s)->len;
        // ... 其他类型
    }
}
```

**二进制安全**：
- 可以存储包含'\0'的数据
- 不依赖C字符串的结束符

**自动扩容**：
```c
sds sdsMakeRoomFor(sds s, size_t addlen) {
    // 扩容逻辑
}
```

## 10. Redis为什么快？

### 10.1 内存操作

- 数据存储在内存中，无磁盘I/O瓶颈
- 内存访问速度比磁盘快几个数量级

### 10.2 高效数据结构

- 哈希表：O(1)时间复杂度
- 跳表：O(logN)范围查询
- 各种优化后的数据结构

### 10.3 单线程模型

**优势**：
- 无锁竞争
- 无上下文切换开销
- 无死锁问题

**瓶颈**：
- CPU密集型操作可能阻塞
- 单个大key操作影响整体性能

### 10.4 I/O多路复用

基于epoll/kqueue实现，单线程处理大量并发连接

## 11. Redis的多线程应用？

### 11.1 后台线程（BIO）

**关闭文件线程**：
```c
bio_init();
bio_create_background_job(BIO_CLOSE_FILE, fd);
```

**AOF刷盘线程**：
- 异步执行fsync操作
- 避免阻塞主线程

**lazyfree线程**：
```redis
UNLINK big_key        # 异步删除
FLUSHDB ASYNC        # 异步清空数据库
```

### 11.2 I/O多线程（6.0+）

**配置**：
```conf
io-threads 4          # 启用4个I/O线程
io-threads-do-reads yes  # 启用读多线程
```

**工作模式**：
- 主线程负责accept连接
- I/O线程负责网络读写
- 命令执行仍在主线程

## 12. I/O多路复用实现？

### 12.1 Reactor模式

**事件循环**：
```c
void aeMain(aeEventLoop *eventLoop) {
    while (!eventLoop->stop) {
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
```

**事件处理**：
```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags) {
    // 获取就绪事件
    numevents = aeApiPoll(eventLoop, tvp);
    
    // 处理文件事件
    for (j = 0; j < numevents; j++) {
        // 处理读/写事件
    }
}
```

### 12.2 支持的多路复用机制

- epoll（Linux）
- kqueue（FreeBSD/ macOS）
- select（跨平台）
- evport（Solaris）

## 13. Redis网络模型？

### 13.1 6.0前：单Reactor单线程

**架构**：
```
Client Requests → I/O Multiplexing → Command Execution → Response
```

**瓶颈**：
- 网络I/O和命令执行都在单线程
- 大value传输可能阻塞其他请求

### 13.2 6.0后：多线程网络I/O

**架构**：
```
Client Requests → I/O Threads (Network) → Main Thread (Command) → Response
```

**优势**：
- 网络读写并行化
- 提升网络密集型应用性能

## 14. Redis如何保证原子性？

### 14.1 单命令原子性

**示例**：
```redis
INCR counter        # 原子操作
HSET user name John # 原子操作
SADD tags "redis"   # 原子操作
```

### 14.2 多命令原子性

**Lua脚本**：
```redis
EVAL "local current = redis.call('GET', KEYS[1])
      if current == ARGV[1] then
          return redis.call('DEL', KEYS[1])
      else
          return 0
      end" 1 lock:order order123
```

**事务**：
```redis
MULTI
SET key1 value1
SET key2 value2
EXEC
```

**WATCH命令**：
```redis
WATCH balance
balance = GET balance
if balance >= 100:
    MULTI
    DECRBY balance 100
    EXEC
```

## 15. Redis持久化方式及优缺点？

### 15.1 RDB（Redis Database）

**配置**：
```conf
save 900 1          # 900秒内至少1个key变化
save 300 10         # 300秒内至少10个key变化  
save 60 10000       # 60秒内至少10000个key变化

dbfilename dump.rdb
dir /var/lib/redis
```

**优点**：
- 文件紧凑，适合备份
- 恢复速度快
- 最大化Redis性能

**缺点**：
- 可能丢失最后一次快照后的数据
- 数据集大时fork可能阻塞服务

### 15.2 AOF（Append Only File）

**配置**：
```conf
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec  # 每秒同步

# 重写配置
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

**优点**：
- 数据安全性高
- 可读性强，可手动修复
- 多种fsync策略可选

**缺点**：
- 文件体积大
- 恢复速度慢
- 写性能受影响

### 15.3 混合持久化（4.0+）

**配置**：
```conf
aof-use-rdb-preamble yes
```

**优势**：
- 快速加载RDB
- 保证数据完整性

## 16. 过期删除与内存淘汰策略区别？

### 16.1 过期删除

**目的**：清理已过期的键值对
**策略**：惰性删除 + 定期删除
**触发**：基于TTL时间

### 16.2 内存淘汰

**目的**：内存不足时释放空间
**策略**：多种淘汰算法选择
**触发**：内存达到maxmemory限制

## 17. 内存淘汰策略？

### 17.1 策略分类

**不淘汰**：
- `noeviction`：返回错误（默认）

**淘汰过期键**：
- `volatile-lru`：LRU算法淘汰
- `volatile-lfu`：LFU算法淘汰  
- `volatile-random`：随机淘汰
- `volatile-ttl`：淘汰TTL小的

**全局淘汰**：
- `allkeys-lru`：所有键LRU淘汰
- `allkeys-lfu`：所有键LFU淘汰
- `allkeys-random`：所有键随机淘汰

### 17.2 策略选择建议

**缓存场景**：`allkeys-lru`
**持久化数据**：`volatile-lru`
**公平性要求**：`allkeys-random`

## 18. 过期删除策略？

### 18.1 惰性删除

**实现**：
```c
int expireIfNeeded(redisDb *db, robj *key) {
    if (!keyIsExpired(db,key)) return 0;
    
    // 删除过期键
    return deleteExpiredKeyAndPropagate(db,key);
}
```

**优势**：CPU友好，只在访问时检查
**劣势**：可能积累大量过期键

### 18.2 定期删除

**实现**：
```c
void activeExpireCycle(int type) {
    // 随机抽查20个键
    // 如果过期比例>25%，继续抽查
}
```

**配置**：
```conf
hz 10  # 每秒执行10次定期删除
```

## 19. 过期键是否立即删除？

**否**，原因：
- 立即删除会占用大量CPU时间
- 影响Redis响应性能
- 采用惰性+定期删除平衡性能与内存

## 20. 主从同步机制？

### 20.1 全量同步

**过程**：
1. 从节点发送`PSYNC ? -1`
2. 主节点执行`BGSAVE`生成RDB
3. 传输RDB文件到从节点
4. 从节点加载RDB
5. 主节点发送复制缓冲区命令
6. 从节点执行缓冲命令

### 20.2 增量同步

**条件**：
- 从节点复制偏移量在复制缓冲区内
- 主节点runid匹配

**过程**：
1. 从节点发送`PSYNC <runid> <offset>`
2. 主节点发送偏移量后的命令

### 20.3 复制缓冲区

**配置**：
```conf
repl-backlog-size 1mb    # 缓冲区大小
repl-backlog-ttl 3600    # 缓冲区存活时间
```

## 21. 主从/集群能否保证数据一致性？

**AP系统**：
- 可用性优先
- 网络分区时可能数据不一致
- 最终一致性

**同步机制**：
- 异步复制（默认）
- 半同步复制（需要等待从节点确认）

## 22. 哨兵机制原理？

### 22.1 哨兵功能

**监控**：定期检查主从节点状态
**通知**：向管理员报告故障
**自动故障转移**：选举新主节点
**配置提供者**：客户端服务发现

### 22.2 故障转移过程

1. **主观下线**：单个哨兵认为主节点不可用
2. **客观下线**：多数哨兵确认主节点不可用
3. **选举Leader**：Raft算法选举哨兵Leader
4. **选主**：根据规则选择新主节点
5. **切换**：提升从节点为主节点

### 22.3 选主规则

1. 网络连接正常的从节点
2. 优先级（slave-priority）高的
3. 复制偏移量大的
4. runid小的

## 23. 集群模式优缺点？

### 23.1 集群架构

**数据分片**：16384个哈希槽
**节点类型**：主节点、从节点
**路由方式**：客户端重定向

### 23.2 优点

- 自动数据分片
- 高可用性
- 线性扩展
- 部分节点故障不影响整体

### 23.3 缺点

- 客户端复杂度增加
- 不支持多键操作（除非在同一槽）
- 数据迁移复杂
- 运维复杂度高

### 23.4 数据分布

```python
def slot(key):
    crc = crc16(key)
    return crc % 16384
```

## 24. Redis使用场景？

### 24.1 缓存

```redis
# 缓存用户信息
SET user:1001:cache '{"name":"John"}' EX 3600

# 缓存穿透保护
SET null:user:9999 "" EX 30
```

### 24.2 分布式锁

```redis
SET lock:resource unique_id NX PX 30000
```

### 24.3 计数器

```redis
INCR page:views:20231017
INCRBY user:1001:points 10
```

### 24.4 消息队列

```redis
# List实现
LPUSH queue task1
BRPOP queue 0

# Stream实现
XADD mystream * field1 value1
XREAD GROUP mygroup consumer1 COUNT 1 STREAMS mystream >
```

## 25. Redis为什么比MySQL快？

### 25.1 架构对比

| 特性 | Redis | MySQL |
|------|-------|-------|
| 存储介质 | 内存 | 磁盘 |
| 数据结构 | 丰富 | 有限 |
| 查询复杂度 | O(1)/O(logN) | O(logN) |
| 并发模型 | 单线程 | 多线程 |

### 25.2 性能数据

- Redis：10W+ QPS
- MySQL：数千QPS

## 26. 本地缓存 vs Redis缓存？

### 26.1 本地缓存

**实现**：HashMap, Guava Cache, Caffeine
**优点**：
- 零网络开销
- 极低延迟
- 不受网络故障影响

**缺点**：
- 内存限制
- 数据不一致
- 集群环境同步困难

### 26.2 Redis缓存

**优点**：
- 数据一致性
- 可扩展性
- 持久化
- 丰富数据结构

**缺点**：
- 网络开销
- 依赖网络稳定性

### 26.3 多级缓存架构

```
请求 → 本地缓存 → Redis缓存 → 数据库
```

## 27. 高并发场景性能？

### 27.1 缓存命中场景

```redis
# 单节点性能
SET key value  # 10W+ QPS
GET key        # 10W+ QPS
```

### 27.2 缓存未命中场景

```sql
-- MySQL单节点
SELECT * FROM users WHERE id = 1001;  # 5000 QPS
```

### 27.3 优化策略

- 缓存预热
- 缓存降级
- 限流保护
- 集群扩展

## 28. 分布式锁实现原理？

### 28.1 加锁

```redis
SET lock:order order123 NX PX 30000
```

**参数说明**：
- `NX`：仅当key不存在时设置
- `PX 30000`：30秒后自动过期
- `order123`：唯一标识，防止误删

### 28.2 解锁

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

### 28.3 锁续期

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("pexpire", KEYS[1], ARGV[2])
else
    return 0
end
```

## 29. 大Key问题与解决？

### 29.1 大Key定义

- String类型：value > 1MB
- 集合类型：元素数量 > 1万
- 哈希/列表：总大小 > 1MB

### 29.2 问题影响

- 内存压力
- 阻塞操作
- 网络拥塞
- 数据倾斜

### 29.3 解决方案

**拆分大Key**：
```redis
# 原始大Hash
HSET user:1001:data field1 value1 ... field10000 value10000

# 拆分后
HSET user:1001:data:1 field1 value1 ... field1000 value1000
HSET user:1001:data:2 field1001 value1001 ... field2000 value2000
```

**异步删除**：
```redis
UNLINK big_key  # 异步删除
```

**监控预警**：
```bash
redis-cli --bigkeys
redis-cli memory usage key_name
```

## 30. 热Key问题与解决？

### 30.1 热Key检测

```redis
# 监控命令调用频率
redis-cli --hotkeys

# 客户端统计
INFO commandstats
```

### 30.2 解决方案

**本地缓存**：
```go
// 客户端本地缓存热key
type LocalCache struct {
    data sync.Map
}

func (lc *LocalCache) Get(key string) (interface{}, bool) {
    return lc.data.Load(key)
}

func (lc *LocalCache) Set(key string, value interface{}) {
    lc.data.Store(key, value)
}
```

**Key复制**：
```redis
# 将热key复制到多个节点
SET hot_key:1 value
SET hot_key:2 value
SET hot_key:3 value
```

**读写分离**：增加从节点分担读压力

## 31. 缓存一致性方案？

### 31.1 读写策略

**读策略**：
```go
func GetWithCache(rdb *redis.Client, key string, fallback func() string) string {
    ctx := context.Background()
    val, err := rdb.Get(ctx, key).Result()
    if err == nil {
        return val
    }
    // 缓存未命中，查DB
    val = fallback()
    rdb.Set(ctx, key, val, time.Hour)
    return val
}
```

**写策略**：
```go
func UpdateWithCache(rdb *redis.Client, key string, update func() error) error {
    // 先更新DB
    if err := update(); err != nil {
        return err
    }
    // 再删缓存
    return rdb.Del(context.Background(), key).Err()
}
```

### 31.2 优化方案

**删除重试**：
```go
// 删除失败时重试
func DeleteWithRetry(rdb *redis.Client, key string, retries int) error {
    ctx := context.Background()
    for i := 0; i < retries; i++ {
        if err := rdb.Del(ctx, key).Err(); err == nil {
            return nil
        }
        time.Sleep(100 * time.Millisecond)
    }
    return errors.New("delete failed after retries")
}
```

**binlog同步**：
```
MySQL → Canal → MQ → 缓存更新服务 → Redis
```

## 32. 缓存雪崩/击穿/穿透？

### 32.1 缓存雪崩

**现象**：大量缓存同时失效
**解决**：
```go
// 随机过期时间
func SetWithRandomExpire(rdb *redis.Client, key, value string, baseExpire time.Duration) {
    randomSec := rand.Intn(60) // 0-59秒随机
    expire := baseExpire + time.Duration(randomSec)*time.Second
    rdb.Set(context.Background(), key, value, expire)
}

// 互斥锁
func GetWithLock(rdb *redis.Client, key string, fallback func() string) string {
    ctx := context.Background()
    val, _ := rdb.Get(ctx, key).Result()
    if val != "" {
        return val
    }
    
    // 获取分布式锁
    lockKey := "lock:" + key
    if rdb.SetNX(ctx, lockKey, "1", 10*time.Second).Val() {
        defer rdb.Del(ctx, lockKey)
        val = fallback()
        rdb.Set(ctx, key, val, time.Hour)
    } else {
        time.Sleep(100 * time.Millisecond)
        return GetWithLock(rdb, key, fallback)
    }
    return val
}
```

### 32.2 缓存击穿

**现象**：热点数据过期瞬间大量请求
**解决**：
```go
// 永不过期 + 异步更新
func GetHotData(rdb *redis.Client, key string, fallback func() string) string {
    ctx := context.Background()
    val, _ := rdb.Get(ctx, key).Result()
    if val == "" {
        // 异步更新
        if rdb.SetNX(ctx, "update:"+key, "1", 30*time.Second).Val() {
            go func() {
                newVal := fallback()
                rdb.Set(ctx, key, newVal, 0)
                rdb.Del(ctx, "update:"+key)
            }()
        }
    }
    return val
}
```

### 32.3 缓存穿透

**现象**：查询不存在的数据
**解决**：
```go
// 布隆过滤器
type BloomFilter struct {
    bitset []bool
    hashes []func(string) uint32
}

func (bf *BloomFilter) MightContain(key string) bool {
    for _, hash := range bf.hashes {
        if !bf.bitset[hash(key)%uint32(len(bf.bitset))] {
            return false
        }
    }
    return true
}

// 缓存空值
func GetWithNullCache(rdb *redis.Client, key string, fallback func() (string, bool)) string {
    ctx := context.Background()
    val, _ := rdb.Get(ctx, key).Result()
    if val == "NULL" {
        return ""
    }
    if val != "" {
        return val
    }
    
    realVal, exists := fallback()
    if !exists {
        rdb.Set(ctx, key, "NULL", 5*time.Minute)
        return ""
    }
    
    rdb.Set(ctx, key, realVal, time.Hour)
    return realVal
}
```

## 33. 布隆过滤器原理？

### 33.1 数据结构

**位数组**：`BitSet`，初始全为0
**哈希函数**：多个独立的哈希函数

### 33.2 操作原理

**添加元素**：
1. 计算元素的k个哈希值
2. 将位数组对应位置设为1

**检查元素**：
1. 计算元素的k个哈希值
2. 检查位数组对应位置是否全为1
   - 全为1：可能存在（可能误判）
   - 不全为1：肯定不存在

### 33.3 Redis实现

```redis
# 使用Redis BitMap实现
SETBIT bloomfilter 1000 1
GETBIT bloomfilter 1000

# 使用RedisBloom模块
BF.ADD users user123
BF.EXISTS users user123
```

## 34. 秒杀场景设计？

### 34.1 数据库方案

```sql
-- 悲观锁
SELECT * FROM products WHERE id = 1001 FOR UPDATE;

-- 乐观锁
UPDATE products 
SET stock = stock - 1, version = version + 1 
WHERE id = 1001 AND stock > 0 AND version = #{version};
```

### 34.2 Redis方案

**库存预减**：
```redis
# 初始化库存
SET stock:1001 1000

# 原子减库存
DECR stock:1001
```

**用户去重**：
```redis
SADD users:seckill:1001 user123
```

**异步下单**：
```redis
LPUSH order:queue '{"userId":123,"productId":1001}'
```

### 34.3 完整流程

1. 校验用户资格
2. Redis预减库存
3. 记录用户购买
4. 异步创建订单
5. 返回结果

## 35. Redis常用命令？

### 35.1 Key操作

```redis
# 查找Key
KEYS user:*              # 模式匹配（生产环境慎用）
SCAN 0 MATCH user:* COUNT 100  # 游标扫描

# 删除Key
DEL key1 key2
UNLINK key1              # 异步删除

# 过期时间
EXPIRE key 60
TTL key
PERSIST key              # 移除过期时间
```

### 35.2 String操作

```redis
# 基本操作
SET key value
GET key
MSET key1 value1 key2 value2
MGET key1 key2

# 数值操作
INCR counter
INCRBY counter 10
DECR counter

# 位操作
SETBIT bitmap 1000 1
GETBIT bitmap 1000
BITCOUNT bitmap
```

### 35.3 Hash操作

```redis
# 字段操作
HSET user:1001 name "John" age 30
HGET user:1001 name
HMGET user:1001 name age
HGETALL user:1001

# 数值操作
HINCRBY user:1001 age 1
HINCRBYFLOAT user:1001 score 1.5

# 扫描
HSCAN user:1001 0
```

### 35.4 List操作

```redis
# 插入弹出
LPUSH list value1
RPUSH list value2
LPOP list
RPOP list

# 范围操作
LRANGE list 0 -1
LTRIM list 0 99         # 保留前100个
LINDEX list 0            # 获取指定位置

# 阻塞操作
BLPOP list 10            # 阻塞10秒
BRPOP list 10
```

### 35.5 Set操作

```redis
# 集合操作
SADD tags "redis" "database"
SREM tags "database"
SMEMBERS tags
SISMEMBER tags "redis"

# 集合运算
SINTER set1 set2         # 交集
SUNION set1 set2         # 并集  
SDIFF set1 set2          # 差集

# 随机元素
SRANDMEMBER tags 2       # 随机2个不重复
SPOP tags 1              # 随机弹出1个
```

### 35.6 ZSet操作

```redis
# 添加查询
ZADD leaderboard 100 "player1" 200 "player2"
ZRANGE leaderboard 0 9 WITHSCORES    # 升序前10
ZREVRANGE leaderboard 0 9 WITHSCORES # 降序前10

# 分数操作
ZINCRBY leaderboard 50 "player1"
ZSCORE leaderboard "player1"

# 范围查询
ZRANGEBYSCORE leaderboard 100 200
ZCOUNT leaderboard 100 200

# 排名查询
ZRANK leaderboard "player1"     # 升序排名
ZREVRANK leaderboard "player1"  # 降序排名
```

### 35.7 事务操作

```redis
# 基本事务
MULTI
SET key1 value1
SET key2 value2
EXEC

# 取消事务
MULTI
SET key1 value1
DISCARD

# 监视键
WATCH key1
MULTI
SET key1 new_value
EXEC                    # 如果key1被修改，事务失败
```

### 35.8 发布订阅

```redis
# 订阅频道
SUBSCRIBE news sports

# 发布消息
PUBLISH news "Breaking news!"

# 模式订阅
PSUBSCRIBE news.*
```

### 35.9 连接管理

```redis
# 连接信息
CLIENT LIST              # 查看客户端连接
CLIENT KILL ip:port      # 断开指定连接

# 服务器信息
INFO memory              # 内存信息
INFO clients             # 客户端信息
INFO stats               # 统计信息

# 配置管理
CONFIG GET maxmemory     # 获取配置
CONFIG SET timeout 300   # 设置配置
```

### 35.10 持久化相关

```redis
# 手动持久化
SAVE                     # 同步保存（阻塞）
BGSAVE                   # 后台保存

# AOF操作
BGREWRITEAOF             # 重写AOF文件

# 最后保存时间
LASTSAVE                 # 获取最后一次保存时间戳
```
