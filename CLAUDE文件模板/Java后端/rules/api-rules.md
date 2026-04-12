# Swagger 接口文档规范

## Controller 层注解

```java
@Tag(name = "用户管理", description = "用户注册、登录、信息查询")
@RestController
@RequestMapping("/api/v1/user")
public class UserController {
    
    @Operation(
        summary = "用户注册",
        description = "手机号注册，24小时内最多3次<br>成功返回token",
        responses = {
            @ApiResponse(responseCode = "200", description = "成功"),
            @ApiResponse(responseCode = "1001", description = "手机号已存在"),
            @ApiResponse(responseCode = "1002", description = "验证码错误")
        }
    )
    @PostMapping("/register")
    public Result<UserRegisterVO> register(
            @Valid @RequestBody 
            @Parameter(description = "注册参数", required = true) 
            UserRegisterDTO dto) {
        // ...
    }
    
    @Operation(summary = "查询用户详情")
    @GetMapping("/detail/{userId}")
    public Result<UserVO> getDetail(
            @Parameter(description = "用户ID", example = "10001") 
            @PathVariable Long userId) {
        // ...
    }
}
```

## DTO 字段注解

```java
@Schema(description = "用户注册请求")
@Data
public class UserRegisterDTO {
    
    @Schema(description = "手机号", requiredMode = RequiredMode.REQUIRED, 
            example = "13800138000", pattern = "^1[3-9]\\d{9}$")
    @NotBlank(message = "手机号不能为空")
    private String phone;
    
    @Schema(description = "验证码", example = "123456")
    @NotBlank
    private String smsCode;
    
    @Schema(description = "邀请码（可选）", example = "ABC123", 
            accessMode = Schema.AccessMode.READ_ONLY)  // 不显示在必填
    private String inviteCode;
}
```

## VO 响应注解

```java
@Schema(description = "用户信息响应")
@Data
public class UserVO {
    @Schema(description = "用户ID", example = "10001")
    private Long id;
    
    @Schema(description = "昵称", example = "张三")
    private String nickname;
    
    @Schema(description = "账户状态：0-正常 1-冻结", example = "0")
    private Integer status;
}
```

## 文档优化技巧
1. **example**: 必须填，方便前端直接复制测试
2. **requiredMode**: 明确必填/选填，减少对接成本
3. **description**: 支持HTML标签（`<br>`换行，`<ul>`列表）
4. **枚举**: 注明含义（`0-待支付 1-已支付`）
