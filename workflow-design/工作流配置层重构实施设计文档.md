# 工作流配置层重构设计文档

> 版本：1.0.0
> 日期：2026-03-31
> 作者：AI Test Platform Team

---

## 一、概述

### 1.1 目标

基于 `workflow-design` 设计文档，重构工作流配置层代码，实现：

- 完整的数据模型（实体类更新）
- RESTful API 接口（复数形式 `/api/workflows`）
- 细粒度节点/连线 CRUD + 批量保存
- 完整的验证逻辑（结构、参数引用、循环依赖、Skill 兼容性）
- Skill 变更触发的兼容性检查

### 1.2 范围

**包含**：
- 数据模型更新（Entity、DTO）
- 服务层重构（CRUD + 批量操作）
- 控制器层重构（RESTful API）
- 验证逻辑实现
- Skill 兼容性检查集成

**不包含**：
- 执行引擎（WorkflowScheduler、ExecutionEngine）
- 节点执行器
- WebSocket 实时推送
- Kafka 分布式执行

### 1.3 前置条件

- 数据库表结构已按设计文档更新
- Skill 模块已存在并可用

---

## 二、数据模型

### 2.1 实体类更新

#### WorkflowEntity 新增字段

```java
// 触发配置
private String triggerType;      // MANUAL/SCHEDULE/API
private String triggerConfig;    // JSON 格式的触发配置
```

#### WorkflowNodeEntity 新增字段

```java
// Skill 引用
private String skillId;              // 引用的 Skill ID
private String skillSnapshot;        // Skill 快照（JSON）

// 端口配置
private String inputPorts;           // 输入端口定义（JSON）
private String outputPorts;          // 输出端口定义（JSON）

// 参数配置
private String inputParams;          // 输入参数定义（JSON）
private String outputParams;         // 输出参数定义（JSON）

// 执行配置
private String executionLocation;    // CLIENT/SERVICE
private String errorStrategy;        // STOP/SKIP/RETRY/ERROR_BRANCH
private Integer retryCount;          // 重试次数
private Integer retryInterval;       // 重试间隔（毫秒）
private Long errorBranchId;          // 错误处理分支节点ID

// 条件节点配置
private String conditionType;        // SIMPLE/MULTI
private String conditions;           // 条件配置（JSON）

// 循环节点配置
private String loopType;             // COUNT/ARRAY/CONDITION
private String loopConfig;           // 循环配置（JSON）

// 批处理/异步/收集配置
private String batchConfig;          // 批处理配置（JSON）
private String asyncConfig;          // 异步处理配置（JSON）
private String collectConfig;        // 结果收集配置（JSON）

// 兼容性状态
private String compatibilityStatus;  // COMPATIBLE/NEEDS_UPDATE/INCOMPATIBLE/INVALID
```

#### WorkflowConnectionEntity 新增字段

```java
private String branchLabel;      // 分支标签（true/false/case1/case2/default）
private Integer branchPriority;  // 分支优先级
```

#### WorkflowAssociationEntity 字段更新

```java
// 字段重命名
private Long containerNodeId;    // 原 loopNodeId，支持多种容器类型
private Long bodyNodeId;         // 保持不变
private String associationType;  // LOOP_BODY/BATCH_BODY/ASYNC_BODY
```

### 2.2 新增枚举类

```java
// WorkflowStatus.java
public enum WorkflowStatus {
    DRAFT, PUBLISHED
}

// TriggerType.java
public enum TriggerType {
    MANUAL, SCHEDULE, API
}

// NodeCategory.java
public enum NodeCategory {
    BASIC, LOGIC, EXECUTION
}

// CompatibilityStatus.java
public enum CompatibilityStatus {
    COMPATIBLE, NEEDS_UPDATE, INCOMPATIBLE, INVALID
}

// ExecutionLocation.java
public enum ExecutionLocation {
    CLIENT, SERVICE
}

// ErrorStrategy.java
public enum ErrorStrategy {
    STOP, SKIP, RETRY, ERROR_BRANCH
}
```

---

## 三、API 接口设计

### 3.1 工作流管理 API

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

### 3.2 节点管理 API

基础路径：`/api/workflows/{workflowId}/nodes`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/` | 获取节点列表 |
| GET | `/{nodeUuid}` | 获取单个节点 |
| POST | `/` | 添加节点 |
| PUT | `/{nodeUuid}` | 更新节点 |
| DELETE | `/{nodeUuid}` | 删除节点 |

### 3.3 连线管理 API

基础路径：`/api/workflows/{workflowId}/connections`

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/` | 获取连线列表 |
| POST | `/` | 添加连线 |
| DELETE | `/{connectionUuid}` | 删除连线 |

### 3.4 验证 API

基础路径：`/api/workflows/{workflowId}`

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/validate` | 验证工作流 |
| GET | `/predecessors/{nodeUuid}` | 获取前置节点列表 |
| GET | `/available-variables/{nodeUuid}` | 获取可引用变量列表 |

---

## 四、服务层设计

### 4.1 服务拆分

```
WorkflowService           - 工作流 CRUD、发布、复制、批量保存
WorkflowNodeService       - 节点 CRUD
WorkflowConnectionService - 连线 CRUD
WorkflowAssociationService - 关联 CRUD
WorkflowValidator         - 验证协调
```

### 4.2 WorkflowService 核心方法

```java
public interface WorkflowService {
    // CRUD
    WorkflowResponse createWorkflow(WorkflowCreateRequest request);
    WorkflowResponse getWorkflowById(Long id);
    Page<WorkflowResponse> getWorkflowList(Pageable pageable);
    WorkflowResponse updateWorkflow(Long id, WorkflowUpdateRequest request);
    void deleteWorkflow(Long id);

    // 状态管理
    WorkflowResponse publishWorkflow(Long id);
    WorkflowResponse unpublishWorkflow(Long id);

    // 复制
    WorkflowResponse copyWorkflow(Long id);

    // 批量操作
    WorkflowResponse saveWorkflowData(Long id, WorkflowDataRequest request);

    // 默认模板
    WorkflowResponse getDefaultWorkflow();
}
```

### 4.3 WorkflowNodeService 核心方法

```java
public interface WorkflowNodeService {
    List<NodeResponse> getNodes(Long workflowId);
    NodeResponse getNode(Long workflowId, String nodeUuid);
    NodeResponse createNode(Long workflowId, NodeCreateRequest request);
    NodeResponse updateNode(Long workflowId, String nodeUuid, NodeUpdateRequest request);
    void deleteNode(Long workflowId, String nodeUuid);
}
```

---

## 五、验证逻辑设计

### 5.1 验证器架构

```java
@Service
public class WorkflowValidator {

    private final StructureValidator structureValidator;
    private final ParameterReferenceValidator paramRefValidator;
    private final CyclicDependencyValidator cyclicValidator;
    private final SkillCompatibilityChecker skillChecker;

    public ValidationResult validate(Long workflowId) {
        ValidationResult result = new ValidationResult();

        // 1. 结构验证
        result.merge(structureValidator.validate(workflowId));

        // 2. 参数引用验证
        result.merge(paramRefValidator.validate(workflowId));

        // 3. 循环依赖检测
        result.merge(cyclicValidator.validate(workflowId));

        // 4. Skill 兼容性检查
        result.merge(skillChecker.check(workflowId));

        return result;
    }
}
```

### 5.2 各验证器职责

**StructureValidator**：
- 检查开始节点存在且唯一
- 检查结束节点存在且唯一
- 检查无孤立节点

**ParameterReferenceValidator**：
- 解析参数引用 `${节点名.参数名}`
- 验证引用的节点是前置节点
- 验证引用的参数存在于目标节点的输出

**CyclicDependencyValidator**：
- 构建有向图
- 使用 Kahn 算法检测环

**SkillCompatibilityChecker**：
- 检查 Skill 是否存在
- 检查参数类型兼容性
- 检查必填参数是否已配置
- 更新节点的 compatibilityStatus

### 5.3 Skill 变更监听

```java
@Component
public class SkillChangeListener {

    @EventListener
    public void onSkillChange(SkillChangeEvent event) {
        String skillId = event.getSkillId();

        // 1. 查找所有使用该 Skill 的节点
        List<WorkflowNodeEntity> nodes = nodeMapper.findBySkillId(skillId);

        // 2. 执行兼容性检查
        for (WorkflowNodeEntity node : nodes) {
            CompatibilityStatus status = skillChecker.checkNode(node);
            node.setCompatibilityStatus(status.name());
            nodeMapper.updateById(node);
        }
    }
}
```

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
├── service/
│   ├── WorkflowService.java
│   ├── WorkflowNodeService.java
│   ├── WorkflowConnectionService.java
│   ├── WorkflowAssociationService.java
│   └── impl/
│       ├── WorkflowServiceImpl.java
│       ├── WorkflowNodeServiceImpl.java
│       ├── WorkflowConnectionServiceImpl.java
│       └── WorkflowAssociationServiceImpl.java
│       └── validation/
│           ├── WorkflowValidator.java
│           ├── StructureValidator.java
│           ├── ParameterReferenceValidator.java
│           ├── CyclicDependencyValidator.java
│           └── SkillCompatibilityChecker.java
│
├── entity/
│   ├── WorkflowEntity.java
│   ├── WorkflowNodeEntity.java
│   ├── WorkflowConnectionEntity.java
│   ├── WorkflowAssociationEntity.java
│   └── ...
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
│   ├── ValidationResult.java
│   └── AvailableVariable.java
│
├── mapper/
│   ├── WorkflowMapper.java
│   ├── WorkflowNodeMapper.java
│   ├── WorkflowConnectionMapper.java
│   └── WorkflowAssociationMapper.java
│
├── event/
│   └── SkillChangeListener.java
│
└── enums/
    ├── WorkflowStatus.java
    ├── TriggerType.java
    ├── NodeCategory.java
    ├── CompatibilityStatus.java
    ├── ExecutionLocation.java
    └── ErrorStrategy.java
```

---

## 七、实施计划

### Phase 1：基础层更新
1. 更新实体类（WorkflowEntity、WorkflowNodeEntity、WorkflowConnectionEntity、WorkflowAssociationEntity）
2. 更新 Mapper 接口和 XML 文件
3. 添加枚举类

### Phase 2：服务层重构
1. 拆分 WorkflowService
2. 实现各服务的 CRUD 方法
3. 保留批量保存接口

### Phase 3：控制器层重构
1. 更新 WorkflowController（路径改为复数）
2. 新增 WorkflowNodeController
3. 新增 WorkflowConnectionController
4. 新增 WorkflowValidationController

### Phase 4：验证逻辑实现
1. 实现 WorkflowValidator
2. 实现各子验证器
3. 实现 SkillCompatibilityChecker

### Phase 5：事件集成
1. 实现 SkillChangeListener
2. 集成到 SkillService

---

## 八、预计文件变更

| 类型 | 操作 | 数量 |
|------|------|------|
| Entity | 更新 | 4 |
| DTO | 新增/更新 | 12 |
| Mapper | 更新 | 4 |
| Service | 新增/更新 | 8 |
| Controller | 新增/更新 | 4 |
| Validator | 新增 | 5 |
| Enum | 新增 | 6 |
| Event | 新增 | 1 |

**总计**：约 44 个文件需要新增或更新
