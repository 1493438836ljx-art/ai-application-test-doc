# Phase 2：服务层重构 - 实施计划

> 版本：1.0.0
> 日期：2026-03-31
> 依赖：Phase 1 基础层更新完成

---

## 一、目标

重构工作流配置层的服务层代码，实现：
- 服务拆分（WorkflowService、NodeService、ConnectionService、AssociationService）
- 完整的 CRUD 方法实现
- 批量保存接口保留
- 事务管理

---

## 二、服务架构

```
WorkflowService           - 工作流 CRUD、发布、复制、批量保存
WorkflowNodeService       - 节点 CRUD
WorkflowConnectionService - 连线 CRUD
WorkflowAssociationService - 关联 CRUD
```

---

## 三、任务清单

### 3.1 WorkflowService

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `WorkflowService.java` | 工作流服务接口 | 待完成 |
| 2 | `WorkflowServiceImpl.java` | 工作流服务实现 | 待完成 |

#### 接口方法定义

```java
public interface WorkflowService {
    // ========== CRUD ==========
    WorkflowResponse createWorkflow(WorkflowCreateRequest request);
    WorkflowResponse getWorkflowById(Long id);
    Page<WorkflowResponse> getWorkflowList(Pageable pageable);
    WorkflowResponse updateWorkflow(Long id, WorkflowUpdateRequest request);
    void deleteWorkflow(Long id);

    // ========== 状态管理 ==========
    WorkflowResponse publishWorkflow(Long id);
    WorkflowResponse unpublishWorkflow(Long id);

    // ========== 复制 ==========
    WorkflowResponse copyWorkflow(Long id);

    // ========== 批量操作 ==========
    WorkflowResponse saveWorkflowData(Long id, WorkflowDataRequest request);

    // ========== 默认模板 ==========
    WorkflowResponse getDefaultWorkflow();
}
```

#### 核心实现逻辑

**创建工作流**：
1. 校验工作流名称唯一性
2. 创建工作流记录（状态为 DRAFT）
3. 自动创建开始节点和结束节点
4. 返回工作流详情

**发布工作流**：
1. 检查工作流状态（必须为 DRAFT）
2. 调用验证器进行完整验证
3. 验证通过后更新状态为 PUBLISHED
4. 记录发布时间和发布人

**取消发布**：
1. 检查工作流状态（必须为 PUBLISHED）
2. 检查是否有正在执行的任务
3. 更新状态为 DRAFT

**复制工作流**：
1. 查询源工作流及所有节点、连线、关联
2. 创建新工作流（名称加 "-副本" 后缀）
3. 复制所有节点（生成新 UUID）
4. 复制所有连线（更新源/目标节点 UUID）
5. 复制所有关联（更新容器/子节点 UUID）

**批量保存数据**：
1. 校验工作流存在且为 DRAFT 状态
2. 处理节点列表（新增/更新/删除）
3. 处理连线列表（新增/删除）
4. 处理关联列表（新增/删除）
5. 更新工作流更新时间

---

### 3.2 WorkflowNodeService

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `WorkflowNodeService.java` | 节点服务接口 | 待完成 |
| 2 | `WorkflowNodeServiceImpl.java` | 节点服务实现 | 待完成 |

#### 接口方法定义

```java
public interface WorkflowNodeService {
    // ========== CRUD ==========
    List<NodeResponse> getNodes(Long workflowId);
    NodeResponse getNode(Long workflowId, String nodeUuid);
    NodeResponse createNode(Long workflowId, NodeCreateRequest request);
    NodeResponse updateNode(Long workflowId, String nodeUuid, NodeUpdateRequest request);
    void deleteNode(Long workflowId, String nodeUuid);

    // ========== 批量操作 ==========
    void batchCreateNodes(Long workflowId, List<NodeCreateRequest> requests);
    void batchUpdateNodes(Long workflowId, List<NodeUpdateRequest> requests);
    void batchDeleteNodes(Long workflowId, List<String> nodeUuids);
}
```

#### 核心实现逻辑

**创建节点**：
1. 校验工作流存在且为 DRAFT 状态
2. 校验节点类型有效性
3. 如果是 Skill 节点，校验 SkillId 有效性
4. 生成节点 UUID
5. 设置节点默认配置
6. 保存节点记录

**更新节点**：
1. 校验节点存在且属于该工作流
2. 校验工作流状态为 DRAFT
3. 更新节点配置
4. 如果更新了 SkillId，触发兼容性检查

**删除节点**：
1. 校验节点存在且属于该工作流
2. 校验工作流状态为 DRAFT
3. 校验节点不是开始或结束节点
4. 删除相关连线
5. 删除相关关联
6. 删除节点记录

---

### 3.3 WorkflowConnectionService

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `WorkflowConnectionService.java` | 连线服务接口 | 待完成 |
| 2 | `WorkflowConnectionServiceImpl.java` | 连线服务实现 | 待完成 |

#### 接口方法定义

```java
public interface WorkflowConnectionService {
    // ========== CRUD ==========
    List<ConnectionResponse> getConnections(Long workflowId);
    ConnectionResponse createConnection(Long workflowId, ConnectionCreateRequest request);
    void deleteConnection(Long workflowId, String connectionUuid);

    // ========== 批量操作 ==========
    void batchCreateConnections(Long workflowId, List<ConnectionCreateRequest> requests);
    void batchDeleteConnections(Long workflowId, List<String> connectionUuids);
}
```

#### 核心实现逻辑

**创建连线**：
1. 校验工作流存在且为 DRAFT 状态
2. 校验源节点和目标节点存在
3. 校验不能自连接（源节点 != 目标节点）
4. 校验连线不重复
5. 生成连线 UUID
6. 保存连线记录

**删除连线**：
1. 校验连线存在且属于该工作流
2. 校验工作流状态为 DRAFT
3. 删除连线记录

---

### 3.4 WorkflowAssociationService

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `WorkflowAssociationService.java` | 关联服务接口 | 待完成 |
| 2 | `WorkflowAssociationServiceImpl.java` | 关联服务实现 | 待完成 |

#### 接口方法定义

```java
public interface WorkflowAssociationService {
    // ========== CRUD ==========
    List<AssociationResponse> getAssociations(Long workflowId);
    AssociationResponse createAssociation(Long workflowId, AssociationCreateRequest request);
    void deleteAssociation(Long workflowId, Long associationId);

    // ========== 批量操作 ==========
    void batchCreateAssociations(Long workflowId, List<AssociationCreateRequest> requests);
    void batchDeleteAssociations(Long workflowId, List<Long> associationIds);

    // ========== 查询 ==========
    List<AssociationResponse> getByContainerNode(Long workflowId, String containerNodeUuid);
    List<AssociationResponse> getByBodyNode(Long workflowId, String bodyNodeUuid);
}
```

#### 核心实现逻辑

**创建关联**：
1. 校验工作流存在且为 DRAFT 状态
2. 校验容器节点和子节点存在
3. 校验关联类型有效性
4. 保存关联记录

**删除关联**：
1. 校验关联存在且属于该工作流
2. 校验工作流状态为 DRAFT
3. 删除关联记录

---

## 四、实施步骤

### Step 1：创建服务接口
1. 创建 `WorkflowService` 接口
2. 创建 `WorkflowNodeService` 接口
3. 创建 `WorkflowConnectionService` 接口
4. 创建 `WorkflowAssociationService` 接口

### Step 2：实现 WorkflowService
1. 实现 CRUD 方法
2. 实现发布/取消发布方法
3. 实现复制方法
4. 实现批量保存方法
5. 实现默认模板方法

### Step 3：实现 WorkflowNodeService
1. 实现 CRUD 方法
2. 实现批量操作方法

### Step 4：实现 WorkflowConnectionService
1. 实现 CRUD 方法
2. 实现批量操作方法

### Step 5：实现 WorkflowAssociationService
1. 实现 CRUD 方法
2. 实现批量操作方法
3. 实现查询方法

### Step 6：事务配置
1. 配置事务管理器
2. 添加事务注解
3. 处理事务传播

---

## 五、代码结构

```
com.example.demo.workflow.service/
├── WorkflowService.java
├── WorkflowNodeService.java
├── WorkflowConnectionService.java
├── WorkflowAssociationService.java
└── impl/
    ├── WorkflowServiceImpl.java
    ├── WorkflowNodeServiceImpl.java
    ├── WorkflowConnectionServiceImpl.java
    └── WorkflowAssociationServiceImpl.java
```

---

## 六、关键代码示例

### 6.1 WorkflowServiceImpl - 发布工作流

```java
@Override
@Transactional(rollbackFor = Exception.class)
public WorkflowResponse publishWorkflow(Long id) {
    // 1. 查询工作流
    WorkflowEntity workflow = workflowMapper.selectById(id);
    if (workflow == null) {
        throw new BusinessException("工作流不存在");
    }

    // 2. 校验状态
    if (!WorkflowStatus.DRAFT.name().equals(workflow.getStatus())) {
        throw new BusinessException("只有草稿状态的工作流才能发布");
    }

    // 3. 执行验证（Phase 4 实现）
    ValidationResult result = workflowValidator.validate(id);
    if (!result.isValid()) {
        throw new BusinessException("工作流验证失败：" + result.getErrors());
    }

    // 4. 更新状态
    workflow.setStatus(WorkflowStatus.PUBLISHED.name());
    workflow.setPublishedAt(LocalDateTime.now());
    workflow.setPublishedBy(SecurityUtils.getCurrentUserId());
    workflowMapper.updateById(workflow);

    // 5. 返回结果
    return buildWorkflowResponse(workflow);
}
```

### 6.2 WorkflowServiceImpl - 批量保存

```java
@Override
@Transactional(rollbackFor = Exception.class)
public WorkflowResponse saveWorkflowData(Long id, WorkflowDataRequest request) {
    // 1. 校验工作流
    WorkflowEntity workflow = validateWorkflowExists(id);
    validateWorkflowStatus(workflow, WorkflowStatus.DRAFT);

    // 2. 处理节点
    if (request.getNodes() != null) {
        // 删除
        if (request.getNodes().getDeleted() != null) {
            nodeService.batchDeleteNodes(id, request.getNodes().getDeleted());
        }
        // 新增
        if (request.getNodes().getCreated() != null) {
            nodeService.batchCreateNodes(id, request.getNodes().getCreated());
        }
        // 更新
        if (request.getNodes().getUpdated() != null) {
            nodeService.batchUpdateNodes(id, request.getNodes().getUpdated());
        }
    }

    // 3. 处理连线
    if (request.getConnections() != null) {
        if (request.getConnections().getDeleted() != null) {
            connectionService.batchDeleteConnections(id, request.getConnections().getDeleted());
        }
        if (request.getConnections().getCreated() != null) {
            connectionService.batchCreateConnections(id, request.getConnections().getCreated());
        }
    }

    // 4. 处理关联
    if (request.getAssociations() != null) {
        if (request.getAssociations().getDeleted() != null) {
            associationService.batchDeleteAssociations(id, request.getAssociations().getDeleted());
        }
        if (request.getAssociations().getCreated() != null) {
            associationService.batchCreateAssociations(id, request.getAssociations().getCreated());
        }
    }

    // 5. 更新工作流时间
    workflow.setUpdatedAt(LocalDateTime.now());
    workflowMapper.updateById(workflow);

    return getWorkflowById(id);
}
```

### 6.3 WorkflowNodeServiceImpl - 创建节点

```java
@Override
public NodeResponse createNode(Long workflowId, NodeCreateRequest request) {
    // 1. 校验工作流
    WorkflowEntity workflow = validateWorkflowExists(workflowId);
    validateWorkflowStatus(workflow, WorkflowStatus.DRAFT);

    // 2. 校验节点类型
    validateNodeType(request.getNodeType());

    // 3. 如果是 Skill 节点，校验 Skill
    if (NodeType.SKILL.name().equals(request.getNodeType())) {
        validateSkillExists(request.getSkillId());
    }

    // 4. 创建节点实体
    WorkflowNodeEntity node = new WorkflowNodeEntity();
    node.setWorkflowId(workflowId);
    node.setNodeUuid(UUID.randomUUID().toString());
    node.setNodeName(request.getNodeName());
    node.setNodeType(request.getNodeType());
    node.setNodeCategory(determineNodeCategory(request.getNodeType()));
    node.setPositionX(request.getPositionX());
    node.setPositionY(request.getPositionY());
    node.setSkillId(request.getSkillId());
    node.setExecutionLocation(ExecutionLocation.SERVICE.name());
    node.setErrorStrategy(ErrorStrategy.STOP.name());
    node.setCompatibilityStatus(CompatibilityStatus.COMPATIBLE.name());

    // 5. 保存
    nodeMapper.insert(node);

    return buildNodeResponse(node);
}
```

---

## 七、验收标准

- [ ] 所有服务接口定义完成
- [ ] 所有服务实现类编译通过
- [ ] CRUD 方法单元测试通过
- [ ] 批量操作方法单元测试通过
- [ ] 事务管理正确配置
- [ ] 异常处理完整

---

## 八、注意事项

1. **事务边界**：批量操作需要在同一事务中执行
2. **并发控制**：使用乐观锁防止并发修改
3. **异常处理**：使用统一的业务异常类
4. **日志记录**：关键操作添加日志
5. **状态校验**：所有修改操作必须校验工作流状态

---

## 九、预计工作量

| 类型 | 数量 | 预计时间 |
|------|------|----------|
| 服务接口 | 4 | 1h |
| 服务实现 | 4 | 6h |
| 单元测试 | 8 | 3h |
| **总计** | **16** | **10h** |

---

## 十、下一阶段

Phase 2 完成后，进入 **Phase 3：控制器层重构**
