# xxl-job 定时任务规范

## JobHandler 位置
`handler` 模块，`com.company.project.handler.job` 包

## 基础模板

```java
@Component
@Slf4j
public class OrderTimeoutCheckJob {
    
    @Autowired
    private OrderService orderService;
    
    /**
     * 超时订单自动取消
     * 调度：每5分钟执行，扫描15分钟前待支付订单
     * 路由策略：分片广播（多实例并行）
     */
    @XxlJob("orderTimeoutCheckJob")  // 唯一JobHandler
    public void execute() {
        // 1. 分片参数（多实例时用）
        int shardTotal = XxlJobHelper.getShardTotal();  // 总分片数
        int shardIndex = XxlJobHelper.getShardIndex();  // 当前分片
        
        log.info("分片：{}/{}", shardIndex, shardTotal);
        
        // 2. 查询（带上分片参数，只查当前分片数据）
        List<Long> orders = orderService.queryTimeoutOrders(
            LocalDateTime.now().minusMinutes(15),
            shardIndex, 
            shardTotal,
            100  // 单次处理100条
        );
        
        if (orders.isEmpty()) {
            XxlJobHelper.log("本分片无数据");  // 记录到xxl-job日志
            return;
        }
        
        // 3. 批量处理（逐条+异常隔离）
        int success = 0, fail = 0;
        for (Long id : orders) {
            try {
                orderService.cancelOrder(id, "SYSTEM_TIMEOUT");
                success++;
            } catch (Exception e) {
                log.error("取消失败，ID: {}", id, e);
                fail++;  // 继续处理其他
            }
        }
        
        // 4. 返回结果（xxl-job控制台可见）
        XxlJobHelper.handleSuccess(
            String.format("成功：%d，失败：%d", success, fail)
        );
    }
}
```

## 调度配置（xxl-job-admin后台）

| 配置项 | 建议值 | 说明 |
|--------|--------|------|
| 执行器 | `order-service-handler` | 与项目名一致 |
| JobHandler | `orderTimeoutCheckJob` | 与@XxlJob值一致 |
| 调度类型 | CRON | `0 */5 * * * ?`（每5分钟） |
| 路由策略 | 分片广播 | 多实例并行处理 |
| 调度过期 | 忽略 | 错过时间不补跑 |
| 超时时间 | 10分钟 | 超过强制终止 |
| 失败重试 | 3次 | 间隔30秒 |

## 设计原则
1. **幂等性**：同一任务多次执行结果一致（用订单ID去重）
2. **分片**：大数据量时用`shardIndex`拆分到不同实例
3. **流控**：单次处理不超过1000条，防止OOM
4. **异常隔离**：单条失败不中断批量（try-catch内）

