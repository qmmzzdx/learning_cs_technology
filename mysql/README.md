# MySQL 相关知识总结

## 1. SQL 基础

### 1.1 NOSQL 和 SQL 的区别

**SQL 数据库（关系型数据库）**：
- 主要代表：SQL Server、Oracle、MySQL、PostgreSQL
- 存储结构化数据，以行列二维表形式存在
- 每一列代表数据的一种属性，每一行代表一个数据实体
- 支持 ACID 特性
- 使用 SQL 语言进行数据操作
- 支持复杂的联表查询和事务处理

**NoSQL 数据库（非关系型数据库）**：
- 主要代表：MongoDB、Redis、Cassandra、HBase
- 存储方式可以是 JSON 文档、哈希表、列族、图等
- 采用 BASE 模型（基本可用、软状态、最终一致性）
- 无固定表结构，灵活性高
- 适合非结构化或半结构化数据

**选择考虑因素**：
- **ACID vs BASE**：银行等需要强一致性用 SQL，社交软件等可用 NoSQL
- **扩展性**：NoSQL 更易水平扩展，SQL 垂直扩展有限
- **数据结构**：结构化数据用 SQL，非结构化用 NoSQL
- **查询复杂度**：复杂查询用 SQL，简单键值查询用 NoSQL
- **数据量**：海量数据用 NoSQL，中等数据量用 SQL

### 1.2 数据库三大范式

**第一范式（1NF）**：
- 要求数据库表的每一列都是不可分割的原子数据项
- 每个字段都是单一值，不能是数组或集合
- 示例：将"家庭信息"拆分为具体字段
- 违反示例：地址字段包含"省市区详细地址"

**第二范式（2NF）**：
- 在 1NF 基础上，非码属性必须完全依赖于候选码
- 消除非主属性对主码的部分函数依赖
- 要求表有主键，其他字段完全依赖于主键
- 示例：订单表拆分为订单主表和订单明细表

**第三范式（3NF）**：
- 在 2NF 基础上，任何非主属性不依赖于其它非主属性
- 消除传递依赖
- 要求字段之间不能有传递依赖关系
- 示例：学生表拆分为学生表和班主任表

**反范式化设计**：
- 适当违反范式以提高查询性能
- 用空间换时间，减少联表查询
- 适用于读多写少的场景

### 1.3 MySQL 连表查询

**内连接（INNER JOIN）**：
```sql
-- 基本内连接
SELECT employees.name, departments.name
FROM employees
INNER JOIN departments ON employees.department_id = departments.id;

-- 多表内连接
SELECT e.name, d.name, p.project_name
FROM employees e
INNER JOIN departments d ON e.department_id = d.id
INNER JOIN projects p ON e.project_id = p.id;
```

**左外连接（LEFT JOIN）**：
```sql
-- 左连接，包含左表所有记录
SELECT employees.name, departments.name
FROM employees
LEFT JOIN departments ON employees.department_id = departments.id;

-- 左连接带条件
SELECT e.name, COALESCE(d.name, '无部门') as department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;
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

**交叉连接（CROSS JOIN）**：
```sql
-- 笛卡尔积
SELECT e.name, d.name
FROM employees e
CROSS JOIN departments d;
```

**自连接（SELF JOIN）**：
```sql
-- 员工和经理关系
SELECT e1.name as employee, e2.name as manager
FROM employees e1
LEFT JOIN employees e2 ON e1.manager_id = e2.id;
```

### 1.4 MySQL 避免重复插入数据

**方式一：使用 UNIQUE 约束**
```sql
-- 单字段唯一约束
CREATE TABLE users (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE,
    name VARCHAR(255)
);

-- 多字段联合唯一约束
CREATE TABLE user_relationships (
    id INT PRIMARY KEY AUTO_INCREMENT,
    user_id INT,
    friend_id INT,
    UNIQUE KEY unique_relationship (user_id, friend_id)
);
```

**方式二：使用 INSERT ... ON DUPLICATE KEY UPDATE**
```sql
-- 存在则更新，不存在则插入
INSERT INTO users (email, name, login_count) 
VALUES ('example@example.com', 'John Doe', 1)
ON DUPLICATE KEY UPDATE 
    name = VALUES(name), 
    login_count = login_count + 1;

-- 基于多个唯一键
INSERT INTO user_scores (user_id, game_type, score)
VALUES (1, 'arcade', 1000)
ON DUPLICATE KEY UPDATE score = GREATEST(score, VALUES(score));
```

**方式三：使用 INSERT IGNORE**
```sql
-- 忽略重复键错误
INSERT IGNORE INTO users (email, name) 
VALUES ('example@example.com', 'John Doe');

-- 批量插入忽略重复
INSERT IGNORE INTO tags (name) VALUES 
('mysql'), ('database'), ('programming');
```

**方式四：使用 REPLACE INTO**
```sql
-- 删除后插入（注意：会删除原记录）
REPLACE INTO users (id, email, name) 
VALUES (1, 'example@example.com', 'John Doe');
```

**方式五：使用 NOT EXISTS**
```sql
-- 使用子查询避免重复
INSERT INTO users (email, name)
SELECT 'example@example.com', 'John Doe'
FROM DUAL
WHERE NOT EXISTS (
    SELECT 1 FROM users WHERE email = 'example@example.com'
);
```

### 1.5 CHAR 和 VARCHAR 区别

| 特性 | CHAR | VARCHAR |
|------|------|---------|
| 长度 | 固定长度 | 可变长度 |
| 存储 | 末尾补足空格 | 按实际长度存储+长度前缀 |
| 最大长度 | 255字符 | 65535字符 |
| 存储效率 | 短字符串效率高 | 长字符串效率高 |
| 适用场景 | 长度固定的数据（代码、状态） | 长度可变的数据（文本、备注） |
| 性能 | 定长，查询稍快 | 变长，需要计算偏移 |
| 空格处理 | 存储时补空格，检索时去掉 | 原样存储和检索 |

**使用建议**：
- 固定长度如身份证、手机号用 CHAR
- 可变长度如姓名、地址用 VARCHAR
- 非常长的文本用 TEXT 类型

### 1.6 数据类型相关问题

**VARCHAR 长度代表字符数**：
- VARCHAR(10) 表示最多存储 10 个字符
- 字节长度取决于字符集：
  - ASCII：每个字符 1 字节
  - UTF-8：每个字符 1-4 字节
  - GBK：每个字符 1-2 字节

**INT(1) 和 INT(10) 的区别**：
- 只是显示宽度的区别，不影响存储范围
- 所有 INT 类型都占用 4 字节存储空间
- 主要在使用 ZEROFILL 时体现差异
```sql
CREATE TABLE test_int (
    id INT(1) ZEROFILL,    -- 显示 0001
    count INT(10) ZEROFILL -- 显示 0000000001
);
```

**Text 数据类型大小**：
- TINYTEXT：255 bytes
- TEXT：65,535 bytes ≈ 64KB
- MEDIUMTEXT：16,777,215 bytes ≈ 16MB
- LONGTEXT：4,294,967,295 bytes ≈ 4GB

**日期时间类型**：
- DATE：'YYYY-MM-DD'
- TIME：'HH:MM:SS'
- DATETIME：'YYYY-MM-DD HH:MM:SS'
- TIMESTAMP：时间戳，范围较小但有时区转换

**IP 地址存储方式**：
```sql
-- 字符串存储（简单直观）
CREATE TABLE ip_records (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ip_address VARCHAR(15)
);

-- 整数存储（节省空间，查询快）
CREATE TABLE ip_records (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ip_address INT UNSIGNED
);

-- 插入和查询
INSERT INTO ip_records (ip_address) VALUES (INET_ATON('192.168.1.1'));
SELECT INET_NTOA(ip_address) FROM ip_records;

-- IPv6 使用 VARBINARY(16)
CREATE TABLE ipv6_records (
    id INT AUTO_INCREMENT PRIMARY KEY,
    ip_address VARBINARY(16)
);
```

### 1.7 外键约束

**作用**：维护表与表之间的关系，确保数据完整性和一致性

**示例**：
```sql
-- 创建带外键的表
CREATE TABLE students (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(50) NOT NULL,
  course_id INT,
  FOREIGN KEY (course_id) REFERENCES courses(id)
    ON DELETE SET NULL
    ON UPDATE CASCADE
);

CREATE TABLE courses (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL
);
```

**外键操作**：
- **RESTRICT**：拒绝删除或更新（默认）
- **CASCADE**：级联删除或更新
- **SET NULL**：设置外键列为 NULL
- **NO ACTION**：标准SQL关键字，同RESTRICT

**外键优势**：
- 数据完整性保证
- 防止孤儿记录
- 自动维护关系一致性

**外键劣势**：
- 性能开销
- 复杂性增加
- 影响分库分表

### 1.8 IN 和 EXISTS

**IN 关键字**：
```sql
-- 值列表
SELECT * FROM Customers WHERE Country IN ('Germany', 'France', 'UK');

-- 子查询
SELECT * FROM Customers 
WHERE CustomerID IN (SELECT CustomerID FROM Orders WHERE Total > 1000);

-- NOT IN
SELECT * FROM Products 
WHERE CategoryID NOT IN (SELECT CategoryID FROM Categories WHERE Active = 1);
```

**EXISTS 关键字**：
```sql
-- 基本 EXISTS
SELECT * FROM Customers c
WHERE EXISTS (
    SELECT 1 FROM Orders o 
    WHERE o.CustomerID = c.CustomerID AND o.OrderDate > '2023-01-01'
);

-- NOT EXISTS
SELECT * FROM Products p
WHERE NOT EXISTS (
    SELECT 1 FROM OrderDetails od WHERE od.ProductID = p.ProductID
);

-- 相关子查询
SELECT * FROM Departments d
WHERE EXISTS (
    SELECT 1 FROM Employees e 
    WHERE e.DepartmentID = d.DepartmentID AND e.Salary > 50000
);
```

**区别对比**：

| 特性 | IN | EXISTS |
|------|----|--------|
| 执行逻辑 | 先执行子查询，再主查询 | 主查询驱动，逐行检查 |
| 性能 | 子查询结果集小且固定时快 | 子查询表大或关联查询时快 |
| NULL处理 | NOT IN 遇到 NULL 返回空结果集 | 不受 NULL 值影响 |
| 适用场景 | 静态值列表或小结果集 | 关联查询，大表查询 |
| 可读性 | 直观易懂 | 相对复杂 |

**性能优化建议**：
- 小结果集用 IN，大结果集用 EXISTS
- 避免在 IN 中使用大子查询
- EXISTS 通常能利用索引更好

### 1.9 MySQL 基本函数

**字符串函数**：
```sql
-- 连接和格式化
SELECT CONCAT('Hello', ' ', 'World'); -- 'Hello World'
SELECT CONCAT_WS(', ', 'John', 'Doe'); -- 'John, Doe'
SELECT FORMAT(1234567.89, 2); -- '1,234,567.89'

-- 截取和填充
SELECT SUBSTRING('Hello World', 1, 5); -- 'Hello'
SELECT LEFT('Hello World', 5); -- 'Hello'
SELECT RIGHT('Hello World', 5); -- 'World'
SELECT LPAD('5', 3, '0'); -- '005'

-- 查找和替换
SELECT LOCATE('World', 'Hello World'); -- 7
SELECT REPLACE('Hello World', 'World', 'MySQL'); -- 'Hello MySQL'
SELECT REPEAT('MySQL ', 3); -- 'MySQL MySQL MySQL '

-- 大小写和修剪
SELECT UPPER('hello'); -- 'HELLO'
SELECT LOWER('HELLO'); -- 'hello'
SELECT TRIM('  hello  '); -- 'hello'
SELECT LTRIM('  hello'); -- 'hello'
SELECT RTRIM('hello  '); -- 'hello'
```

**数值函数**：
```sql
-- 基本数学函数
SELECT ABS(-10); -- 10
SELECT CEIL(10.1); -- 11
SELECT FLOOR(10.9); -- 10
SELECT ROUND(10.567, 2); -- 10.57

-- 指数和对数
SELECT POWER(2, 3); -- 8
SELECT SQRT(16); -- 4
SELECT EXP(1); -- 2.718
SELECT LN(2.718); -- 1.0

-- 随机数和符号
SELECT RAND(); -- 0-1随机数
SELECT SIGN(-10); -- -1
SELECT MOD(10, 3); -- 1
```

**日期函数**：
```sql
-- 当前时间
SELECT NOW(); -- '2023-10-17 10:30:45'
SELECT CURDATE(); -- '2023-10-17'
SELECT CURTIME(); -- '10:30:45'
SELECT UNIX_TIMESTAMP(); -- 1697517045

-- 日期计算
SELECT DATE_ADD(NOW(), INTERVAL 1 DAY); -- 加1天
SELECT DATE_SUB(NOW(), INTERVAL 1 MONTH); -- 减1月
SELECT DATEDIFF('2023-10-20', '2023-10-17'); -- 3
SELECT TIMESTAMPDIFF(HOUR, '2023-10-17 08:00:00', '2023-10-17 18:00:00'); -- 10

-- 日期提取
SELECT YEAR(NOW()); -- 2023
SELECT MONTH(NOW()); -- 10
SELECT DAY(NOW()); -- 17
SELECT DAYNAME(NOW()); -- 'Tuesday'
SELECT WEEK(NOW()); -- 42
```

**聚合函数**：
```sql
-- 基本聚合
SELECT COUNT(*) as total_count FROM orders;
SELECT SUM(amount) as total_amount FROM orders;
SELECT AVG(amount) as avg_amount FROM orders;
SELECT MAX(amount) as max_amount FROM orders;
SELECT MIN(amount) as min_amount FROM orders;

-- 分组聚合
SELECT department, COUNT(*), AVG(salary)
FROM employees
GROUP BY department
HAVING AVG(salary) > 5000;

-- 统计函数
SELECT STD(salary) as salary_std FROM employees;
SELECT VARIANCE(salary) as salary_var FROM employees;
SELECT GROUP_CONCAT(name SEPARATOR ', ') FROM employees GROUP BY department;
```

**条件函数**：
```sql
-- CASE WHEN
SELECT name, 
       CASE 
           WHEN salary > 10000 THEN '高薪'
           WHEN salary > 5000 THEN '中薪'
           ELSE '低薪'
       END as salary_level
FROM employees;

-- IF 函数
SELECT name, IF(salary > 5000, '高', '低') as salary_level FROM employees;

-- IFNULL 和 COALESCE
SELECT name, IFNULL(bonus, 0) as bonus FROM employees;
SELECT name, COALESCE(phone, mobile, '无联系方式') as contact FROM employees;
```

### 1.10 SQL 查询语句执行顺序

**完整执行顺序**：
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

**详细解释**：
1. **FROM**：确定数据来源表
2. **ON**：连接条件过滤
3. **JOIN**：根据连接类型合并表
4. **WHERE**：行级条件过滤
5. **GROUP BY**：分组操作
6. **聚合函数**：计算分组聚合值
7. **HAVING**：分组后条件过滤
8. **SELECT**：选择输出列
9. **DISTINCT**：去重操作
10. **ORDER BY**：结果排序
11. **LIMIT**：结果数量限制

**示例分析**：
```sql
SELECT department, AVG(salary) as avg_salary
FROM employees
WHERE hire_date > '2020-01-01'
GROUP BY department
HAVING AVG(salary) > 5000
ORDER BY avg_salary DESC
LIMIT 10;
```
执行顺序：employees表 → WHERE过滤 → GROUP BY分组 → HAVING过滤 → SELECT选择 → ORDER BY排序 → LIMIT限制

## 2. 存储引擎

### 2.1 常见存储引擎比较

| 特性 | InnoDB | MyISAM | Memory | Archive |
|------|--------|--------|--------|---------|
| 事务支持 | ✅ | ❌ | ❌ | ❌ |
| 行级锁 | ✅ | ❌ | ✅ | ✅ |
| 外键约束 | ✅ | ❌ | ❌ | ❌ |
| 崩溃恢复 | ✅ | ❌ | ❌ | ❌ |
| 存储限制 | 64TB | 256TB | 内存大小 | 无限制 |
| 压缩存储 | ✅ | ✅ | ❌ | ✅ |
| 全文索引 | ✅ (5.6+) | ✅ | ❌ | ❌ |
| 适用场景 | 高并发读写 | 大量读操作 | 临时数据 | 日志归档 |

### 2.2 InnoDB 成为默认引擎的原因

1. **事务支持**：支持 ACID 事务，适合业务系统
2. **并发性能**：行级锁定提供更好的并发控制
3. **崩溃恢复**：通过 redo log 实现崩溃恢复，数据更安全
4. **数据完整性**：支持外键约束，保证数据一致性
5. **热备份**：支持在线热备份
6. **优化器支持**：MySQL优化器对InnoDB优化更好

### 2.3 InnoDB vs MyISAM 详细区别

**事务和锁**：
- InnoDB：支持事务，行级锁，适合高并发写
- MyISAM：不支持事务，表级锁，并发写性能差

**索引结构**：
- InnoDB：聚簇索引，数据文件本身就是索引文件
- MyISAM：非聚簇索引，索引和数据文件分离（.MYI和.MYD）

**存储特性**：
- InnoDB：支持外键，支持在线热备份
- MyISAM：支持全文索引（5.6前），压缩表

**性能特点**：
- InnoDB：写操作性能好，支持事务回滚
- MyISAM：读操作性能好，COUNT(*) 速度快

**崩溃恢复**：
- InnoDB：通过 redo log 支持崩溃恢复
- MyISAM：不支持崩溃恢复，需要修复表

### 2.4 数据文件类型

**数据库目录结构**：
```
/var/lib/mysql/
├── my_test/
│   ├── db.opt           # 数据库字符集配置
│   ├── t_order.frm      # 表结构定义
│   └── t_order.ibd      # 表数据文件（独占表空间）
├── ibdata1              # 共享表空间文件
├── ib_logfile0          # redo log 文件
├── ib_logfile1          # redo log 文件
└── mysql-bin.000001     # binlog 文件
```

**文件说明**：
- **.frm**：存储表结构定义，所有引擎都有
- **.ibd**：InnoDB表数据文件（innodb_file_per_table=1时）
- **ibdata1**：共享表空间，存储系统表空间、undo log等
- **.MYD**：MyISAM数据文件
- **.MYI**：MyISAM索引文件
- **db.opt**：数据库默认字符集和校验规则

**表空间管理**：
```sql
-- 查看表空间配置
SHOW VARIABLES LIKE 'innodb_file_per_table';

-- 启用独立表空间（推荐）
SET GLOBAL innodb_file_per_table=1;

-- 查看表空间信息
SELECT * FROM information_schema.INNODB_TABLESPACES;
```

## 3. 索引

### 3.1 索引分类详解

**按数据结构分类**：
- **B+Tree索引**：最常用，支持范围查询和排序
- **Hash索引**：精确匹配快，不支持范围查询
- **Full-text索引**：全文搜索，支持文本内容搜索
- **R-Tree索引**：空间数据索引，GIS数据

**按物理存储分类**：
- **聚簇索引**：叶子节点包含实际数据行，InnoDB主键索引
- **二级索引**：叶子节点包含主键值，需要回表查询

**按字段特性分类**：
- **主键索引**：建立在主键字段上，不允许空值
- **唯一索引**：建立在UNIQUE字段上，允许空值
- **普通索引**：建立在普通字段上，允许重复值
- **前缀索引**：对字符类型字段前几个字符建立索引

**按字段个数分类**：
- **单列索引**：单个字段上的索引
- **联合索引**：多个字段组合的索引，遵循最左匹配原则

### 3.2 聚簇索引 vs 非聚簇索引

| 特性 | 聚簇索引 | 非聚簇索引 |
|------|---------|-----------|
| 数据存储 | 索引叶子节点包含实际数据 | 叶子节点包含指针或主键值 |
| 索引数量 | 每个表只能有一个 | 可以有多个 |
| 查询效率 | 主键查询快，直接获取数据 | 需要回表查询，多一次IO |
| 范围查询 | 效率高，连续存储 | 效率相对较低 |
| 插入性能 | 主键顺序插入快，随机插入慢 | 相对稳定 |
| 更新影响 | 可能引起页分裂 | 影响较小 |

**InnoDB聚簇索引选择**：
1. 有主键：使用主键作为聚簇索引
2. 无主键有唯一索引：使用第一个非空唯一索引
3. 都没有：使用隐藏的ROWID作为聚簇索引

### 3.3 B+树详细特性

**数据结构特点**：
- 多叉树结构，所有叶子节点在同一层
- 非叶子节点只存储索引信息（键值和指针）
- 叶子节点存储实际数据或主键值
- 叶子节点形成双向链表，支持顺序访问

**B+树结构示例**：
```
         [10,   20,   30]          -- 根节点
         /      |       \
[5,8,9]  [15,18,19]  [25,28,29]    -- 中间节点
  |        |          |
[1,2,3] [11,12,13] [21,22,23]      -- 叶子节点（包含数据）
```

**搜索复杂度**：O(logdN)，其中d为节点最大子节点数

**优势**：
- **高度低**：千万级数据只需3-4层
- **磁盘I/O少**：查询只需3-4次磁盘I/O
- **范围查询高效**：叶子节点链表支持顺序扫描
- **查询稳定**：所有查询都要到叶子节点

**节点大小**：通常为16KB，与InnoDB页大小一致

### 3.4 B+树 vs B树 vs 二叉树

**B+树 vs B树**：
- B+树非叶子节点不存数据，B树存
- B+树叶子节点链表连接，B树无
- B+树查询性能更稳定，都要到叶子节点
- B+树范围查询更高效

**B+树 vs 二叉树**：
- B+树更矮胖，磁盘I/O更少
- 二叉树可能退化为链表，性能不稳定
- B+树更适合磁盘存储，减少随机I/O

**B+树 vs Hash**：
- B+树支持范围查询，Hash只支持等值查询
- B+树支持排序，Hash无序
- Hash在精确匹配时更快

### 3.5 联合索引与最左匹配原则

**创建联合索引**：
```sql
-- 创建姓名和年龄的联合索引
CREATE INDEX idx_name_age ON users(name, age);

-- 创建姓名、年龄、性别的联合索引
CREATE INDEX idx_name_age_gender ON users(name, age, gender);
```

**最左匹配原则**：
- 查询必须从索引的最左列开始
- 不能跳过索引中的列
- 范围查询后面的列无法使用索引

**示例分析**：
```sql
-- 能使用索引的情况
WHERE name = 'John'                    -- ✅ 使用name
WHERE name = 'John' AND age = 25       -- ✅ 使用name,age
WHERE name = 'John' AND age > 20       -- ✅ 使用name,age
WHERE name = 'John' AND age = 25 AND gender = 'M' -- ✅ 使用所有列

-- 不能完全使用索引的情况
WHERE age = 25                         -- ❌ 未从最左列开始
WHERE name = 'John' AND gender = 'M'   -- ❌ 跳过了age列
WHERE age = 25 AND gender = 'M'        -- ❌ 未从最左列开始
WHERE name LIKE 'J%' AND age = 25      -- ✅ 使用name,age
```

**索引跳跃扫描**（MySQL 8.0+）：
```sql
-- MySQL 8.0支持索引跳跃扫描
WHERE gender = 'F' AND age = 25        -- 可能使用idx_gender_age索引
```

### 3.6 索引失效的 6 种情况

1. **模糊匹配**：
   ```sql
   -- 失效
   SELECT * FROM users WHERE name LIKE '%John%';
   SELECT * FROM users WHERE name LIKE '%John';
   
   -- 有效
   SELECT * FROM users WHERE name LIKE 'John%';
   ```

2. **使用函数**：
   ```sql
   -- 失效
   SELECT * FROM users WHERE UPPER(name) = 'JOHN';
   SELECT * FROM users WHERE DATE(create_time) = '2023-01-01';
   
   -- 有效
   SELECT * FROM users WHERE name = UPPER('john');
   SELECT * FROM users WHERE create_time >= '2023-01-01' AND create_time < '2023-01-02';
   ```

3. **表达式计算**：
   ```sql
   -- 失效
   SELECT * FROM products WHERE price + 10 > 100;
   SELECT * FROM users WHERE id + 1 = 10;
   
   -- 有效
   SELECT * FROM products WHERE price > 90;
   SELECT * FROM users WHERE id = 9;
   ```

4. **隐式类型转换**：
   ```sql
   -- 失效（name是varchar，123是数字）
   SELECT * FROM users WHERE name = 123;
   
   -- 有效
   SELECT * FROM users WHERE name = '123';
   ```

5. **违反最左匹配**：
   ```sql
   -- idx_name_age索引
   SELECT * FROM users WHERE age = 25;  -- 失效
   ```

6. **OR 条件不当**：
   ```sql
   -- 失效（name有索引，age无索引）
   SELECT * FROM users WHERE name = 'John' OR age = 25;
   
   -- 有效（都有索引）
   SELECT * FROM users WHERE name = 'John' OR name = 'Jane';
   ```

### 3.7 索引优化策略

**前缀索引优化**：
```sql
-- 对长字符串字段创建前缀索引
CREATE INDEX idx_email_prefix ON users(email(10));

-- 计算合适的前缀长度
SELECT 
    COUNT(DISTINCT LEFT(email, 5)) / COUNT(*) as selectivity_5,
    COUNT(DISTINCT LEFT(email, 10)) / COUNT(*) as selectivity_10,
    COUNT(DISTINCT LEFT(email, 15)) / COUNT(*) as selectivity_15
FROM users;
```

**覆盖索引优化**：
```sql
-- 创建覆盖索引
CREATE INDEX idx_covering ON users(name, age, email);

-- 使用覆盖索引，避免回表
SELECT name, age, email FROM users WHERE name = 'John';  -- ✅ 覆盖索引
SELECT * FROM users WHERE name = 'John';                 -- ❌ 需要回表
```

**索引选择性优化**：
```sql
-- 计算字段的选择性
SELECT 
    COUNT(DISTINCT gender) / COUNT(*) as gender_selectivity,
    COUNT(DISTINCT name) / COUNT(*) as name_selectivity
FROM users;

-- 选择性高的字段更适合建索引
```

**索引创建原则**：
- 区分度高的字段适合建索引（选择性 > 0.1）
- 频繁查询的字段适合建索引
- 频繁作为WHERE条件的字段适合建索引
- 避免在更新频繁的字段上建索引
- 考虑字段的数据类型，优先选择小的数据类型

**索引维护**：
```sql
-- 查看索引使用情况
SELECT * FROM information_schema.STATISTICS 
WHERE TABLE_NAME = 'users';

-- 分析索引使用
EXPLAIN SELECT * FROM users WHERE name = 'John';

-- 优化表（重建索引）
OPTIMIZE TABLE users;

-- 删除冗余索引
DROP INDEX redundant_index ON users;
```

## 4. 事务

### 4.1 事务特性（ACID）及实现

**原子性（Atomicity）**：
- **定义**：事务的所有操作要么全部完成，要么全部不完成
- **实现**：通过 undo log 实现回滚
- **机制**：事务执行过程中，所有修改都记录到undo log，回滚时反向执行

**一致性（Consistency）**：
- **定义**：事务操作前后数据满足完整性约束
- **实现**：通过持久性+原子性+隔离性保证
- **约束**：主键约束、外键约束、唯一约束、非空约束等

**隔离性（Isolation）**：
- **定义**：多个事务并发执行时相互隔离
- **实现**：通过 MVCC 或锁机制实现
- **级别**：读未提交、读提交、可重复读、串行化

**持久性（Durability）**：
- **定义**：事务提交后修改永久保存
- **实现**：通过 redo log 保证
- **机制**：先写日志，后写数据，崩溃时重放redo log

### 4.2 并发问题

**脏读（Dirty Read）**：
- **定义**：一个事务读到另一个未提交事务修改的数据
- **发生级别**：读未提交
- **示例**：事务A修改数据未提交，事务B读取到未提交数据

**不可重复读（Non-Repeatable Read）**：
- **定义**：同一事务内多次读取同一数据结果不一致
- **发生级别**：读未提交、读提交
- **示例**：事务A读取数据后，事务B修改并提交，事务A再次读取数据变化

**幻读（Phantom Read）**：
- **定义**：同一事务内多次查询记录数量不一致
- **发生级别**：读未提交、读提交、可重复读
- **示例**：事务A查询符合条件的记录数，事务B插入新记录并提交，事务A再次查询记录数增加

**丢失更新（Lost Update）**：
- **定义**：两个事务同时更新同一数据，后提交的覆盖先提交的更新
- **解决**：通过锁机制或乐观锁解决

### 4.3 隔离级别详解

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 实现方式 | 性能 |
|---------|------|-----------|------|----------|------|
| 读未提交 | ✅ | ✅ | ✅ | 直接读最新数据 | 最高 |
| 读提交 | ❌ | ✅ | ✅ | 每个语句前生成Read View | 高 |
| 可重复读 | ❌ | ❌ | ✅ | 事务开始时生成Read View | 中 |
| 串行化 | ❌ | ❌ | ❌ | 加锁 | 最低 |

**设置隔离级别**：
```sql
-- 查看当前隔离级别
SELECT @@transaction_isolation;

-- 设置会话隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 设置全局隔离级别
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

**MySQL 默认隔离级别**：可重复读（REPEATABLE READ）

### 4.4 MVCC 实现原理

**核心组件**：

**隐藏列**：
- `DB_TRX_ID`：最近修改事务ID
- `DB_ROLL_PTR`：回滚指针，指向undo log记录
- `DB_ROW_ID`：行ID（隐藏主键）

**Undo Log**：
- 存储历史版本数据
- 形成版本链，通过回滚指针连接
- 用于回滚和MVCC读

**Read View**：
- 事务可见性判断依据
- 包含：活跃事务列表、最小活跃事务ID、下一个事务ID等

**Read View 字段**：
- `m_ids`：活跃事务ID列表
- `min_trx_id`：最小活跃事务ID
- `max_trx_id`：下一个事务ID
- `creator_trx_id`：创建者事务ID

**可见性判断规则**：
1. `trx_id < min_trx_id`：可见（事务已提交）
2. `trx_id >= max_trx_id`：不可见（事务后启动）
3. `min_trx_id <= trx_id < max_trx_id`：
   - 在 `m_ids` 中：不可见（事务未提交）
   - 不在 `m_ids` 中：可见（事务已提交）

**MVCC 读操作流程**：
1. 获取事务Read View
2. 遍历数据行版本链
3. 根据可见性规则找到合适版本
4. 返回可见版本数据

## 5. 锁

### 5.1 锁类型详解

**全局锁**：
```sql
-- 加全局读锁
FLUSH TABLES WITH READ LOCK;

-- 释放锁
UNLOCK TABLES;

-- 全局锁使用场景：全库备份
```

**表级锁**：
```sql
-- 显式表锁
LOCK TABLES table_name READ;   -- 读锁
LOCK TABLES table_name WRITE;  -- 写锁

-- 解锁
UNLOCK TABLES;

-- 元数据锁（MDL）：自动加锁，防止表结构变更
```

**意向锁**：
- **意向共享锁（IS）**：事务打算在表中的行上加共享锁
- **意向排他锁（IX）**：事务打算在表中的行上加排他锁
- **作用**：避免表级锁和行级锁冲突检查

**行级锁**：
- **记录锁（Record Lock）**：锁定单条记录
- **间隙锁（Gap Lock）**：锁定记录之间的间隙
- **临键锁（Next-Key Lock）**：记录锁+间隙锁
- **插入意向锁**：插入操作时使用的间隙锁

### 5.2 锁应用场景

**表锁适用场景**：
- 全表数据迁移或备份
- 表结构变更（ALTER TABLE）
- 大批量数据操作（历史数据清理）

**行锁适用场景**：
- 高并发单行操作（账户余额更新）
- 需要精细并发控制的场景
- 在线事务处理（OLTP）系统

**间隙锁作用**：
- 防止幻读
- 在可重复读隔离级别下使用
- 锁定不存在的记录范围

**死锁处理**：
```sql
-- 查看死锁信息
SHOW ENGINE INNODB STATUS;

-- 设置死锁超时
SET innodb_lock_wait_timeout = 50;

-- 死锁检测
SET innodb_deadlock_detect = ON;
```

### 5.3 锁兼容性矩阵

| 当前锁 | IS | IX | S | X |
|--------|----|----|---|---|
| IS | ✅ | ✅ | ✅ | ❌ |
| IX | ✅ | ✅ | ❌ | ❌ |
| S | ✅ | ❌ | ✅ | ❌ |
| X | ❌ | ❌ | ❌ | ❌ |

## 6. 日志系统

### 6.1 日志类型及作用

**redo log（重做日志）**：
- **作用**：保证事务持久性，崩溃恢复
- **特点**：物理日志，顺序写入，循环使用
- **存储**：记录数据页的物理修改
- **文件**：ib_logfile0, ib_logfile1

**undo log（回滚日志）**：
- **作用**：保证事务原子性，实现回滚和MVCC
- **特点**：逻辑日志，随机写入
- **存储**：记录更新前的数据状态
- **位置**：共享表空间或独立表空间

**binlog（二进制日志）**：
- **作用**：数据备份和主从复制
- **特点**：逻辑日志，追加写入
- **格式**：STATEMENT、ROW、MIXED
- **文件**：mysql-bin.000001等

**relay log（中继日志）**：
- **作用**：主从复制中间存储
- **特点**：从库接收的binlog暂存
- **处理**：SQL线程回放relay log

**slow query log（慢查询日志）**：
- **作用**：记录执行时间超过阈值的SQL
- **配置**：long_query_time参数
- **分析工具**：mysqldumpslow, pt-query-digest

**error log（错误日志）**：
- **作用**：记录MySQL启动、运行、停止过程中的错误信息
- **位置**：通常为hostname.err

### 6.2 两阶段提交

**过程**：

**prepare 阶段**：
1. 写入redo log，状态为prepare
2. 持久化redo log到磁盘
3. 事务处于预备状态

**commit 阶段**：
1. 写入binlog
2. 持久化binlog到磁盘
3. 提交事务，redo log状态改为commit

**崩溃恢复**：
- **场景1**：binlog有XID，redo log状态prepare → 提交事务
- **场景2**：binlog无XID，redo log状态prepare → 回滚事务
- **场景3**：redo log状态commit → 完成事务提交

**配置优化**：
```sql
-- 提高日志写入性能
SET sync_binlog = 0;        -- 依赖OS刷盘
SET innodb_flush_log_at_trx_commit = 2; -- 每秒刷盘

-- 保证数据安全
SET sync_binlog = 1;        -- 每次提交刷盘
SET innodb_flush_log_at_trx_commit = 1; -- 每次提交刷盘
```

### 6.3 Double Write Buffer

**作用**：防止数据页损坏，保证数据页写入的原子性

**过程**：
1. 数据页先拷贝到Doublewrite Buffer（内存）
2. Doublewrite Buffer分两次顺序写入共享表空间（磁盘）
3. 数据页写入实际表空间文件
4. 如果步骤3失败，从Doublewrite Buffer恢复数据页

**优势**：
- 防止部分写（partial write）问题
- 数据页损坏时可以从Doublewrite Buffer恢复
- 保证数据页写入的原子性

**配置**：
```sql
-- 查看Doublewrite Buffer状态
SHOW STATUS LIKE 'Innodb_dblwr%';

-- 禁用Doublewrite Buffer（不推荐）
SET innodb_doublewrite = OFF;
```

## 7. 性能调优

### 7.1 EXPLAIN 详细分析

**EXPLAIN 输出字段详解**：

**id**：
- 查询标识符
- 相同id表示同一查询块
- 不同id按从大到小执行

**select_type**：
- SIMPLE：简单查询
- PRIMARY：最外层查询
- SUBQUERY：子查询
- DERIVED：派生表
- UNION：UNION查询

**table**：
- 访问的表名
- `<derivedN>`：派生表
- `<unionM,N>`：UNION结果

**partitions**：
- 匹配的分区

**type**（重要，性能从好到坏）：
- **system**：表只有一行
- **const**：主键或唯一索引等值查询
- **eq_ref**：唯一索引关联查询
- **ref**：非唯一索引查询
- **range**：索引范围扫描
- **index**：全索引扫描
- **ALL**：全表扫描

**possible_keys**：
- 可能使用的索引

**key**：
- 实际使用的索引

**key_len**：
- 使用的索引长度

**ref**：
- 与索引比较的列

**rows**：
- 预估扫描行数

**filtered**：
- 条件过滤的百分比

**Extra**（重要）：
- **Using index**：覆盖索引
- **Using where**：WHERE条件过滤
- **Using temporary**：使用临时表
- **Using filesort**：文件排序
- **Using join buffer**：使用连接缓冲

### 7.2 查询优化实战

**索引优化**：
```sql
-- 创建合适索引
CREATE INDEX idx_name_age ON users(name, age);
CREATE INDEX idx_email ON users(email);
CREATE INDEX idx_created_at ON orders(created_at);

-- 使用覆盖索引
SELECT id, name FROM users WHERE name = 'John';  -- ✅
SELECT * FROM users WHERE name = 'John';         -- ❌

-- 避免索引失效
-- 错误：使用函数
SELECT * FROM users WHERE DATE(create_time) = '2023-01-01';
-- 正确：范围查询
SELECT * FROM users WHERE create_time >= '2023-01-01' AND create_time < '2023-01-02';
```

**分页优化**：
```sql
-- 传统分页（性能差）
SELECT * FROM orders ORDER BY id LIMIT 100000, 20;

-- 优化分页（使用游标）
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 20;

-- 延迟关联
SELECT * FROM orders 
INNER JOIN (SELECT id FROM orders ORDER BY id LIMIT 100000, 20) AS tmp 
ON orders.id = tmp.id;
```

**联表优化**：
```sql
-- 小表驱动大表
SELECT * FROM small_table s 
JOIN large_table l ON s.id = l.small_id;

-- 确保被驱动表有索引
CREATE INDEX idx_small_id ON large_table(small_id);

-- 使用STRAIGHT_JOIN强制连接顺序
SELECT STRAIGHT_JOIN s.*, l.* 
FROM small_table s 
JOIN large_table l ON s.id = l.small_id;
```

**子查询优化**：
```sql
-- 低效：相关子查询
SELECT * FROM users u 
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- 高效：使用JOIN
SELECT DISTINCT u.* FROM users u 
JOIN orders o ON u.id = o.user_id;

-- 使用派生表
SELECT u.* FROM users u
JOIN (SELECT DISTINCT user_id FROM orders) o ON u.id = o.user_id;
```

### 7.3 慢查询解决方案

**1. 分析执行计划**：
```sql
-- 使用EXPLAIN分析
EXPLAIN SELECT * FROM users WHERE name = 'John';

-- 使用EXPLAIN FORMAT=JSON获取详细信息
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE name = 'John';

-- 使用EXPLAIN ANALYZE（MySQL 8.0+）
EXPLAIN ANALYZE SELECT * FROM users WHERE name = 'John';
```

**2. 优化索引**：
```sql
-- 创建缺失索引
CREATE INDEX idx_missing ON table_name(column1, column2);

-- 删除冗余索引
DROP INDEX redundant_index ON table_name;

-- 重建索引
ALTER TABLE table_name ENGINE=InnoDB;
```

**3. 重写SQL**：
```sql
-- 避免复杂子查询
-- 优化前
SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE amount > 100);
-- 优化后
SELECT u.* FROM users u 
JOIN orders o ON u.id = o.user_id 
WHERE o.amount > 100 
GROUP BY u.id;
```

**4. 数据库设计优化**：
- 适当反范式化，减少联表
- 分区表处理大数据量
- 使用合适的数据类型

**5. 架构优化**：
- 读写分离
- 分库分表
- 使用缓存

**6. 缓存策略**：
```sql
-- 查询缓存（MySQL 8.0已移除）
-- 使用应用程序缓存（Redis, Memcached）
```

## 8. 架构设计

### 8.1 主从复制原理

**复制过程**：

**主库（Master）**：
1. 事务提交，写入binlog
2. log dump线程读取binlog
3. 发送binlog事件到从库

**从库（Slave）**：
1. I/O线程接收binlog，写入relay log
2. SQL线程读取relay log
3. 回放SQL事件，更新数据

**复制配置**：
```sql
-- 主库配置
[mysqld]
server_id = 1
log_bin = mysql-bin
binlog_format = ROW

-- 从库配置
[mysqld]
server_id = 2
relay_log = mysql-relay-bin
read_only = 1
```

**创建复制用户**：
```sql
-- 主库执行
CREATE USER 'repl'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
```

**启动复制**：
```sql
-- 从库执行
CHANGE MASTER TO
MASTER_HOST='master_host',
MASTER_USER='repl',
MASTER_PASSWORD='password',
MASTER_LOG_FILE='mysql-bin.000001',
MASTER_LOG_POS=107;

START SLAVE;

-- 查看复制状态
SHOW SLAVE STATUS\G
```

### 8.2 主从延迟处理

**延迟原因**：
- 从库性能较差
- 网络延迟或带宽不足
- 大事务执行（长时间未提交）
- 从库过多，主库负载高
- 串行复制，SQL线程单线程

**解决方案**：

**强制走主库**：
```sql
-- 关键业务读主库
-- 根据业务场景拆分读写
```

**并行复制**：
```sql
-- 启用并行复制
SET GLOBAL slave_parallel_workers = 4;
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
```

**缓存层**：
```java
// 读取缓存，减少数据库压力
// 缓存穿透时读主库
```

**业务拆分**：
- 延迟不敏感业务走从库
- 实时性要求高业务走主库

**监控延迟**：
```sql
-- 查看复制延迟
SHOW SLAVE STATUS\G

-- 查看Seconds_Behind_Master
SELECT * FROM performance_schema.replication_applier_status_by_worker;
```

### 8.3 分库分表策略

**垂直分库**：
```sql
-- 按业务模块拆分
-- 用户库
CREATE DATABASE user_db;
-- 订单库  
CREATE DATABASE order_db;
-- 商品库
CREATE DATABASE product_db;
```

**垂直分表**：
```sql
-- 大表拆小表
-- 用户基础表
CREATE TABLE user_basic (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    email VARCHAR(100)
);

-- 用户扩展表
CREATE TABLE user_ext (
    user_id INT PRIMARY KEY,
    profile TEXT,
    preferences JSON
);
```

**水平分库**：
```sql
-- 按用户ID分库
-- user_db_0: user_id % 4 = 0
-- user_db_1: user_id % 4 = 1
-- user_db_2: user_id % 4 = 2  
-- user_db_3: user_id % 4 = 3
```

**水平分表**：
```sql
-- 按月分表
CREATE TABLE orders_202301 (...);
CREATE TABLE orders_202302 (...);
CREATE TABLE orders_202303 (...);
```

**分片策略**：

**范围分片**：
```sql
-- 按时间范围
-- 按ID范围
```

**哈希分片**：
```sql
-- 一致性哈希
-- 用户ID取模
```

**地理位置分片**：
```sql
-- 按地区分片
-- 华北、华东、华南等
```

**分片中间件**：
- MyCAT
- ShardingSphere
- Vitess

## 9. 高级特性

### 9.1 窗口函数

**基本语法**：
```sql
SELECT 
    column1,
    column2,
    window_function() OVER (
        [PARTITION BY partition_expression]
        [ORDER BY sort_expression [ASC|DESC]]
        [frame_clause]
    ) as window_column
FROM table_name;
```

**排名函数**：
```sql
-- 员工薪资排名
SELECT 
    name,
    salary,
    department,
    ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as row_num,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rank,
    DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) as dense_rank,
    NTILE(4) OVER (ORDER BY salary DESC) as quartile
FROM employees;
```

**聚合窗口函数**：
```sql
-- 计算移动平均
SELECT 
    order_date,
    amount,
    AVG(amount) OVER (
        ORDER BY order_date 
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) as moving_avg,
    SUM(amount) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) as running_total
FROM orders;
```

**前后值函数**：
```sql
-- 计算环比增长
SELECT 
    month,
    revenue,
    LAG(revenue, 1) OVER (ORDER BY month) as prev_revenue,
    revenue - LAG(revenue, 1) OVER (ORDER BY month) as growth,
    LEAD(revenue, 1) OVER (ORDER BY month) as next_revenue
FROM monthly_sales;
```

**首尾值函数**：
```sql
-- 计算部门内最高最低薪资
SELECT 
    name,
    department,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department 
        ORDER BY salary DESC
    ) as highest_salary,
    LAST_VALUE(salary) OVER (
        PARTITION BY department 
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as lowest_salary
FROM employees;
```

### 9.2 通用表表达式（CTE）

**基本CTE**：
```sql
-- 简单CTE
WITH department_stats AS (
    SELECT 
        department,
        COUNT(*) as emp_count,
        AVG(salary) as avg_salary
    FROM employees
    GROUP BY department
)
SELECT * FROM department_stats WHERE emp_count > 10;
```

**递归CTE**：
```sql
-- 组织架构树形查询
WITH RECURSIVE org_tree AS (
    -- 锚点：顶级部门
    SELECT 
        id,
        name,
        parent_id,
        1 as level
    FROM departments
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- 递归：子部门
    SELECT 
        d.id,
        d.name,
        d.parent_id,
        ot.level + 1
    FROM departments d
    INNER JOIN org_tree ot ON d.parent_id = ot.id
)
SELECT * FROM org_tree ORDER BY level, id;
```

**多CTE**：
```sql
WITH 
dept_summary AS (
    SELECT department, COUNT(*) as cnt 
    FROM employees 
    GROUP BY department
),
high_salary_dept AS (
    SELECT department 
    FROM employees 
    GROUP BY department 
    HAVING AVG(salary) > 5000
)
SELECT * FROM dept_summary 
WHERE department IN (SELECT department FROM high_salary_dept);
```

### 9.3 JSON 支持

**创建JSON数据**：
```sql
-- 创建带JSON列的表
CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100),
    attributes JSON,
    tags JSON
);

-- 插入JSON数据
INSERT INTO products (name, attributes, tags) VALUES (
    'Laptop',
    '{"brand": "Dell", "specs": {"cpu": "i7", "ram": "16GB"}}',
    '["electronics", "computer"]'
);
```

**JSON查询**：
```sql
-- 提取JSON字段
SELECT 
    name,
    attributes->>'$.brand' as brand,
    attributes->'$.specs.cpu' as cpu,
    JSON_EXTRACT(attributes, '$.specs.ram') as ram
FROM products;

-- JSON路径查询
SELECT * FROM products 
WHERE attributes->>'$.brand' = 'Dell';

-- 检查JSON包含
SELECT * FROM products 
WHERE JSON_CONTAINS(tags, '"electronics"');
```

**JSON函数**：
```sql
-- JSON对象操作
SELECT 
    JSON_OBJECT('name', name, 'price', price) as product_json,
    JSON_ARRAY('tag1', 'tag2') as tags_array,
    JSON_MERGE(attributes, '{"warranty": "2 years"}') as merged_attrs
FROM products;

-- JSON聚合
SELECT 
    department,
    JSON_ARRAYAGG(JSON_OBJECT('name', name, 'salary', salary)) as employees
FROM employees
GROUP BY department;
```

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

-- 方法3：使用集合操作
SELECT s.sid, s.sname
FROM Student s
WHERE s.sid IN (SELECT sid FROM Score WHERE cid = '02')
AND s.sid NOT IN (SELECT sid FROM Score WHERE cid = '01');
```

**题目2：查询总分排名5-10名的学生**
```sql
-- 方法1：使用窗口函数（MySQL 8.0+）
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

-- 方法2：使用变量（MySQL 5.7）
SELECT 
    stu_id,
    total_score,
    ranking
FROM (
    SELECT 
        stu_id,
        total_score,
        @rank := @rank + 1 as ranking
    FROM (
        SELECT 
            stu_id,
            SUM(score) as total_score
        FROM student_score
        GROUP BY stu_id
        ORDER BY total_score DESC
    ) t, (SELECT @rank := 0) r
) ranked
WHERE ranking BETWEEN 5 AND 10;
```

**题目3：查询每个班级分数前3名的学生**
```sql
-- 使用窗口函数
SELECT 
    class_id,
    student_id,
    student_name,
    score,
    ranking
FROM (
    SELECT 
        c.class_id,
        s.student_id,
        s.student_name,
        sc.score,
        ROW_NUMBER() OVER (PARTITION BY c.class_id ORDER BY sc.score DESC) as ranking
    FROM students s
    JOIN classes c ON s.class_id = c.class_id
    JOIN scores sc ON s.student_id = sc.student_id
) ranked
WHERE ranking <= 3;
```

**题目4：连续登录用户查询**
```sql
-- 查询连续登录7天以上的用户
WITH UserLoginGroups AS (
    SELECT 
        user_id,
        login_date,
        DATE_SUB(login_date, INTERVAL ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY login_date) DAY) as group_id
    FROM user_logins
    GROUP BY user_id, login_date
),
LoginStreaks AS (
    SELECT 
        user_id,
        MIN(login_date) as start_date,
        MAX(login_date) as end_date,
        COUNT(*) as streak_days
    FROM UserLoginGroups
    GROUP BY user_id, group_id
    HAVING COUNT(*) >= 7
)
SELECT * FROM LoginStreaks ORDER BY streak_days DESC;
```

### 10.2 性能优化案例

**案例1：深分页优化**
```sql
-- 优化前（性能差）
SELECT * FROM orders ORDER BY id LIMIT 100000, 20;

-- 优化后1：使用游标（基于ID）
SELECT * FROM orders WHERE id > 100000 ORDER BY id LIMIT 20;

-- 优化后2：延迟关联
SELECT * FROM orders 
INNER JOIN (SELECT id FROM orders ORDER BY id LIMIT 100000, 20) AS tmp 
ON orders.id = tmp.id;

-- 优化后3：使用覆盖索引
CREATE INDEX idx_covering ON orders(id, status, amount);
SELECT id FROM orders ORDER BY id LIMIT 100000, 20;  -- 获取ID
SELECT * FROM orders WHERE id IN (...);              -- 根据ID查询详情
```

**案例2：大数据量统计优化**
```sql
-- 优化前：全表扫描
SELECT COUNT(*) FROM users WHERE status = 1;

-- 优化后1：使用索引
CREATE INDEX idx_status ON users(status);
SELECT COUNT(*) FROM users WHERE status = 1;

-- 优化后2：使用汇总表
CREATE TABLE user_stats (
    date DATE PRIMARY KEY,
    active_count INT,
    total_count INT
);

-- 定期更新汇总表
REPLACE INTO user_stats 
SELECT 
    CURDATE(),
    COUNT(CASE WHEN status = 1 THEN 1 END),
    COUNT(*)
FROM users;

-- 查询时直接使用汇总表
SELECT active_count FROM user_stats WHERE date = CURDATE();
```

**案例3：联表查询优化**
```sql
-- 优化前：无索引联表
SELECT * FROM orders o 
JOIN users u ON o.user_id = u.id 
WHERE u.name = 'John';

-- 优化后1：确保索引
CREATE INDEX idx_user_id ON orders(user_id);
CREATE INDEX idx_name ON users(name);

-- 优化后2：小表驱动大表
SELECT STRAIGHT_JOIN o.* 
FROM users u 
JOIN orders o ON u.id = o.user_id 
WHERE u.name = 'John';

-- 优化后3：使用覆盖索引
CREATE INDEX idx_covering ON users(name, id);
SELECT o.* 
FROM orders o 
WHERE o.user_id IN (SELECT id FROM users WHERE name = 'John');
```

**案例4：大数据删除优化**
```sql
-- 优化前：单次删除大量数据
DELETE FROM logs WHERE created_at < '2023-01-01';  -- 可能锁表

-- 优化后：分批删除
DELIMITER $$
CREATE PROCEDURE BatchDeleteLogs()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE batch_size INT DEFAULT 1000;
    
    WHILE NOT done DO
        DELETE FROM logs 
        WHERE created_at < '2023-01-01' 
        LIMIT batch_size;
        
        IF ROW_COUNT() = 0 THEN
            SET done = TRUE;
        END IF;
        
        -- 避免锁表，稍作休息
        DO SLEEP(0.1);
    END WHILE;
END$$
DELIMITER ;

CALL BatchDeleteLogs();
```

### 10.3 高并发场景解决方案

**乐观锁实现**：
```sql
-- 商品库存更新（避免超卖）
CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    stock INT,
    version INT DEFAULT 0
);

-- 乐观锁更新
UPDATE products 
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND stock > 0 AND version = #{current_version};
```

**悲观锁实现**：
```sql
-- 使用SELECT FOR UPDATE
START TRANSACTION;

SELECT * FROM accounts 
WHERE user_id = 1 FOR UPDATE;

UPDATE accounts SET balance = balance - 100 
WHERE user_id = 1;

COMMIT;
```

**分布式锁实现**：
```sql
-- 基于数据库的分布式锁
CREATE TABLE distributed_lock (
    lock_name VARCHAR(100) PRIMARY KEY,
    lock_holder VARCHAR(100),
    expires_at DATETIME
);

-- 获取锁
INSERT INTO distributed_lock (lock_name, lock_holder, expires_at)
VALUES ('order_lock', 'server_1', DATE_ADD(NOW(), INTERVAL 30 SECOND))
ON DUPLICATE KEY UPDATE 
    lock_holder = IF(expires_at < NOW(), VALUES(lock_holder), lock_holder),
    expires_at = IF(expires_at < NOW(), VALUES(expires_at), expires_at);
```

**限流方案**：
```sql
-- 基于数据库的限流
CREATE TABLE api_limits (
    api_key VARCHAR(100) PRIMARY KEY,
    request_count INT DEFAULT 0,
    last_reset TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 检查限流
START TRANSACTION;

SELECT request_count, last_reset 
FROM api_limits 
WHERE api_key = 'key123' FOR UPDATE;

-- 如果超过1分钟重置
UPDATE api_limits 
SET request_count = CASE 
    WHEN TIMESTAMPDIFF(SECOND, last_reset, NOW()) > 60 THEN 1
    ELSE request_count + 1
END,
last_reset = CASE 
    WHEN TIMESTAMPDIFF(SECOND, last_reset, NOW()) > 60 THEN NOW()
    ELSE last_reset
END
WHERE api_key = 'key123' AND request_count < 100;

COMMIT;
```
