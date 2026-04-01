# Phase 3：控制器层重构 - 实施计划

> 版本：1.0.0
> 日期：2026-03-31
> 依赖：Phase 2 服务层重构完成

---

## 一、目标

重构工作流配置层的控制器代码，实现：
- RESTful API 接口（复数形式 `/api/workflows`）
- 细粒度节点/连线 CRUD
- 验证相关 API
- 统一响应格式
- 参数校验

---

## 二、API 设计总览

### 2.1 工作流管理 API

基础路径：`/api/workflows`

| 方法 | 路径 | 说明 | 请求体 | 响应体 |
|------|------|------|--------|--------|
| GET | `/` | 获取工作流列表 | - | Page<WorkflowResponse> |
| GET | `/{id}` | 获取工作流详情 | - | WorkflowResponse |
| POST | `/` | 创建工作流 | WorkflowCreateRequest | WorkflowResponse |
| PUT | `/{id}` | 更新工作流 | WorkflowUpdateRequest | WorkflowResponse |
| DELETE | `/{id}` | 删除工作流 | - | 204 No Content |
| POST | `/{id}/publish` | 发布工作流 | - | WorkflowResponse |
| POST | `/{id}/unpublish` | 取消发布 | - | WorkflowResponse |
| POST | `/{id}/copy` | 复制工作流 | - | WorkflowResponse |
| POST | `/{id}/data` | 批量保存数据 | WorkflowDataRequest | WorkflowResponse |
| GET | `/default` | 获取默认模板 | - | WorkflowResponse |

### 2.2 节点管理 API

基础路径：`/api/workflows/{workflowId}/nodes`

| 方法 | 路径 | 说明 | 请求体 | 响应体 |
|------|------|------|--------|--------|
| GET | `/` | 获取节点列表 | - | List<NodeResponse> |
| GET | `/{nodeUuid}` | 获取单个节点 | - | NodeResponse |
| POST | `/` | 添加节点 | NodeCreateRequest | NodeResponse |
| PUT | `/{nodeUuid}` | 更新节点 | NodeUpdateRequest | NodeResponse |
| DELETE | `/{nodeUuid}` | 删除节点 | - | 204 No Content |

### 2.3 连线管理 API

基础路径：`/api/workflows/{workflowId}/connections`

| 方法 | 路径 | 说明 | 请求体 | 响应体 |
|------|------|------|--------|--------|
| GET | `/` | 获取连线列表 | - | List<ConnectionResponse> |
| POST | `/` | 添加连线 | ConnectionCreateRequest | ConnectionResponse |
| DELETE | `/{connectionUuid}` | 删除连线 | - | 204 No Content |

### 2.4 验证 API

基础路径：`/api/workflows/{workflowId}`

| 方法 | 路径 | 说明 | 请求体 | 响应体 |
|------|------|------|--------|--------|
| POST | `/validate` | 验证工作流 | - | ValidationResult |
| GET | `/predecessors/{nodeUuid}` | 获取前置节点 | - | List<NodeResponse> |
| GET | `/available-variables/{nodeUuid}` | 获取可引用变量 | - | List<AvailableVariable> |

---

## 三、任务清单

### 3.1 WorkflowController

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `WorkflowController.java` | 工作流管理控制器 | 待完成 |

#### 接口定义

```java
@RestController
@RequestMapping("/api/workflows")
public class WorkflowController {

    // ========== CRUD ==========
    @GetMapping
    public Result<Page<WorkflowResponse>> list(@RequestParam(defaultValue = "0") int page,
                                                @RequestParam(defaultValue = "10") int size);

    @GetMapping("/{id}")
    public Result<WorkflowResponse> getById(@PathVariable Long id);

    @PostMapping
    public Result<WorkflowResponse> create(@RequestBody @Valid WorkflowCreateRequest request);

    @PutMapping("/{id}")
    public Result<WorkflowResponse> update(@PathVariable Long id,
                                            @RequestBody @Valid WorkflowUpdateRequest request);

    @DeleteMapping("/{id}")
    public Result<Void> delete(@PathVariable Long id);

    // ========== 状态管理 ==========
    @PostMapping("/{id}/publish")
    public Result<WorkflowResponse> publish(@PathVariable Long id);

    @PostMapping("/{id}/unpublish")
    public Result<WorkflowResponse> unpublish(@PathVariable Long id);

    // ========== 复制 ==========
    @PostMapping("/{id}/copy")
    public Result<WorkflowResponse> copy(@PathVariable Long id);

    // ========== 批量操作 ==========
    @PostMapping("/{id}/data")
    public Result<WorkflowResponse> saveData(@PathVariable Long id,
                                              @RequestBody @Valid WorkflowDataRequest request);

    // ========== 默认模板 ==========
    @GetMapping("/default")
    public Result<WorkflowResponse> getDefault();
}
```

---

### 3.2 WorkflowNodeController

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `WorkflowNodeController.java` | 节点管理控制器 | 待完成 |

#### 接口定义

```java
@RestController
@RequestMapping("/api/workflows/{workflowId}/nodes")
public class WorkflowNodeController {

    @GetMapping
    public Result<List<NodeResponse>> list(@PathVariable Long workflowId);

    @GetMapping("/{nodeUuid}")
    public Result<NodeResponse> getByUuid(@PathVariable Long workflowId,
                                           @PathVariable String nodeUuid);

    @PostMapping
    public Result<NodeResponse> create(@PathVariable Long workflowId,
                                        @RequestBody @Valid NodeCreateRequest request);

    @PutMapping("/{nodeUuid}")
    public Result<NodeResponse> update(@PathVariable Long workflowId,
                                        @PathVariable String nodeUuid,
                                        @RequestBody @Valid NodeUpdateRequest request);

    @DeleteMapping("/{nodeUuid}")
    public Result<Void> delete(@PathVariable Long workflowId,
                                @PathVariable String nodeUuid);
}
```

---

### 3.3 WorkflowConnectionController

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `WorkflowConnectionController.java` | 连线管理控制器 | 待完成 |

#### 接口定义

```java
@RestController
@RequestMapping("/api/workflows/{workflowId}/connections")
public class WorkflowConnectionController {

    @GetMapping
    public Result<List<ConnectionResponse>> list(@PathVariable Long workflowId);

    @PostMapping
    public Result<ConnectionResponse> create(@PathVariable Long workflowId,
                                              @RequestBody @Valid ConnectionCreateRequest request);

    @DeleteMapping("/{connectionUuid}")
    public Result<Void> delete(@PathVariable Long workflowId,
                                @PathVariable String connectionUuid);
}
```

---

### 3.4 WorkflowValidationController

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `WorkflowValidationController.java` | 验证控制器 | 待完成 |

#### 接口定义

```java
@RestController
@RequestMapping("/api/workflows/{workflowId}")
public class WorkflowValidationController {

    @PostMapping("/validate")
    public Result<ValidationResult> validate(@PathVariable Long workflowId);

    @GetMapping("/predecessors/{nodeUuid}")
    public Result<List<NodeResponse>> getPredecessors(@PathVariable Long workflowId,
                                                       @PathVariable String nodeUuid);

    @GetMapping("/available-variables/{nodeUuid}")
    public Result<List<AvailableVariable>> getAvailableVariables(@PathVariable Long workflowId,
                                                                  @PathVariable String nodeUuid);
}
```

---

## 四、DTO 详细设计

### 4.1 WorkflowCreateRequest

```java
@Data
public class WorkflowCreateRequest {

    @NotBlank(message = "工作流名称不能为空")
    @Size(max = 100, message = "工作流名称不能超过100个字符")
    private String name;

    @Size(max = 500, message = "描述不能超过500个字符")
    private String description;

    @Size(max = 50, message = "分类不能超过50个字符")
    private String category;

    private String triggerType;      // MANUAL/SCHEDULE/API
    private String triggerConfig;    // JSON 配置
}
```

### 4.2 WorkflowUpdateRequest

```java
@Data
public class WorkflowUpdateRequest {

    @Size(max = 100, message = "工作流名称不能超过100个字符")
    private String name;

    @Size(max = 500, message = "描述不能超过500个字符")
    private String description;

    @Size(max = 50, message = "分类不能超过50个字符")
    private String category;

    private String triggerType;
    private String triggerConfig;
}
```

### 4.3 WorkflowResponse

```java
@Data
public class WorkflowResponse {
    private Long id;
    private String name;
    private String description;
    private String category;
    private String status;           // DRAFT/PUBLISHED
    private String triggerType;
    private String triggerConfig;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private LocalDateTime publishedAt;
    private String publishedBy;

    // 关联数据
    private List<NodeResponse> nodes;
    private List<ConnectionResponse> connections;
    private List<AssociationResponse> associations;
}
```

### 4.4 WorkflowDataRequest

```java
@Data
public class WorkflowDataRequest {

    private NodeDataRequest nodes;
    private ConnectionDataRequest connections;
    private AssociationDataRequest associations;

    @Data
    public static class NodeDataRequest {
        private List<NodeCreateRequest> created;
        private List<NodeUpdateRequest> updated;
        private List<String> deleted;  // UUID 列表
    }

    @Data
    public static class ConnectionDataRequest {
        private List<ConnectionCreateRequest> created;
        private List<String> deleted;  // UUID 列表
    }

    @Data
    public static class AssociationDataRequest {
        private List<AssociationCreateRequest> created;
        private List<Long> deleted;  // ID 列表
    }
}
```

### 4.5 NodeCreateRequest

```java
@Data
public class NodeCreateRequest {

    @NotBlank(message = "节点名称不能为空")
    @Size(max = 100, message = "节点名称不能超过100个字符")
    private String nodeName;

    @NotBlank(message = "节点类型不能为空")
    private String nodeType;

    @NotNull(message = "X坐标不能为空")
    private Integer positionX;

    @NotNull(message = "Y坐标不能为空")
    private Integer positionY;

    // Skill 节点
    private String skillId;

    // 端口配置
    private String inputPorts;
    private String outputPorts;

    // 参数配置
    private String inputParams;
    private String outputParams;

    // 执行配置
    private String executionLocation;
    private String errorStrategy;
    private Integer retryCount;
    private Integer retryInterval;

    // 条件配置
    private String conditionType;
    private String conditions;

    // 循环配置
    private String loopType;
    private String loopConfig;

    // 其他配置
    private String batchConfig;
    private String asyncConfig;
    private String collectConfig;
}
```

### 4.6 NodeUpdateRequest

```java
@Data
public class NodeUpdateRequest {

    @NotBlank(message = "节点UUID不能为空")
    private String nodeUuid;

    @Size(max = 100, message = "节点名称不能超过100个字符")
    private String nodeName;

    private Integer positionX;
    private Integer positionY;

    // 其他字段同 NodeCreateRequest，均为可选
    private String skillId;
    private String inputPorts;
    private String outputPorts;
    private String inputParams;
    private String outputParams;
    private String executionLocation;
    private String errorStrategy;
    private Integer retryCount;
    private Integer retryInterval;
    private String conditionType;
    private String conditions;
    private String loopType;
    private String loopConfig;
    private String batchConfig;
    private String asyncConfig;
    private String collectConfig;
}
```

### 4.7 NodeResponse

```java
@Data
public class NodeResponse {
    private Long id;
    private String nodeUuid;
    private String nodeName;
    private String nodeType;
    private String nodeCategory;
    private Integer positionX;
    private Integer positionY;

    // Skill 引用
    private String skillId;
    private String skillSnapshot;

    // 端口配置
    private String inputPorts;
    private String outputPorts;

    // 参数配置
    private String inputParams;
    private String outputParams;

    // 执行配置
    private String executionLocation;
    private String errorStrategy;
    private Integer retryCount;
    private Integer retryInterval;
    private Long errorBranchId;

    // 条件配置
    private String conditionType;
    private String conditions;

    // 循环配置
    private String loopType;
    private String loopConfig;

    // 其他配置
    private String batchConfig;
    private String asyncConfig;
    private String collectConfig;

    // 兼容性状态
    private String compatibilityStatus;
}
```

### 4.8 ConnectionCreateRequest

```java
@Data
public class ConnectionCreateRequest {

    @NotBlank(message = "源节点UUID不能为空")
    private String sourceNodeUuid;

    @NotBlank(message = "目标节点UUID不能为空")
    private String targetNodeUuid;

    // 端口配置
    private String sourcePort;
    private String targetPort;

    // 分支配置
    private String branchLabel;
    private Integer branchPriority;
}
```

### 4.9 ConnectionResponse

```java
@Data
public class ConnectionResponse {
    private Long id;
    private String connectionUuid;
    private String sourceNodeUuid;
    private String targetNodeUuid;
    private String sourcePort;
    private String targetPort;
    private String branchLabel;
    private Integer branchPriority;
}
```

### 4.10 ValidationResult

```java
@Data
public class ValidationResult {
    private boolean valid;
    private List<ValidationError> errors;
    private List<ValidationWarning> warnings;

    @Data
    public static class ValidationError {
        private String code;
        private String message;
        private String nodeUuid;      // 关联的节点
        private String field;          // 关联的字段
    }

    @Data
    public static class ValidationWarning {
        private String code;
        private String message;
        private String nodeUuid;
    }
}
```

### 4.11 AvailableVariable

```java
@Data
public class AvailableVariable {
    private String nodeUuid;
    private String nodeName;
    private String paramName;
    private String paramType;
    private String description;
}
```

---

## 五、实施步骤

### Step 1：创建 DTO 类
1. 创建请求类（CreateRequest、UpdateRequest）
2. 创建响应类（Response）
3. 添加校验注解

### Step 2：实现 WorkflowController
1. 创建控制器类
2. 注入 WorkflowService
3. 实现所有接口方法
4. 添加 Swagger 注解

### Step 3：实现 WorkflowNodeController
1. 创建控制器类
2. 注入 WorkflowNodeService
3. 实现所有接口方法

### Step 4：实现 WorkflowConnectionController
1. 创建控制器类
2. 注入 WorkflowConnectionService
3. 实现所有接口方法

### Step 5：实现 WorkflowValidationController
1. 创建控制器类
2. 注入 WorkflowValidator（Phase 4 实现）
3. 实现所有接口方法

### Step 6：全局异常处理
1. 创建 GlobalExceptionHandler
2. 处理业务异常
3. 处理参数校验异常
4. 统一响应格式

---

## 六、代码结构

```
com.example.demo.workflow/
├── controller/
│   ├── WorkflowController.java
│   ├── WorkflowNodeController.java
│   ├── WorkflowConnectionController.java
│   └── WorkflowValidationController.java
│
├── dto/
│   ├── WorkflowCreateRequest.java
│   ├── WorkflowUpdateRequest.java
│   ├── WorkflowResponse.java
│   ├── WorkflowDataRequest.java
│   ├── NodeCreateRequest.java
│   ├── NodeUpdateRequest.java
│   ├── NodeResponse.java
│   ├── ConnectionCreateRequest.java
│   ├── ConnectionResponse.java
│   ├── AssociationCreateRequest.java
│   ├── AssociationResponse.java
│   ├── ValidationResult.java
│   └── AvailableVariable.java
│
└── exception/
    ├── BusinessException.java
    └── GlobalExceptionHandler.java
```

---

## 七、关键代码示例

### 7.1 WorkflowController 完整实现

```java
@RestController
@RequestMapping("/api/workflows")
@Tag(name = "工作流管理", description = "工作流的增删改查及发布等操作")
@RequiredArgsConstructor
public class WorkflowController {

    private final WorkflowService workflowService;

    @GetMapping
    @Operation(summary = "获取工作流列表")
    public Result<Page<WorkflowResponse>> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(required = false) String status,
            @RequestParam(required = false) String category) {

        Pageable pageable = PageRequest.of(page, size);
        Page<WorkflowResponse> result = workflowService.getWorkflowList(pageable);
        return Result.success(result);
    }

    @GetMapping("/{id}")
    @Operation(summary = "获取工作流详情")
    public Result<WorkflowResponse> getById(@PathVariable Long id) {
        WorkflowResponse response = workflowService.getWorkflowById(id);
        return Result.success(response);
    }

    @PostMapping
    @Operation(summary = "创建工作流")
    public Result<WorkflowResponse> create(
            @RequestBody @Valid WorkflowCreateRequest request) {
        WorkflowResponse response = workflowService.createWorkflow(request);
        return Result.success(response);
    }

    @PutMapping("/{id}")
    @Operation(summary = "更新工作流")
    public Result<WorkflowResponse> update(
            @PathVariable Long id,
            @RequestBody @Valid WorkflowUpdateRequest request) {
        WorkflowResponse response = workflowService.updateWorkflow(id, request);
        return Result.success(response);
    }

    @DeleteMapping("/{id}")
    @Operation(summary = "删除工作流")
    public Result<Void> delete(@PathVariable Long id) {
        workflowService.deleteWorkflow(id);
        return Result.success();
    }

    @PostMapping("/{id}/publish")
    @Operation(summary = "发布工作流")
    public Result<WorkflowResponse> publish(@PathVariable Long id) {
        WorkflowResponse response = workflowService.publishWorkflow(id);
        return Result.success(response);
    }

    @PostMapping("/{id}/unpublish")
    @Operation(summary = "取消发布")
    public Result<WorkflowResponse> unpublish(@PathVariable Long id) {
        WorkflowResponse response = workflowService.unpublishWorkflow(id);
        return Result.success(response);
    }

    @PostMapping("/{id}/copy")
    @Operation(summary = "复制工作流")
    public Result<WorkflowResponse> copy(@PathVariable Long id) {
        WorkflowResponse response = workflowService.copyWorkflow(id);
        return Result.success(response);
    }

    @PostMapping("/{id}/data")
    @Operation(summary = "批量保存工作流数据")
    public Result<WorkflowResponse> saveData(
            @PathVariable Long id,
            @RequestBody @Valid WorkflowDataRequest request) {
        WorkflowResponse response = workflowService.saveWorkflowData(id, request);
        return Result.success(response);
    }

    @GetMapping("/default")
    @Operation(summary = "获取默认工作流模板")
    public Result<WorkflowResponse> getDefault() {
        WorkflowResponse response = workflowService.getDefaultWorkflow();
        return Result.success(response);
    }
}
```

### 7.2 GlobalExceptionHandler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusinessException(BusinessException e) {
        return Result.fail(e.getCode(), e.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleValidationException(MethodArgumentNotValidException e) {
        String message = e.getBindingResult().getFieldErrors().stream()
                .map(FieldError::getDefaultMessage)
                .collect(Collectors.joining(", "));
        return Result.fail(ErrorCode.PARAM_ERROR, message);
    }

    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.fail(ErrorCode.SYSTEM_ERROR, "系统异常，请稍后重试");
    }
}
```

---

## 八、验收标准

- [ ] 所有控制器类编译通过
- [ ] 所有 DTO 类包含完整字段
- [ ] 参数校验注解配置正确
- [ ] Swagger 注解配置完整
- [ ] 接口测试通过（Postman/curl）
- [ ] 异常处理正确返回

---

## 九、注意事项

1. **RESTful 规范**：路径使用复数形式，遵循 HTTP 方法语义
2. **参数校验**：使用 `@Valid` 和 JSR-303 注解
3. **统一响应**：使用统一的 `Result` 包装类
4. **Swagger 文档**：添加 `@Operation`、`@Parameter` 等注解
5. **日志记录**：关键操作添加日志

---

## 十、预计工作量

| 类型 | 数量 | 预计时间 |
|------|------|----------|
| DTO 类 | 12 | 2h |
| 控制器 | 4 | 3h |
| 异常处理 | 2 | 1h |
| 接口测试 | 12 接口 | 2h |
| **总计** | **30** | **8h** |

---

## 十一、下一阶段

Phase 3 完成后，进入 **Phase 4：验证逻辑实现**
