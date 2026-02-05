# 数据迁移方案参考

## 迁移方案对比

| 方案 | 停机时间 | 复杂度 | 风险 | 适用场景 |
|------|---------|-------|------|---------|
| 停机迁移 | 需要 | 低 | 低 | 业务可中断、数据量小 |
| 双写迁移 | 不需要 | 高 | 中 | 业务不能中断、在线迁移 |
| 增量同步 | 不需要 | 中 | 中 | 大数据量、有限窗口期 |

## 方案一：停机迁移

### 适用场景
- 业务可以接受短暂停机（如凌晨 2-4 点）
- 数据量较小（< 100GB）
- 迁移时间窗口充足

### 迁移步骤

```
┌────────────────────────────────────────────────────────────┐
│                    停机迁移流程                             │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  1. 停止应用写入                                           │
│     │                                                      │
│     ▼                                                      │
│  2. 数据全量导出（mysqldump）                              │
│     │                                                      │
│     ▼                                                      │
│  3. 数据导入到新库                                         │
│     │                                                      │
│     ▼                                                      │
│  4. 数据一致性校验                                         │
│     │                                                      │
│     ▼                                                      │
│  5. 应用切换配置                                           │
│     │                                                      │
│     ▼                                                      │
│  6. 启动应用验证                                           │
│     │                                                      │
│     ▼                                                      │
│  7. 确认正常后下线旧库                                     │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### 实施脚本

```bash
#!/bin/bash
# 停机迁移脚本

# 配置
OLD_HOST="old-db.example.com"
NEW_HOST="new-db.example.com"
DATABASE="mydb"
BACKUP_DIR="/backup/$(date +%Y%m%d)"

# 1. 停止应用
echo "停止应用..."
systemctl stop myapp

# 2. 全量导出
echo "导出数据..."
mkdir -p $BACKUP_DIR
mysqldump -h $OLD_HOST -u root -p$OLD_PASS \
    --single-transaction \
    --routines \
    --triggers \
    --events \
    $DATABASE > $BACKUP_DIR/dump.sql

# 3. 导入到新库
echo "导入数据..."
mysql -h $NEW_HOST -u root -p$NEW_PASS $DATABASE < $BACKUP_DIR/dump.sql

# 4. 数据校验
echo "数据校验..."
OLD_COUNT=$(mysql -h $OLD_HOST -u root -p$OLD_PASS -N \
    -e "SELECT SUM(TABLE_ROWS) FROM information_schema.TABLES WHERE TABLE_SCHEMA='$DATABASE'")
NEW_COUNT=$(mysql -h $NEW_HOST -u root -p$NEW_PASS -N \
    -e "SELECT SUM(TABLE_ROWS) FROM information_schema.TABLES WHERE TABLE_SCHEMA='$DATABASE'")

echo "旧库行数: $OLD_COUNT, 新库行数: $NEW_COUNT"

if [ "$OLD_COUNT" == "$NEW_COUNT" ]; then
    echo "数据校验通过"

    # 5. 切换应用配置
    echo "切换应用配置..."
    # 更新数据库连接配置

    # 6. 启动应用
    echo "启动应用..."
    systemctl start myapp

    # 7. 验证
    sleep 10
    curl http://localhost:8080/health

    echo "迁移完成！"
else
    echo "数据校验失败，请检查！"
    exit 1
fi
```

### 注意事项

1. **提前演练**：在测试环境完整演练一遍
2. **时间估算**：留出 30% 的缓冲时间
3. **回滚准备**：准备好快速回滚脚本
4. **监控准备**：迁移后密切监控

---

## 方案二：双写迁移（推荐）

### 适用场景
- 业务不能停机
- 数据一致性要求高
- 可接受短期性能损耗

### 迁移步骤

```
┌─────────────────────────────────────────────────────────────────────┐
│                        双写迁移流程                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  阶段 1: 应用改造（支持双写）                                        │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │                                                         │        │
│  │   应用                                                   │        │
│  │    │                                                     │        │
│  │    ├─ 写入旧库（主写）                                   │        │
│  │    │                                                     │        │
│  │    └─ 写入新库（副写，异步）                             │        │
│  │                                                         │        │
│  └─────────────────────────────────────────────────────────┘        │
│                                                                     │
│  阶段 2: 数据校验                                                    │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │   全量数据校验（对比新旧库数据）                          │        │
│  │   增量数据校验（实时对比写入数据）                        │        │
│  └─────────────────────────────────────────────────────────┘        │
│                                                                     │
│  阶段 3: 灰度切读                                                   │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │   5% 读流量 → 新库                                       │        │
│  │   20% 读流量 → 新库                                      │        │
│  │   50% 读流量 → 新库                                      │        │
│  │   100% 读流量 → 新库                                     │        │
│  └─────────────────────────────────────────────────────────┘        │
│                                                                     │
│  阶段 4: 灰度切写                                                   │
│  ┌─────────────────────────────────────────────────────────┐        │
│  │   5% 写流量 → 新库（主写）                               │        │
│  │   20% 写流量 → 新库                                      │        │
│  │   50% 写流量 → 新库                                      │        │
│  │   100% 写流量 → 新库                                     │        │
│  └─────────────────────────────────────────────────────────┘        │
│                                                                     │
│  阶段 5: 下线旧库                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 应用改造示例

```java
@Service
public class OrderService {

    @Autowired
    private OrderMapper oldOrderMapper;

    @Autowired
    private OrderMapper newOrderMapper;

    @Autowired
    private AsyncExecutor asyncExecutor;

    // 配置：新库读写开关
    @Value("${migration.read.new:false}")
    private boolean readFromNew;

    @Value("${migration.write.new:false}")
    private boolean writeToNew;

    /**
     * 创建订单（双写）
     */
    @Transactional
    public void createOrder(Order order) {
        // 主写旧库
        oldOrderMapper.insert(order);

        // 异步写新库（如果开启）
        if (writeToNew || migrationPhase != MigrationPhase.OLD_ONLY) {
            asyncExecutor.execute(() -> {
                try {
                    newOrderMapper.insert(order);
                } catch (Exception e) {
                    // 记录失败日志，后续补偿
                    log.error("写新库失败", e);
                    migrationLogService.logWriteFailure(order);
                }
            });
        }
    }

    /**
     * 查询订单（根据配置路由）
     */
    public Order getOrderById(Long orderId) {
        if (readFromNew) {
            return newOrderMapper.selectById(orderId);
        } else {
            return oldOrderMapper.selectById(orderId);
        }
    }
}
```

### 数据校验

```java
@Service
public class DataValidationService {

    @Autowired
    private OrderMapper oldOrderMapper;

    @Autowired
    private OrderMapper newOrderMapper;

    /**
     * 全量数据校验
     */
    public void validateAllData() {
        long maxId = oldOrderMapper.selectMaxId();
        long batchSize = 10000;

        for (long i = 0; i < maxId; i += batchSize) {
            List<Long> ids = oldOrderMapper.selectIdsInRange(i, i + batchSize);

            for (Long id : ids) {
                Order oldOrder = oldOrderMapper.selectById(id);
                Order newOrder = newOrderMapper.selectById(id);

                if (!isEqual(oldOrder, newOrder)) {
                    log.error("数据不一致: id={}", id);
                    // 记录不一致的数据
                    dataDiffService.recordDiff(id, oldOrder, newOrder);
                }
            }
        }
    }

    /**
     * 增量数据校验（实时）
     */
    @Scheduled(fixedRate = 60000) // 每分钟
    public void validateIncrementalData() {
        // 校验最近 10 分钟写入的数据
        LocalDateTime since = LocalDateTime.now().minusMinutes(10);

        List<Order> oldOrders = oldOrderMapper.selectByTimeRange(since, LocalDateTime.now());

        for (Order oldOrder : oldOrders) {
            Order newOrder = newOrderMapper.selectById(oldOrder.getId());

            if (newOrder == null) {
                log.error("新库缺失数据: id={}", oldOrder.getId());
                // 补偿：写入新库
                newOrderMapper.insert(oldOrder);
            } else if (!isEqual(oldOrder, newOrder)) {
                log.error("数据不一致: id={}", oldOrder.getId());
                // 补偿：更新新库
                newOrderMapper.updateById(oldOrder);
            }
        }
    }
}
```

### 灰度配置

```yaml
# application.yml
migration:
  # 阶段：OLD_ONLY, DUAL_WRITE, SWITCH_READ, SWITCH_WRITE, NEW_ONLY
  phase: DUAL_WRITE

  # 读流量路由百分比
  read:
    new-percent: 0  # 0-100

  # 写流量路由百分比
  write:
    new-percent: 0  # 0-100
```

### 切换控制器

```java
@RestController
@RequestMapping("/migration")
public class MigrationController {

    @Value("${migration.read.new-percent:0}")
    private int readNewPercent;

    @Value("${migration.write.new-percent:0}")
    private int writeNewPercent;

    /**
     * 调整读流量百分比
     */
    @PostMapping("/read/percent")
    public String setReadPercent(@RequestParam int percent) {
        if (percent < 0 || percent > 100) {
            return "百分比必须在 0-100 之间";
        }

        readNewPercent = percent;
        log.info("读流量切换到新库: {}%", percent);

        return "读流量已调整为 " + percent + "%";
    }

    /**
     * 调整写流量百分比
     */
    @PostMapping("/write/percent")
    public String setWritePercent(@RequestParam int percent) {
        if (percent < 0 || percent > 100) {
            return "百分比必须在 0-100 之间";
        }

        writeNewPercent = percent;
        log.info("写流量切换到新库: {}%", percent);

        return "写流量已调整为 " + percent + "%";
    }

    /**
     * 紧急回滚
     */
    @PostMapping("/rollback")
    public String rollback() {
        readNewPercent = 0;
        writeNewPercent = 0;
        log.warn("紧急回滚，所有流量切回旧库");

        return "已紧急回滚";
    }
}
```

---

## 方案三：基于 Binlog 增量同步

### 适用场景
- 大数据量（> 500GB）
- 停机窗口有限
- 需要准实时同步

### 技术方案

**工具选择**：
- **Canal**：阿里开源，推荐
- **Maxwell**：轻量级
- **Debezium**：Kafka Connect
- **自研**：基于 Binlog 解析

### 使用 Canal 同步

```bash
# 1. 开启 MySQL Binlog
[mysqld]
server-id=1
log-bin=mysql-bin
binlog_format=ROW
binlog_row_image=FULL

# 2. 配置 Canal
canal.destinations=example
canal.destinations.example.position=journal

canal.instance.master.address=old-db.example.com:3306
canal.instance.master.journal.name=
canal.instance.master.journal.pos=
canal.instance.master.timestamp=
canal.instance.master.gtid=

canal.instance.dbUsername=canal
canal.instance.dbPassword=canal
canal.instance.connectionCharset=UTF-8
canal.instance.filter.regex=.*\\..*

# 3. 启动 Canal
sh bin/startup.sh

# 4. 数据同步客户端
public class CanalClient {

    public void sync() {
        CanalConnector connector = CanalConnectors.newSingleConnector(
            new InetSocketAddress("localhost", 11111),
            "example", "", ""
        );

        try {
            connector.connect();
            connector.subscribe(".*\\..*");
            connector.rollback();

            while (true) {
                Message message = connector.getWithoutAck(100);
                long batchId = message.getId();
                int size = message.getEntries().size();

                if (batchId != -1 && size > 0) {
                    for (Entry entry : message.getEntries()) {
                        if (entry.getEntryType() == EntryType.ROWDATA) {
                            RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());

                            // 处理数据变更
                            handleRowChange(rowChange);
                        }
                    }
                }

                connector.ack(batchId);
            }
        } finally {
            connector.disconnect();
        }
    }

    private void handleRowChange(RowChange rowChange) {
        for (RowData rowData : rowChange.getRowDatasList()) {
            switch (rowChange.getEventType()) {
                case INSERT:
                    handleInsert(rowData.getAfterColumnsList());
                    break;
                case UPDATE:
                    handleUpdate(rowData.getBeforeColumnsList(), rowData.getAfterColumnsList());
                    break;
                case DELETE:
                    handleDelete(rowData.getBeforeColumnsList());
                    break;
            }
        }
    }
}
```

---

## 数据一致性保障

### 1. 对账机制

```java
@Service
public class ReconciliationService {

    /**
     * 每日对账
     */
    @Scheduled(cron = "0 0 2 * * ?") // 每天凌晨 2 点
    public void dailyReconciliation() {
        LocalDate yesterday = LocalDate.now().minusDays(1);

        // 对账订单表
        reconcileOrder(yesterday);

        // 对账支付表
        reconcilePayment(yesterday);
    }

    private void reconcileOrder(LocalDate date) {
        LocalDateTime start = date.atStartOfDay();
        LocalDateTime end = date.plusDays(1).atStartOfDay();

        // 统计旧库
        Long oldCount = oldOrderMapper.countByTimeRange(start, end);
        BigDecimal oldAmount = oldOrderMapper.sumAmountByTimeRange(start, end);

        // 统计新库
        Long newCount = newOrderMapper.countByTimeRange(start, end);
        BigDecimal newAmount = newOrderMapper.sumAmountByTimeRange(start, end);

        // 对比
        if (!oldCount.equals(newCount) || oldAmount.compareTo(newAmount) != 0) {
            log.error("订单对账失败: date={}, oldCount={}, newCount={}, oldAmount={}, newAmount={}",
                date, oldCount, newCount, oldAmount, newAmount);

            // 发送告警
            alertService.sendAlert("订单对账失败");

            // 详细对比
            detailReconciliation(start, end);
        } else {
            log.info("订单对账成功: date={}, count={}, amount={}", date, newCount, newAmount);
        }
    }

    /**
     * 详细对账（逐条对比）
     */
    private void detailReconciliation(LocalDateTime start, LocalDateTime end) {
        List<Order> oldOrders = oldOrderMapper.selectByTimeRange(start, end);

        for (Order oldOrder : oldOrders) {
            Order newOrder = newOrderMapper.selectById(oldOrder.getId());

            if (newOrder == null) {
                log.error("新库缺失订单: id={}", oldOrder.getId());
                // 补偿
                newOrderMapper.insert(oldOrder);
            } else if (!isEqual(oldOrder, newOrder)) {
                log.error("订单数据不一致: id={}", oldOrder.getId());
                // 补偿
                newOrderMapper.updateById(oldOrder);
            }
        }
    }
}
```

### 2. 幂等性设计

```java
@Service
public class IdempotentService {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    /**
     * 幂等写入
     */
    public void writeWithIdempotent(Order order) {
        String key = "order:write:" + order.getId();

        // 尝试加锁
        Boolean locked = redisTemplate.opsForValue()
            .setIfAbsent(key, "1", Duration.ofMinutes(10));

        if (Boolean.TRUE.equals(locked)) {
            try {
                // 检查是否已存在
                Order existing = newOrderMapper.selectById(order.getId());
                if (existing == null) {
                    newOrderMapper.insert(order);
                } else {
                    // 已存在，更新
                    newOrderMapper.updateById(order);
                }
            } finally {
                redisTemplate.delete(key);
            }
        } else {
            log.warn("重复写入: id={}", order.getId());
        }
    }
}
```

---

## 回滚方案

### 回滚决策条件

遇到以下情况立即回滚：
1. 错误率超过 1%
2. P99 响应时间增加超过 50%
3. 数据一致性校验失败
4. 应用报错率异常上升

### 回滚脚本

```bash
#!/bin/bash
# 快速回滚脚本

echo "开始回滚..."

# 1. 切换配置
echo "切换配置到旧库..."
cp /etc/myapp/old-db.yml /etc/myapp/db.yml

# 2. 重启应用
echo "重启应用..."
systemctl restart myapp

# 3. 验证
echo "验证应用..."
sleep 10
for i in {1..10}; do
    curl -f http://localhost:8080/health || {
        echo "健康检查失败，回滚失败！"
        exit 1
    }
done

echo "回滚完成！"
```

---

## 监控指标

迁移过程中需要监控的关键指标：

| 指标 | 说明 | 告警阈值 |
|------|------|---------|
| 数据一致率 | 新旧库数据一致的比例 | < 99.9% |
| 写入延迟 | 新库写入延迟 | > 100ms |
| 查询响应时间 | 接口响应时间 | 增加 50% |
| 错误率 | 请求错误率 | > 0.1% |
| 慢查询数量 | 慢查询数量 | 增加 20% |
| 连接池使用率 | 数据库连接池使用率 | > 80% |
