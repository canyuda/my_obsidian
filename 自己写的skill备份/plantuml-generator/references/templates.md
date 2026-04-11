# PlantUML Syntax Templates

## Table of Contents

- [PlantUML Syntax Templates](#plantuml-syntax-templates)
  - [Table of Contents](#table-of-contents)
  - [Sequence Diagram](#sequence-diagram)
  - [Use Case Diagram](#use-case-diagram)
  - [Class Diagram](#class-diagram)
  - [Object Diagram](#object-diagram)
  - [Activity Diagram](#activity-diagram)
  - [Architecture Diagram](#architecture-diagram)

---

## Sequence Diagram

```plantuml
@startuml user-login-sequence
autonumber
actor "用户" as user
participant "AuthController" as ctrl
participant "AuthService" as svc
participant "UserRepository" as repo
database "MySQL" as db
participant "Redis" as redis

== 认证流程 ==
user -> ctrl : POST /auth/login
ctrl -> svc : login(username, password)
svc -> repo : findByUsername()
repo -> db : SELECT * FROM user
db --> repo : ResultSet
repo --> svc : User entity

alt 密码正确
    svc -> redis : SET token
    redis --> svc : OK
    svc --> ctrl : LoginResponse(token)
    ctrl --> user : 200 + token
else 密码错误
    svc --> ctrl : AuthException
    ctrl --> user : 401 Unauthorized
end
@enduml
```

**关键语法：**

| 语法 | 含义 |
|------|------|
| `->` | 同步消息（实线箭头） |
| `-->` | 返回消息（虚线箭头） |
| `->>` | 异步消息（细箭头） |
| `autonumber` | 自动编号 |
| `actor` | 参与者（人形） |
| `participant` | 参与者（方框） |
| `database` | 数据库 |
| `queue` | 消息队列 |
| `boundary` | 边界/网关 |
| `control` | 控制器 |
| `entity` | 实体 |
| `== Label ==` | 分隔线 |
| `note over` | 跨参与者注释 |
| `note left/right of` | 侧边注释 |
| `alt/else/end` | 条件分支 |
| `opt` | 可选块 |
| `loop` | 循环块 |
| `par/end` | 并行块 |
| `break` | 中断块 |

---

## Use Case Diagram

```plantuml
@startuml order-usecase
left to right direction
skinparam packageStyle rectangle

actor "普通用户" as user
actor "管理员" as admin
actor "支付系统" as payment

rectangle "订单系统" {
  usecase "创建订单" as UC1
  usecase "查看订单" as UC2
  usecase "取消订单" as UC3
  usecase "支付订单" as UC4
  usecase "管理订单" as UC5
  usecase "导出订单" as UC6
  usecase "登录" as UC7
}

user --> UC1
user --> UC2
user --> UC3
user --> UC4
user --> UC7
admin --> UC2
admin --> UC5
admin --> UC6
admin --> UC7
payment --> UC4

UC1 ..> UC7 : <<include>>
UC4 ..> UC1 : <<extend>>
UC3 ..> UC7 : <<include>>
@enduml
```

**关键语法：**

| 语法 | 含义 |
|------|------|
| `actor` | 参与者 |
| `usecase` | 用例 |
| `rectangle` | 系统边界 |
| `-->` | 关联 |
| `..>` + `<<include>>` | 包含关系 |
| `..>` + `<<extend>>` | 扩展关系 |
| `left to right direction` | 从左到右布局 |
| `top to bottom direction` | 从上到下布局 |

---

## Class Diagram

```plantuml
@startuml order-class
package "com.example.order" {
  abstract class BaseService<T, ID> {
    # repository: JpaRepository<T, ID>
    + findById(id): Optional<T>
    + save(entity): T
    + deleteById(id): void
  }

  class OrderService {
    - orderRepository: OrderRepository
    - paymentClient: PaymentClient
    + createOrder(dto): OrderDTO
    + cancelOrder(id): void
    + queryOrders(query): Page<OrderDTO>
  }

  interface OrderRepository {
    + findByUserId(userId): List<Order>
    + findByStatus(status): List<Order>
    + countByCreateTimeBetween(start, end): Long
  }

  class Order {
    - id: Long
    - userId: Long
    - amount: BigDecimal
    - status: OrderStatus
    - items: List<OrderItem>
    - createTime: LocalDateTime
    + getTotalAmount(): BigDecimal
  }

  class OrderItem {
    - id: Long
    - productId: Long
    - quantity: Integer
    - price: BigDecimal
  }

  enum OrderStatus {
    PENDING
    PAID
    SHIPPED
    COMPLETED
    CANCELLED
  }
}

BaseService <|-- OrderService
OrderService ..> OrderRepository : <<uses>>
OrderService ..> Order : <<creates>>
Order --> OrderItem : contains *
Order --> OrderStatus
@enduml
```

**关键语法：**

| 语法 | 含义 |
|------|------|
| `class` | 普通类 |
| `abstract class` | 抽象类 |
| `interface` | 接口 |
| `enum` | 枚举 |
| `+` | public |
| `-` | private |
| `#` | protected |
| `<|--` | 继承 |
| `..|>` | 实现 |
| `-->` | 关联 |
| `*--` | 组合 |
| `o--` | 聚合 |
| `..>` | 依赖 |
| `package` | 包 |
| `--` 隐藏标签 | 省略关系标签 |

---

## Object Diagram

```plantuml
@startuml order-instance
object "orderService" as svc {
  cacheEnabled = true
  maxRetryCount = 3
}

object "order1" as o1 {
  id = 1001
  userId = 2001
  amount = 299.00
  status = PAID
  createTime = "2024-01-15 10:30:00"
}

object "item1" as i1 {
  productId = 3001
  quantity = 2
  price = 99.50
}

object "item2" as i2 {
  productId = 3002
  quantity = 1
  price = 100.00
}

object "user1" as u1 {
  id = 2001
  name = "张三"
  level = VIP
}

svc --> o1 : currentOrder
o1 --> i1 : contains
o1 --> i2 : contains
u1 --> o1 : placed
@enduml
```

**关键语法：**

| 语法 | 含义 |
|------|------|
| `object "name" as alias` | 对象声明 |
| `{ fields }` | 对象属性 |
| `-->` | 链接 |
| `: label` | 链接标签 |

---

## Activity Diagram

```plantuml
@startuml order-create-activity
start

:接收创建订单请求;

fork
  :验证用户身份;
fork again
  :检查商品库存;
end fork

if (库存充足?) then (否)
  :返回库存不足错误;
  stop
else (是)
  :计算订单金额;
endif

:创建订单记录;

switch (支付方式?)
case (支付宝)
  :调用支付宝接口;
case (微信)
  :调用微信支付接口;
case (余额)
  :扣减用户余额;
endswitch

if (支付成功?) then (是)
  :更新订单状态为 PAID;
  :发送订单确认通知;
  :扣减库存;
else (否)
  :更新订单状态为 CANCELLED;
  :记录支付失败日志;
endif

stop
@enduml
```

**关键语法：**

| 语法 | 含义 |
|------|------|
| `start` / `stop` | 开始/结束 |
| `:action;` | 动作 |
| `if/then/else/endif` | 条件分支 |
| `switch/case/endswitch` | 多分支 |
| `fork/fork again/end fork` | 并行 |
| `repeat/repeat while (cond)` | repeat-until 循环 |
| `while (cond) is (label)/endwhile` | while 循环 |
| `|Swimlane|` | 泳道 |
| `detach` | 分离箭头 |
| `-->` | 转换箭头 |

**泳道示例：**

```plantuml
|用户|
start
:提交订单;

|订单服务|
:创建订单;

|支付服务|
:处理支付;

|用户|
:收到确认;
stop
```

---

## Architecture Diagram

```plantuml
@startuml system-architecture
skinparam componentStyle rectangle
skinparam backgroundColor #FEFEFE
skinparam shadowing false

!define CLIENTS #LightBlue
!define GATEWAY #LightGreen
!define SERVICES #LightYellow
!define INFRA #LightGray

package "客户端" CLIENTS {
  [Web App] as web
  [Mobile App] as mobile
  [Open API] as openapi
}

package "API 网关层" GATEWAY {
  [Gateway\n(Spring Cloud Gateway)] as gw
}

package "业务服务层" SERVICES {
  package "用户服务" {
    [UserController] as uc
    [UserService] as us
  }
  package "订单服务" {
    [OrderController] as oc
    [OrderService] as os
  }
  package "商品服务" {
    [ProductController] as pc
    [ProductService] as ps
  }
}

package "基础设施层" INFRA {
  database "MySQL\n(主从)" as db
  queue "RabbitMQ" as mq
  [Redis Cluster] as redis
  [ElasticSearch] as es
  participant "OSS" as oss
}

web --> gw : HTTPS
mobile --> gw : HTTPS
openapi --> gw : HTTPS

gw --> uc
gw --> oc
gw --> pc

uc --> us
oc --> os
pc --> ps

us --> db
os --> db
ps --> db

os --> mq : 下单事件
os --> redis : 缓存
ps --> es : 搜索索引
ps --> oss : 商品图片
@enduml
```

**关键语法：**

| 语法 | 含义 |
|------|------|
| `[Component]` | 组件 |
| `package` | 分组/包 |
| `database` | 数据库 |
| `queue` | 消息队列 |
| `participant` | 云服务 |
| `node` | 部署节点 |
| `interface` | 端口/接口 |
| `skinparam` | 全局样式 |
| `!define` | 宏定义（用于颜色等） |
| `-->` | 实线连接 |
| `..>` | 虚线连接（弱依赖） |
| `\n` | 组件内换行 |

**多层架构简化模板：**

```plantuml
@startuml layers
skinparam componentStyle rectangle

package "Controller 层" as ctrl {
  [API 1]
  [API 2]
}

package "Service 层" as svc {
  [Service A]
  [Service B]
}

package "DAO 层" as dao {
  [Repository A]
  [Repository B]
}

[API 1] --> [Service A]
[API 2] --> [Service B]
[Service A] --> [Repository A]
[Service B] --> [Repository B]
@enduml
```
