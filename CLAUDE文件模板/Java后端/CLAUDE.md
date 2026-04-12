# `[项目名]`- 项目导航

## 1. 架构速查

### 模块结构
```
api → dao → service → handler/web
```
- **api**: DTO、Feign、枚举常量（无依赖）
- **dao**: Entity、Mapper、XML（依赖api）
- **service**: 业务逻辑、事务、BizException（依赖api/dao）
- **handler**: MQ监听、JobHandler（依赖service）
- **web**: Controller、启动类（依赖所有）

### 技术栈
JDK17 | SpringBoot 3.2 | Maven | Nacos | Apollo | RocketMQ | Redisson | xxl-job | Sa-Token | MyBatis-Plus | Swagger

## 2. 快速决策表

| 我要做... | 参考文件 | 关键类 |
|-----------|----------|--------|
| 写业务代码（Service/Controller） | `/rules/coding-rules.md` | `BizException` |
| 发/收MQ消息 | `/rules/mq-rules.md` | `RocketMQTemplate` |
| 用缓存或分布式锁 | `/rules/cache-rules.md` | `RedissonClient` |
| 写定时任务 | `/rules/job-rules.md` | `@XxlJob` |
| 写接口文档 | `/rules/api-rules.md` | `@Operation` |

## 3. 命名铁律（无需查阅子文件）

```java
// Controller: XxxController, POST /create, GET /detail/{id}
// Service: XxxService/XxxServiceImpl, 方法: createOrder()
// Mapper: XxxMapper extends BaseMapper<XxxEntity>
// Entity: XxxEntity, @TableName("t_xxx")
// MQ Listener: XxxMQListener implements RocketMQListener<DTO>
// Job: XxxJob, @XxlJob("xxxJob")
```

## 4. AI协作指令模板

### 生成代码
```
任务：实现[功能]
模块：[service/web/handler]
输入：[参数]
规则：[必须/禁止事项]
输出：设计思路(100字) + 代码 + 自测用例
```

### 审查代码
```
审查以下代码的[异常/事务/并发]问题：
[代码片段]
关注：是否用BizException？是否有分布式锁？事务边界？
输出：CRITICAL/WARNING/INFO分级
```

### 排查故障
```
现象：[具体现象]
已有信息：[日志/指标/变更]
排查方向：数据库慢查询？MQ堆积？Redis热点？
```

## 5. 其他规范

- 编码规范见 `/rules/coding-rules.md`
- 缓存规范见 `/rules/cache-rules.md`
- api规范见 `/rules/api-rules.md`
- 消息规范见 `/rules/mq-rules.md`
- 定时任务规范见 `/rules/job-rules.md`
- 数据库设计规范见 `/rules/database-rules.md`
- 测试规范见 `/rules/testing-rules.md`
- 代码提交规范见 `/rules/git-rules.md`
