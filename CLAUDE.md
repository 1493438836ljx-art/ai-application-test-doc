# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是 AI 应用测试平台的设计文档仓库，包含两个核心模块的系统设计文档：

1. **Skill库系统** - 管理可复用技能能力的模块
2. **工作流系统** - 可视化工作流编排和执行框架

## 技术栈

| 层级 | 技术 |
|------|------|
| 前端框架 | Vue 3 + Vite |
| UI组件库 | Element Plus |
| 状态管理 | Pinia |
| 后端框架 | Spring Boot 3.2.3 + Java 17 |
| ORM框架 | MyBatis-Plus |
| 数据库 | MySQL (生产) / H2 (开发) |
| 消息队列 | Kafka |
| 实时通信 | WebSocket |
| API文档 | Swagger/OpenAPI 3 |

## 目录结构

```
skill-design/                    # Skill库系统设计文档
├── Skill库系统设计文档.md        # 总体设计文档
├── 01-数据库设计文档.md          # 数据库表结构设计
├── 02-API接口文档.md             # REST API 接口设计
├── 03-前端组件设计文档.md        # Vue 组件设计
└── 04-后端代码实现文档.md        # Spring Boot 实现设计

workflow-design/                 # 工作流系统设计文档
├── 01-workflow-configuration-design.md   # 配置层逻辑框架
├── database-tables-summary.md            # 数据库表结构汇总
├── 工作流配置层重构实施设计文档.md         # 重构实施方案
├── execution-framework-chapter[1-7].md   # 执行框架设计（7章）
└── phase[1-5]-*-implementation-plan.md   # 分阶段实施计划
```

## 核心架构

### 工作流系统分层架构

1. **配置层** - 工作流定义、节点配置、参数绑定、验证
2. **调度层** - 执行调度、流程编排、状态管理、实时推送
3. **执行层** - 节点执行器（基础节点、逻辑节点、技能节点）
4. **分布式层** - Kafka 通信、Client 执行机管理

### 节点类型体系

- **BASIC (基础节点)**: start, end
- **LOGIC (逻辑控制节点)**: condition_simple, condition_multi, loop, batch, async, collect
- **EXECUTION (执行节点)**: skill (从 Skill 库动态加载)

### 参数引用机制

```
${节点名称.参数名}
```

只能引用前置节点的输出，保证执行可确定性。

## 数据库表结构

### Skill库相关表
- `skill` - Skill 主表
- `skill_parameter` - 参数表（入参/出参统一）
- `skill_access_control` - 访问控制表

### 工作流相关表
- `workflow` - 工作流主表
- `workflow_node` - 节点表
- `workflow_connection` - 连线表
- `workflow_association` - 容器节点关联表
- `workflow_execution` - 执行记录表
- `workflow_node_execution` - 节点执行记录表
- `execution_log` - 执行日志表
- `workflow_error_log` - 错误日志表
- `client_registry` - Client 执行机注册表

## 文档编写规范

- 使用中文编写设计文档
- 使用 Markdown 格式
- 数据库设计包含完整的 SQL DDL 语句
- API 设计遵循 RESTful 规范
- 架构图使用 ASCII 图或代码块表示
