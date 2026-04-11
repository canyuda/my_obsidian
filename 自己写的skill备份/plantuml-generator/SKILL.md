---
name: plantuml-generator
description: |
  Generate PlantUML diagram files (.puml) by analyzing existing code and project structure.
  Supports 6 diagram types: sequence diagrams (序列图/时序图), use case diagrams (用例图),
  class diagrams (类图), object diagrams (对象图), activity diagrams (活动图/流程图),
  and architecture diagrams (架构图).

  TRIGGER when user says things like:
  - "生成序列图", "生成时序图", "画一个序列图", "这个业务的调用链"
  - "生成类图", "画出类的关系", "类的继承结构"
  - "生成架构图", "系统架构图", "模块架构"
  - "生成用例图", "用户能做什么"
  - "生成对象图", "对象关系"
  - "生成活动图", "生成流程图", "业务流程"
  - "生成PlantUML", "画个图", "出个图"
  - Any mention of generating diagrams for code, modules, systems, or business logic
  - Even if user doesn't explicitly mention "PlantUML", use this skill for any code-to-diagram request.
    For example, "帮我梳理一下这个模块的结构" or "这个接口的调用流程是什么" should trigger this skill
    when the output would benefit from a visual diagram.
allowed-tools: 
  - Read
  - Grep
  - Glob
---

# PlantUML Diagram Generator

从现有代码和项目结构生成 PlantUML (.puml) 图表文件。

## Workflow

1. **确认图表类型和范围** — 从用户请求中提取图表类型和关注范围（哪个模块/包/类）
2. **分析代码** — 使用 Glob/Grep/Read 工具有策略地读取相关代码
3. **生成 PlantUML** — 基于代码分析结果编写 .puml 内容
4. **保存文件** — 保存为 .puml 文件

## Code Analysis Strategy by Diagram Type

### Sequence Diagram / 序列图

目标：展示方法调用链和消息流。

**分析步骤：**
1. 找到入口点 — Controller 方法、API endpoint、或用户指定的方法
2. 用 Grep 追踪调用链 — 搜索方法调用、service 注入
3. 识别参与者 — 每个被调用的 service/component/repository 作为 participant
4. 标注异步调用 — `@Async`、`CompletableFuture`、消息队列用异步箭头
5. 标注分支逻辑 — `if/else` 用 `alt` 块

**关注：** Controller → Service → Repository/Mapper → DB 的完整链路。

### Class Diagram / 类图

目标：展示类的结构关系（继承、实现、组合、依赖）。

**分析步骤：**
1. 用 Glob 扫描目标包下所有 `.java` 文件
2. 用 Grep 找 `extends`、`implements` 关系
3. 读取类文件提取字段和方法签名
4. 通过 `@Autowired`/`@Resource` 识别依赖关系
5. 识别泛型和集合类型

**关注：** 不要超过 15 个类，聚焦核心类。省略 getter/setter 和简单字段。

### Use Case Diagram / 用例图

目标：展示系统功能和参与者。

**分析步骤：**
1. 扫描所有 Controller 的 `@RequestMapping`/`@GetMapping`/`@PostMapping`
2. 从 `@PreAuthorize`/`@RolesAllowed` 推断角色和参与者
3. 将每个 endpoint 映射为一个用例
4. 识别 `include`（公共流程如认证）和 `extend`（可选流程）关系

### Object Diagram / 对象图

目标：展示特定场景下的对象实例和状态。

**分析步骤：**
1. 识别核心业务对象（实体类）
2. 从测试代码或配置中推断典型实例数据
3. 展示对象间的引用链

### Activity Diagram / 活动图

目标：展示业务逻辑的控制流和决策分支。

**分析步骤：**
1. 找到业务逻辑入口方法
2. 读取方法体，追踪 if/else/switch/for/while
3. 识别并行处理 — 线程池、`@Async`、`CompletableFuture.allOf`
4. 识别异常处理 — try/catch 块作为分支
5. 用泳道区分不同组件的处理

### Architecture Diagram / 架构图

目标：展示系统整体结构和模块间关系。

**分析步骤：**
1. 用 Glob 扫描顶层包结构 — `ls` 或 `find` 目录层级
2. 从 `pom.xml`/`build.gradle` 识别模块依赖
3. 从配置文件识别外部依赖 — 数据库、消息队列、缓存、第三方 API
4. 识别分层 — controller/service/dao/config/common 等包
5. 绘制组件图，标注通信协议

**关注：** 保持高层抽象，不展示具体类。

## PlantUML Syntax Reference

详细的语法模板见 `references/templates.md`。生成前请先阅读对应图表类型的模板。

## Output Rules

- **文件后缀**: `.puml`
- **编码**: UTF-8
- **文件开头**: `@startuml {filename}`，结尾 `@enduml`
- **默认保存位置**: 用户指定路径，或当前工作目录
- **文件命名**: `{主题}-{图表类型}.puml`，例如：
  - `user-service-sequence.puml`
  - `order-module-class.puml`
  - `system-architecture.puml`
- **命名约定**: 图表中使用代码原始命名（英文），注释用中文
- **语法**: 严格遵守plantuml语法规则，禁止生成无法生成图片的文件, 必须通过`plantuml --check-syntax <puml文件路径>`的校验

## Important Notes

- 生成图表前，**必须先充分阅读和分析相关代码**，不要基于猜测生成
- 控制复杂度：序列图参与者 ≤ 10 个，类图 ≤ 15 个类，超过时聚焦核心部分
- 确保生成的 PlantUML 语法正确可渲染 — 注意分号、括号匹配
- 如果用户没有指定保存路径，保存到当前目录并告知用户文件位置
- 如果代码量很大，先确认用户关注的核心模块再分析，避免读取过多无关代码
