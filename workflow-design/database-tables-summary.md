# 工作流数据库表结构汇总

> 更新时间：2026-03-31
> 数据库：ai_test_platform

---

## 一、表清单

| 表名 | 说明 | 状态 |
|------|------|------|
| workflow | 工作流主表 | ✅ 已更新 |
| workflow_node | 工作流节点表 | ✅ 已更新 |
| workflow_connection | 工作流连线表 | ✅ 已更新 |
| workflow_association | 工作流关联表 | ✅ 已存在 |
| workflow_execution | 工作流执行记录表 | ✅ 已更新 |
| workflow_node_execution | 节点执行记录表 | ✅ 新建 |
| workflow_error_log | 错误日志表 | ✅ 新建 |
| execution_log | 执行日志表 | ✅ 新建 |
| client_registry | Client执行机注册表 | ✅ 新建 |
| workflow_node_type | 节点类型定义表 | ✅ 已存在 |
| workflow_variable_type | 变量类型定义表 | ✅ 已存在 |

---

## 二、表结构详情

### 2.1 workflow（工作流主表）

```sql
CREATE TABLE workflow (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    name            VARCHAR(100) NOT NULL COMMENT '工作流名称',
    description     VARCHAR(500) COMMENT '工作流描述',
    status          VARCHAR(20) NOT NULL DEFAULT 'DRAFT' COMMENT '状态: DRAFT/PUBLISHED',
    published       TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否已发布',
    has_run         TINYINT(1) NOT NULL DEFAULT 0 COMMENT '是否已运行过',
    version         INT NOT NULL DEFAULT 1 COMMENT '版本号',
    trigger_type    VARCHAR(20) DEFAULT 'MANUAL' COMMENT '触发类型: MANUAL/SCHEDULE/API',
    trigger_config  JSON COMMENT '触发配置(如定时规则)',
    created_by      VARCHAR(64) COMMENT '创建人',
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_by      VARCHAR(64) COMMENT '更新人',
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted         TINYINT(1) NOT NULL DEFAULT 0 COMMENT '逻辑删除标记',
    INDEX idx_status (status),
    INDEX idx_created_by (created_by)
) COMMENT '工作流主表';
```

### 2.2 workflow_node（工作流节点表）

```sql
CREATE TABLE workflow_node (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    workflow_id         BIGINT NOT NULL COMMENT '所属工作流ID',
    node_uuid           VARCHAR(36) NOT NULL COMMENT '节点UUID(前端生成)',
    type                VARCHAR(50) NOT NULL COMMENT '节点类型',
    type_id             BIGINT COMMENT '节点类型ID',
    name                VARCHAR(100) NOT NULL COMMENT '节点名称(显示名)',
    skill_id            VARCHAR(50) COMMENT '引用的Skill ID',
    skill_snapshot      JSON COMMENT 'Skill快照(创建时的参数定义)',
    position_x          INT NOT NULL COMMENT '画布X坐标',
    position_y          INT NOT NULL COMMENT '画布Y坐标',
    input_ports         TEXT COMMENT '输入端口配置',
    output_ports        TEXT COMMENT '输出端口配置',
    input_params        TEXT COMMENT '输入参数配置',
    output_params       TEXT COMMENT '输出参数配置',
    config              TEXT COMMENT '节点配置(JSON)',
    execution_location  VARCHAR(20) COMMENT '执行位置: CLIENT/SERVICE',
    error_strategy      VARCHAR(20) DEFAULT 'STOP' COMMENT '错误策略: STOP/SKIP/RETRY/ERROR_BRANCH',
    retry_count         INT DEFAULT 3 COMMENT '重试次数',
    retry_interval      INT DEFAULT 1000 COMMENT '重试间隔(毫秒)',
    error_branch_id     BIGINT COMMENT '错误处理分支节点ID',
    condition_type      VARCHAR(20) COMMENT '条件类型: SIMPLE/MULTI',
    conditions          JSON COMMENT '条件表达式配置',
    loop_type           VARCHAR(20) COMMENT '循环类型: COUNT/ARRAY/CONDITION',
    loop_config         JSON COMMENT '循环配置',
    batch_config        JSON COMMENT '批处理配置',
    async_config        JSON COMMENT '异步处理配置',
    collect_config      JSON COMMENT '结果收集配置',
    compatibility_status VARCHAR(20) DEFAULT 'COMPATIBLE' COMMENT '兼容性状态',
    parent_node_id      BIGINT COMMENT '父节点ID(循环/批处理节点)',
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_workflow_id (workflow_id),
    INDEX idx_skill_id (skill_id),
    INDEX idx_parent_node (parent_node_id),
    FOREIGN KEY (workflow_id) REFERENCES workflow(id)
) COMMENT '工作流节点表';
```

### 2.3 workflow_connection（工作流连线表）

```sql
CREATE TABLE workflow_connection (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    workflow_id         BIGINT NOT NULL COMMENT '所属工作流ID',
    connection_uuid     VARCHAR(36) NOT NULL COMMENT '连线UUID',
    source_node_id      BIGINT NOT NULL COMMENT '源节点ID',
    source_port_id      VARCHAR(50) NOT NULL COMMENT '源端口ID',
    target_node_id      BIGINT NOT NULL COMMENT '目标节点ID',
    target_port_id      VARCHAR(50) NOT NULL COMMENT '目标端口ID',
    branch_label        VARCHAR(50) COMMENT '分支标签(如: true/false/case1/case2/default)',
    branch_priority     INT COMMENT '分支优先级(多路分支时使用)',
    label               VARCHAR(100) COMMENT '连线标签',
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_workflow_id (workflow_id),
    INDEX idx_source_node (source_node_id),
    INDEX idx_target_node (target_node_id),
    FOREIGN KEY (workflow_id) REFERENCES workflow(id)
) COMMENT '工作流连线表';
```

### 2.4 workflow_association（工作流关联表）

```sql
CREATE TABLE workflow_association (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    workflow_id         BIGINT NOT NULL COMMENT '所属工作流ID',
    loop_node_id        BIGINT NOT NULL COMMENT '容器节点ID(循环/批处理/异步)',
    body_node_id        BIGINT NOT NULL COMMENT '内部节点ID',
    association_type    VARCHAR(50) NOT NULL COMMENT '关联类型: LOOP_BODY/BATCH_BODY/ASYNC_BODY',
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_workflow_id (workflow_id),
    INDEX idx_container_node (loop_node_id),
    FOREIGN KEY (workflow_id) REFERENCES workflow(id)
) COMMENT '工作流关联表(容器节点与内部节点的关系)';
```

### 2.5 workflow_execution（工作流执行记录表）

```sql
CREATE TABLE workflow_execution (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    workflow_id         BIGINT NOT NULL COMMENT '工作流ID',
    execution_uuid      VARCHAR(36) NOT NULL COMMENT '执行UUID',
    status              VARCHAR(20) NOT NULL COMMENT '状态: PENDING/RUNNING/SUCCESS/FAILED/...',
    trigger_type        VARCHAR(20) NOT NULL DEFAULT 'MANUAL' COMMENT '触发类型',
    triggered_by        VARCHAR(64) COMMENT '触发人',
    input_data          JSON COMMENT '输入参数',
    output_data         JSON COMMENT '输出结果',
    error_message       TEXT COMMENT '错误信息',
    node_executions     TEXT COMMENT '节点执行摘要',
    progress            INT NOT NULL DEFAULT 0 COMMENT '执行进度(0-100)',
    start_time          DATETIME COMMENT '开始时间',
    end_time            DATETIME COMMENT '结束时间',
    duration_ms         BIGINT COMMENT '执行耗时(毫秒)',
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_workflow_id (workflow_id),
    INDEX idx_execution_uuid (execution_uuid),
    INDEX idx_status (status),
    INDEX idx_start_time (start_time)
) COMMENT '工作流执行记录表';
```

### 2.6 workflow_node_execution（节点执行记录表）

```sql
CREATE TABLE workflow_node_execution (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    execution_id        BIGINT NOT NULL COMMENT '执行记录ID',
    workflow_id         BIGINT NOT NULL COMMENT '工作流ID',
    node_uuid           VARCHAR(50) NOT NULL COMMENT '节点UUID',
    node_name           VARCHAR(100) COMMENT '节点名称',
    node_type           VARCHAR(50) COMMENT '节点类型',
    status              VARCHAR(20) NOT NULL COMMENT '状态: PENDING/RUNNING/SUCCESS/FAILED/SKIPPED/TIMEOUT',
    input_data          JSON COMMENT '输入参数',
    output_data         JSON COMMENT '输出参数',
    error_message       TEXT COMMENT '错误信息',
    error_stack         TEXT COMMENT '错误堆栈',
    retry_count         INT DEFAULT 0 COMMENT '已重试次数',
    start_time          DATETIME COMMENT '开始时间',
    end_time            DATETIME COMMENT '结束时间',
    duration_ms         BIGINT COMMENT '执行耗时(毫秒)',
    client_id           VARCHAR(100) COMMENT '执行的Client ID',
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_execution_id (execution_id),
    INDEX idx_workflow_id (workflow_id),
    INDEX idx_node_uuid (node_uuid),
    INDEX idx_status (status),
    FOREIGN KEY (execution_id) REFERENCES workflow_execution(id) ON DELETE CASCADE
) COMMENT '节点执行记录表';
```

### 2.7 workflow_error_log（错误日志表）

```sql
CREATE TABLE workflow_error_log (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    execution_id        BIGINT COMMENT '执行记录ID',
    workflow_id         BIGINT COMMENT '工作流ID',
    node_uuid           VARCHAR(50) COMMENT '节点UUID',
    node_name           VARCHAR(100) COMMENT '节点名称',
    error_type          VARCHAR(50) COMMENT '错误类型: RECOVERABLE/BUSINESS/SYSTEM',
    error_code          INT COMMENT '错误代码',
    error_message       VARCHAR(2000) COMMENT '错误消息',
    error_stack         TEXT COMMENT '错误堆栈',
    context_json        JSON COMMENT '上下文信息',
    retry_count         INT COMMENT '重试次数',
    timestamp           DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '发生时间',
    INDEX idx_execution_id (execution_id),
    INDEX idx_workflow_id (workflow_id),
    INDEX idx_error_type (error_type),
    INDEX idx_timestamp (timestamp)
) COMMENT '工作流错误日志表';
```

### 2.8 execution_log（执行日志表）

```sql
CREATE TABLE execution_log (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    execution_id        BIGINT NOT NULL COMMENT '执行记录ID',
    node_uuid           VARCHAR(50) COMMENT '节点UUID',
    log_level           VARCHAR(20) NOT NULL COMMENT '日志级别: DEBUG/INFO/WARN/ERROR',
    message             TEXT NOT NULL COMMENT '日志消息',
    timestamp           DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '时间戳',
    INDEX idx_execution_id (execution_id),
    INDEX idx_timestamp (timestamp),
    FOREIGN KEY (execution_id) REFERENCES workflow_execution(id) ON DELETE CASCADE
) COMMENT '执行日志表';
```

### 2.9 client_registry（Client执行机注册表）

```sql
CREATE TABLE client_registry (
    id                      BIGINT PRIMARY KEY AUTO_INCREMENT,
    client_id               VARCHAR(100) NOT NULL UNIQUE COMMENT 'Client ID',
    client_name             VARCHAR(200) COMMENT 'Client名称',
    status                  VARCHAR(20) DEFAULT 'IDLE' COMMENT '状态: IDLE/BUSY/OFFLINE',
    running_tasks           INT DEFAULT 0 COMMENT '正在执行的任务数',
    max_concurrency         INT DEFAULT 5 COMMENT '最大并发数',
    cpu_usage               DOUBLE COMMENT 'CPU使用率',
    memory_usage            DOUBLE COMMENT '内存使用率',
    supported_execution_types JSON COMMENT '支持的执行类型',
    version                 VARCHAR(50) COMMENT 'Client版本',
    labels                  JSON COMMENT '标签(用于任务匹配)',
    last_heartbeat          DATETIME COMMENT '最后心跳时间',
    created_at              DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at              DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_status (status),
    INDEX idx_last_heartbeat (last_heartbeat)
) COMMENT 'Client执行机注册表';
```

---

## 三、表关系图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          工作流数据库表关系                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐       ┌─────────────────┐       ┌─────────────────────┐   │
│  │  workflow   │──────▶│ workflow_node   │◀──────│ workflow_connection │   │
│  │  (工作流)   │ 1:N   │   (节点)        │  N:N  │     (连线)          │   │
│  └─────────────┘       └─────────────────┘       └─────────────────────┘   │
│         │                      │                                            │
│         │ 1:N                  │ 1:N (parent_node)                          │
│         ▼                      ▼                                            │
│  ┌─────────────────┐   ┌─────────────────────┐                              │
│  │workflow_execu-  │   │workflow_association │                              │
│  │    tion         │   │     (关联)          │                              │
│  │  (执行记录)     │   └─────────────────────┘                              │
│  └─────────────────┘                                                        │
│         │                                                                   │
│         │ 1:N                                                               │
│         ├──────────────────────────┬────────────────────────┐               │
│         ▼                          ▼                        ▼               │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐          │
│  │workflow_node_   │   │ execution_log   │   │workflow_error_  │          │
│  │    execution    │   │  (执行日志)      │   │      log        │          │
│  │  (节点执行)     │   └─────────────────┘   │  (错误日志)     │          │
│  └─────────────────┘                         └─────────────────┘          │
│                                                                             │
│  ┌─────────────────┐                                                       │
│  │client_registry  │  (独立表，通过 Kafka 心跳维护)                         │
│  │  (执行机注册)   │                                                       │
│  └─────────────────┘                                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 四、设计说明

### 4.1 配置层表（配置与执行分离）

| 表名 | 用途 |
|------|------|
| workflow | 存储工作流的基本信息和触发配置 |
| workflow_node | 存储节点定义、参数配置、错误处理配置 |
| workflow_connection | 存储节点间的连线关系和分支标签 |
| workflow_association | 存储容器节点与内部节点的父子关系 |

### 4.2 执行层表（运行时数据）

| 表名 | 用途 |
|------|------|
| workflow_execution | 记录每次工作流执行的完整信息 |
| workflow_node_execution | 记录每个节点的执行详情 |
| execution_log | 记录执行过程中的日志 |
| workflow_error_log | 记录错误详情，用于分析和告警 |

### 4.3 基础设施表

| 表名 | 用途 |
|------|------|
| client_registry | 维护 Client 执行机的在线状态和元数据 |
| workflow_node_type | 预定义的节点类型定义 |
| workflow_variable_type | 预定义的变量类型定义 |

---

## 五、更新记录

| 日期 | 操作 | 说明 |
|------|------|------|
| 2026-03-31 | 更新 workflow 表 | 添加 trigger_type, trigger_config 字段 |
| 2026-03-31 | 更新 workflow_node 表 | 添加 skill_id, skill_snapshot, 执行位置, 错误策略, 条件/循环/批处理/异步/收集配置字段 |
| 2026-03-31 | 更新 workflow_connection 表 | 添加 branch_label, branch_priority 字段 |
| 2026-03-31 | 新建 workflow_node_execution 表 | 节点执行记录 |
| 2026-03-31 | 新建 workflow_error_log 表 | 错误日志 |
| 2026-03-31 | 新建 execution_log 表 | 执行日志 |
| 2026-03-31 | 新建 client_registry 表 | Client执行机注册 |
