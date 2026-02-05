# MySQL 性能优化指南

## 慢查询分析

### 1. 开启慢查询日志

```sql
-- 查看慢查询配置
SHOW VARIABLES LIKE 'slow_query%';
SHOW VARIABLES LIKE 'long_query_time';

-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1; -- 超过 1 秒的查询记录

-- 查看 slow_query_log_file 文件位置
SHOW VARIABLES LIKE 'slow_query_log_file';
```

### 2. 分析慢查询日志

```bash
# 使用 mysqldumpslow 工具
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# -s t: 按查询时间排序
# -t 10: 显示前 10 条
```

### 3. 使用 EXPLAIN 分析

```sql
-- 分析 SQL 执行计划
EXPLAIN SELECT * FROM order WHERE user_id = 123;

EXPLAIN 的关键指标：
┌─────────────┬────────────────────────────────────────┐
│ 列名         │ 说明                                   │
├─────────────┼────────────────────────────────────────┤
│ type        │ 访问类型（性能从好到坏）                │
│             │ system > const > eq_ref > ref > range  │
│             │ > index > ALL                          │
├─────────────┼────────────────────────────────────────┤
│ key         │ 实际使用的索引                         │
├─────────────┼────────────────────────────────────────┤
│ rows        │ 预估扫描的行数                         │
├─────────────┼────────────────────────────────────────┤
│ Extra       │ 额外信息                               │
│             │ Using index: 索引覆盖                 │
│             │ Using filesort: 需要文件排序           │
│             │ Using temporary: 使用临时表           │
└─────────────┴────────────────────────────────────────┘
```

## 索引优化

### 1. 索引设计原则

**适合建索引的列**：
- 经常用于 WHERE 条件的列
- 经常用于 JOIN 的列
- 经常用于 ORDER BY 的列
- 经常用于 GROUP BY 的列

**不适合建索引的列**：
- 频繁更新的列
- 数据区分度低的列（如性别）
- TEXT/BLOB 等大字段
- 很少被查询的列

### 2. 索引类型选择

```sql
-- 主键索引（聚簇索引）
PRIMARY KEY (id)

-- 唯一索引
UNIQUE KEY uk_email (email)

-- 普通索引
KEY idx_user_id (user_id)

-- 组合索引（注意最左前缀原则）
KEY idx_user_status_time (user_id, status, create_time)

-- 组合索引生效场景：
✅ WHERE user_id = 1
✅ WHERE user_id = 1 AND status = 2
✅ WHERE user_id = 1 AND status = 2 AND create_time > '2024-01-01'
✅ ORDER BY user_id, status
❌ WHERE status = 2  -- 跳过了 user_id
❌ WHERE create_time > '2024-01-01'  -- 跳过了前面两列

-- 覆盖索引（包含所有查询字段）
KEY idx_user_id_status_time (user_id, status, create_time, id)
-- 这样查询就不需要回表了
EXPLAIN SELECT id, status, create_time
FROM order WHERE user_id = 1;
-- Extra: Using index
```

### 3. 索引优化案例

**案例 1: 避免索引失效**
```sql
-- ❌ 对索引列进行函数运算
SELECT * FROM user WHERE YEAR(create_time) = 2024;

-- ✅ 改写为范围查询
SELECT * FROM user WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';

-- ❌ 隐式类型转换
SELECT * FROM user WHERE phone = 13800138000;  -- phone 是 VARCHAR

-- ✅ 显式类型转换
SELECT * FROM user WHERE phone = '13800138000';
```

**案例 2: 优化 LIKE 查询**
```sql
-- ❌ 前缀模糊匹配无法使用索引
SELECT * FROM user WHERE name LIKE '%张%';

-- ✅ 后缀模糊匹配可以使用索引
SELECT * FROM user WHERE name LIKE '张%';

-- 如果必须前缀模糊，考虑全文索引
ALTER TABLE user ADD FULLTEXT INDEX ft_name (name);
SELECT * FROM user WHERE MATCH(name) AGAINST('张' IN BOOLEAN MODE);
```

**案例 3: 优化 OR 查询**
```sql
-- ❌ 不同列的 OR 无法使用索引
SELECT * FROM user WHERE user_id = 1 OR email = 'test@example.com';

-- ✅ 改写为 UNION
SELECT * FROM user WHERE user_id = 1
UNION
SELECT * FROM user WHERE email = 'test@example.com';
```

## 表结构优化

### 1. 数据类型选择

```sql
-- 整数类型选择
TINYINT:    1 字节, -128 ~ 127
SMALLINT:   2 字节, -32768 ~ 32767
INT:        4 字节, -21亿 ~ 21亿  -- 最常用
BIGINT:     8 字节, 超大整数

-- 字符串类型选择
CHAR(n):    固定长度, 速度快, 最大 255
VARCHAR(n): 变长长度, 省空间, 最大 65535
TEXT:       大文本, 最大 65535

-- 示例：手机号用 VARCHAR(11) 而不是 BIGINT
phone VARCHAR(11)  -- 考虑前导 0 和特殊字符
```

### 2. 字段长度优化

```sql
-- ❌ 过长的 VARCHAR
name VARCHAR(255)  -- 实际名字很少超过 20 个字

-- ✅ 合理的长度
name VARCHAR(50)   -- 节省空间，提高缓存命中率
```

### 3. NULL 值处理

```sql
-- ❌ 允许 NULL
age INT DEFAULT NULL

-- ✅ 设置默认值
age INT NOT NULL DEFAULT 0

-- 原因：
-- 1. NULL 值占用额外空间
-- 2. 索引统计更复杂
-- 3. 查询时需要特殊处理 IS NULL
```

### 4. 范式化 vs 反范式化

**范式化（第三范式）**：
- 优点：减少数据冗余，避免更新异常
- 缺点：需要多表 JOIN

**反范式化**：
- 优点：减少 JOIN，提高查询性能
- 缺点：数据冗余，需要同步

**建议**：核心业务用范式化，查询密集场景适当反范式化

```sql
-- 范式化
CREATE TABLE user (
    id BIGINT PRIMARY KEY,
    name VARCHAR(50)
);

CREATE TABLE order (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    amount DECIMAL(10,2)
);

-- 反范式化（冗余用户名）
CREATE TABLE order (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    user_name VARCHAR(50),  -- 冗余字段
    amount DECIMAL(10,2)
);
```

## 查询优化

### 1. 避免 SELECT *

```sql
-- ❌ 查询所有列
SELECT * FROM user WHERE id = 1;

-- ✅ 只查询需要的列
SELECT id, name, email FROM user WHERE id = 1;

-- 原因：
-- 1. 减少网络传输
-- 2. 可能使用覆盖索引
-- 3. 减少内存占用
```

### 2. 分页优化

```sql
-- ❌ 深分页性能差
SELECT * FROM order ORDER BY id LIMIT 1000000, 10;

-- ✅ 使用上次查询的最大 ID
SELECT * FROM order WHERE id > 1000000 ORDER BY id LIMIT 10;

-- ✅ 如果必须用 OFFSET，先查询 ID
SELECT * FROM order WHERE id IN (
    SELECT id FROM order ORDER BY id LIMIT 1000000, 10
);
```

### 3. 子查询优化

```sql
-- ❌ 子查询可能导致多次扫描
SELECT * FROM user WHERE id IN (
    SELECT user_id FROM order WHERE amount > 1000
);

-- ✅ 改写为 JOIN
SELECT u.* FROM user u
INNER JOIN order o ON u.id = o.user_id
WHERE o.amount > 1000;
```

### 4. 批量插入

```sql
-- ❌ 单条插入
INSERT INTO user (name, email) VALUES ('test1', 'test1@example.com');
INSERT INTO user (name, email) VALUES ('test2', 'test2@example.com');
INSERT INTO user (name, email) VALUES ('test3', 'test3@example.com');

-- ✅ 批量插入
INSERT INTO user (name, email) VALUES
    ('test1', 'test1@example.com'),
    ('test2', 'test2@example.com'),
    ('test3', 'test3@example.com');

-- 或者使用 LOAD DATA INFILE
LOAD DATA INFILE '/tmp/users.csv'
INTO TABLE user
FIELDS TERMINATED BY ','
(name, email);
```

## 配置优化

### 1. InnoDB 缓冲池

```ini
# my.cnf
# InnoDB 缓冲池大小（建议物理内存的 50-70%）
innodb_buffer_pool_size = 4G

# 缓冲池实例数（建议等于 CPU 核数）
innodb_buffer_pool_instances = 8
```

### 2. 连接数配置

```ini
# 最大连接数
max_connections = 500

# 连接超时
wait_timeout = 28800  -- 8 小时
interactive_timeout = 28800
```

### 3. 日志配置

```ini
# Redo Log 大小
innodb_log_file_size = 512M
innodb_log_files_in_group = 2

# 刷盘策略
# 0: 每秒刷盘，可能丢失 1 秒数据
# 1: 每次事务刷盘，最安全
# 2: 每次事务写缓存，每秒刷盘
innodb_flush_log_at_trx_commit = 1
```

## 监控指标

### 1. 关键指标

```sql
-- 查看 QPS
SHOW GLOBAL STATUS LIKE 'Questions';
SHOW GLOBAL STATUS LIKE 'Uptime';

-- 查看 TPS
SHOW GLOBAL STATUS LIKE 'Com_commit';
SHOW GLOBAL STATUS LIKE 'Com_rollback';

-- 查看连接数
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Max_used_connections';

-- 查看缓冲池命中率
SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_read%';
-- 命中率 = 1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)
-- 应该 > 95%
```

### 2. 慢查询监控

```sql
-- 查看慢查询数量
SHOW GLOBAL STATUS LIKE 'Slow_queries';

-- 开启慢查询日志
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
```

### 3. 锁等待监控

```sql
-- 查看锁等待
SELECT * FROM information_schema.INNODB_LOCKS;
SELECT * FROM information_schema.INNODB_LOCK_WAITS;

-- 查看事务
SELECT * FROM information_schema.INNODB_TRX;
```

## 常见性能问题

### 1. 连接池耗尽

**现象**：应用报 "Too many connections"

**原因**：
- 连接没有正确释放
- 连接池配置太小
- 慢查询导致连接长时间占用

**解决**：
```java
// Druid 配置示例
spring.datasource.druid.initial-size=10
spring.datasource.druid.max-active=100
spring.datasource.druid.max-wait=60000
spring.datasource.druid.test-while-idle=true
spring.datasource.druid.validation-query=SELECT 1
```

### 2. 磁盘 IO 高

**现象**：iowait 持续高

**原因**：
- 没有命中索引，全表扫描
- 缓冲池太小
- 大量排序操作

**解决**：
- 优化 SQL，添加索引
- 增加 innodb_buffer_pool_size
- 调整排序缓冲区 sort_buffer_size

### 3. CPU 高

**现象**：CPU 使用率高

**原因**：
- 大量复杂的 JOIN
- 排序和分组
- 函数运算

**解决**：
- 优化 SQL，减少复杂度
- 使用覆盖索引
- 将计算移到应用层
