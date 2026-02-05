# 水平拆分策略详解

## 什么是水平拆分

水平拆分（Horizontal Sharding）是将同一个表的数据按照某种规则分散存储在多个数据库或表中，每个分片只包含部分数据。

## 常见分片策略

### 1. 范围分片（Range-based）

#### 按时间分片
```sql
-- 订单表按月分表
order_202401, order_202402, order_202403, ...
```

**优点**：
- 时间范围查询高效（查最近一个月只需扫一个表）
- 历史数据可以归档
- 分片数量和时间线性增长，易于理解

**缺点**：
- 可能数据倾斜（某段时间订单特别多）
- 跨时间范围查询需要扫多个表

**适用场景**：
- 日志表（log_202401, log_202402...）
- 订单表（按月或按季度）
- 流水表（transaction_202401...）

#### 按数值范围分片
```sql
-- 用户 ID 按范围分库
user_db_0: 0 - 1000000
user_db_1: 1000001 - 2000000
user_db_2: 2000001 - 3000000
```

**优点**：
- 实现简单
- 范围查询只需要扫部分分片

**缺点**：
- 数据分布不均（热点数据集中在某个范围）
- 扩容需要数据迁移

**适用场景**：
- 数据分布相对均匀
- 有明显的范围查询需求

---

### 2. 哈希分片（Hash-based）

#### 简单哈希
```java
// 计算分片
int shardCount = 16;
int shardIndex = Math.abs(userId.hashCode()) % shardCount;

// 路由到对应表
String tableName = "user_" + shardIndex;
```

**优点**：
- 数据均匀分布
- 集群负载均衡
- 实现简单

**缺点**：
- 不支持范围查询
- 扩容需要大量数据迁移
- 哈希算法变更影响所有数据

**适用场景**：
- 单条记录查询（按 ID）
- 点查询场景
- 数据分布均匀的场景

#### 一致性哈希
```java
// 使用一致性哈希解决扩容问题
ConsistentHash consistentHash = new ConsistentHash(virtualNodes);
String dbName = consistentHash.get(userId.toString());
```

**优点**：
- 扩容时只需要迁移部分数据
- 数据分布相对均匀
- 减少数据迁移量

**缺点**：
- 实现复杂
- 可能数据分布不均（取决于虚拟节点数量）

**适用场景**：
- 需要频繁扩容
- 对数据迁移敏感的场景

---

### 3. 地理位置/地域分片

```sql
-- 按省份分库
user_db_beijing, user_db_shanghai, user_db_guangdong, ...
```

**优点**：
- 数据就近访问，降低延迟
- 符合数据合规要求（数据本地化）
- 可以针对不同地域独立优化

**缺点**：
- 可能数据分布不均
- 跨地域查询性能差

**适用场景**：
- 有地域属性的业务
- 需要数据本地化合规
- 多地域部署

---

### 4. 复合分片键

```java
// 组合分片键：用户ID + 时间
String tableName = String.format("order_%d_%02d",
    userId % 10,  // 10 个库
    month % 12    // 每个库 12 张表（按月）
);
```

**优点**：
- 灵活支持多种查询模式
- 可以平衡多种查询需求

**缺点**：
- 实现复杂
- 路由逻辑复杂

**适用场景**：
- 需要支持多种查询模式
- 有多个查询维度

---

## 分片数量规划

### 估算公式

```
分片数量 = (预计 3 年后数据总量 / 单表容量上限) / 冗余系数

其中：
- 单表容量上限：建议 1000-2000 万行
- 冗余系数：1.5-2（预留扩容空间）
```

### 示例计算

假设：
- 当前订单数据：1000 万
- 年增长率：100%
- 预计 3 年后：8000 万
- 单表上限：2000 万
- 冗余系数：1.5

```
分片数量 = (8000 / 2000) / 1.5 = 2.6 ≈ 4 个分片
```

建议：**从 4 个分片开始，预留扩展到 8 个分片的空间**

## 分片算法选择

| 分片算法 | 数据均匀性 | 范围查询 | 扩容难度 | 实现复杂度 | 推荐指数 |
|---------|----------|---------|---------|----------|---------|
| 范围分片 | ⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| 简单哈希 | ⭐⭐⭐⭐⭐ | ⭐ | ⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 一致性哈希 | ⭐⭐⭐⭐ | ⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 地理位置分片 | ⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ |

## 分片键选择原则

### 1. 查询维度优先
选择最常用的查询条件作为分片键：
- 如果 90% 的查询是按用户 ID 查，就用 user_id
- 如果 90% 的查询是按时间查，就用时间

### 2. 数据均匀性
确保数据均匀分布在各个分片：
- 避免热点分片
- 定期检查数据分布情况

### 3. 扩容友好性
选择便于扩容的分片算法：
- 一致性哈希 > 简单哈希 > 范围分片

### 4. 业务连续性
分片键必须是业务上稳定、不变的：
- ❌ 不要用手机号（用户可能换号）
- ❌ 不要用部门 ID（用户可能换部门）
- ✅ 使用用户 ID
- ✅ 使用订单 ID

## 常见问题处理

### 1. 跨分片查询

**问题**：查询多个分片的数据
```java
// 需要查询多个用户的订单
List<Order> orders = orderMapper.findByUserIds(Arrays.asList(1, 2, 3, ...));
```

**方案**：
```java
// 并行查询
List<CompletableFuture<List<Order>>> futures = userIds.stream()
    .map(userId -> CompletableFuture.supplyAsync(() ->
        orderMapper.findByUserId(userId), executor))
    .collect(Collectors.toList());

// 合并结果
List<Order> allOrders = futures.stream()
    .flatMap(future -> future.join().stream())
    .collect(Collectors.toList());
```

### 2. 分片排序

**问题**：跨分片排序
```sql
-- 需要对所有分片的数据排序
SELECT * FROM order ORDER BY create_time LIMIT 10;
```

**方案**：
1. 每个分片排序取 TOP N
2. 应用层归并排序
3. 最终返回 TOP N

```java
// 每个分片查询 TOP 10
List<List<Order>> shardResults = shards.stream()
    .map(shard -> orderMapper.findTop10(shard))
    .collect(Collectors.toList());

// 归并排序
List<Order> top10 = shardResults.stream()
    .flatMap(List::stream)
    .sorted(Comparator.comparing(Order::getCreateTime))
    .limit(10)
    .collect(Collectors.toList());
```

### 3. 分片扩容

**问题**：从 4 个分片扩容到 8 个分片

**方案一：停机扩容**
```
1. 停止写入
2. 导出所有数据
3. 按新分片规则重新导入
4. 验证数据
5. 切换应用
6. 启动写入
```

**方案二：双写扩容（推荐）**
```
1. 应用支持双写（旧分片 + 新分片）
2. 数据校验（旧数据同步到新分片）
3. 灰度切读（逐步切换读流量）
4. 灰度切写（逐步切换写流量）
5. 下线旧分片
```

**方案三：在线迁移（使用一致性哈希）**
```
1. 新增分片节点
2. 调整一致性哈希环
3. 自动迁移部分数据到新节点
4. 逐步迁移完成
```

## 最佳实践

### 1. 从小规模开始
- 初期从 2-4 个分片开始
- 观察数据分布和访问模式
- 根据实际情况调整

### 2. 分片中间件选型
- **ShardingSphere**：功能全面，推荐复杂场景
- **MyCAT**：简单场景，部署方便
- **Vitess**：云原生场景
- **自研**：特殊需求定制

### 3. 路由规则集中管理
```java
public class ShardingRuleConfig {
    private static final Map<String, ShardingRule> RULES = new HashMap<>();

    static {
        RULES.put("order", new OrderShardingRule());
        RULES.put("user", new UserShardingRule());
        // ...
    }

    public static String getTableName(String table, Object shardingKey) {
        return RULES.get(table).shard(shardingKey);
    }
}
```

### 4. 监控指标
- 各分片的数据量
- 各分片的 QPS
- 慢查询分布
- 数据分布倾斜度

### 5. 预案准备
- 制定扩容预案
- 准备数据迁移工具
- 准备回滚方案
