# 测试规范

## 测试架构

```
src/test/java/
├── unit/               # 单元测试（纯Java，无Spring）
│   ├── service/        # Service层业务逻辑（Mock依赖）
│   └── util/           # 工具类测试
└── integration/        # 集成测试（@SpringBootTest）
    ├── controller/     # API接口测试（MockMvc）
    ├── mapper/         # 数据库测试（@MybatisPlusTest）
    └── mq/             # 消息队列测试（Testcontainers）
```

## 依赖配置（pom.xml）

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mysql</artifactId>
    <scope>test</scope>
</dependency>
```

## 单元测试（Service层）

```java
@ExtendWith(MockitoExtension.class)  // 纯Mockito，不加载Spring
class OrderServiceImplTest {
    
    @Mock
    private OrderMapper orderMapper;
    
    @Mock
    private InventoryService inventoryService;  // 外部服务Mock
    
    @InjectMocks
    private OrderServiceImpl orderService;
    
    @Test
    @DisplayName("创建订单成功：库存充足")
    void createOrder_Success() {
        // Given
        OrderCreateDTO dto = new OrderCreateDTO();
        dto.setUserId(10001L);
        dto.setSkuId(20001L);
        dto.setQuantity(2);
        
        when(inventoryService.checkStock(20001L, 2)).thenReturn(true);
        
        // When
        OrderCreateVO result = orderService.createOrder(dto);
        
        // Then
        assertNotNull(result.getOrderNo());  // 验证生成订单号
        assertEquals(19900, result.getAmount());  // 金额计算正确（分）
        verify(orderMapper).insert(any(OrderEntity.class));  // 验证插入DB
    }
    
    @Test
    @DisplayName("创建订单失败：库存不足抛异常")
    void createOrder_InsufficientStock() {
        when(inventoryService.checkStock(any(), any())).thenReturn(false);
        
        assertThrows(InsufficientStockException.class, () -> {
            orderService.createOrder(dto);
        });
    }
}
```

## 集成测试（Controller层）

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
@TestPropertySource(locations = "classpath:application-test.yml")  // 独立配置
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @MockBean
    private UserService userService;  // Mock Service层
    
    @Test
    @DisplayName("POST /api/v1/user/register - 注册成功")
    void register_Success() throws Exception {
        // Given
        UserRegisterDTO dto = new UserRegisterDTO();
        dto.setPhone("13800138000");
        dto.setSmsCode("123456");
        
        when(userService.register(any())).thenReturn(
            UserRegisterVO.builder().token("mock_token_123").build()
        );
        
        // When & Then
        mockMvc.perform(post("/api/v1/user/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(dto)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.code").value(200))
            .andExpect(jsonPath("$.data.token").value("mock_token_123"))
            .andDo(print());
    }
    
    @Test
    @DisplayName("参数校验失败返回400")
    void register_InvalidPhone() throws Exception {
        UserRegisterDTO dto = new UserRegisterDTO();
        dto.setPhone("invalid");  // 错误手机号
        
        mockMvc.perform(post("/api/v1/user/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(dto)))
            .andExpect(status().isOk())  // 业务封装后HTTP还是200
            .andExpect(jsonPath("$.code").value(400))  // 业务码400
            .andExpect(jsonPath("$.message").value(containsString("手机号格式")));
    }
}
```

## 数据库集成测试（Mapper层）

```java
@MybatisPlusTest  // 只加载MyBatis-Plus，不加载其他Bean
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers  // 自动启动Docker容器
class OrderMapperTest {
    
    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("test_db")
        .withUsername("test")
        .withPassword("test");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }
    
    @Autowired
    private OrderMapper orderMapper;
    
    @Test
    @DisplayName("插入订单并查询")
    void insertAndSelect() {
        // Given
        OrderEntity order = new OrderEntity();
        order.setOrderNo("TEST20240115001");
        order.setUserId(10001L);
        order.setStatus(0);
        
        // When
        orderMapper.insert(order);
        
        // Then
        OrderEntity found = orderMapper.selectById(order.getId());
        assertEquals("TEST20240115001", found.getOrderNo());
        assertNotNull(found.getCreateTime());  // 自动填充验证
    }
    
    @Test
    @Sql("/sql/cleanup_order.sql")  // 测试前清理数据（可选）
    @DisplayName("分页查询")
    void selectPage() {
        // 插入10条测试数据
        for (int i = 0; i < 10; i++) {
            // ...
        }
        
        Page<OrderEntity> page = new Page<>(1, 5);
        Page<OrderEntity> result = orderMapper.selectPage(page, 
            Wrappers.<OrderEntity>lambdaQuery().eq(OrderEntity::getUserId, 10001L));
        
        assertEquals(5, result.getRecords().size());
        assertEquals(10, result.getTotal());
    }
}
```

## MQ测试（使用嵌入式RocketMQ或Testcontainers）

```java
@SpringBootTest
@Testcontainers
class OrderPaidMQListenerTest {
    
    @Container
    static GenericContainer<?> rocketmq = new GenericContainer<>("apache/rocketmq:5.1.4")
        .withExposedPorts(9876, 10911);
    
    @Autowired
    private RocketMQTemplate rocketMQTemplate;
    
    @Autowired
    private InventoryService inventoryService;  // 真实服务或Spy
    
    @Test
    @DisplayName("消费订单支付事件并扣减库存")
    void consumeOrderPaid() throws InterruptedException {
        // Given：发送测试消息
        OrderPaidEvent event = new OrderPaidEvent();
        event.setOrderNo("TEST001");
        event.setSkuId(10001L);
        event.setQuantity(2);
        
        rocketMQTemplate.syncSend("trade_order_paid:paid", 
            MessageBuilder.withPayload(event).build());
        
        // When：等待消费（最多10秒）
        Thread.sleep(5000);
        
        // Then：验证库存已扣减
        verify(inventoryService, timeout(10000)).deductStock(10001L, 2);
    }
}
```

## 测试约定

1. **命名**：`被测方法名_场景_预期结果`，如`createOrder_StockOk_ReturnOrderNo`
2. **Given-When-Then**：注释明确三段逻辑（可选BDD风格的`@Given`注解）
3. **数据隔离**：每个测试方法独立，禁止测试间数据依赖
4. **执行速度**：单元测试<10ms，集成测试<5s，Testcontainers启动复用
5. **覆盖率**：Service层行覆盖率>80%，分支覆盖率>70%（JaCoCo检查）

## CI集成（GitLab CI示例）

```yaml
test:
  stage: test
  script:
    - ./mvnw test -Dtest=!*IntegrationTest  # 先跑单元测试（快）
    - ./mvnw test -Dtest=*IntegrationTest    # 再跑集成测试（慢，需Docker）
  coverage: '/Total.*?(\d+%)/'
  artifacts:
    reports:
      junit: target/surefire-reports/*.xml
```

## 测试数据管理

- **单条数据**：代码内Builder模式构建（`OrderEntity.builder().build()`）
- **批量数据**：`@Sql`导入classpath下的SQL文件（`src/test/resources/sql/`）
- **随机数据**：使用`java-faker`生成手机号、姓名等（避免硬编码）
```

---

**更新后的主文件导航区**（在CLAUDE.md中更新）：

```markdown
| 我要做... | 参考文件 | 关键类 |
|-----------|----------|--------|
| 写业务代码 | `/rules/coding-rules.md` | `BizException` |
| 发/收MQ消息 | `/rules/mq-rules.md` | `RocketMQTemplate` |
| 用缓存或锁 | `/rules/cache-rules.md` | `RedissonClient` |
| 写定时任务 | `/rules/job-rules.md` | `@XxlJob` |
| 写接口文档 | `/rules/api-rules.md` | `@Operation` |
| **设计数据库** | **`/rules/database-rules.md`** | **`@TableName`** |
| **写测试代码** | **`/rules/testing-rules.md`** | **`@SpringBootTest`** |