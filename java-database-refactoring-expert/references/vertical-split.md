# 垂直拆分最佳实践

## 什么是垂直拆分

垂直拆分（Vertical Sharding）是按照业务模块将数据库拆分成多个独立的数据库，每个数据库包含相关的表集合。

## 适用场景

### 1. 业务模块清晰划分
- 系统包含多个独立的业务域（用户、订单、商品、支付等）
- 业务模块之间关联查询较少
- 不同模块有不同的访问模式和增长速度

### 2. 性能瓶颈
- 单库数据量过大，影响整体性能
- 某个业务模块的查询影响其他模块
- 需要针对不同模块进行不同的优化策略

### 3. 团队协作
- 不同团队负责不同业务模块
- 需要独立发布和部署
- 减少团队间的耦合

## 拆分原则

### 1. 业务高内聚
将业务上紧密相关的表放在同一个库中：
- 用户库：user、user_profile、user_auth
- 订单库：order、order_item、order_log
- 商品库：product、category、inventory
- 支付库：payment、payment_record、refund

### 2. 数据关联最小化
- 避免跨库 JOIN，如果必须 JOIN，考虑：
  - 冗余字段（将常用字段冗余到本地）
  - 应用层组装（分两次查询再组装）
  - 数据同步（通过消息队列异步同步）

### 3. 访问模式独立
不同业务模块的访问特征：
- 读写比例不同
- 峰值时段不同
- 数据增长速度不同

## 拆分步骤

### 步骤 1: 业务分析
1. 梳理所有业务模块
2. 分析表之间的关联关系
3. 统计各模块的访问量和数据量
4. 识别跨模块的依赖和调用

### 步骤 2: 拆分设计
```
原数据库：
┌─────────────────────────────┐
│      single_database        │
├─────────────────────────────┤
│ - user                      │
│ - user_profile              │
│ - order                     │
│ - order_item                │
│ - product                   │
│ - payment                   │
└─────────────────────────────┘

拆分后：
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│  user_db     │  │  order_db    │  │ product_db   │
├──────────────┤  ├──────────────┤  ├──────────────┤
│ - user       │  │ - order      │  │ - product    │
│ - user_profile│  │ - order_item │  │ - category   │
│ - user_auth  │  │ - order_log  │  │ - inventory  │
└──────────────┘  └──────────────┘  └──────────────┘
```

### 步骤 3: 应用改造
```java
// 拆分前：单数据源
@Autowired
private JdbcTemplate jdbcTemplate;

// 拆分后：多数据源
@Autowired
@Qualifier("userDataSource")
private JdbcTemplate userJdbcTemplate;

@Autowired
@Qualifier("orderDataSource")
private JdbcTemplate orderJdbcTemplate;

@Autowired
@Qualifier("productDataSource")
private JdbcTemplate productJdbcTemplate;
```

### 步骤 4: 数据迁移
1. 创建新的数据库和表结构
2. 按业务模块导出数据
3. 导入到对应的新库
4. 数据一致性校验
5. 应用切换和验证

## 跨库问题处理

### 1. 跨库 JOIN

**方案一：冗余字段**
```sql
-- order 表冗余用户姓名
CREATE TABLE order (
    id BIGINT PRIMARY KEY,
    user_id BIGINT,
    user_name VARCHAR(50),  -- 冗余字段
    ...
);
```

**方案二：应用层组装**
```java
// 先查订单
List<Order> orders = orderMapper.selectByUserId(userId);

// 再查用户
User user = userMapper.selectById(userId);

// 组装结果
orders.forEach(order -> order.setUserName(user.getName()));
```

**方案三：数据同步**
```java
// 通过消息队列同步数据
@KafkaListener(topics = "user-update")
public void handleUserUpdate(User user) {
    // 同步到订单库
    orderMapper.updateUserName(user.getId(), user.getName());
}
```

### 2. 分布式事务

**TCC 模式示例**：
```java
@Compensable
public void createOrder(Order order) {
    // Try: 预扣库存
    inventoryService.tryDeduct(order.getProductId(), order.getQuantity());

    // Confirm: 确认扣库存
    // Cancel: 取消扣库存（如果订单创建失败）
}
```

### 3. 全局表

对于一些数据量小且被多个业务模块访问的表，可以在每个库中都保存一份：
- 字典表（dict、dict_item）
- 配置表（config、system_config）
- 地区表（area、city、province）

## 注意事项

### 1. 拆分不是越早越好
- 初期业务量不大时，不要拆分
- 拆分会增加系统复杂度
- 拆分后运维成本增加

### 2. 预留扩展空间
- 拆分时要考虑未来 2-3 年的增长
- 预留水平和垂直二次拆分的空间

### 3. 监控和回滚
- 做好性能监控，对比拆分前后的效果
- 准备回滚方案，必要时可以回退

## 常见问题

### Q1: 拆分后如何保证数据一致性？
A: 采用最终一致性，通过消息队列、定时对账等方式保证数据最终一致。

### Q2: 跨库查询怎么处理？
A: 优先使用冗余字段避免跨库查询；必须跨库时，使用应用层组装或数据同步。

### Q3: 如何判断是否需要垂直拆分？
A: 当单库数据量超过 500 万，且业务模块清晰、跨模块关联少时，可以考虑垂直拆分。
