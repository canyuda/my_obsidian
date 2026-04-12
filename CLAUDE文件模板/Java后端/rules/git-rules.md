# Git 提交规范

## 提交信息格式（Angular规范）

```
<type>(<scope>): <subject>

<body>

<footer>
```

### 类型（Type）

| 标识 | 含义 | 示例 |
|------|------|------|
| `feat` | 新功能 | `feat(order): 增加订单取消接口` |
| `fix` | Bug修复 | `fix(mq): 修复消息重复消费问题` |
| `docs` | 文档更新 | `docs(api): 补充Swagger注解说明` |
| `style` | 代码格式 | `style(service): 统一代码缩进` |
| `refactor` | 重构 | `refactor(dao): 优化SQL查询性能` |
| `test` | 测试相关 | `test(unit): 补充订单服务单元测试` |
| `chore` | 构建/工具 | `chore(maven): 升级MyBatis-Plus版本` |
| `perf` | 性能优化 | `perf(cache): 优化Redis批量读取` |

### 作用域（Scope）

与项目模块对应：
- `api` / `dao` / `service` / `handler` / `web`
- `mq` / `job` / `cache` / `db`
- `common` / `config` / `utils`

## 提交示例

```bash
# 正确示例
git commit -m "feat(order): 实现订单超时自动取消功能

- 添加xxl-job定时任务扫描15分钟前订单
- 使用Redisson分布式锁防止并发取消
- 释放库存时发送RocketMQ消息

Closes #123"

# 修复示例
git commit -m "fix(redis): 修复缓存击穿导致的DB压力激增

问题：并发查询同一用户时，大量请求穿透到数据库
解决：添加Redisson分布式锁控制并发查询
影响：接口响应时间从2s降至50ms

Fixes #456"

# 简单提交（单文件/单功能）
git commit -m "docs(claude): 补充git提交规范文档"
```

## 禁止的提交信息

```bash
# 错误示例
git commit -m "update"                    # 无类型和范围
git commit -m "修改了一些代码"              # 中文描述模糊
git commit -m "feat: feat: 修复bug"       # 重复/格式错误
git commit -m "临时提交"                   # 提交未完成代码（除非WIP）
```

## 分支管理规范

### 分支命名

| 分支类型 | 命名格式 | 来源 | 合并目标 |
|----------|----------|------|----------|
| 功能开发 | `feat/{模块}-{描述}` | `develop` | `develop` |
| Bug修复 | `fix/{issue号}-{描述}` | `develop` | `develop` |
| 热修复 | `hotfix/{描述}` | `main` | `main` + `develop` |
| 发布 | `release/{版本号}` | `develop` | `main` |

```bash
# 示例
git checkout -b feat/order-timeout-cancel
git checkout -b fix/456-redis-concurrent
git checkout -b hotfix/fix-payment-bug
git checkout -b release/1.2.0
```

### 工作流程

```bash
# 1. 功能开发（每日推进）
git checkout develop
git pull origin develop
git checkout -b feat/user-cache-optimization

# 2. 频繁提交（原子性原则：一个提交只做一件事）
git add src/main/java/.../UserService.java
git commit -m "feat(service): 添加用户缓存查询方法"

git add src/main/java/.../UserController.java  
git commit -m "feat(web): 添加用户详情查询接口"

# 3. 推送前rebase（保持线性历史）
git fetch origin
git rebase origin/develop  # 解决冲突后再推送
git push origin feat/user-cache-optimization

# 4. 创建Merge Request（ develop ← feat/xxx ）
# 5. Code Review通过后Squash合并
```

## 提交原子性原则

**一个提交只包含一个逻辑变更**

```bash
# 错误：混合提交（功能+修复+格式调整）
git add .
git commit -m "提交代码"

# 正确：拆分提交
git add UserService.java OrderService.java
git commit -m "feat(service): 实现用户订单关联查询"

git add UserController.java
git commit -m "feat(web): 增加订单详情接口"

git add pom.xml
git commit -m "chore(deps): 升级RocketMQ客户端至5.1.4"

git add UserService.java  # 格式化调整
git commit -m "style(service): 优化UserService代码格式"
```

## 提交前自检（IDE集成）

```bash
# Maven项目提交前自动检查（推荐配置到IDE pre-commit）
./mvnw clean compile  # 编译通过
./mvnw test -Dtest=!*IntegrationTest  # 单元测试通过
./mvnw spotless:check  # 代码格式检查（如有配置）
```

## 特殊场景

### WIP（Work In Progress）提交

```bash
# 临时保存未完成工作（禁止推送到远程）
git commit -m "WIP: feat(order): 订单退款功能开发中 - 待联调支付接口"

# 完成后修正（变基修改提交信息）
git rebase -i HEAD~3
# 将pick改为reword，修改为规范提交信息
```

### 回滚提交

```bash
# 生产环境紧急回滚（保留历史）
git revert abc1234  # 自动生成"Revert xxx"提交

# 错误提交修正（未推送前）
git commit --amend -m "fix(cache): 正确修复Redis连接池配置"
```

## 关联Issue

```bash
# 自动关联GitLab/GitHub Issue（提交信息底部）
git commit -m "feat(mq): 实现订单状态变更消息推送

- 添加OrderStatusChangeEvent事件对象
- 配置RocketMQ生产者确认机制
- 添加消息发送失败重试逻辑

Relates to #123  # 相关
Closes #456      # 关闭
Fixes #789       # 修复并关闭
"
```

## 快速参考卡

```bash
# 新功能
git commit -m "feat(模块): 简短描述"

# 修复Bug  
git commit -m "fix(模块): 修复xx问题"

# 文档更新
git commit -m "docs(模块): 更新xx文档"

# 紧急热修复
git checkout main
git checkout -b hotfix/修复描述
# ...修复代码...
git commit -m "hotfix(模块): 修复xx问题"
git push origin hotfix/xx
# 合并到main和develop
```

**原则：提交信息是写给一个月后的自己和队友看的，必须一眼看懂这次做了什么。**
