# Vibe IM - AI 代理开发指南

**项目**: Vibe IM - 实时即时通讯应用  
**语言**: 混合（后端：中文注释 + 英文代码，前端：英文）  
**最后更新**: 2026-03-31

---

## 1. 项目概述

Vibe IM 是一个全栈实时即时通讯应用，包含：

- **后端**: Spring Boot 3.2 REST API + WebSocket 服务器，使用 MySQL 8.0 和 Redis 7.x
- **前端**: Flutter 3.x 跨平台聊天客户端
- **核心功能**: 用户注册/登录、实时消息、消息持久化、离线消息投递、WebSocket 心跳

项目遵循分层架构模式，控制器、服务和仓库之间职责清晰分离。

---

## 2. 技术栈

### 后端 (`backend/`)
| 技术 | 版本 | 用途 |
|------------|---------|---------|
| Java | 21 | 编程语言，支持虚拟线程 |
| Spring Boot | 3.2.0 | 应用框架 |
| Spring Data JPA | 3.2+ | 数据库访问 ORM |
| Spring WebSocket | 3.2+ | 实时双向通信 |
| Spring Security Crypto | 6.2+ | 密码加密（BCrypt） |
| MySQL | 8.0+ | 主数据库 |
| Redis | 7.x | 会话管理和缓存 |
| Lombok | 1.18+ | 减少样板代码 |
| Maven | 3.9+ | 构建工具 |
| TestContainers | 1.19+ | 集成测试 |

### 前端 (`frontend/`)
| 技术 | 版本 | 用途 |
|------------|---------|---------|
| Flutter | 3.24+ | 跨平台 UI 框架 |
| Dart | 3.0+ | 编程语言 |
| Provider | 6.1+ | 状态管理 |
| HTTP | 1.1+ | REST API 客户端 |
| web_socket_channel | 2.4+ | WebSocket 客户端 |
| shared_preferences | 2.2+ | 本地存储 |
| intl | 0.18+ | 国际化和格式化 |

### 基础设施
| 服务 | 端口 | 描述 |
|---------|------|-------------|
| 后端 API | 8080 | HTTP + WebSocket 服务器 |
| MySQL | 3306 | 数据库 |
| Redis | 6379 | 缓存和会话存储 |

---

## 3. 项目结构

```
vibeCodingDemo/
├── backend/                          # Java Spring Boot 后端
│   ├── src/main/java/com/vibe/im/
│   │   ├── VibeImApplication.java    # 主入口
│   │   ├── config/                   # 配置类
│   │   │   ├── SecurityConfig.java   # BCrypt 密码编码器
│   │   │   ├── WebSocketConfig.java  # WebSocket 端点注册
│   │   │   ├── RedisConfig.java      # Redis 配置
│   │   │   └── WebMvcConfig.java     # CORS 和 Web 配置
│   │   ├── controller/               # REST API 控制器
│   │   │   ├── AuthController.java   # /auth/* 端点
│   │   │   └── ChatController.java   # /chat/* 端点
│   │   ├── service/                  # 业务逻辑层
│   │   │   ├── AuthService.java      # 认证和会话管理
│   │   │   └── ChatService.java      # 消息发送和历史记录
│   │   ├── repository/               # JPA 数据访问层
│   │   │   ├── UserRepository.java
│   │   │   └── MessageRepository.java
│   │   ├── entity/                   # JPA 实体
│   │   │   ├── User.java             # 用户实体
│   │   │   ├── Message.java          # 消息实体
│   │   │   └── enums/                # 枚举类型
│   │   │       ├── UserStatus.java   # ONLINE, OFFLINE
│   │   │       ├── MessageStatus.java # SENDING, SENT, DELIVERED
│   │   │       └── MessageType.java  # TEXT, IMAGE, FILE
│   │   ├── dto/                      # 数据传输对象
│   │   │   ├── request/              # 请求 DTO
│   │   │   │   ├── LoginRequest.java
│   │   │   │   ├── RegisterRequest.java
│   │   │   │   └── SendMessageRequest.java
│   │   │   └── response/             # 响应 DTO
│   │   │       ├── LoginResponse.java
│   │   │       ├── UserResponse.java
│   │   │       ├── MessageResponse.java
│   │   │       └── PageResponse.java
│   │   ├── websocket/                # WebSocket 处理器
│   │   │   ├── WebSocketHandler.java # WebSocket 消息处理器
│   │   │   └── ConnectionManager.java # 连接管理和心跳
│   │   ├── exception/                # 异常处理
│   │   │   ├── BusinessException.java
│   │   │   ├── ErrorCode.java
│   │   │   └── GlobalExceptionHandler.java
│   │   └── util/                     # 工具类
│   │       └── SnowflakeIdGenerator.java
│   ├── src/main/resources/
│   │   └── application.yml           # 应用配置
│   ├── src/test/java/com/vibe/im/    # 测试类
│   │   ├── service/AuthServiceTest.java
│   │   ├── controller/AuthControllerTest.java
│   │   ├── entity/UserTest.java
│   │   └── integration/ChatFlowIntegrationTest.java
│   ├── pom.xml                       # Maven 依赖
│   ├── Dockerfile                    # Docker 镜像定义
│   └── docker-compose.yml            # Docker Compose 配置
│
├── frontend/                         # Flutter 前端
│   ├── lib/
│   │   ├── main.dart                 # 应用入口
│   │   ├── pages/                    # UI 页面
│   │   │   ├── login_page.dart       # 登录/注册页面
│   │   │   ├── chat_list_page.dart   # 聊天列表页面
│   │   │   └── chat_page.dart        # 单聊页面
│   │   ├── providers/                # 状态管理（Provider 模式）
│   │   │   ├── auth_provider.dart    # 认证状态
│   │   │   └── chat_provider.dart    # 聊天消息状态
│   │   ├── services/                 # API 和 WebSocket 服务
│   │   │   ├── api_service.dart      # HTTP REST API 客户端
│   │   │   └── websocket_service.dart # WebSocket 客户端
│   │   ├── models/                   # 数据模型
│   │   │   ├── user.dart
│   │   │   ├── message.dart
│   │   │   └── contact.dart
│   │   └── widgets/                  # 可复用组件（如有）
│   ├── pubspec.yaml                  # Flutter 依赖
│   ├── analysis_options.yaml         # Dart  lint 规则
│   └── test/widget_test.dart         # 组件测试
│
└── docs/                             # 文档
    ├── research/project/report.md    # 技术评估报告
    ├── superpowers/specs/            # 设计规格
    └── superpowers/plans/            # 实施计划
```

---

## 4. 构建和运行命令

### 前置条件
- **JDK 21+** - [下载](https://www.oracle.com/java/technologies/downloads/)
- **Maven 3.9+** - [下载](https://maven.apache.org/download.cgi)
- **Docker & Docker Compose** - [下载](https://www.docker.com/products/docker-desktop)
- **Flutter 3.24+** - [安装](https://docs.flutter.dev/get-started/install)

### 后端

```bash
cd backend

# 构建项目
mvn clean package

# 运行单元测试
mvn test

# 使用 TestContainers 运行集成测试
mvn verify

# 使用本地 MySQL/Redis 运行
mvn spring-boot:run

# Docker Compose - 全栈启动（MySQL + Redis + 后端）
docker-compose up -d

# 查看日志
docker-compose logs -f backend

# 停止所有服务
docker-compose down
```

### 前端

```bash
cd frontend

# 获取依赖
flutter pub get

# 在已连接设备/模拟器上运行
flutter run

# 运行测试
flutter test

# 代码分析
flutter analyze

# 构建 Web 版本
flutter build web

# 构建特定平台版本
flutter build ios
flutter build android
flutter build macos
flutter build windows
flutter build linux
```

---

## 5. API 文档

所有 API 端点前缀为 `/api`。

### 认证端点

| 方法 | 端点 | 描述 | 需要认证 |
|--------|----------|-------------|---------------|
| POST | `/api/auth/register` | 注册新用户 | 否 |
| POST | `/api/auth/login` | 用户登录 | 否 |
| POST | `/api/auth/logout` | 用户登出 | 可选 |
| GET | `/api/auth/me` | 获取当前用户 | 是 |

### 聊天端点

| 方法 | 端点 | 描述 | 需要认证 |
|--------|----------|-------------|---------------|
| POST | `/api/chat/send` | 发送消息 | 是（Session-Id 请求头） |
| GET | `/api/chat/messages` | 获取聊天记录 | 是（Session-Id 请求头） |

### WebSocket

- **URL**: `ws://localhost:8080/api/ws?sessionId={sessionId}`
- **心跳**: 每 15-30 秒发送 `{"type": "PING"}`
- **消息格式**: JSON

---

## 6. 测试策略

### 后端测试

**单元测试** (`src/test/java/com/vibe/im/`):
- 使用 JUnit 5 和 Mockito 进行模拟
- 使用 `@Nested` 和 `@DisplayName` 对测试分组
- 使用 `@ParameterizedTest` 处理多组测试用例

```bash
# 运行所有测试
mvn test

# 运行指定测试类
mvn test -Dtest=AuthServiceTest

# 运行集成测试
mvn verify
```

**集成测试**:
- 使用 TestContainers 启动 MySQL 和 Redis
- 测试完整用户流程（注册 → 登录 → 发消息 → 登出）
- 位于 `src/test/java/com/vibe/integration/`

### 前端测试

```bash
# 运行组件测试
flutter test

# 带覆盖率运行
flutter test --coverage
```

**注意**: 当前前端测试覆盖率较低。`test/widget_test.dart` 是模板文件，需要更新。

---

## 7. 代码风格指南

### 后端 (Java)

- **语言**: 文档注释使用中文，代码使用英文
- **包名**: `com.vibe.im`
- **命名规范**: 
  - 类名: PascalCase
  - 方法/变量: camelCase
  - 常量: UPPER_SNAKE_CASE
- **注解**: 使用 Lombok (`@Data`, `@Builder`, `@RequiredArgsConstructor`)
- **文档**: 所有公共类和方法必须有中文 Javadoc 注释

示例:
```java
/**
 * 用户注册
 *
 * <p>实现逻辑：</p>
 * <ol>
 *   <li>检查用户名是否已存在</li>
 *   <li>使用BCrypt加密密码</li>
 *   <li>创建用户实体</li>
 * </ol>
 *
 * @param request 注册请求
 * @return 用户响应信息
 * @throws BusinessException 当用户名已存在时抛出
 */
@Transactional
public UserResponse register(RegisterRequest request) {
    // 实现
}
```

### 前端 (Dart)

- **语言**: 英文注释和文档
- **命名规范**:
  - 类名: PascalCase
  - 方法/变量: camelCase
  - 常量: lowerCamelCase 或顶级常量使用 k 前缀
  - 私有成员: _leadingUnderscore
- **状态管理**: 使用 Provider 模式
- **文档**: 使用 Dart 文档注释 (`///`)

示例:
```dart
/// Register a new user
/// @param username The username for registration
/// @param password The password for registration
/// @param nickname The user's nickname
/// @return User object containing the registered user information
/// @throws Exception if registration fails
Future<User> register({
  required String username,
  required String password,
  required String nickname,
}) async {
  // Implementation
}
```

---

## 8. 安全考虑

### 密码安全
- 使用 BCrypt（强度 10）加密密码
- 禁止存储明文密码
- 密码校验：最少 6 个字符

### 会话管理
- 会话存储在 Redis 中，TTL 为 7 天
- 会话 ID 使用 UUID 生成
- 每个认证请求都进行会话校验
- 活跃时自动续期会话

### WebSocket 安全
- WebSocket 握手期间校验会话
- 无效会话将被拒绝，返回 `CloseStatus.NOT_ACCEPTABLE`
- 心跳超时：30 秒（30 秒无活动连接将被关闭）

### CORS 配置
- 开发环境当前配置为允许所有来源 (`*`)
- 生产环境应加以限制

---

## 9. 开发约定

### Git 工作流
1. 从 main 分支创建功能分支
2. 使用约定式提交: `feat:`, `fix:`, `docs:`, `refactor:`
3. 编写有意义的提交信息

### 错误处理

**后端**:
- 预期业务错误使用 `BusinessException`
- 使用 `ErrorCode` 枚举定义错误码（400, 401, 404, 500）
- 全局异常处理器返回统一错误格式:
```json
{
  "code": 401,
  "message": "认证失败",
  "timestamp": "2026-03-30T10:00:00Z"
}
```

**前端**:
- 异步操作使用 try-catch
- 向用户展示友好的错误信息
- 仅在调试模式下记录错误

### 数据库表结构

**用户表**:
- `id`: BIGINT 主键 自增
- `username`: VARCHAR(50) 唯一 非空
- `password`: VARCHAR(255) 非空（BCrypt 哈希）
- `nickname`: VARCHAR(50)
- `avatar`: VARCHAR(500)
- `status`: VARCHAR(20) - ONLINE/OFFLINE
- `create_time`: DATETIME
- `update_time`: DATETIME

**消息表**:
- `id`: BIGINT 主键 自增
- `sender_id`: BIGINT 非空
- `receiver_id`: BIGINT 非空
- `content`: VARCHAR(1000) 非空
- `message_type`: VARCHAR(20) - TEXT/IMAGE/FILE
- `status`: VARCHAR(20) - SENDING/SENT/DELIVERED
- `send_time`: DATETIME
- `create_time`: DATETIME
- `update_time`: DATETIME

### Redis 键命名规范
- `session:{sessionId}` - 会话数据（Hash，7 天 TTL）
- `user:online:{userId}` - 在线状态（String，30 分钟 TTL）

---

## 10. 常见开发任务

### 新增 API 端点

1. 在 `dto/request/` 或 `dto/response/` 中创建 DTO
2. 在对应 Service 类中添加业务逻辑
3. 在 Controller 类中创建端点
4. 添加单元测试
5. 更新本文档

### 新增实体

1. 在 `entity/` 中创建带 JPA 注解的实体类
2. 如需枚举，在 `entity/enums/` 中创建
3. 创建继承 JpaRepository 的 Repository 接口
4. 添加数据库迁移（开发环境已启用 Hibernate auto-ddl）

### WebSocket 消息流转

1. 客户端连接: `ws://localhost:8080/api/ws?sessionId=xxx`
2. 服务器校验会话，加入 ConnectionManager
3. 服务器发送离线消息（status=SENT）
4. 客户端定期发送 PING，服务器回复 PONG
5. 通过 REST API 发送消息时，服务器通过 WebSocket 推送给接收方
6. 断开连接时，ConnectionManager 移除连接

---

## 11. 故障排除

### 后端问题

**数据库连接错误**:
```bash
# 检查 MySQL 容器
docker-compose ps mysql
docker-compose logs mysql

# 校验 application.yml 中的连接配置
```

**Redis 连接错误**:
```bash
docker-compose ps redis
docker-compose logs redis
```

**端口被占用**:
- 后端: 修改 `application.yml` 中的 `server.port`
- MySQL: 修改 `docker-compose.yml` 中的端口映射
- Redis: 修改 `docker-compose.yml` 中的端口映射

### 前端问题

**热重载不生效**:
```bash
flutter clean
flutter pub get
flutter run
```

**依赖问题**:
```bash
flutter pub upgrade
flutter clean
flutter pub get
```

---

## 12. 资源

- **后端 README**: `backend/README.md` - 详细 API 文档
- **设计规格**: `docs/superpowers/specs/2026-03-30-im-first-phase-design.md`
- **技术报告**: `docs/research/project/report.md`
- **Flutter 文档**: https://docs.flutter.dev/
- **Spring Boot 文档**: https://spring.io/projects/spring-boot

---

**AI 代理注意事项**: 修改代码时，请务必：
1. 遵循现有代码模式和约定
2. 更新对应测试
3. 添加适当的文档注释
4. 考虑并发操作的线程安全
5. 确保前后端 API 契约一致
