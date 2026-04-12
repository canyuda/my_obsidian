# 编码规范

## 包结构
```
com.company.project.{模块}.{分层}.{业务域}
```
- api: `dto.xx` / `feign.xx` / `enums.xx`
- dao: `entity.xx` / `mapper.xx`
- service: `xx` / `impl.xx` / `exception`
- handler: `rocketmq.xx` / `job.xx`
- web: `controller.xx` / `config` / `advice`

## 注释规范（强制中文）

```java
/**
 * 创建订单并预占库存
 * 
 * 流程：1.校验库存 2.生成订单号(雪花算法) 3.预占库存 4.发送延时消息(15分钟)
 * 异常：库存不足抛InsufficientStockException，用户状态异常抛UserStatusException
 * 事务：全部写操作@Transactional(rollbackFor = Exception.class)
 * 
 * @param dto 订单创建参数
 * @return OrderCreateVO 订单号、支付金额、截止时间
 * @throws BizException 业务校验失败
 */
public OrderCreateVO createOrder(OrderCreateDTO dto) { }
```

## 数据访问规范

```java
// 查询：用MP LambdaQuery
userMapper.selectOne(Wrappers.<UserEntity>lambdaQuery()
    .eq(UserEntity::getPhone, phone)
    .eq(UserEntity::getDeleted, 0)
    .last("LIMIT 1"));

// 分页：IPage + 自动COUNT
Page<UserEntity> page = userMapper.selectPage(
    new Page<>(dto.getPageNum(), dto.getPageSize()),
    buildWrapper(dto)
);

// 更新：先更新DB再删缓存（延迟双删）
```

## 异常体系

```java
// 继承BizException（ErrorCodeEnum定义异常码）
public class OrderNotFoundException extends BizException {
    public OrderNotFoundException(Long id) {
        super(2001, "订单不存在，ID: " + id);  // 2xxx=订单域
    }
}

// Web层全局拦截（GlobalExceptionHandler）
// 业务异常WARN日志，系统异常ERROR日志+打印堆栈
```

## 依赖注入
```java
@Autowired  // 字段注入（项目内统一）
// 或构造器注入（Lombok @RequiredArgsConstructor）
// 禁止：@Resource，保持统一
```