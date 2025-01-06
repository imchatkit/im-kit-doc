# 服务端API设计方案

## 一、OpenAPI定义

### 1.1 基础配置
```yaml
openapi: 3.0.0
info:
  title: IM Cloud API
  version: 1.0.0
  description: IM云服务API文档
servers:
  - url: https://api.im.example.com/v1
    description: 生产环境
  - url: https://api-test.im.example.com/v1
    description: 测试环境

components:
  securitySchemes:
    AppAuth:
      type: apiKey
      in: header
      name: X-App-Key
      description: 应用密钥
    UserAuth:
      type: bearer
      scheme: bearer
      bearerFormat: JWT
      description: 用户Token
```

### 1.2 群组接口定义
```yaml
paths:
  /app/groups:
    post:
      tags:
        - APP群组接口
      summary: APP创建群组
      security:
        - UserAuth: []
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateGroupRequest'
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ApiResponse'
  
  /server/groups:
    post:
      tags:
        - 服务端群组接口
      summary: 服务端创建群组
      security:
        - AppAuth: []
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateServerGroupRequest'
      responses:
        '200':
          description: 成功
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ApiResponse'

components:
  schemas:
    CreateGroupRequest:
      type: object
      required:
        - name
        - memberIds
      properties:
        name:
          type: string
          description: 群组名称
        memberIds:
          type: array
          items:
            type: string
          description: 成员ID列表
        announcement:
          type: string
          description: 群公告
    
    CreateServerGroupRequest:
      type: object
      required:
        - name
        - ownerId
        - memberIds
      properties:
        name:
          type: string
          description: 群组名称
        ownerId:
          type: string
          description: 群主ID
        memberIds:
          type: array
          items:
            type: string
          description: 成员ID列表
        settings:
          $ref: '#/components/schemas/GroupSettings'
```

## 二、代码实现

### 2.1 Controller层
```java
// APP接口控制器
@RestController
@RequestMapping("/api/v1/app/groups")
@Tag(name = "APP群组接口")
@RequiredArgsConstructor
public class AppGroupController {
    
    private final GroupService groupService;
    
    @Operation(summary = "APP创建群组")
    @PostMapping
    public ApiResponse<GroupDTO> createGroup(
            @RequestHeader(USER_TOKEN_HEADER) String token,
            @RequestBody @Valid CreateGroupRequest request) {
        String userId = tokenService.getUserId(token);
        GroupDTO group = groupService.createGroupFromApp(userId, request);
        return ApiResponse.success(group);
    }
}

// 服务端接口控制器
@RestController
@RequestMapping("/api/v1/server/groups")
@Tag(name = "服务端群组接口")
@RequiredArgsConstructor
public class ServerGroupController {
    
    private final GroupService groupService;
    
    @Operation(summary = "服务端创建群组")
    @PostMapping
    public ApiResponse<GroupDTO> createGroup(
            @RequestHeader(APP_KEY_HEADER) String appKey,
            @RequestBody @Valid CreateServerGroupRequest request) {
        String appId = appKeyService.getAppId(appKey);
        GroupDTO group = groupService.createGroupFromServer(appId, request);
        return ApiResponse.success(group);
    }
}
```

### 2.2 Service层
```java
@Service
@Slf4j
public class GroupService {
    
    private final GroupRepository groupRepository;
    private final UserService userService;
    private final MessageService messageService;
    private final GroupValidator groupValidator;
    
    /**
     * APP端创建群组
     */
    @Transactional
    public GroupDTO createGroupFromApp(String userId, CreateGroupRequest request) {
        // 1. 业务验证
        groupValidator.validateAppCreateGroup(userId, request);
        
        // 2. 构建创建上下文
        GroupCreateContext context = GroupCreateContext.builder()
            .source(GroupSource.APP)
            .creator(userId)
            .name(request.getName())
            .memberIds(request.getMemberIds())
            .settings(GroupSettings.defaultAppSettings())
            .build();
            
        // 3. 创建群组
        Group group = createGroupCore(context);
        
        // 4. 发送群创建事件
        messageService.sendGroupCreateMessage(group);
        
        return GroupDTO.from(group);
    }
    
    /**
     * 服务端创建群组
     */
    @Transactional
    public GroupDTO createGroupFromServer(String appId, CreateServerGroupRequest request) {
        // 1. 业务验证
        groupValidator.validateServerCreateGroup(appId, request);
        
        // 2. 构建创建上下文
        GroupCreateContext context = GroupCreateContext.builder()
            .source(GroupSource.SERVER)
            .appId(appId)
            .creator(request.getOwnerId())
            .name(request.getName())
            .memberIds(request.getMemberIds())
            .settings(request.getSettings())
            .build();
            
        // 3. 创建群组
        Group group = createGroupCore(context);
        
        // 4. 发送群创建事件
        messageService.sendGroupCreateMessage(group);
        
        return GroupDTO.from(group);
    }
    
    /**
     * 核心创建群组逻辑
     */
    private Group createGroupCore(GroupCreateContext context) {
        // 1. 创建群组实体
        Group group = new Group();
        group.setId(generateGroupId());
        group.setName(context.getName());
        group.setOwnerId(context.getCreator());
        group.setSettings(context.getSettings());
        group.setSource(context.getSource());
        group.setCreateTime(LocalDateTime.now());
        
        // 2. 创建群成员
        List<GroupMember> members = createGroupMembers(group, context);
        group.setMembers(members);
        
        // 3. 保存群组
        return groupRepository.save(group);
    }
}
```

### 2.3 验证层
```java
@Component
@Slf4j
public class GroupValidator {
    
    private final UserService userService;
    private final AppService appService;
    
    /**
     * APP创建群组验证
     */
    public void validateAppCreateGroup(String userId, CreateGroupRequest request) {
        // 1. 验证用户
        User user = userService.getUser(userId);
        if (user.getStatus() != UserStatus.NORMAL) {
            throw new BusinessException("用户状态异常");
        }
        
        // 2. 验证成员数量
        if (request.getMemberIds().size() > 100) {
            throw new BusinessException("群成员不能超过100人");
        }
        
        // 3. 验证成员有效性
        validateMembers(request.getMemberIds());
    }
    
    /**
     * 服务端创建群组验证
     */
    public void validateServerCreateGroup(String appId, CreateServerGroupRequest request) {
        // 1. 验证应用
        App app = appService.getApp(appId);
        if (!app.isEnabled()) {
            throw new BusinessException("应用已禁用");
        }
        
        // 2. 验证成员数量
        if (request.getMemberIds().size() > 500) {
            throw new BusinessException("群成员不能超过500人");
        }
        
        // 3. 验证群主
        if (!request.getMemberIds().contains(request.getOwnerId())) {
            throw new BusinessException("群主必须是群成员");
        }
        
        // 4. 验证成员有效性
        validateMembers(request.getMemberIds());
    }
    
    /**
     * 验证群成员有效性
     */
    private void validateMembers(List<String> memberIds) {
        List<User> users = userService.getUsers(memberIds);
        if (users.size() != memberIds.size()) {
            throw new BusinessException("存在无效的用户ID");
        }
    }
}
```

### 2.4 数据模型
```java
@Data
@Entity
@Table(name = "im_group")
public class Group {
    @Id
    private String id;
    
    private String name;
    
    private String ownerId;
    
    @Enumerated(EnumType.STRING)
    private GroupSource source;
    
    @Convert(converter = GroupSettingsConverter.class)
    private GroupSettings settings;
    
    @OneToMany(mappedBy = "group", cascade = CascadeType.ALL)
    private List<GroupMember> members;
    
    private LocalDateTime createTime;
    
    private LocalDateTime updateTime;
}

@Data
@Builder
public class GroupCreateContext {
    private GroupSource source;
    private String appId;
    private String creator;
    private String name;
    private List<String> memberIds;
    private GroupSettings settings;
}

public enum GroupSource {
    APP,    // APP创建
    SERVER  // 服务端创建
}
```

## 三、接口规范

### 3.1 统一响应格式
```java
@Data
@Builder
public class ApiResponse<T> {
    private int code;
    private String message;
    private T data;
    
    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
            .code(200)
            .message("success")
            .data(data)
            .build();
    }
    
    public static <T> ApiResponse<T> error(int code, String message) {
        return ApiResponse.<T>builder()
            .code(code)
            .message(message)
            .build();
    }
}
```

### 3.2 错误码定义
```java
public enum ErrorCode {
    PARAM_ERROR(400, "参数错误"),
    UNAUTHORIZED(401, "未授权"),
    FORBIDDEN(403, "无权限"),
    NOT_FOUND(404, "资源不存在"),
    INTERNAL_ERROR(500, "系统错误");
    
    private final int code;
    private final String message;
}
```

### 3.3 异常处理
```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(BusinessException.class)
    public ApiResponse<Void> handleBusinessException(BusinessException e) {
        log.warn("业务异常: {}", e.getMessage());
        return ApiResponse.error(e.getCode(), e.getMessage());
    }
    
    @ExceptionHandler(Exception.class)
    public ApiResponse<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return ApiResponse.error(
            ErrorCode.INTERNAL_ERROR.getCode(),
            ErrorCode.INTERNAL_ERROR.getMessage()
        );
    }
}
```

## 四、最佳实践

### 4.1 接口设计原则
1. 分离APP和服务端接口
2. 统一的响应格式
3. 清晰的错误码
4. 详细的接口文档
5. 完善的参数校验

### 4.2 代码分层原则
1. Controller只负责请求处理
2. Service分离业务逻辑
3. Validator专注参数校验
4. Repository处理数据访问
5. DTO做数据转换

### 4.3 安全考虑
1. 不同的认证方式
2. 严格的权限控制
3. 参数校验和过滤
4. 日志审计
5. 限流控制 

## 五、Spring Cloud OpenAPI SDK生成

### 5.1 项目依赖配置
```gradle
// build.gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'org.openapi.generator' version '7.1.0'  // OpenAPI生成器插件
}

dependencies {
    // Spring Cloud
    implementation 'org.springframework.cloud:spring-cloud-starter-openfeign'
    
    // OpenAPI文档
    implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0'
    implementation 'org.springdoc:springdoc-openapi-starter-common:2.3.0'
    
    // OpenAPI注解处理
    implementation 'io.swagger.core.v3:swagger-annotations:2.2.19'
    implementation 'io.swagger.core.v3:swagger-models:2.2.19'
    
    // 验证
    implementation 'org.springframework.boot:spring-boot-starter-validation'
}
```

### 5.2 OpenAPI配置
```java
@Configuration
public class OpenAPIConfig {
    
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("IM Cloud API")
                .version("1.0.0")
                .description("IM云服务接口文档")
                .contact(new Contact()
                    .name("Your Company")
                    .email("support@example.com")))
            .addSecurityItem(new SecurityRequirement().addList("bearer-jwt"))
            .components(new Components()
                .addSecuritySchemes("bearer-jwt", 
                    new SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")));
    }
}
```

### 5.3 SDK生成配置
```gradle
// openapi-generator.gradle
openApiGenerate {
    generatorName = "java"
    inputSpec = "$rootDir/openapi.json"
    outputDir = "$buildDir/generated/openapi"
    apiPackage = "com.your.im.api"
    modelPackage = "com.your.im.model"
    configOptions = [
        dateLibrary: "java8",
        library: "feign",
        interfaceOnly: "true",
        useRuntimeException: "true",
        openApiNullable: "false"
    ]
}

// 添加生成的源码到编译路径
sourceSets {
    main {
        java {
            srcDir "$buildDir/generated/openapi/src/main/java"
        }
    }
}
```

### 5.4 生成SDK的Maven配置
```xml
<!-- sdk/pom.xml -->
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.your.im</groupId>
    <artifactId>im-sdk</artifactId>
    <version>1.0.0</version>
    
    <properties>
        <feign.version>12.5</feign.version>
        <jackson.version>2.15.3</jackson.version>
    </properties>
    
    <dependencies>
        <!-- Feign -->
        <dependency>
            <groupId>io.github.openfeign</groupId>
            <artifactId>feign-core</artifactId>
            <version>${feign.version}</version>
        </dependency>
        <dependency>
            <groupId>io.github.openfeign</groupId>
            <artifactId>feign-jackson</artifactId>
            <version>${feign.version}</version>
        </dependency>
        
        <!-- Jackson -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version>
        </dependency>
    </dependencies>
</project>
```

### 5.5 生成的SDK示例
```java
// 生成的API接口
@javax.annotation.Generated(value = "org.openapitools.codegen.languages.JavaClientCodegen")
public interface GroupApi {
    
    /**
     * APP创建群组
     * @param createGroupRequest 创建群组请求 (required)
     * @return GroupDTO
     */
    @RequestLine("POST /api/v1/app/groups")
    @Headers({
        "Content-Type: application/json",
        "Accept: application/json"
    })
    ApiResponse<GroupDTO> createGroup(
        @HeaderParam("X-User-Token") String token,
        @RequestBody CreateGroupRequest createGroupRequest
    );
    
    /**
     * 服务端创建群组
     * @param createServerGroupRequest 服务端创建群组请求 (required)
     * @return GroupDTO
     */
    @RequestLine("POST /api/v1/server/groups")
    @Headers({
        "Content-Type: application/json",
        "Accept: application/json"
    })
    ApiResponse<GroupDTO> createServerGroup(
        @HeaderParam("X-App-Key") String appKey,
        @RequestBody CreateServerGroupRequest createServerGroupRequest
    );
}

// SDK配置类
@Configuration
public class ImSdkConfig {
    
    @Bean
    public GroupApi groupApi(
            @Value("${im.api.url}") String apiUrl,
            @Value("${im.api.timeout:5000}") int timeout) {
        return Feign.builder()
            .encoder(new JacksonEncoder())
            .decoder(new JacksonDecoder())
            .options(new Request.Options(timeout, timeout))
            .target(GroupApi.class, apiUrl);
    }
}
```

### 5.6 SDK使用示例
```java
// APP端使用示例
@Service
@RequiredArgsConstructor
public class GroupServiceImpl {
    
    private final GroupApi groupApi;
    
    public GroupDTO createGroup(String token, CreateGroupRequest request) {
        ApiResponse<GroupDTO> response = groupApi.createGroup(token, request);
        if (response.getCode() != 200) {
            throw new BusinessException(response.getMessage());
        }
        return response.getData();
    }
}

// 服务端使用示例
@Service
@RequiredArgsConstructor
public class ServerGroupServiceImpl {
    
    private final GroupApi groupApi;
    private final AppKeyProvider appKeyProvider;
    
    public GroupDTO createServerGroup(CreateServerGroupRequest request) {
        String appKey = appKeyProvider.getAppKey();
        ApiResponse<GroupDTO> response = groupApi.createServerGroup(appKey, request);
        if (response.getCode() != 200) {
            throw new BusinessException(response.getMessage());
        }
        return response.getData();
    }
}
```

### 5.7 生成多语言SDK
```bash
# 生成OpenAPI文档
curl http://localhost:8080/v3/api-docs > openapi.json

# 生成Java SDK
openapi-generator generate \
  -i openapi.json \
  -g java \
  -o sdk/java \
  --additional-properties=library=feign

# 生成Python SDK
openapi-generator generate \
  -i openapi.json \
  -g python \
  -o sdk/python

# 生成TypeScript SDK
openapi-generator generate \
  -i openapi.json \
  -g typescript-axios \
  -o sdk/typescript

# 生成Go SDK
openapi-generator generate \
  -i openapi.json \
  -g go \
  -o sdk/go
```

### 5.8 SDK发布流程
```groovy
// 发布配置
publishing {
    publications {
        mavenJava(MavenPublication) {
            from components.java
            groupId = 'com.your.im'
            artifactId = 'im-sdk'
            version = '1.0.0'
        }
    }
    repositories {
        maven {
            url = "https://maven.pkg.github.com/your-org/im-sdk"
            credentials {
                username = project.findProperty("gpr.user") ?: System.getenv("USERNAME")
                password = project.findProperty("gpr.key") ?: System.getenv("TOKEN")
            }
        }
    }
}
```

### 5.9 最佳实践
1. SDK版本管理
   - 使用语义化版本
   - 保持向后兼容
   - 完整的变更日志

2. 文档生成
   - 自动生成API文档
   - 示例代码
   - 接口说明

3. 错误处理
   - 统一的异常体系
   - 错误码说明
   - 重试机制

4. 性能优化
   - 连接池管理
   - 超时配置
   - 并发控制