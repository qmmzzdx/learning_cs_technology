# MySQL 相关知识总结

## 1. SQL 基础

### 1.1 NOSQL 和 SQL 的区别

**SQL 数据库（关系型数据库）**：
- 主要代表：SQL Server、Oracle、MySQL、PostgreSQL
- 存储结构化数据，以行列二维表形式存在
- 每一列代表数据的一种属性，每一行代表一个数据实体
- 支持 ACID 特性

**NoSQL 数据库（非关系型数据库）**：
- 主要代表：MongoDB、Redis
- 存储方式可以是 JSON 文档、哈希表等
- 采用 BASE 模型（基本可用、软状态、最终一致性）

**选择考虑因素**：
- **ACID vs BASE**：银行等需要强一致性用 SQL，社交软件等可用 NoSQL
- **扩展性**：NoSQL 更易水平扩展
- **数据结构**：结构化数据用 SQL，非结构化用 NoSQL

### 1.2 数据库三大范式

**第一范式（1NF）**：
- 要求数据库表的每一列都是不可分割的原子数据项
- 示例：将"家庭信息"拆分为具体字段

**第二范式（2NF）**：
- 在 1NF 基础上，非码属性必须完全依赖于候选码
- 消除非主属性对主码的部分函数依赖
- 示例：订单表拆分为订单主表和订单明细表

**第三范式（3NF）**：
- 在 2NF 基础上，任何非主属性不依赖于其它非主属性
- 消除传递依赖
- 示例：学生表拆分为学生表和班主任表

### 1.3 MySQL 连表查询

**内连接（INNER JOIN）**：
```sql
SELECT employees.name, departments.name
FROM employees
INNER JOIN departments ON employees.department_id = departments.id;
```

**左外连接（LEFT JOIN）**：
```sql
SELECT employees.name, departments.name
FROM employees
LEFT JOIN departments ON employees.department_id = departments.id;
```

**右外连接（RIGHT JOIN）**：
```sql
SELECT employees.name, departments.name
FROM employees
RIGHT JOIN departments ON employees.department_id = departments.id;
```

**全外连接（FULL JOIN）**：
```sql
-- MySQL 中使用 UNION 实现
SELECT employees.name, departments.name
FROM employees
LEFT JOIN departments ON employees.department_id = departments.id

UNION

SELECT employees.name, departments.name
FROM employees
RIGHT JOIN departments ON employees.department_id = departments.id;
```

### 1.4 MySQL 避免重复插入数据

**方式一：使用 UNIQUE 约束**
```sql
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE,
    name VARCHAR(255)
);
```

**方式二：使用 INSERT ... ON DUPLICATE KEY UPDATE**
```sql
INSERT INTO users (email, name) 
VALUES ('example@example.com', 'John Doe')
ON DUPLICATE KEY UPDATE name = VALUES(name);
```

**方式三：使用 INSERT IGNORE**
```sql
INSERT IGNORE INTO users (email, name) 
VALUES ('example@example.com', 'John Doe');
```

### 1.5 CHAR 和 VARCHAR 区别

| 特性 | CHAR | VARCHAR |
|------|------|---------|
| 长度 | 固定长度 | 可变长度 |
| 存储 | 末尾补足空格 | 按实际长度存储 |
| 适用场景 | 长度固定的数据（代码、状态） | 长度可变的数据（文本、备注） |
| 效率 | 短字符串效率高 | 节约存储空间 |

### 1.6 数据类型相关问题

**VARCHAR 长度代表字符数**：
- VARCHAR(10) 表示最多存储 10 个字符
- 字节长度取决于字符集（ASCII 每个字符 1 字节，UTF-8 每个字符 1-4 字节）

**INT(1) 和 INT(10) 的区别**：
- 只是显示宽度的区别，不影响存储范围
- 所有 INT 类型都占用 4 字节存储空间
- 主要在使用 ZEROFILL 时体现差异

**Text 数据类型大小**：
- TEXT：65,535 bytes ≈ 64KB
- MEDIUMTEXT：16,777,215 bytes ≈ 16MB
- LONGTEXT：4,294,967,295 bytes ≈ 4GB

**IP 地址存储方式**：
```sql
-- 字符串存储
CREATE TABLE ip_records (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ip_address VARCHAR(15)
);

-- 整数存储
CREATE TABLE ip_records (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ip_address INT UNSIGNED
);
INSERT INTO ip_records (ip_address) VALUES (INET_ATON('192.168.1.1'));
SELECT INET_NTOA(ip_address) FROM ip_records;
```

### 1.7 外键约束

**作用**：维护表与表之间的关系，确保数据完整性和一致性

**示例**：
```sql
CREATE TABLE students (
  id INT PRIMARY KEY,
  name VARCHAR(50),
  course_id INT,
  FOREIGN KEY (course_id) REFERENCES courses(id)
);
```

### 1.8 IN 和 EXISTS

**IN 关键字**：
```sql
SELECT * FROM Customers WHERE Country IN ('Germany', 'France');
SELECT * FROM Customers WHERE Country IN (SELECT Country FROM Suppliers);
```

**EXISTS 关键字**：
```sql
SELECT * FROM Customers 
WHERE EXISTS (SELECT 1 FROM Orders WHERE Orders.CustomerID = Customers.CustomerID);
```

**区别**：
- **性能**：EXISTS 通常优于 IN，特别是子查询表很大时
- **使用场景**：IN 适合子查询结果集小，EXISTS 适合关联查询
- **NULL 值处理**：IN 能处理 NULL，EXISTS 不受 NULL 影响

### 1.9 MySQL 基本函数

**字符串函数**：
```sql
SELECT CONCAT('Hello', ' ', 'World');
SELECT LENGTH('Hello');
SELECT SUBSTRING('Hello World', 1, 5);
SELECT REPLACE('Hello World', 'World', 'MySQL');
```

**数值函数**：
```sql
SELECT ABS(-10);
SELECT POWER(2, 3);
```

**日期函数**：
```sql
SELECT NOW();
SELECT CURDATE();
```

**聚合函数**：
```sql
SELECT COUNT(*), SUM(price), AVG(price), MAX(price), MIN(price) FROM orders;
```

### 1.10 SQL 查询语句执行顺序

```sql
(1) FROM <left_table>
    (3) <join_type> JOIN <right_table>
    (2) ON <join_condition>
(4) WHERE <where_condition>
(5) GROUP BY <group_by_list>
(6) AGG_FUNC <column>
(7) WITH {CUBE|ROLLUP}
(8) HAVING <having_condition>
(9) SELECT 
(10) DISTINCT <column>
(11) ORDER BY <order_by_list>
(12) LIMIT <limit_number>
```

## 2. 存储引擎

### 2.1 常见存储引擎比较

| 特性 | InnoDB | MyISAM | Memory |
|------|--------|--------|--------|
| 事务支持 | ✅ | ❌ | ❌ |
| 行级锁 | ✅ | ❌ | ❌ |
| 外键约束 | ✅ | ❌ | ❌ |
| 崩溃恢复 | ✅ | ❌ | ❌ |
| 存储限制 | 64TB | 256TB | 内存大小 |
| 适用场景 | 高并发读写 | 大量读操作 | 高性能读操作 |

### 2.2 InnoDB 成为默认引擎的原因

1. **事务支持**：支持 ACID 事务
2. **并发性能**：行级锁定提供更好的并发
3. **崩溃恢复**：通过 redo log 实现崩溃恢复
4. **数据完整性**：支持外键约束

### 2.3 InnoDB vs MyISAM 详细区别

**事务**：
- InnoDB：支持事务，可回滚
- MyISAM：不支持事务

**索引结构**：
- InnoDB：聚簇索引，数据文件本身就是索引文件
- MyISAM：非聚簇索引，索引和数据文件分离

**锁粒度**：
- InnoDB：行级锁，支持更细粒度的并发控制
- MyISAM：表级锁，并发访问受限

**COUNT 效率**：
- InnoDB：不保存具体行数，需要全表扫描
- MyISAM：维护行数计数器，查询速度快

**崩溃恢复**：
- InnoDB：通过 redo log 支持崩溃恢复
- MyISAM：不支持崩溃恢复

### 2.4 数据文件类型

**数据库目录结构**：
```
/var/lib/mysql/my_test/
├── db.opt          # 数据库字符集配置
├── t_order.frm     # 表结构定义
└── t_order.ibd     # 表数据文件（独占表空间）
```

**文件说明**：
- **.frm**：存储表结构定义
- **.ibd**：表数据文件（innodb_file_per_table=1 时）
- **ibdata1**：共享表空间文件
- **db.opt**：数据库默认字符集和校验规则

## 3. 索引

### 3.1 索引分类详解

**按数据结构分类**：
- B+tree 索引：最常用，支持范围查询
- Hash 索引：精确匹配快，不支持范围查询
- Full-text 索引：全文搜索

**按物理存储分类**：
- **聚簇索引**：叶子节点包含实际数据行
- **二级索引**：叶子节点包含主键值，需要回表

**按字段特性分类**：
- **主键索引**：建立在主键字段上，不允许空值
- **唯一索引**：建立在 UNIQUE 字段上，允许空值
- **普通索引**：建立在普通字段上
- **前缀索引**：对字符类型字段前几个字符建立索引

**按字段个数分类**：
- **单列索引**：单个字段上的索引
- **联合索引**：多个字段组合的索引

### 3.2 聚簇索引 vs 非聚簇索引

| 特性 | 聚簇索引 | 非聚簇索引 |
|------|---------|-----------|
| 数据存储 | 索引叶子节点包含实际数据 | 叶子节点包含指针或主键值 |
| 索引数量 | 每个表只能有一个 | 可以有多个 |
| 查询效率 | 直接获取数据，无需回表 | 需要回表查询 |
| 范围查询 | 效率高 | 效率相对较低 |

### 3.3 B+树详细特性

**数据结构特点**：
- 多叉树结构，叶子节点在同一层
- 非叶子节点只存储索引信息（键值和指针）
- 叶子节点存储实际数据或主键值
- 叶子节点形成双向链表，支持顺序访问

**搜索复杂度**：O(logdN)，其中 d 为节点最大子节点数

**优势**：
- 高度低：千万级数据只需 3-4 层
- 磁盘 I/O 少：查询只需 3-4 次磁盘 I/O
- 范围查询高效：叶子节点链表支持顺序扫描

### 3.4 B+树 vs B树 vs 二叉树

**B+树 vs B树**：
- B+树非叶子节点不存数据，B树存
- B+树叶子节点链表连接，B树无
- B+树查询性能更稳定

**B+树 vs 二叉树**：
- B+树更矮胖，磁盘 I/O 更少
- 二叉树可能退化为链表，性能不稳定

**B+树 vs Hash**：
- B+树支持范围查询，Hash 只支持等值查询

### 3.5 联合索引与最左匹配原则

**创建联合索引**：
```sql
CREATE INDEX idx_product_no_name ON product(product_no, name);
```

**最左匹配原则**：
- 查询必须从索引的最左列开始
- 不能跳过索引中的列
- 范围查询后面的列无法使用索引

**示例**：
```sql
-- 能使用索引的情况
WHERE a = 1
WHERE a = 1 AND b = 2
WHERE a = 1 AND b = 2 AND c = 3

-- 不能使用索引的情况
WHERE b = 2
WHERE c = 3
WHERE b = 2 AND c = 3
```

### 3.6 索引失效的 6 种情况

1. **模糊匹配**：LIKE '%xx' 或 LIKE '%xx%'
2. **使用函数**：对索引列使用函数
3. **表达式计算**：对索引列进行表达式计算
4. **隐式类型转换**：字符串与数字比较
5. **违反最左匹配**：联合索引使用不当
6. **OR 条件不当**：OR 前后条件列索引情况不同

### 3.7 索引优化策略

**前缀索引优化**：
```sql
CREATE INDEX idx_name ON table_name(column_name(10));
```

**覆盖索引优化**：
```sql
-- 查询字段都在索引中，无需回表
SELECT id, name FROM users WHERE name = 'John';
```

**自增主键优势**：
- 插入性能高，避免页分裂
- 存储紧凑，减少碎片

**索引创建原则**：
- 区分度高的字段适合建索引
- 频繁查询的字段适合建索引
- 避免在更新频繁的字段上建索引

## 4. 事务

### 4.1 事务特性（ACID）及实现

**原子性（Atomicity）**：
- 定义：事务的所有操作要么全部完成，要么全部不完成
- 实现：通过 undo log 实现回滚

**一致性（Consistency）**：
- 定义：事务操作前后数据满足完整性约束
- 实现：通过持久性+原子性+隔离性保证

**隔离性（Isolation）**：
- 定义：多个事务并发执行时相互隔离
- 实现：通过 MVCC 或锁机制实现

**持久性（Durability）**：
- 定义：事务提交后修改永久保存
- 实现：通过 redo log 保证

### 4.2 并发问题

**脏读**：
- 一个事务读到另一个未提交事务修改的数据
- 发生在读未提交隔离级别

**不可重复读**：
- 同一事务内多次读取同一数据结果不一致
- 发生在读未提交、读提交隔离级别

**幻读**：
- 同一事务内多次查询记录数量不一致
- 发生在读未提交、读提交、可重复读隔离级别

### 4.3 隔离级别详解

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 |
|---------|------|-----------|------|----------|
| 读未提交 | ✅ | ✅ | ✅ | 直接读最新数据 |
| 读提交 | ❌ | ✅ | ✅ | 每个语句前生成 Read View |
| 可重复读 | ❌ | ❌ | ✅ | 事务开始时生成 Read View |
| 串行化 | ❌ | ❌ | ❌ | 加锁 |

**MySQL 默认隔离级别**：可重复读

### 4.4 MVCC 实现原理

**核心组件**：
- **隐藏列**：trx_id（事务ID）、roll_pointer（回滚指针）
- **Undo Log**：存储历史版本数据
- **Read View**：事务可见性判断

**Read View 字段**：
- m_ids：活跃事务ID列表
- min_trx_id：最小活跃事务ID
- max_trx_id：下一个事务ID
- creator_trx_id：创建者事务ID

**可见性判断规则**：
1. trx_id < min_trx_id：可见（事务已提交）
2. trx_id >= max_trx_id：不可见（事务后启动）
3. min_trx_id <= trx_id < max_trx_id：
   - 在 m_ids 中：不可见（事务未提交）
   - 不在 m_ids 中：可见（事务已提交）

## 5. 锁

### 5.1 锁类型详解

**全局锁**：
```sql
FLUSH TABLES WITH READ LOCK;  -- 加锁
UNLOCK TABLES;                -- 释放锁
```

**表级锁**：
- **表锁**：LOCK TABLES table_name READ/WRITE
- **元数据锁（MDL）**：自动加锁，防止表结构变更
- **意向锁**：表明事务打算在表中的行上加什么类型的锁

**行级锁**：
- **记录锁（Record Lock）**：锁定单条记录
- **间隙锁（Gap Lock）**：锁定记录之间的间隙
- **临键锁（Next-Key Lock）**：记录锁+间隙锁

### 5.2 锁应用场景

**表锁适用场景**：
- 全表数据迁移
- 表结构变更
- 大批量数据操作

**行锁适用场景**：
- 高并发单行操作
- 需要精细并发控制的场景

**间隙锁作用**：
- 防止幻读
- 在可重复读隔离级别下使用

## 6. 日志系统

### 6.1 日志类型及作用

**redo log（重做日志）**：
- 作用：保证事务持久性，崩溃恢复
- 特点：物理日志，顺序写入
- 存储：记录数据页的物理修改

**undo log（回滚日志）**：
- 作用：保证事务原子性，实现回滚和 MVCC
- 特点：逻辑日志
- 存储：记录更新前的数据状态

**binlog（二进制日志）**：
- 作用：数据备份和主从复制
- 特点：逻辑日志，追加写入
- 格式：STATEMENT、ROW、MIXED

**relay log（中继日志）**：
- 作用：主从复制中间存储
- 特点：从库接收的 binlog 暂存

### 6.2 两阶段提交

**过程**：
1. **prepare 阶段**：
   - 写入 redo log，状态为 prepare
   - 持久化 redo log 到磁盘

2. **commit 阶段**：
   - 写入 binlog
   - 持久化 binlog 到磁盘
   - 提交事务，redo log 状态改为 commit

**崩溃恢复**：
- binlog 有 XID：提交事务
- binlog 无 XID：回滚事务

### 6.3 Double Write Buffer

**作用**：防止数据页损坏

**过程**：
1. 数据页先拷贝到 Doublewrite Buffer
2. Doublewrite Buffer 刷到磁盘共享表空间
3. 数据页刷到实际表空间

**优势**：
- 数据页损坏时可以从 Doublewrite Buffer 恢复
- 保证数据页写入的原子性

## 7. 性能调优

### 7.1 EXPLAIN 详细分析

**关键字段解读**：

**type**（访问类型，性能从好到坏）：
- const：主键或唯一索引等值查询
- eq_ref：唯一索引关联查询
- ref：非唯一索引查询
- range：索引范围扫描
- index：全索引扫描
- ALL：全表扫描

**Extra**（额外信息）：
- Using index：覆盖索引
- Using where：WHERE 条件过滤
- Using temporary：使用临时表
- Using filesort：文件排序
- Using join buffer：使用连接缓冲

### 7.2 查询优化实战

**索引优化**：
```sql
-- 创建合适索引
CREATE INDEX idx_name_age ON users(name, age);

-- 使用覆盖索引
SELECT id, name FROM users WHERE name = 'John';

-- 避免索引失效
SELECT * FROM users WHERE DATE(create_time) = '2023-01-01';  -- 错误
SELECT * FROM users WHERE create_time >= '2023-01-01' AND create_time < '2023-01-02';  -- 正确
```

**分页优化**：
```sql
-- 传统分页（性能差）
SELECT * FROM table ORDER BY id LIMIT 10000, 10;

-- 优化分页
SELECT * FROM table WHERE id > 10000 ORDER BY id LIMIT 10;
```

**联表优化**：
```sql
-- 小表驱动大表
SELECT * FROM small_table s 
JOIN large_table l ON s.id = l.small_id;

-- 确保被驱动表有索引
CREATE INDEX idx_small_id ON large_table(small_id);
```

### 7.3 慢查询解决方案

1. **分析执行计划**：使用 EXPLAIN
2. **优化索引**：创建缺失索引，删除冗余索引
3. **重写SQL**：避免复杂子查询，使用 JOIN
4. **数据库设计**：适当反范式化
5. **架构优化**：读写分离，分库分表
6. **缓存策略**：使用 Redis 缓存热点数据

## 8. 架构设计

### 8.1 主从复制原理

**复制过程**：
1. **主库**：
   - 写入 binlog
   - 提交事务
   - log dump 线程发送 binlog

2. **从库**：
   - I/O 线程接收 binlog 写入 relay log
   - SQL 线程回放 relay log
   - 更新存储引擎数据

**复制模式**：
- **异步复制**：默认模式，性能好，可能丢失数据
- **半同步复制**：至少一个从库确认收到日志
- **全同步复制**：所有从库确认收到日志

### 8.2 主从延迟处理

**延迟原因**：
- 从库性能较差
- 网络延迟
- 大事务执行
- 从库过多

**解决方案**：
- **强制走主库**：关键业务读主库
- **并行复制**：启用多线程复制
- **缓存层**：读取缓存减少数据库压力
- **业务拆分**：延迟不敏感业务走从库

### 8.3 分库分表策略

**垂直分库**：
- 按业务模块拆分
- 示例：订单库、用户库、商品库

**垂直分表**：
- 大表拆小表
- 示例：用户基础表、用户扩展表

**水平分库**：
- 同一表数据分布到不同数据库
- 示例：用户表按 user_id 取模分库

**水平分表**：
- 同一数据库内拆分表
- 示例：订单表按月份分表

**分片策略**：
- 范围分片：按时间、ID 范围
- 哈希分片：按字段哈希值
- 地理位置分片：按地区

## 9. 高级特性

### 9.1 可重入锁实现

**数据库表设计**：
```sql
CREATE TABLE `lock_table` (
    `id` INT AUTO_INCREMENT PRIMARY KEY,
    `lock_name` VARCHAR(255) NOT NULL,
    `holder_thread` VARCHAR(255),
    `reentry_count` INT DEFAULT 0
);
```

**加锁逻辑**：
1. 开启事务
2. 查询锁记录
3. 记录不存在：插入新记录
4. 记录存在且同一线程：增加重入次数
5. 提交事务

**解锁逻辑**：
1. 开启事务
2. 查询锁记录
3. 重入次数 > 1：减少次数
4. 重入次数 = 1：删除记录
5. 提交事务

### 9.2 UPDATE 语句执行过程

1. **执行器**：调用存储引擎接口获取数据
2. **Buffer Pool**：检查数据页是否在内存中
3. **Undo Log**：记录更新前的数据状态
4. **更新内存**：修改 Buffer Pool 中的数据页
5. **Redo Log**：记录数据页的物理修改
6. **Binlog**：记录逻辑操作
7. **两阶段提交**：保证 redo log 和 binlog 一致性
8. **刷脏页**：后台线程将脏页刷入磁盘

### 9.3 数据不丢失保障机制

**WAL（Write-Ahead Logging）技术**：
- 先写日志，后写数据
- 保证崩溃恢复能力

**Redo Log 作用**：
- 顺序写入，性能高
- 崩溃时重放日志恢复数据
- 避免随机写的数据页损坏风险

**Double Write Buffer**：
- 防止数据页部分写入损坏
- 提供数据页备份恢复

## 10. 实战问题解决方案

### 10.1 经典 SQL 题目

**题目1：查询不存在01课程但存在02课程的学生成绩**
```sql
-- 方法1：使用 LEFT JOIN
SELECT s.sid, s.sname, sc2.cid, sc2.score
FROM Student s
LEFT JOIN Score AS sc1 ON s.sid = sc1.sid AND sc1.cid = '01'
LEFT JOIN Score AS sc2 ON s.sid = sc2.sid AND sc2.cid = '02'
WHERE sc1.cid IS NULL AND sc2.cid IS NOT NULL;

-- 方法2：使用 NOT EXISTS
SELECT s.sid, s.sname, sc.cid, sc.score
FROM Student s
JOIN Score sc ON s.sid = sc.sid AND sc.cid = '02'
WHERE NOT EXISTS (
    SELECT 1 FROM Score sc1 WHERE sc1.sid = s.sid AND sc1.cid = '01'
);
```

**题目2：查询总分排名5-10名的学生**
```sql
WITH StudentTotalScores AS (
    SELECT 
        stu_id,
        SUM(score) AS total_score
    FROM 
        student_score
    GROUP BY 
        stu_id
),
RankedStudents AS (
    SELECT
        stu_id,
        total_score,
        RANK() OVER (ORDER BY total_score DESC) AS ranking
    FROM
        StudentTotalScores
)
SELECT
    stu_id,
    total_score
FROM
    RankedStudents
WHERE
    ranking BETWEEN 5 AND 10;
```

**题目3：查询班级下所有学生选课情况**
```sql
SELECT 
    s.student_id,
    s.student_name,
    cs.course_name
FROM 
    students s
JOIN 
    course_selections cs ON s.student_id = cs.student_id
JOIN 
    classes c ON s.class_id = c.class_id
WHERE 
    c.class_name = 'Class A';
```

### 10.2 性能优化案例

**案例1：深分页优化**
```sql
-- 优化前（性能差）
SELECT * FROM orders ORDER BY id LIMIT 100000, 20;

-- 优化后（性能好）
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 20;
```

**案例2：大数据量统计优化**
```sql
-- 优化前（全表扫描）
SELECT COUNT(*) FROM users WHERE status = 1;

-- 优化后（使用索引）
CREATE INDEX idx_status ON users(status);
SELECT COUNT(*) FROM users WHERE status = 1;
```

**案例3：联表查询优化**
```sql
-- 优化前（无索引）
SELECT * FROM orders o 
JOIN users u ON o.user_id = u.id 
WHERE u.name = 'John';

-- 优化后（确保驱动表小，被驱动表有索引）
CREATE INDEX idx_user_id ON orders(user_id);
CREATE INDEX idx_name ON users(name);
```
