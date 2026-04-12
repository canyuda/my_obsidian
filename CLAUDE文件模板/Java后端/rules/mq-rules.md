# RocketMQ 规范

## Topic 命名
`{系统}_{域}_{事件}`，Tag做二级过滤
- `trade_order_paid` (Tag: paid/cancelled)
- `inventory_deducted` (Tag: success/failed)

## 生产者（Service层）

```java
@Service
public class OrderEventPublisher {
    
    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    /**
     * 同步发送（重要事件）+ 业务KEYS
     */
    public void sendOrderPaid(OrderEntity order) {
        Message<OrderPaidEvent> message = MessageBuilder
            .withPayload(new OrderPaidEvent(order))
            .setHeader("KEYS", order.getOrderNo())  // 用于查询/幂等
            .build();
            
        SendResult result = rocketMQTemplate.syncSend(
            "trade_order_paid:paid",  // Topic:Tag
            message,
            3000,  // 超时3秒
            3      // 重试3次
        );
        
        if (!result.getSendStatus().equals(SendStatus.SEND_OK)) {
            throw new BizException(2005, "订单事件发送失败");
        }
    }
    
    /**
     * 延时消息（超时取消）Level 14=30分钟，Level 16=1小时
     */
    public void sendDelayCancel(OrderEntity order) {
        rocketMQTemplate.syncSendDelayTimeSeconds(
            "trade_order_cancel:timeout",
            MessageBuilder.withPayload(order.getId()).build(),
            900  // 15分钟 = 900秒
        );
    }
}
```

## 消费者（Handler层）

```java
@Component
@RocketMQMessageListener(
    topic = "trade_order_paid",
    consumerGroup = "order-service-paid-consumer",  // 唯一
    selectorExpression = "paid",  // 只消费paid Tag
    consumeMode = ConsumeMode.CONCURRENTLY  // 默认并发
)
public class OrderPaidMQListener implements RocketMQListener<OrderPaidEvent> {
    
    @Autowired
    private InventoryService inventoryService;
    @Autowired
    private RedissonClient redisson;

    @Override
    public void onMessage(OrderPaidEvent event) {
        String orderNo = event.getOrderNo();
        
        // 幂等：分布式锁 + 业务状态校验
        RLock lock = redisson.getLock("mq:consume:order:" + orderNo);
        try {
            if (!lock.tryLock(5, 30, TimeUnit.SECONDS)) {
                return;  // 获取失败，消息会重试
            }
            
            // 检查是否已处理（查DB状态）
            if (alreadyProcessed(orderNo)) {
                return;
            }
            
            inventoryService.deductStock(event);
            
        } catch (Exception e) {
            log.error("消费失败: {}", orderNo, e);
            throw new RuntimeException("库存扣减失败");  // 抛异常触发重试
        } finally {
            lock.unlock();
        }
    }
}
```

## 幂等策略（按优先级）
1. **数据库唯一索引**（推荐，consumerId+msgId）
2. **Redis SETNX**（`mq:consumed:{msgId}`，TTL=24h）
3. **业务状态机**（已处理状态直接返回）
