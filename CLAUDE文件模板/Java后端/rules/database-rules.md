# 数据库设计规范

## 命名规范

| 对象 | 规则 | 示例 |
|------|------|------|
| 表名 | `t_{业务域}_{实体}`，全小写下划线 | `t_order_item`, `t_user_profile` |
| 字段 | 下划线命名，避免关键字 | `user_name`, `order_status` |
| 主键 | `id`，BigInt，雪花算法 | 不推荐使用自增ID（分表冲突） |
| 索引 | `idx_{字段}` / `uk_{字段}`（唯一） | `idx_user_phone`, `uk_order_no` |
| 时间字段 | `create_time` / `update_time` | 必备，LocalDateTime映射 |

## 必备字段（所有表）

```sql
CREATE TABLE `t_example` (
  `id` BIGINT NOT NULL COMMENT '主键（雪花算法）',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `deleted` TINYINT NOT NULL DEFAULT 0 COMMENT '逻辑删除：0-正常 1-删除',
  PRIMARY KEY (`id`),
  KEY `idx_create_time` (`create_time`)  -- 时间范围查询优化
) ENGINE=InnoDB COMMENT='表示例';
```

## 字段类型选择

| 业务场景 | MySQL类型 | Java类型 | 说明 |
|----------|-----------|----------|------|
| 主键/外键 | `BIGINT(20)` | `Long` | 雪花算法ID，永不自增 |
| 金额/精确计算 | `DECIMAL(18,2)` | `BigDecimal` | 禁止用float/double |
| 状态/类型 | `TINYINT` | `Integer` | 0-255足够枚举 |
| 长文本 | `VARCHAR(2000)` | `String` | 超过2KB用TEXT |
| JSON扩展字段 | `JSON` | `String`/`Map` | MySQL 5.7+，可建虚拟列索引 |
| 时间戳 | `DATETIME(3)` | `LocalDateTime` | 毫秒精度，时区存UTC或统一+8 |

## 索引设计原则

```sql
-- 1. 单表索引不超过5个（写放大控制）
-- 2. 区分度高的在前（手机号 > 状态）
-- 3. 覆盖索引减少回表（SELECT字段全在索引内）

-- 正确示例：订单查询优化
CREATE TABLE `t_order` (
  `id` BIGINT NOT NULL,
  `user_id` BIGINT NOT NULL COMMENT '买家ID',
  `order_no` VARCHAR(32) NOT NULL COMMENT '订单号',
  `status` TINYINT NOT NULL COMMENT '0-待支付 1-已支付',
  `create_time` DATETIME NOT NULL,
  
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_order_no` (`order_no`),  -- 订单号唯一
  KEY `idx_user_status` (`user_id`, `status`, `create_time`)  -- 用户订单列表（覆盖索引）
) COMMENT='订单表';
```

## Entity映射规范（MyBatis-Plus）

```java
@Data
@TableName("t_order")
public class OrderEntity {
    @TableId(type = IdType.ASSIGN_ID)  // 雪花算法
    private Long id;
    
    @TableField("user_id")  // 显式映射（下划线转驼峰开启时可省略）
    private Long userId;
    
    @TableField("order_no")
    private String orderNo;
    
    @TableField("status")
    private Integer status;
    
    @TableField(value = "create_time", fill = FieldFill.INSERT)
    private LocalDateTime createTime;
    
    @TableField(value = "update_time", fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;
    
    @TableLogic  // 逻辑删除（与deleted字段绑定）
    @TableField("deleted")
    private Integer deleted;
}
```

## 分表预留（未来扩展）

```sql
-- 当前不分表，但字段设计预留分表能力：
-- 1. 主键不用自增（雪花算法）
-- 2. 分片键冗余存储（如user_id冗余到order表用于分表键）
-- 3. 全局唯一键用业务码+随机数（如订单号），不依赖数据库自增

-- 分表后查询原则：
-- - 带分片键（user_id）：直接路由到具体表
-- - 不带分片键（order_no）：扫描所有分表（Parallel Query或映射表）
```

## 变更规范

1. **加字段**：允许DDL，但必须加在表尾，禁止`AFTER xxx`指定位置（减少锁表时间）
2. **加索引**：使用`pt-online-schema-change`（Percona工具），避免主从延迟
3. **删字段**：先注释标记，双版本后物理删除（Apollo配置开关控制）
4. **SQL审查**：所有SQL必须EXPLAIN验证索引（重点看type列，避免ALL）

## MyBatis-Plus 自动填充配置

```java
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {
    @Override
    public void insertFill(MetaObject metaObject) {
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "deleted", Integer.class, 0);
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());
    }
}
```