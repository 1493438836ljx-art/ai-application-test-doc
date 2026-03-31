# 工作流配置设计（逻辑框架）

> 版本：1.0.0
> 日期：2026-03-30
> 作者：AI Test Platform Team

---

## 一、概述

### 1.1 设计目标

设计一个灵活、可扩展的工作流配置框架，支持：

- 从 Skill 库动态加载功能节点
- 可视化拖拽式工作流编排
- 灵活的参数绑定和引用机制
- 多种控制节点（循环、批处理、异步、条件分支等）
- 完整的工作流生命周期管理

### 1.2 设计原则

1. **混合节点模式**：基础节点固定，功能节点从 Skill 库选择
2. **参数前置引用**：只能引用前置节点的输出，保证执行可确定性
3. **配置与执行分离**：配置层只负责定义和验证，不关心执行细节
4. **兼容性优先**：Skill 变更时进行兼容性检查，保护已有工作流

---

## 二、节点类型体系

### 2.1 节点类型总览

| 节点类型 | 代码 | 分类 | 说明 |
|---------|------|------|------|
| 开始节点 | `start` | BASIC | 工作流入口，定义初始参数 |
| 结束节点 | `end` | BASIC | 工作流出口，定义输出参数 |
| 简单条件 | `condition_simple` | LOGIC | 二分支条件判断 |
| 多路条件 | `condition_multi` | LOGIC | 多分支条件判断 |
| 循环节点 | `loop` | LOGIC | 顺序遍历执行 |
| 批处理节点 | `batch` | LOGIC | 并发批量执行，等待全部完成 |
| 异步处理节点 | `async` | LOGIC | 异步执行子工作流，不阻塞主流程 |
| 结果收集节点 | `collect` | LOGIC | 等待并收集异步任务的输出结果 |
| 技能节点 | `skill` | EXECUTION | 执行 Skill 库中的技能 |

### 2.2 节点分类

```
节点类型
├── BASIC (基础节点)
│   ├── start    - 开始节点
│   └── end      - 结束节点
│
├── LOGIC (逻辑控制节点)
│   ├── condition_simple - 简单条件分支
│   ├── condition_multi  - 多路条件分支
│   ├── loop             - 循环处理
│   ├── batch            - 批处理
│   ├── async            - 异步处理
│   └── collect          - 结果收集
│
└── EXECUTION (执行节点)
    └── skill             - 技能执行（从 Skill 库动态加载）
```

---

## 三、数据模型设计

### 3.1 数据库表结构

#### 3.1.1 工作流主表 (workflow)

```sql
CREATE TABLE workflow (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    name            VARCHAR(100) NOT NULL COMMENT '工作流名称',
    description     TEXT COMMENT '工作流描述',
    status          VARCHAR(20) NOT NULL DEFAULT 'DRAFT' COMMENT '状态: DRAFT/PUBLISHED',
    version         INT NOT NULL DEFAULT 1 COMMENT '版本号',
    trigger_type    VARCHAR(20) DEFAULT 'MANUAL' COMMENT '触发类型: MANUAL/SCHEDULE/API',
    trigger_config  JSON COMMENT '触发配置(如定时规则)',
    created_by      VARCHAR(100) COMMENT '创建人',
    updated_by      VARCHAR(100) COMMENT '更新人',
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted         BOOLEAN DEFAULT FALSE COMMENT '逻辑删除标记',
    INDEX idx_status (status),
    INDEX idx_created_by (created_by)
) COMMENT '工作流主表';
```

#### 3.1.2 工作流节点表 (workflow_node)

```sql
CREATE TABLE workflow_node (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    workflow_id     BIGINT NOT NULL COMMENT '所属工作流ID',
    node_uuid       VARCHAR(50) NOT NULL COMMENT '节点UUID(前端生成)',
    node_type       VARCHAR(50) NOT NULL COMMENT '节点类型',
    node_name       VARCHAR(100) NOT NULL COMMENT '节点名称(显示名)',

    -- Skill 引用(仅 skill 类型节点使用)
    skill_id        VARCHAR(50) COMMENT '引用的Skill ID',
    skill_snapshot  JSON COMMENT 'Skill快照(创建时的参数定义)',

    -- 位置信息
    position_x      INT COMMENT '画布X坐标',
    position_y      INT COMMENT '画布Y坐标',

    -- 参数配置(JSON格式)
    input_config    JSON COMMENT '输入参数配置',
    output_config   JSON COMMENT '输出参数配置',

    -- 执行位置配置
    execution_location VARCHAR(20) COMMENT '执行位置: CLIENT/SERVICE',

    -- 错误处理配置
    error_strategy  VARCHAR(20) DEFAULT 'STOP' COMMENT '错误策略: STOP/SKIP/RETRY/ERROR_BRANCH',
    retry_count     INT DEFAULT 3 COMMENT '重试次数',
    retry_interval  INT DEFAULT 1000 COMMENT '重试间隔(毫秒)',
    error_branch_id BIGINT COMMENT '错误处理分支节点ID',

    -- 条件节点专用
    condition_type  VARCHAR(20) COMMENT '条件类型: SIMPLE/MULTI',
    conditions      JSON COMMENT '条件表达式配置',

    -- 循环节点专用
    loop_type       VARCHAR(20) COMMENT '循环类型: COUNT/ARRAY/CONDITION',
    loop_config     JSON COMMENT '循环配置',

    -- 批处理节点专用
    batch_config    JSON COMMENT '批处理配置',

    -- 异步处理节点专用
    async_config    JSON COMMENT '异步处理配置',

    -- 结果收集节点专用
    collect_config  JSON COMMENT '结果收集配置',

    -- 父节点(用于循环体/批处理体内节点)
    parent_node_id  BIGINT COMMENT '父节点ID(循环/批处理节点)',

    -- 兼容性状态
    compatibility_status VARCHAR(20) DEFAULT 'COMPATIBLE' COMMENT '兼容性状态: COMPATIBLE/NEEDS_UPDATE/INCOMPATIBLE/INVALID',

    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_workflow_id (workflow_id),
    INDEX idx_skill_id (skill_id),
    INDEX idx_parent_node (parent_node_id),
    FOREIGN KEY (workflow_id) REFERENCES workflow(id)
) COMMENT '工作流节点表';
```

#### 3.1.3 工作流连线表 (workflow_connection)

```sql
CREATE TABLE workflow_connection (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    workflow_id     BIGINT NOT NULL COMMENT '所属工作流ID',
    connection_uuid VARCHAR(50) NOT NULL COMMENT '连线UUID',
    source_node_id  BIGINT NOT NULL COMMENT '源节点ID',
    target_node_id  BIGINT NOT NULL COMMENT '目标节点ID',

    -- 条件分支专用
    branch_label    VARCHAR(50) COMMENT '分支标签(如: true/false/case1/case2/default)',
    branch_priority INT COMMENT '分支优先级(多路分支时使用)',

    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_workflow_id (workflow_id),
    INDEX idx_source_node (source_node_id),
    INDEX idx_target_node (target_node_id),
    FOREIGN KEY (workflow_id) REFERENCES workflow(id),
    FOREIGN KEY (source_node_id) REFERENCES workflow_node(id),
    FOREIGN KEY (target_node_id) REFERENCES workflow_node(id)
) COMMENT '工作流连线表';
```

#### 3.1.4 工作流关联表 (workflow_association)

```sql
CREATE TABLE workflow_association (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    workflow_id     BIGINT NOT NULL COMMENT '所属工作流ID',
    container_node_id BIGINT NOT NULL COMMENT '容器节点ID(循环/批处理/异步)',
    body_node_id    BIGINT NOT NULL COMMENT '内部节点ID',
    association_type VARCHAR(20) NOT NULL COMMENT '关联类型: LOOP_BODY/BATCH_BODY/ASYNC_BODY',
    display_order   INT COMMENT '显示顺序',
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_workflow_id (workflow_id),
    INDEX idx_container_node (container_node_id),
    FOREIGN KEY (workflow_id) REFERENCES workflow(id)
) COMMENT '工作流关联表(容器节点与内部节点的关系)';
```

---

### 3.2 JSON 字段结构定义

#### 3.2.1 input_config (输入参数配置)

```json
{
  "parameters": [
    {
      "name": "input_file",
      "type": "File",
      "fileType": "Excel",
      "valueSource": {
        "type": "reference",
        "value": "${表格提取.output_file}"
      },
      "required": true
    },
    {
      "name": "cols",
      "type": "String",
      "valueSource": {
        "type": "literal",
        "value": "A,B,C"
      },
      "required": true
    }
  ]
}
```

**valueSource 类型说明：**

| type | 说明 | value 示例 |
|------|------|-----------|
| `reference` | 引用前置节点输出 | `${节点名.参数名}` |
| `literal` | 固定值 | 任意 JSON 值 |

#### 3.2.2 output_config (输出参数配置)

```json
{
  "parameters": [
    {
      "name": "output_file",
      "type": "File",
      "fileType": "Excel",
      "description": "清洗后的文件"
    }
  ]
}
```

#### 3.2.3 conditions (条件配置 - 简单分支)

```json
{
  "type": "SIMPLE",
  "expression": {
    "leftOperand": "${节点A.output}",
    "operator": "equals",
    "rightOperand": "success"
  }
}
```

#### 3.2.4 conditions (条件配置 - 多路分支)

```json
{
  "type": "MULTI",
  "cases": [
    {
      "id": "case1",
      "label": "分支A",
      "priority": 1,
      "expression": {
        "leftOperand": "${判断节点.score}",
        "operator": "greaterThan",
        "rightOperand": 80
      }
    },
    {
      "id": "case2",
      "label": "分支B",
      "priority": 2,
      "expression": {
        "leftOperand": "${判断节点.score}",
        "operator": "between",
        "rightOperand": [60, 80]
      }
    }
  ],
  "defaultCase": {
    "id": "default",
    "label": "默认分支"
  }
}
```

**支持的操作符：**

| 类别 | 操作符 |
|------|--------|
| 字符串 | `equals`, `notEquals`, `contains`, `startsWith`, `endsWith`, `isEmpty`, `isNotEmpty` |
| 数值 | `equals`, `notEquals`, `greaterThan`, `lessThan`, `greaterThanOrEqual`, `lessThanOrEqual`, `between` |
| 布尔 | `isTrue`, `isFalse` |
| 数组 | `isEmpty`, `isNotEmpty`, `contains`, `sizeEquals`, `sizeGreaterThan` |

#### 3.2.5 loop_config (循环配置)

```json
// 计数循环
{
  "type": "COUNT",
  "times": 5,
  "maxIterations": 1000,
  "outputParams": [
    {
      "name": "processed_files",
      "type": "Array",
      "elementType": "File",
      "source": "${循环体节点.output_file}"
    }
  ]
}

// 数组遍历
{
  "type": "ARRAY",
  "arraySource": {
    "type": "reference",
    "value": "${表格提取.output}"
  },
  "maxIterations": 1000,
  "outputParams": [...]
}

// 条件循环
{
  "type": "CONDITION",
  "condition": {
    "leftOperand": "${计数器.value}",
    "operator": "lessThan",
    "rightOperand": "10"
  },
  "maxIterations": 1000,
  "outputParams": [...]
}
```

#### 3.2.6 batch_config (批处理配置)

```json
{
  "type": "ARRAY",
  "arraySource": {
    "type": "reference",
    "value": "${表格提取.output}"
  },
  "concurrency": 5,
  "failureStrategy": "CONTINUE_OTHERS",
  "outputMode": "MERGE",
  "maxIterations": 100
}
```

**批处理配置说明：**

| 字段 | 说明 | 可选值 |
|------|------|--------|
| `type` | 固定为 `ARRAY` | `ARRAY` |
| `arraySource` | 输入数组来源 | - |
| `concurrency` | 最大并发数 | 1-100 |
| `failureStrategy` | 失败策略 | `STOP_ALL`, `CONTINUE_OTHERS` |
| `outputMode` | 输出模式 | `MERGE`, `SEPARATE` |

#### 3.2.7 async_config (异步处理配置)

```json
{
  "type": "ARRAY",
  "arraySource": {
    "type": "reference",
    "value": "${表格提取.output}"
  },
  "concurrency": 10,
  "triggerMode": "FIRE_AND_FORGET",
  "callbackWorkflowId": null,
  "timeout": 3600000
}
```

**异步处理配置说明：**

| 字段 | 说明 | 可选值 |
|------|------|--------|
| `type` | 执行类型 | `ARRAY`(遍历), `SINGLE`(单次) |
| `arraySource` | 输入数组来源 | - |
| `concurrency` | 最大并发数 | 1-100 |
| `triggerMode` | 触发模式 | `FIRE_AND_FORGET`, `CALLBACK` |
| `callbackWorkflowId` | 回调工作流ID | - |
| `timeout` | 子工作流超时(毫秒) | - |

#### 3.2.8 collect_config (结果收集配置)

```json
{
  "asyncNodeId": "async-uuid-001",
  "waitStrategy": "ALL",
  "waitCount": null,
  "timeout": 300000,
  "outputMode": "MERGE",
  "failureHandling": "PARTIAL_SUCCESS"
}
```

**结果收集配置说明：**

| 字段 | 说明 | 可选值 |
|------|------|--------|
| `asyncNodeId` | 关联的异步处理节点UUID | - |
| `waitStrategy` | 等待策略 | `ALL`, `FIRST`, `N` |
| `waitCount` | 等待完成的数量(N模式) | - |
| `timeout` | 等待超时(毫秒) | - |
| `outputMode` | 输出模式 | `MERGE`, `SEPARATE`, `FIRST_SUCCESS` |
| `failureHandling` | 失败处理 | `FAIL_FAST`, `PARTIAL_SUCCESS`, `IGNORE_FAILURES` |

---

## 四、控制节点详细设计

### 4.1 循环节点 (Loop)

#### 4.1.1 节点结构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         循环节点结构                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   主工作流画布:                                                          │
│   ┌──────┐      ┌────────────────────┐      ┌──────┐                   │
│   │ 开始 │─────▶│   循环节点 (Loop)   │─────▶│ 结束 │                   │
│   └──────┘      │  ┌──────────────┐  │      └──────┘                   │
│                 │  │  循环体画布   │  │                                 │
│                 │  │ ┌────┐ ┌────┐│  │                                 │
│                 │  │ │节点1│→│节点2││  │                                 │
│                 │  │ └────┘ └────┘│  │                                 │
│                 │  └──────────────┘  │                                 │
│                 └────────────────────┘                                 │
│                                                                         │
│   循环体内部可用变量:                                                    │
│   - current_item: 当前遍历元素 (数组遍历时)                              │
│   - current_index: 当前索引 (从 0 开始)                                  │
│   - 外部变量: 通过 ${外部节点名.变量名} 引用                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 4.1.2 循环类型对比

| 循环类型 | 配置 | 可用变量 | 适用场景 |
|---------|------|---------|---------|
| 计数循环 | `times=N` | `current_index` | 固定次数重复 |
| 数组遍历 | `arraySource` | `current_item`, `current_index` | 遍历数组元素 |
| 条件循环 | `condition` | `current_index` | 不确定次数循环 |

#### 4.1.3 循环输出收集

循环节点可以定义输出参数，收集每次迭代的输出：

```json
{
  "outputParams": [
    {
      "name": "processed_files",
      "type": "Array",
      "elementType": "File",
      "source": "${循环体节点.output_file}"
    }
  ]
}
```

最终输出为 `Array<elementType>` 类型。

---

### 4.2 批处理节点 (Batch)

#### 4.2.1 节点结构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        批处理节点结构                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   主工作流画布:                                                          │
│   ┌──────┐      ┌────────────────────┐      ┌──────┐                   │
│   │ 开始 │─────▶│  批处理节点(Batch)  │─────▶│ 结束 │                   │
│   └──────┘      │  并发执行子工作流   │      └──────┘                   │
│                 │                    │                                 │
│                 │  ┌────┐ ┌────┐    │                                 │
│                 │  │子1 │ │子2 │ ...│  (并发执行)                      │
│                 │  └────┘ └────┘    │                                 │
│                 └────────────────────┘                                 │
│                                                                         │
│   批处理与循环的区别:                                                    │
│   - 循环 (Loop): 顺序执行，等待每次完成                                  │
│   - 批处理 (Batch): 并发执行，等待全部完成                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 4.2.2 批处理与循环节点对比

| 特性 | 循环节点 (Loop) | 批处理节点 (Batch) |
|------|----------------|-------------------|
| 执行方式 | 顺序执行 | 并发执行 |
| 数组遍历 | ✅ | ✅ |
| current_item | ✅ | ✅ |
| current_index | ✅ | ✅ |
| 并发控制 | ❌ | ✅ 可配置并发数 |
| 失败策略 | 节点级配置 | 支持全局失败策略 |
| 输出收集 | 顺序收集 | 异步收集后合并 |
| 阻塞主流程 | ✅ 阻塞 | ✅ 阻塞 |
| 适用场景 | 有依赖的顺序处理 | 独立任务的批量处理 |

---

### 4.3 异步处理节点 (Async)

#### 4.3.1 节点结构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        异步处理节点结构                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   主工作流:                                                              │
│   ┌──────┐   ┌────────┐   ┌────────┐   ┌────────────┐   ┌──────┐      │
│   │ 开始 │──▶│ 数据准备│──▶│ 异步A  │──▶│ 结果收集A  │──▶│ 结束 │      │
│   └──────┘   └────────┘   └────────┘   └────────────┘   └──────┘      │
│                            │               ▲                            │
│                            │               │                            │
│                            ▼               │                            │
│                    ┌───────────────┐       │                            │
│                    │ 子工作流 1    │       │                            │
│                    │ 子工作流 2    │───────┘                            │
│                    │ 子工作流 N    │  完成后输出                         │
│                    └───────────────┘  到收集节点                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 4.3.2 异步处理 vs 循环 vs 批处理

| 特性 | 循环节点 | 批处理节点 | 异步处理节点 |
|------|---------|-----------|-------------|
| 执行方式 | 顺序执行 | 并发执行 | 异步触发 |
| 阻塞主流程 | ✅ 阻塞 | ✅ 阻塞 | ❌ 不阻塞 |
| 等待子流程完成 | ✅ 等待 | ✅ 等待全部 | ❌ 触发后立即继续 |
| 数组遍历 | ✅ | ✅ | ✅ |
| 并发控制 | ❌ | ✅ | ✅ |
| current_item/index | ✅ | ✅ | ✅ |
| 输出收集 | 顺序收集 | 合并收集 | 不收集/回调收集 |
| 适用场景 | 有依赖的顺序处理 | 独立任务批量处理 | 后台任务/通知/日志记录 |

---

### 4.4 结果收集节点 (Collect)

#### 4.4.1 节点功能

结果收集节点用于等待并收集异步处理节点触发的子工作流输出。

#### 4.4.2 等待策略

| 策略 | 说明 |
|------|------|
| `ALL` | 等待所有子任务完成 |
| `FIRST` | 等待第一个子任务完成 |
| `N` | 等待指定数量的子任务完成 |

#### 4.4.3 输出示例

假设异步处理节点触发了 3 个子工作流：

```json
{
  "async_results": [
    { "id": 1, "status": "success", "data": "result_1" },
    { "id": 2, "status": "success", "data": "result_2" },
    { "id": 3, "status": "success", "data": "result_3" }
  ],
  "total_count": 3,
  "success_count": 3,
  "failed_count": 0
}
```

---

### 4.5 条件分支节点 (Condition)

#### 4.5.1 简单条件分支

```
        ┌─ true ─→ 分支A
条件判断 ┤
        └─ false ─→ 分支B
```

#### 4.5.2 多路条件分支

```
        ┌─ 条件1 ─→ 分支A
        ├─ 条件2 ─→ 分支B
条件判断 ┤
        ├─ 条件3 ─→ 分支C
        └─ 默认 ─→ 分支D
```

---

## 五、参数引用机制

### 5.1 引用格式

```
${节点名称.参数名}
```

示例：
- `${开始.input_file}` - 引用开始节点的 input_file 参数
- `${文本清洗.output_file}` - 引用文本清洗节点的 output_file 参数
- `${循环.current_item}` - 引用循环节点的当前元素

### 5.2 前置节点计算

只有前置节点的输出才能被引用。前置节点的计算规则：

1. **线性流程**：直接前驱节点
2. **分支流程**：所有可能执行到当前节点的路径上的节点
3. **循环体内**：循环节点 + 循环体内在当前节点之前的节点

### 5.3 前置节点计算算法

```
算法：计算前置节点集合

输入：工作流定义 W, 目标节点 T
输出：前置节点集合 P

1. P = ∅
2. 构建反向邻接表 G（从连线中提取）
3. DFS(T, G):
   - 如果 T 已访问，返回
   - 标记 T 为已访问
   - 对于 G[T] 中的每个前驱节点 N:
     - DFS(N, G)
   - P = P ∪ {T}
4. 返回 P - {T}
```

---

## 六、Skill 集成设计

### 6.1 Skill 节点动态加载

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Skill 节点动态加载流程                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   前端 (工作流编辑器)                                                    │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │  1. 加载 Skill 列表                                              │  │
│   │     GET /api/skills?status=PUBLISHED                             │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                            │                                            │
│                            ▼                                            │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │  2. 转换为节点类型定义                                           │  │
│   │     {                                                            │  │
│   │       type: 'skill_${skillId}',                                  │  │
│   │       name: skill.name,                                          │  │
│   │       category: 'EXECUTION',                                     │  │
│   │       skillId: skill.id,                                         │  │
│   │       inputParams: skill.inputParameters,                        │  │
│   │       outputParams: skill.outputParameters                       │  │
│   │     }                                                            │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                            │                                            │
│                            ▼                                            │
│   ┌─────────────────────────────────────────────────────────────────┐  │
│   │  3. 渲染到节点面板                                               │  │
│   └─────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 Skill 节点配置面板

```
┌─────────────────────────────────────────────────────────────────────┐
│  节点配置面板                                                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 基本信息                                                     │   │
│  │  节点名称: [文本清洗-步骤1        ]                          │   │
│  │  执行位置: ○ Service端  ● Client端                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 输入参数                                                     │   │
│  │  input_file (File<Excel>) * 必填                             │   │
│  │  ┌─────────────────────────────────────────────────────────┐│   │
│  │  │ 🔗 ${表格提取.output_file}                       [选择] ││   │
│  │  └─────────────────────────────────────────────────────────┘│   │
│  │                                                              │   │
│  │  cols (String) * 必填                                        │   │
│  │  ┌─────────────────────────────────────────────────────────┐│   │
│  │  │ A,B,C                                                   ││   │
│  │  └─────────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 输出参数 (只读)                                              │   │
│  │  output_file: File<Excel>  → 可供后续节点引用                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │ 错误处理                                                     │   │
│  │  策略: [终止工作流 ▼]                                        │   │
│  │  重试次数: [3]  重试间隔: [1000]ms                           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 6.3 Skill 兼容性检查

#### 6.3.1 兼容性规则

| 变更类型 | 兼容性 | 说明 |
|---------|--------|------|
| 新增可选参数 | ✅ 兼容 | 工作流无需修改 |
| 新增必填参数 | ⚠️ 部分兼容 | 需要为该参数配置值 |
| 修改参数类型 | ❌ 不兼容 | 必须更新工作流配置 |
| 删除参数 | ❌ 不兼容 | 引用该参数的配置失效 |
| 修改参数名称 | ❌ 不兼容 | 引用该参数的配置失效 |
| Skill 被删除 | 🔴 失效 | 节点完全失效 |

#### 6.3.2 节点兼容性状态

| 状态 | 说明 |
|------|------|
| `COMPATIBLE` | 兼容，可正常执行 |
| `NEEDS_UPDATE` | 部分兼容，需要更新配置 |
| `INCOMPATIBLE` | 不兼容，必须更新才能执行 |
| `INVALID` | Skill 已删除，节点失效 |

---

## 七、工作流验证

### 7.1 验证规则

#### 7.1.1 结构验证

- 必须有且仅有一个开始节点
- 必须有且仅有一个结束节点
- 不能有孤立节点（未连接任何其他节点）

#### 7.1.2 节点配置验证

- Skill 节点必须关联已发布的 Skill
- 循环节点必须有循环体（可选警告）
- 所有必填参数必须配置

#### 7.1.3 连线验证

- 不能有重复连线
- 连线的源节点和目标节点必须存在
- 条件分支必须有对应的分支标签

#### 7.1.4 参数引用验证

- 引用的节点必须是前置节点
- 引用的参数名必须存在于目标节点的输出参数中
- 参数类型必须兼容

---

## 八、工作流生命周期

### 8.1 状态流转

```
┌─────────┐     保存      ┌─────────┐     发布      ┌──────────┐
│  新建   │ ───────────▶ │  草稿   │ ───────────▶ │  已发布  │
└─────────┘              └─────────┘              └──────────┘
                              │                        │
                              │ 取消发布               │ 复制为新版本
                              ▼                        ▼
                         ┌─────────┐              ┌─────────┐
                         │  草稿   │              │  草稿   │
                         └─────────┘              └─────────┘
```

### 8.2 版本管理

采用简单版本管理：
- 工作流有"草稿"和"已发布"两种状态
- 已发布的工作流不可修改
- 需要修改时，复制为新版本（草稿状态）

---

## 九、API 设计

### 9.1 工作流管理 API

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/workflows` | 获取工作流列表 |
| GET | `/api/workflows/{id}` | 获取工作流详情 |
| POST | `/api/workflows` | 创建工作流 |
| PUT | `/api/workflows/{id}` | 更新工作流 |
| DELETE | `/api/workflows/{id}` | 删除工作流 |
| POST | `/api/workflows/{id}/publish` | 发布工作流 |
| POST | `/api/workflows/{id}/unpublish` | 取消发布 |
| POST | `/api/workflows/{id}/copy` | 复制工作流 |

### 9.2 节点管理 API

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/workflows/{id}/nodes` | 获取工作流节点列表 |
| POST | `/api/workflows/{id}/nodes` | 添加节点 |
| PUT | `/api/workflows/{id}/nodes/{nodeId}` | 更新节点 |
| DELETE | `/api/workflows/{id}/nodes/{nodeId}` | 删除节点 |

### 9.3 连线管理 API

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/api/workflows/{id}/connections` | 获取连线列表 |
| POST | `/api/workflows/{id}/connections` | 添加连线 |
| DELETE | `/api/workflows/{id}/connections/{connId}` | 删除连线 |

### 9.4 验证 API

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/workflows/validate` | 验证工作流定义 |
| GET | `/api/workflows/{id}/predecessors/{nodeUuid}` | 获取节点的前置节点列表 |
| GET | `/api/workflows/{id}/available-variables/{nodeUuid}` | 获取节点可引用的变量列表 |

---

## 十、前端组件设计

### 10.1 工作流编辑器组件结构

```
WorkflowEditorView.vue
├── Toolbar.vue                    # 工具栏（保存、发布、运行等）
├── NodePanel.vue                  # 节点面板（可拖拽节点列表）
│   ├── BasicNodes.vue            # 基础节点
│   ├── LogicNodes.vue            # 逻辑控制节点
│   └── SkillNodes.vue            # 技能节点（动态加载）
├── Canvas.vue                     # 画布（节点拖放、连线）
│   ├── FlowNode.vue              # 节点组件
│   ├── FlowConnection.vue        # 连线组件
│   └── LoopBodyCanvas.vue        # 循环体画布
├── ConfigPanel.vue                # 配置面板
│   ├── NodeConfig.vue            # 节点配置
│   ├── ParameterConfig.vue       # 参数配置
│   └── VariableSelector.vue      # 变量选择器
└── ExecutionMonitor.vue           # 执行监控面板
```

### 10.2 关键交互设计

#### 10.2.1 节点拖放

1. 从节点面板拖拽节点到画布
2. 释放时创建节点实例
3. 自动生成节点 UUID

#### 10.2.2 连线创建

1. 从源节点输出端口拖拽
2. 连接到目标节点输入端口
3. 自动验证连线合法性（不能形成环）

#### 10.2.3 参数配置

1. 点击节点打开配置面板
2. 为每个参数选择值来源（固定值/引用）
3. 引用时只能选择前置节点的输出

---

## 十一、总结

本文档定义了工作流配置的完整逻辑框架，包括：

1. **节点类型体系**：基础节点、逻辑控制节点、执行节点
2. **数据模型**：工作流、节点、连线、关联表结构
3. **控制节点**：循环、批处理、异步、结果收集、条件分支
4. **参数引用**：前置节点计算、引用格式
5. **Skill 集成**：动态加载、配置面板、兼容性检查
6. **工作流验证**：结构、配置、连线、引用验证
7. **生命周期**：状态流转、版本管理
8. **API 设计**：RESTful 接口定义
9. **前端组件**：编辑器结构、交互设计

该框架为工作流的执行提供了完整的配置基础，执行框架详见《工作流执行设计》文档。
