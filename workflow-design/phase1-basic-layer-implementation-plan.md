# Phase 1：基础层更新 - 实施计划

> 版本：1.0.0
> 日期：2026-03-31
> 依赖：数据库表结构已更新

---

## 一、目标

更新工作流配置层的基础代码结构，包括：
- 实体类（Entity）字段更新
- 数据传输对象（DTO）创建/更新
- 枚举类添加
- Mapper 接口和 XML 更新

---

## 二、任务清单

### 2.1 枚举类创建

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `WorkflowStatus.java` | 工作流状态：DRAFT, PUBLISHED | 待完成 |
| 2 | `TriggerType.java` | 触发类型：MANUAL, SCHEDULE, API | 待完成 |
| 3 | `NodeCategory.java` | 节点分类：BASIC, LOGIC, EXECUTION | 待完成 |
| 4 | `CompatibilityStatus.java` | 兼容性状态：COMPATIBLE, NEEDS_UPDATE, INCOMPATIBLE, INVALID | 待完成 |
| 5 | `ExecutionLocation.java` | 执行位置：CLIENT, SERVICE | 待完成 |
| 6 | `ErrorStrategy.java` | 错误策略：STOP, SKIP, RETRY, ERROR_BRANCH | 待完成 |

**目标目录**: `com.example.demo.workflow.enums`

### 2.2 实体类更新

#### 2.2.1 WorkflowEntity 新增字段

```java
// 触发配置
private String triggerType;      // MANUAL/SCHEDULE/API
private String triggerConfig;    // JSON 格式的触发配置
```

#### 2.2.2 WorkflowNodeEntity 新增字段

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

#### 2.2.3 WorkflowConnectionEntity 新增字段

```java
private String branchLabel;      // 分支标签（true/false/case1/case2/default）
private Integer branchPriority;  // 分支优先级
```

#### 2.2.4 WorkflowAssociationEntity 字段更新

```java
// 字段重命名
private Long containerNodeId;    // 原 loopNodeId，支持多种容器类型
private Long bodyNodeId;         // 保持不变
private String associationType;  // LOOP_BODY/BATCH_BODY/ASYNC_BODY
```

### 2.3 DTO 创建/更新

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `WorkflowCreateRequest.java` | 创建工作流请求 | 待完成 |
| 2 | `WorkflowUpdateRequest.java` | 更新工作流请求 | 待完成 |
| 3 | `WorkflowResponse.java` | 工作流响应 | 待完成 |
| 4 | `WorkflowDataRequest.java` | 批量保存数据请求 | 待完成 |
| 5 | `NodeCreateRequest.java` | 创建节点请求 | 待完成 |
| 6 | `NodeUpdateRequest.java` | 更新节点请求 | 待完成 |
| 7 | `NodeResponse.java` | 节点响应 | 待完成 |
| 8 | `ConnectionCreateRequest.java` | 创建连线请求 | 待完成 |
| 9 | `ConnectionResponse.java` | 连线响应 | 待完成 |
| 10 | `ValidationResult.java` | 验证结果 | 待完成 |
| 11 | `AvailableVariable.java` | 可用变量 | 待完成 |

**目标目录**: `com.example.demo.workflow.dto`

### 2.4 Mapper 更新

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `WorkflowMapper.java` | 工作流 Mapper 接口 | 待完成 |
| 2 | `WorkflowMapper.xml` | 工作流 Mapper XML | 待完成 |
| 3 | `WorkflowNodeMapper.java` | 节点 Mapper 接口 | 待完成 |
| 4 | `WorkflowNodeMapper.xml` | 节点 Mapper XML | 待完成 |
| 5 | `WorkflowConnectionMapper.java` | 连线 Mapper 接口 | 待完成 |
| 6 | `WorkflowConnectionMapper.xml` | 连线 Mapper XML | 待完成 |
| 7 | `WorkflowAssociationMapper.java` | 关联 Mapper 接口 | 待完成 |
| 8 | `WorkflowAssociationMapper.xml` | 关联 Mapper XML | 待完成 |

**目标目录**:
- 接口: `com.example.demo.workflow.mapper`
- XML: `resources/mapper/workflow/`

---

## 三、实施步骤

### Step 1：创建枚举类
1. 创建 `enums` 包
2. 依次创建 6 个枚举类
3. 确保枚举值与设计文档一致

### Step 2：更新实体类
1. 更新 `WorkflowEntity` 添加触发配置字段
2. 更新 `WorkflowNodeEntity` 添加所有新字段
3. 更新 `WorkflowConnectionEntity` 添加分支字段
4. 更新 `WorkflowAssociationEntity` 重命名字段并添加类型字段

### Step 3：创建 DTO 类
1. 创建请求类（CreateRequest、UpdateRequest）
2. 创建响应类（Response）
3. 创建验证相关 DTO

### Step 4：更新 Mapper
1. 更新 Mapper 接口添加新方法
2. 更新 Mapper XML 添加新字段映射
3. 添加按 SkillId 查询节点的方法

---

## 四、验收标准

- [ ] 所有枚举类编译通过
- [ ] 所有实体类包含新增字段
- [ ] 所有 DTO 类创建完成
- [ ] Mapper 接口和 XML 更新完成
- [ ] 单元测试通过（如有）

---

## 五、注意事项

1. **字段类型**：JSON 字段使用 String 类型存储
2. **字段命名**：保持与数据库表字段一致（驼峰转下划线）
3. **兼容性**：保留原有字段，仅新增不删除
4. **注解**：使用 Lombok 注解简化代码

---

## 六、预计工作量

| 类型 | 数量 | 预计时间 |
|------|------|----------|
| 枚举类 | 6 | 0.5h |
| 实体类更新 | 4 | 1h |
| DTO 类 | 11 | 2h |
| Mapper 更新 | 8 | 1.5h |
| **总计** | **29** | **5h** |

---

## 七、下一阶段

Phase 1 完成后，进入 **Phase 2：服务层重构**
