# 系统设计 相关知识总结

## 1. 秒杀系统设计

### 核心挑战
- **高并发处理**：应对瞬时流量峰值
- **库存一致性**：防止超卖问题
- **系统稳定性**：保证服务可用性
- **公平性保障**：防止机器人和恶意请求

### 分层解决方案

#### 前端层
```go
// Go 示例：前端限流和验证
type SeckillRequest struct {
    UserID    int64  `json:"user_id"`
    ProductID int64  `json:"product_id"`
    Timestamp int64  `json:"timestamp"`
    Token     string `json:"token"` // 防刷令牌
}

func (s *SeckillService) PreCheck(req *SeckillRequest) error {
    // 1. 活动时间校验
    if !s.isActivityActive() {
        return errors.New("活动未开始或已结束")
    }
    
    // 2. 用户频率限制
    if s.isUserRateLimited(req.UserID) {
        return errors.New("请求过于频繁")
    }
    
    // 3. 令牌验证
    if !s.validateToken(req.Token) {
        return errors.New("非法请求")
    }
    
    return nil
}
```

#### 网关层
```python
# Python 示例：网关层限流
from redis import Redis
from functools import wraps

class GatewayLimiter:
    def __init__(self, redis_client: Redis):
        self.redis = redis_client
    
    def ip_limit(self, max_requests: int = 100, window: int = 60):
        def decorator(func):
            @wraps(func)
            def wrapper(ip_address, *args, **kwargs):
                key = f"rate_limit:ip:{ip_address}"
                current = self.redis.get(key)
                
                if current and int(current) >= max_requests:
                    raise Exception("IP请求频率超限")
                
                pipeline = self.redis.pipeline()
                pipeline.incr(key, 1)
                pipeline.expire(key, window)
                pipeline.execute()
                
                return func(ip_address, *args, **kwargs)
            return wrapper
        return decorator
```

#### 缓存层 - Redis库存管理
```go
// Go 示例：Redis原子操作扣减库存
type InventoryManager struct {
    redisClient *redis.Client
}

func (im *InventoryManager) DeductStock(productID int64, quantity int) (bool, error) {
    // 使用Lua脚本保证原子性
    script := `
        local key = KEYS[1]
        local deduct = tonumber(ARGV[1])
        local current = tonumber(redis.call('GET', key) or '0')
        
        if current >= deduct then
            redis.call('DECRBY', key, deduct)
            return 1
        else
            return 0
        end
    `
    
    result, err := im.redisClient.Eval(script, 
        []string{fmt.Sprintf("stock:%d", productID)}, quantity).Result()
    
    if err != nil {
        return false, err
    }
    
    return result.(int64) == 1, nil
}
```

#### 消息队列异步处理
```python
# Python 示例：消息队列削峰
import asyncio
from kafka import KafkaProducer
import json

class SeckillOrderProcessor:
    def __init__(self):
        self.producer = KafkaProducer(
            bootstrap_servers=['localhost:9092'],
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )
    
    async def process_seckill_request(self, user_id: int, product_id: int):
        # 1. Redis预扣库存
        stock_key = f"stock:{product_id}"
        if not await self.redis_client.decr(stock_key):
            return {"success": False, "message": "库存不足"}
        
        # 2. 发送消息到队列
        message = {
            "user_id": user_id,
            "product_id": product_id,
            "timestamp": int(time.time()),
            "request_id": self.generate_request_id()
        }
        
        self.producer.send('seckill_orders', message)
        return {"success": True, "message": "抢购成功，处理中"}
    
    async def consume_orders(self):
        # 后台消费者处理订单
        consumer = KafkaConsumer('seckill_orders')
        for message in consumer:
            order_data = json.loads(message.value)
            await self.create_order(order_data)
```

## 2. 订单超时取消方案

### 多种技术方案对比

#### 方案一：Redis有序集合 + 定时任务
```go
// Go 示例：Redis实现订单超时
type OrderTimeoutManager struct {
    redisClient *redis.Client
}

func (otm *OrderTimeoutManager) AddTimeoutOrder(orderID string, timeout int64) error {
    // 将订单ID和超时时间添加到有序集合
    return otm.redisClient.ZAdd("timeout_orders", redis.Z{
        Score:  float64(time.Now().Unix() + timeout),
        Member: orderID,
    }).Err()
}

func (otm *OrderTimeoutManager) CheckTimeoutOrders() {
    // 定时扫描超时订单
    max := strconv.FormatInt(time.Now().Unix(), 10)
    
    results, err := otm.redisClient.ZRangeByScore("timeout_orders", 
        redis.ZRangeBy{Min: "0", Max: max}).Result()
    
    if err != nil {
        return
    }
    
    for _, orderID := range results {
        otm.processTimeoutOrder(orderID)
    }
}

func (otm *OrderTimeoutManager) processTimeoutOrder(orderID string) {
    // 处理超时订单：取消订单、恢复库存等
    // 使用事务保证操作原子性
    tx := otm.redisClient.TxPipeline()
    tx.ZRem("timeout_orders", orderID)
    tx.Set(fmt.Sprintf("order:status:%s", orderID), "cancelled", 0)
    tx.Exec()
}
```

#### 方案二：消息队列延迟队列
```python
# Python 示例：RabbitMQ延迟队列
import pika
import json

class OrderTimeoutService:
    def __init__(self):
        self.connection = pika.BlockingConnection(
            pika.ConnectionParameters('localhost')
        )
        self.channel = self.connection.channel()
        
        # 声明延迟交换器
        self.channel.exchange_declare(
            exchange='delayed_exchange',
            exchange_type='x-delayed-message',
            arguments={'x-delayed-type': 'direct'}
        )
    
    def schedule_order_timeout(self, order_id: str, delay_ms: int):
        message = json.dumps({
            'order_id': order_id,
            'action': 'cancel'
        })
        
        self.channel.basic_publish(
            exchange='delayed_exchange',
            routing_key='order_timeout',
            body=message,
            properties=pika.BasicProperties(
                headers={'x-delay': delay_ms}
            )
        )
    
    def start_consumer(self):
        def callback(ch, method, properties, body):
            data = json.loads(body)
            self.handle_timeout_order(data['order_id'])
            ch.basic_ack(delivery_tag=method.delivery_tag)
        
        self.channel.basic_consume(
            queue='order_timeout_queue',
            on_message_callback=callback
        )
        self.channel.start_consuming()
```

## 3. Redis扩展方案

### 读写分离架构
```go
// Go 示例：Redis读写分离配置
type RedisCluster struct {
    Master *redis.Client   // 写节点
    Slaves []*redis.Client // 读节点池
    currentSlave int
    mutex sync.Mutex
}

func (rc *RedisCluster) GetSlave() *redis.Client {
    rc.mutex.Lock()
    defer rc.mutex.Unlock()
    
    slave := rc.Slaves[rc.currentSlave]
    rc.currentSlave = (rc.currentSlave + 1) % len(rc.Slaves)
    return slave
}

func (rc *RedisCluster) Write(key string, value interface{}) error {
    return rc.Master.Set(key, value, 0).Err()
}

func (rc *RedisCluster) Read(key string) (string, error) {
    return rc.GetSlave().Get(key).Result()
}
```

### Redis Cluster数据分片
```python
# Python 示例：Redis Cluster客户端
from rediscluster import RedisCluster

class DistributedRedis:
    def __init__(self, startup_nodes):
        self.cluster = RedisCluster(
            startup_nodes=startup_nodes,
            decode_responses=True,
            skip_full_coverage_check=True
        )
    
    def get_slot_key(self, key):
        """根据key计算槽位"""
        return self.cluster.keyslot(key)
    
    def set_data(self, key, value):
        """自动路由到正确的节点"""
        return self.cluster.set(key, value)
    
    def get_data(self, key):
        """自动从正确的节点读取"""
        return self.cluster.get(key)
```

## 4. 分布式可重入锁设计

### Redis实现方案
```go
// Go 示例：可重入分布式锁
type ReentrantLock struct {
    redisClient *redis.Client
    lockKey     string
    ownerID     string
    expiration  time.Duration
}

func NewReentrantLock(redisClient *redis.Client, lockKey string, ownerID string) *ReentrantLock {
    return &ReentrantLock{
        redisClient: redisClient,
        lockKey:     lockKey,
        ownerID:     ownerID,
        expiration:  30 * time.Second,
    }
}

func (rl *ReentrantLock) Lock() (bool, error) {
    script := `
        local key = KEYS[1]
        local owner = ARGV[1]
        local expiration = ARGV[2]
        
        local current = redis.call('HGETALL', key)
        if next(current) == nil then
            -- 锁不存在，直接获取
            redis.call('HMSET', key, 'owner', owner, 'count', 1)
            redis.call('PEXPIRE', key, expiration)
            return 1
        end
        
        local currentOwner = redis.call('HGET', key, 'owner')
        if currentOwner == owner then
            -- 重入：增加计数
            local count = tonumber(redis.call('HGET', key, 'count')) + 1
            redis.call('HSET', key, 'count', count)
            redis.call('PEXPIRE', key, expiration)
            return 1
        else
            return 0
        end
    `
    
    result, err := rl.redisClient.Eval(script, 
        []string{rl.lockKey}, 
        rl.ownerID, 
        rl.expiration.Milliseconds()).Result()
    
    if err != nil {
        return false, err
    }
    
    return result.(int64) == 1, nil
}

func (rl *ReentrantLock) Unlock() error {
    script := `
        local key = KEYS[1]
        local owner = ARGV[1]
        
        local currentOwner = redis.call('HGET', key, 'owner')
        if currentOwner ~= owner then
            return 0
        end
        
        local count = tonumber(redis.call('HGET', key, 'count')) - 1
        if count <= 0 then
            redis.call('DEL', key)
            return 1
        else
            redis.call('HSET', key, 'count', count)
            return 1
        end
    `
    
    _, err := rl.redisClient.Eval(script, 
        []string{rl.lockKey}, rl.ownerID).Result()
    
    return err
}

func (rl *ReentrantLock) TryLock(timeout time.Duration) (bool, error) {
    deadline := time.Now().Add(timeout)
    
    for time.Now().Before(deadline) {
        acquired, err := rl.Lock()
        if err != nil {
            return false, err
        }
        if acquired {
            return true, nil
        }
        time.Sleep(100 * time.Millisecond)
    }
    
    return false, nil
}
```

## 5. 负载均衡策略

### 软件负载均衡 - Nginx配置
```nginx
# Nginx负载均衡配置示例
upstream backend_servers {
    # 1. 轮询策略 (默认)
    server 192.168.1.10:8080;
    server 192.168.1.11:8080;
    
    # 2. 权重策略
    server 192.168.1.12:8080 weight=3;
    server 192.168.1.13:8080 weight=1;
    
    # 3. IP哈希策略
    ip_hash;
    
    # 4. 最少连接策略
    least_conn;
}

server {
    listen 80;
    server_name example.com;
    
    location / {
        proxy_pass http://backend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # 健康检查
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503;
    }
    
    # 静态资源缓存
    location ~* \.(js|css|png|jpg|jpeg|gif|ico)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### 应用层负载均衡 - Go实现
```go
// Go 示例：简单的负载均衡器
type LoadBalancer struct {
    servers []*Server
    current int
    mutex   sync.Mutex
}

type Server struct {
    URL    string
    Weight int
    Alive  bool
}

func (lb *LoadBalancer) NextServer() *Server {
    lb.mutex.Lock()
    defer lb.mutex.Unlock()
    
    // 轮询算法
    server := lb.servers[lb.current]
    lb.current = (lb.current + 1) % len(lb.servers)
    
    // 健康检查
    if !server.Alive {
        return lb.NextServer()
    }
    
    return server
}

func (lb *LoadBalancer) HealthCheck() {
    for _, server := range lb.servers {
        go func(s *Server) {
            resp, err := http.Get(s.URL + "/health")
            if err != nil || resp.StatusCode != 200 {
                s.Alive = false
            } else {
                s.Alive = true
            }
        }(server)
    }
}
```

## 关键设计原则总结

### 1. 分层设计原则
- **前端层**：静态化、限流、验证码
- **网关层**：鉴权、限流、缓存
- **服务层**：异步化、队列化、无状态
- **数据层**：分库分表、读写分离、缓存策略

### 2. 一致性保障
- **最终一致性**：通过消息队列和补偿机制
- **数据同步**：定期核对、异常告警、自动修复
- **事务管理**：分布式事务、补偿事务

### 3. 性能优化
- **缓存策略**：多级缓存、缓存预热、缓存击穿防护
- **数据库优化**：索引优化、查询优化、连接池管理
- **异步处理**：非阻塞IO、事件驱动、批量处理

### 4. 容错与降级
- **服务降级**：核心功能优先、非核心功能可降级
- **熔断机制**：故障隔离、自动恢复
- **监控告警**：实时监控、智能告警、快速响应
