# Skill库系统设计文档

## 1. 概述

### 1.1 文档信息
- **文档版本**: 1.0.0
- **创建日期**: 2026-03-31
- **项目名称**: AI Application Test Platform - Skill库模块

### 1.2 系统简介
Skill库是一个用于管理和配置可复用技能能力的模块，支持创建、编辑、发布、复制和删除Skill。每个Skill可以定义输入参数、输出参数、执行套件文件以及访问控制策略。

### 1.3 技术栈
| 层级 | 技术 |
|------|------|
| 前端框架 | Vue 3 + Vite |
| UI组件库 | Element Plus |
| 状态管理 | Pinia |
| 后端框架 | Spring Boot 3.2.3 + Java 17 |
| ORM框架 | MyBatis-Plus |
| 数据库 | MySQL (生产) / H2 (开发) |
| API文档 | Swagger/OpenAPI 3 |

---

## 2. 系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                         前端 (Vue 3)                             │
├─────────────────────────────────────────────────────────────────┤
│  SkillLibraryView.vue                                           │
│  ├── SkillFormDialog.vue (创建/编辑表单)                         │
│  │   └── ParamConfigTable.vue (参数配置表格)                     │
│  └── SkillDetailDialog.vue (详情展示)                            │
├─────────────────────────────────────────────────────────────────┤
│  API Layer: src/api/skill.js                                    │
└───────────────────────────┬─────────────────────────────────────┘
                            │ HTTP/REST
┌───────────────────────────▼─────────────────────────────────────┐
│                       后端 (Spring Boot)                         │
├─────────────────────────────────────────────────────────────────┤
│  Controller Layer                                               │
│  └── SkillController.java                                       │
├─────────────────────────────────────────────────────────────────┤
│  Service Layer                                                  │
│  ├── SkillService.java (接口)                                   │
│  └── SkillServiceImpl.java (实现)                               │
├─────────────────────────────────────────────────────────────────┤
│  Mapper Layer (MyBatis-Plus)                                    │
│  ├── SkillMapper.java                                           │
│  ├── SkillParameterMapper.java                                  │
│  └── SkillAccessControlMapper.java                              │
├─────────────────────────────────────────────────────────────────┤
│  Entity Layer                                                   │
│  ├── SkillEntity.java                                           │
│  ├── SkillParameterEntity.java                                  │
│  └── SkillAccessControlEntity.java                              │
├─────────────────────────────────────────────────────────────────┤
│  Utility & Scheduler                                            │
│  ├── ParameterTypeValidator.java (参数类型验证)                  │
│  └── SkillAgingCleanupScheduler.java (老化清理定时任务)          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                         数据库 (MySQL)                           │
├─────────────────────────────────────────────────────────────────┤
│  skill (主表)                                                    │
│  skill_parameter (参数表)                                        │
│  skill_access_control (访问控制表)                               │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 模块依赖关系

```
PersonalCenterView.vue (个人中心容器)
    │
    └── SkillLibraryView.vue (Skill库主视图)
            │
            ├── SkillFormDialog.vue (表单弹窗)
            │       │
            │       └── ParamConfigTable.vue (参数配置组件)
            │
            └── SkillDetailDialog.vue (详情弹窗)
```

---

## 3. 数据库设计

### 3.1 ER图

```
┌─────────────────────┐
│       skill         │
├─────────────────────┤
│ PK id               │
│    name (UK)        │
│    description      │
│    suite_path       │
│    suite_filename   │
│    execution_type   │
│    category         │
│    access_type      │
│    is_container     │
│    allow_add_input  │
│    allow_add_output │
│    status           │
│    created_by       │
│    updated_by       │
│    created_at       │
│    updated_at       │
│    deleted          │
│    deleted_at       │
└──────────┬──────────┘
           │ 1:N
           │
┌──────────▼──────────┐     ┌─────────────────────┐
│   skill_parameter   │     │skill_access_control │
├─────────────────────┤     ├─────────────────────┤
│ PK id               │     │ PK id               │
│ FK skill_id         │     │ FK skill_id         │
│    param_direction  │     │    target_type      │
│    param_order      │     │    target_id        │
│    param_type       │     └─────────────────────┘
│    param_name       │
│    default_value    │
│    description      │
│    required         │
└─────────────────────┘
```

### 3.2 表结构详细设计

#### 3.2.1 skill (Skill主表)

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | VARCHAR(36) | PK | UUID主键 |
| name | VARCHAR(100) | UK, NOT NULL | Skill名称，全局唯一 |
| description | VARCHAR(2000) | | Skill描述 |
| suite_path | VARCHAR(500) | | 执行套件文件存储路径 |
| suite_filename | VARCHAR(500) | | 执行套件原始文件名 |
| execution_type | VARCHAR(20) | NOT NULL | 执行类型: AUTOMATED/AI |
| category | VARCHAR(20) | NOT NULL | 分类: SYSTEM/USER |
| access_type | VARCHAR(20) | NOT NULL | 访问控制: PUBLIC/PRIVATE/WHITELIST/PROJECT |
| is_container | TINYINT(1) | NOT NULL, DEFAULT 0 | 是否容器 |
| allow_add_input_params | TINYINT(1) | DEFAULT 0 | 是否支持增加入参 |
| allow_add_output_params | TINYINT(1) | DEFAULT 0 | 是否支持增加出参 |
| status | VARCHAR(20) | NOT NULL | 状态: PUBLISHED/DRAFT |
| created_by | VARCHAR(100) | NOT NULL | 创建人 |
| updated_by | VARCHAR(100) | NOT NULL | 更新人 |
| created_at | DATETIME | NOT NULL | 创建时间 |
| updated_at | DATETIME | NOT NULL | 更新时间 |
| deleted | TINYINT(1) | NOT NULL, DEFAULT 0 | 逻辑删除标记 |
| deleted_at | DATETIME | | 删除时间(用于老化清理) |

**索引设计**:
- `idx_name`: UNIQUE INDEX on name
- `idx_category`: INDEX on category
- `idx_status`: INDEX on status
- `idx_created_by`: INDEX on created_by

#### 3.2.2 skill_parameter (参数表)

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | VARCHAR(36) | PK | UUID主键 |
| skill_id | VARCHAR(36) | FK, NOT NULL | 关联的Skill ID |
| param_direction | VARCHAR(10) | NOT NULL | 参数方向: INPUT/OUTPUT |
| param_order | INT | NOT NULL | 参数顺序(从1开始) |
| param_type | VARCHAR(50) | | 参数类型 |
| param_name | VARCHAR(100) | | 参数名称 |
| default_value | VARCHAR(1000) | | 默认值(仅入参使用) |
| description | VARCHAR(500) | | 参数描述 |
| required | TINYINT(1) | DEFAULT 0 | 是否必填 |

**索引设计**:
- `idx_skill_id`: INDEX on skill_id
- `idx_skill_direction`: INDEX on (skill_id, param_direction)

#### 3.2.3 skill_access_control (访问控制表)

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | VARCHAR(36) | PK | UUID主键 |
| skill_id | VARCHAR(36) | FK, NOT NULL | 关联的Skill ID |
| target_type | VARCHAR(20) | NOT NULL | 目标类型: USER/PROJECT |
| target_id | VARCHAR(36) | NOT NULL | 目标ID |

**索引设计**:
- `idx_skill_id`: INDEX on skill_id
- `idx_target`: INDEX on (target_type, target_id)
- `idx_skill_target`: UNIQUE INDEX on (skill_id, target_type, target_id)

### 3.3 数据库迁移历史

| 版本 | 文件名 | 说明 |
|------|--------|------|
| V1.3 | V1.3__add_skill_library.sql | 创建初始表结构(分离入参/出参表) |
| V1.4 | V1.4__update_skill_library_schema.sql | 添加allow_add字段，移除variadic字段 |
| V2024.01.01 | V2024.01.01__merge_skill_parameter_tables.sql | 合并入参/出参表为统一的skill_parameter表 |

---

## 4. 后端设计

### 4.1 包结构

```
com.example.demo.skill/
├── controller/
│   └── SkillController.java          # REST API控制器
├── service/
│   ├── SkillService.java             # 服务接口
│   └── impl/
│       └── SkillServiceImpl.java     # 服务实现
├── mapper/
│   ├── SkillMapper.java              # Skill数据访问
│   ├── SkillParameterMapper.java     # 参数数据访问
│   └── SkillAccessControlMapper.java # 访问控制数据访问
├── entity/
│   ├── SkillEntity.java              # Skill实体
│   ├── SkillParameterEntity.java     # 参数实体
│   ├── SkillAccessControlEntity.java # 访问控制实体
│   ├── SkillStatus.java              # 状态枚举
│   ├── SkillCategory.java            # 分类枚举
│   ├── SkillAccessType.java          # 访问类型枚举
│   ├── SkillExecutionType.java       # 执行类型枚举
│   └── SkillParamDirection.java      # 参数方向枚举
├── dto/
│   ├── SkillCreateRequest.java       # 创建请求DTO
│   ├── SkillUpdateRequest.java       # 更新请求DTO
│   ├── SkillQueryRequest.java        # 查询请求DTO
│   └── SkillResponse.java            # 响应DTO
├── util/
│   └── ParameterTypeValidator.java   # 参数类型验证器
└── scheduler/
    └── SkillAgingCleanupScheduler.java # 老化清理定时任务
```

### 4.2 API接口设计

#### 4.2.1 接口列表

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | /api/skill | 创建Skill(支持文件上传) |
| GET | /api/skill/list | 获取Skill列表(分页) |
| GET | /api/skill/search | 搜索Skill(多条件) |
| GET | /api/skill/{id} | 获取Skill详情 |
| PUT | /api/skill/{id} | 更新Skill(支持文件上传) |
| DELETE | /api/skill/{id} | 删除Skill(逻辑删除) |
| POST | /api/skill/{id}/publish | 发布Skill |
| POST | /api/skill/{id}/unpublish | 取消发布Skill |
| POST | /api/skill/{id}/copy | 复制Skill |
| GET | /api/skill/{id}/download | 下载执行套件 |

#### 4.2.2 接口详细说明

##### POST /api/skill - 创建Skill

**请求**: multipart/form-data
- `file`: 执行套件文件(ZIP格式) - 可选
- `data`: SkillCreateRequest JSON

**SkillCreateRequest 结构**:
```json
{
  "name": "string (必填, max:100)",
  "description": "string (max:2000)",
  "executionType": "AUTOMATED | AI (必填)",
  "category": "SYSTEM | USER (必填)",
  "accessType": "PUBLIC | PRIVATE | WHITELIST | PROJECT (必填)",
  "isContainer": "boolean (必填)",
  "allowAddInputParams": "boolean",
  "allowAddOutputParams": "boolean",
  "createdBy": "string",
  "inputParameters": [
    {
      "paramType": "string",
      "paramName": "string",
      "defaultValue": "string",
      "description": "string",
      "required": "boolean"
    }
  ],
  "outputParameters": [
    {
      "paramType": "string",
      "paramName": "string",
      "description": "string",
      "required": "boolean"
    }
  ],
  "accessControls": [
    {
      "targetType": "USER | PROJECT",
      "targetId": "string"
    }
  ]
}
```

**响应**: SkillResponse (201 Created)

##### GET /api/skill/search - 搜索Skill

**查询参数**:
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| name | String | 否 | 名称关键字(模糊匹配) |
| executionType | String | 否 | 执行类型 |
| category | String | 否 | 分类 |
| accessType | String | 否 | 访问控制类型 |
| status | String | 否 | 状态 |
| createdBy | String | 否 | 创建人 |
| isContainer | Boolean | 否 | 是否容器 |
| page | int | 否 | 页码(默认1) |
| size | int | 否 | 每页大小(默认10) |

**响应**: Page<SkillResponse>

##### GET /api/skill/{id} - 获取详情

**响应**: SkillResponse (包含完整的入参、出参、访问控制列表)

##### PUT /api/skill/{id} - 更新Skill

**请求**: multipart/form-data
- `file`: 新的执行套件文件 - 可选
- `data`: SkillUpdateRequest JSON

**响应**: SkillResponse (200 OK)

##### DELETE /api/skill/{id} - 删除Skill

**说明**: 执行逻辑删除，设置deleted=true和deleted_at时间戳

**响应**: 204 No Content

### 4.3 枚举类型定义

#### SkillStatus (状态)
```java
public enum SkillStatus {
    PUBLISHED,  // 已发布
    DRAFT       // 草稿
}
```

#### SkillCategory (分类)
```java
public enum SkillCategory {
    SYSTEM,  // 系统定义
    USER     // 用户定义
}
```

#### SkillAccessType (访问类型)
```java
public enum SkillAccessType {
    PUBLIC,    // 公开
    PRIVATE,   // 私有(仅创建者)
    WHITELIST, // 白名单
    PROJECT    // 项目级
}
```

#### SkillExecutionType (执行类型)
```java
public enum SkillExecutionType {
    AUTOMATED,  // 自动化
    AI          // AI驱动
}
```

#### SkillParamDirection (参数方向)
```java
public enum SkillParamDirection {
    INPUT,   // 入参
    OUTPUT   // 出参
}
```

### 4.4 参数类型系统

#### 支持的参数类型

**基本类型**:
- String
- Boolean
- Integer
- Object
- Times

**复合类型**:
- `Array<ElementType>` - 数组类型，元素可以是基本类型
- `File<FileType>` - 文件类型
- `Array<File<FileType>>` - 文件数组

**文件类型**:
- Zip, Doc, Docx, Excel, Pdf, Txt

**类型验证器** (ParameterTypeValidator):
```java
// 验证正则表达式
BASIC_TYPES = ["String", "Boolean", "Integer", "Object", "Times"]
FILE_TYPES = ["Zip", "Doc", "Docx", "Excel", "Pdf", "Txt"]
ARRAY_ELEMENT_TYPES = ["String", "Boolean", "Integer", "Time", "Object"]

// 匹配模式
Array<Element>         → ^Array<([A-Za-z]+)>$
File<Type>             → ^File<(.+)>$
Array<File<Type>>      → ^Array<File<(.+)>>$
```

### 4.5 定时任务设计

#### SkillAgingCleanupScheduler

**功能**: 物理删除已逻辑删除超过365天的Skill数据

**执行周期**: 每天凌晨0点5分 (cron: `0 5 0 * * ?`)

**处理流程**:
1. 查询deleted=true且deleted_at超过365天的Skill
2. 逐个删除关联的参数记录
3. 逐个删除关联的访问控制记录
4. 物理删除Skill主记录

---

## 5. 前端设计

### 5.1 目录结构

```
src/views/skill/
├── SkillLibraryView.vue       # 主视图页面
└── components/
    ├── SkillFormDialog.vue    # 创建/编辑表单弹窗
    ├── SkillDetailDialog.vue  # 详情展示弹窗
    └── ParamConfigTable.vue   # 参数配置表格组件
```

### 5.2 组件设计

#### 5.2.1 SkillLibraryView.vue (主视图)

**功能**:
- Skill列表展示(卡片网格布局)
- 搜索和筛选(名称、分类、状态)
- 分页
- 新建/编辑/复制/删除/发布操作

**状态管理**:
```javascript
// 核心状态
const loading = ref(false)
const skillList = ref([])
const total = ref(0)
const currentPage = ref(1)
const pageSize = ref(10)
const searchKeyword = ref('')
const categoryFilter = ref('')
const statusFilter = ref('')

// 弹窗状态
const dialogVisible = ref(false)
const isEdit = ref(false)
const currentSkill = ref(null)
const detailDialogVisible = ref(false)
const detailSkill = ref(null)
```

**配置映射**:
```javascript
// 状态配置
const statusConfig = {
  PUBLISHED: { label: '已发布', type: 'success' },
  DRAFT: { label: '草稿', type: 'warning' },
}

// 分类配置
const categoryConfig = {
  SYSTEM: { label: '系统', color: '#409EFF' },
  USER: { label: '自定义', color: '#67C23A' },
}

// 访问类型配置
const accessTypeConfig = {
  PUBLIC: { label: '公开', color: '#67C23A' },
  PRIVATE: { label: '私有', color: '#909399' },
  WHITELIST: { label: '白名单', color: '#E6A23C' },
  PROJECT: { label: '项目级', color: '#409EFF' },
}

// 执行类型配置
const executionTypeConfig = {
  AUTOMATED: '自动化',
  AI: 'AI驱动',
}
```

#### 5.2.2 SkillFormDialog.vue (表单弹窗)

**功能**:
- 创建新Skill
- 编辑已有Skill
- 执行套件文件上传
- 入参/出参配置

**表单字段**:
```javascript
const formData = ref({
  name: '',
  description: '',
  executionType: 'AUTOMATED',
  category: 'USER',
  accessType: 'PRIVATE',
  isContainer: false,
  allowAddInputParams: false,
  allowAddOutputParams: false,
  inputParameters: [],
  outputParameters: [],
})
```

**文件上传处理**:
- 支持拖拽上传
- 仅接受ZIP格式
- 编辑时显示已上传文件，支持下载和重新上传

#### 5.2.3 ParamConfigTable.vue (参数配置表格)

**功能**:
- 参数列表的增删改
- 参数类型级联选择
- 入参/出参数据结构适配

**类型选择器设计**:
```javascript
// 级联选择器选项结构
const typeOptions = [
  // 基本类型
  { value: 'String', label: 'String' },
  { value: 'Boolean', label: 'Boolean' },
  // ...

  // Array复合类型
  {
    value: 'Array',
    label: 'Array',
    children: [
      { value: 'String', label: 'String' },
      // ...
      {
        value: 'File',
        label: 'File',
        children: [/* 文件类型选项 */]
      }
    ]
  },

  // File复合类型
  {
    value: 'File',
    label: 'File',
    children: [/* 文件类型选项 */]
  }
]
```

**类型解析**:
```javascript
// 解析类型字符串为级联选择器值
const parseType = (typeStr) => {
  // Array<File<SubType>> → { baseType: 'Array', elementType: 'File', fileType: 'SubType' }
  // Array<Element>       → { baseType: 'Array', elementType: 'Element', fileType: '' }
  // File<SubType>        → { baseType: 'File', elementType: '', fileType: 'SubType' }
  // BasicType            → { baseType: 'BasicType', elementType: '', fileType: '' }
}
```

#### 5.2.4 SkillDetailDialog.vue (详情弹窗)

**功能**:
- 展示Skill完整信息
- 参数表格展示
- 执行套件下载
- 快捷编辑/发布操作

### 5.3 API层设计

**src/api/skill.js**:
```javascript
const BASE_URL = 'http://localhost:8080/api/skill'

// 获取Skill列表(分页搜索)
export async function getSkillList(params)

// 获取Skill详情
export async function getSkillDetail(id)

// 创建Skill(支持文件上传)
export async function createSkill(data, file)

// 更新Skill(支持文件上传)
export async function updateSkill(id, data, file)

// 删除Skill
export async function deleteSkill(id)

// 发布Skill
export async function publishSkill(id)

// 取消发布Skill
export async function unpublishSkill(id)

// 复制Skill
export async function copySkill(id)

// 获取下载URL
export function getDownloadSuiteUrl(id)
```

### 5.4 路由集成

Skill库页面嵌入在个人中心视图中:
```javascript
// PersonalCenterView.vue
import SkillLibraryView from '@/views/skill/SkillLibraryView.vue'

const menuItems = [
  // ...
  { key: 'skill-library', label: 'Skill库' },
]

// 条件渲染
<template v-if="activeMenu === 'skill-library'">
  <SkillLibraryView />
</template>
```

---

## 6. 业务流程

### 6.1 创建Skill流程

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ 点击新建按钮 │───►│ 填写表单数据 │───►│ 上传执行套件 │
└──────────────┘    └──────────────┘    └──────────────┘
                                               │
                                               ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ 显示成功提示 │◄───│ 返回新记录   │◄───│ 提交到后端   │
└──────────────┘    └──────────────┘    └──────────────┘
```

**后端处理**:
1. 验证Skill名称唯一性
2. 保存执行套件文件到本地(./uploads/suites/)
3. 创建Skill主记录
4. 批量插入入参记录
5. 批量插入出参记录
6. 批量插入访问控制记录

### 6.2 发布/取消发布流程

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ 检查当前状态 │───►│ 确认操作     │───►│ 更新状态字段 │
└──────────────┘    └──────────────┘    └──────────────┘
        │                                       │
        ▼                                       ▼
┌──────────────┐                       ┌──────────────┐
│ DRAFT        │                       │ PUBLISHED    │
│ (可发布)     │                       │ (可取消发布) │
└──────────────┘                       └──────────────┘
```

### 6.3 复制Skill流程

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ 确认复制操作 │───►│ 复制主记录   │───►│ 复制参数记录 │
└──────────────┘    │ (名称+副本)  │    └──────────────┘
                    └──────────────┘            │
                                                ▼
                    ┌──────────────┐    ┌──────────────┐
                    │ 返回新记录   │◄───│ 复制访问控制 │
                    └──────────────┘    └──────────────┘
```

**注意**: 复制后的Skill状态始终为DRAFT

### 6.4 删除流程(逻辑删除)

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ 确认删除操作 │───►│ 删除关联参数 │───►│ 删除访问控制 │
└──────────────┘    └──────────────┘    └──────────────┘
                                                │
                                                ▼
                    ┌──────────────┐    ┌──────────────┐
                    │ 设置deleted  │◄───│ 删除执行套件 │
                    │ 和deleted_at │    │ 文件         │
                    └──────────────┘    └──────────────┘
```

---

## 7. 安全设计

### 7.1 输入验证

**前端验证**:
- Skill名称: 必填，1-100字符
- 描述: 最大2000字符
- 参数名称: 最大100字符
- 参数类型: 枚举值校验
- 文件类型: 仅接受ZIP格式

**后端验证**:
- 使用Jakarta Validation注解
- 参数类型使用ParameterTypeValidator正则验证
- Skill名称唯一性校验

### 7.2 文件上传安全

- 文件存储路径: `./uploads/suites/`
- 文件重命名: UUID + 原始扩展名
- 文件类型限制: 仅接受ZIP

### 7.3 访问控制

- PUBLIC: 所有用户可见
- PRIVATE: 仅创建者可见
- WHITELIST: 白名单用户可见
- PROJECT: 项目成员可见

---

## 8. 性能优化

### 8.1 数据库优化

- 列表查询仅返回基本字段和参数计数，不返回完整参数列表
- 详情查询才返回完整参数和访问控制
- 使用索引优化常用查询条件

### 8.2 前端优化

- 列表使用卡片网格布局，CSS Grid自适应
- 参数配置表格使用虚拟滚动(大量参数时)
- 文件上传使用FormData，支持大文件

---

## 9. 扩展性设计

### 9.1 参数类型扩展

如需添加新的参数类型:
1. 后端: 更新ParameterTypeValidator的类型列表和正则
2. 前端: 更新ParamConfigTable.vue的typeOptions

### 9.2 访问控制扩展

如需添加新的访问控制类型:
1. 后端: 添加新的枚举值到SkillAccessType
2. 前端: 更新accessTypeConfig配置

### 9.3 执行类型扩展

如需添加新的执行类型:
1. 后端: 添加新的枚举值到SkillExecutionType
2. 前端: 更新executionTypeOptions配置

---

## 10. 部署配置

### 10.1 后端配置

```yaml
# application.yml
skill:
  suite:
    upload-path: ./uploads/suites  # 执行套件存储路径
```

### 10.2 数据库初始化

1. 执行Flyway迁移脚本(自动)
2. 迁移脚本位于: `src/main/resources/db/migration/`

---

## 11. 附录

### 11.1 文件清单

**前端文件**:
| 文件路径 | 说明 |
|----------|------|
| src/views/skill/SkillLibraryView.vue | 主视图 |
| src/views/skill/components/SkillFormDialog.vue | 表单弹窗 |
| src/views/skill/components/SkillDetailDialog.vue | 详情弹窗 |
| src/views/skill/components/ParamConfigTable.vue | 参数配置组件 |
| src/api/skill.js | API封装 |

**后端文件**:
| 文件路径 | 说明 |
|----------|------|
| controller/SkillController.java | 控制器 |
| service/SkillService.java | 服务接口 |
| service/impl/SkillServiceImpl.java | 服务实现 |
| mapper/SkillMapper.java | Skill Mapper |
| mapper/SkillParameterMapper.java | 参数 Mapper |
| mapper/SkillAccessControlMapper.java | 访问控制 Mapper |
| entity/SkillEntity.java | Skill实体 |
| entity/SkillParameterEntity.java | 参数实体 |
| entity/SkillAccessControlEntity.java | 访问控制实体 |
| entity/SkillStatus.java | 状态枚举 |
| entity/SkillCategory.java | 分类枚举 |
| entity/SkillAccessType.java | 访问类型枚举 |
| entity/SkillExecutionType.java | 执行类型枚举 |
| entity/SkillParamDirection.java | 参数方向枚举 |
| dto/SkillCreateRequest.java | 创建请求DTO |
| dto/SkillUpdateRequest.java | 更新请求DTO |
| dto/SkillQueryRequest.java | 查询请求DTO |
| dto/SkillResponse.java | 响应DTO |
| util/ParameterTypeValidator.java | 参数类型验证器 |
| scheduler/SkillAgingCleanupScheduler.java | 老化清理任务 |
| resources/mapper/skill/SkillMapper.xml | Skill SQL映射 |
| resources/mapper/skill/SkillParameterMapper.xml | 参数SQL映射 |
| resources/mapper/skill/SkillAccessControlMapper.xml | 访问控制SQL映射 |

**数据库迁移文件**:
| 文件路径 | 说明 |
|----------|------|
| V1.3__add_skill_library.sql | 初始表结构 |
| V1.4__update_skill_library_schema.sql | Schema更新 |
| V2024.01.01__merge_skill_parameter_tables.sql | 合并参数表 |

### 11.2 术语表

| 术语 | 说明 |
|------|------|
| Skill | 技能，可复用的能力单元 |
| Suite | 执行套件，Skill的可执行文件包 |
| Parameter | 参数，Skill的输入输出定义 |
| Access Control | 访问控制，Skill的权限管理 |
| Aging Cleanup | 老化清理，定期清理已删除数据的机制 |
