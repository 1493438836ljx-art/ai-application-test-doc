# Skill库 API接口文档

## 1. 接口概述

### 1.1 基础信息
- **Base URL**: `http://localhost:8080/api/skill`
- **Content-Type**: `application/json` (除文件上传外)
- **认证**: 暂无(后续可扩展)

### 1.2 通用响应格式

#### 成功响应
```json
{
  "id": "uuid",
  "name": "string",
  ...
}
```

#### 分页响应
```json
{
  "content": [...],
  "pageable": {
    "pageNumber": 1,
    "pageSize": 10,
    "totalElements": 100,
    "totalPages": 10
  },
  "totalElements": 100,
  "totalPages": 10,
  "size": 10,
  "number": 1,
  "first": true,
  "last": false
}
```

#### 错误响应
```json
{
  "timestamp": "2026-03-31T10:00:00.000+00",
  "status": 400/404/409/500,
  "error": "错误信息",
  "path": "/api/skill/xxx"
}
```

---

## 2. 接口详情

### 2.1 创建Skill

**POST** `/api/skill`

#### 请求
- **Content-Type**: `multipart/form-data`
- **Body**:
  - `file`: (File, 可选) 执行套件文件(ZIP格式)
  - `data`: (JSON, 必填) Skill创建请求

```json
{
  "name": "data-processor",
  "description": "数据处理技能，用于Excel文件的读取和转换",
  "executionType": "AUTOMATED",
  "category": "USER",
  "accessType": "PRIVATE",
  "isContainer": false,
  "allowAddInputParams": false,
  "allowAddOutputParams": false,
  "createdBy": "admin",
  "inputParameters": [
    {
      "paramType": "String",
      "paramName": "inputFile",
      "defaultValue": "",
      "description": "输入文件路径",
      "required": true
    },
    {
      "paramType": "Array<String>",
      "paramName": "columns",
      "defaultValue": "",
      "description": "需要处理的列名",
      "required": false
    }
  ],
  "outputParameters": [
    {
      "paramType": "Array<Object>",
      "paramName": "result",
      "description": "处理结果数组",
      "required": true
    }
  ],
  "accessControls": []
}
```

#### 响应
- **Status**: `201 Created`
- **Body**: SkillResponse

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "data-processor",
  "description": "数据处理技能，用于Excel文件的读取和转换",
  "suitePath": "/uploads/suites/uuid.zip",
  "suiteFilename": "data-processor.zip",
  "executionType": "AUTOMATED",
  "category": "USER",
  "accessType": "PRIVATE",
  "isContainer": false,
  "allowAddInputParams": false,
  "allowAddOutputParams": false,
  "status": "DRAFT",
  "createdBy": "admin",
  "createdAt": "2026-03-31T10:00:00",
  "updatedBy": "admin",
  "updatedAt": "2026-03-31T10:00:00",
  "inputParamCount": 2,
  "outputParamCount": 1,
  "inputParameters": [...],
  "outputParameters": [...],
  "accessControls": []
}
```

#### 错误码
| 状态码 | 说明 |
|--------|------|
| 400 | 参数验证失败 |
| 409 | Skill名称已存在 |

---

### 2.2 获取Skill列表

**GET** `/api/skill/list`

#### 请求参数
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| page | int | 否 | 1 | 页码(从1开始) |
| size | int | 否 | 10 | 每页大小 |
| sort | string | 否 | createdAt | 排序字段 |
| direction | string | 否 | DESC | 排序方向(ASC/DESC) |

#### 响应
- **Status**: `200 OK`
- **Body**: Page<SkillResponse>

```json
{
  "content": [
    {
      "id": "uuid",
      "name": "data-processor",
      "description": "描述信息",
      "executionType": "AUTOMATED",
      "category": "USER",
      "accessType": "PRIVATE",
      "isContainer": false,
      "status": "PUBLISHED",
      "inputParamCount": 2,
      "outputParamCount": 1,
      "createdAt": "2026-03-31T10:00:00",
      "createdBy": "admin"
    }
  ],
  "totalElements": 100,
  "totalPages": 10
}
```

---

### 2.3 搜索Skill

**GET** `/api/skill/search`

#### 请求参数
| 参数 | 类型 | 必填 | 默认值 | 说明 |
|------|------|------|--------|------|
| name | string | 否 | - | 名称关键字(模糊匹配) |
| executionType | string | 否 | - | 执行类型: AUTOMATED/AI |
| category | string | 否 | - | 分类: SYSTEM/USER |
| accessType | string | 否 | - | 访问类型: PUBLIC/PRIVATE/WHITELIST/PROJECT |
| status | string | 否 | - | 状态: PUBLISHED/DRAFT |
| createdBy | string | 否 | - | 创建人 |
| isContainer | boolean | 否 | - | 是否容器 |
| page | int | 否 | 1 | 页码(从1开始) |
| size | int | 否 | 10 | 每页大小 |

#### 响应
- **Status**: `200 OK`
- **Body**: Page<SkillResponse>

---

### 2.4 获取Skill详情

**GET** `/api/skill/{id}`

#### 路径参数
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | string | 是 | Skill ID (UUID) |

#### 响应
- **Status**: `200 OK`
- **Body**: SkillResponse (包含完整参数和访问控制)

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "data-processor",
  "description": "数据处理技能",
  "suitePath": "/uploads/suites/uuid.zip",
  "suiteFilename": "data-processor.zip",
  "executionType": "AUTOMATED",
  "category": "USER",
  "accessType": "PRIVATE",
  "isContainer": false,
  "allowAddInputParams": false,
  "allowAddOutputParams": false,
  "status": "DRAFT",
  "createdBy": "admin",
  "createdAt": "2026-03-31T10:00:00",
  "updatedBy": "admin",
  "updatedAt": "2026-03-31T10:00:00",
  "inputParamCount": 2,
  "outputParamCount": 1,
  "inputParameters": [
    {
      "id": "param-uuid-1",
      "paramOrder": 1,
      "paramType": "String",
      "paramName": "inputFile",
      "defaultValue": "",
      "description": "输入文件路径",
      "required": true
    }
  ],
  "outputParameters": [
    {
      "id": "param-uuid-2",
      "paramOrder": 1,
      "paramType": "Array<Object>",
      "paramName": "result",
      "description": "处理结果数组",
      "required": true
    }
  ],
  "accessControls": []
}
```

#### 错误码
| 状态码 | 说明 |
|--------|------|
| 404 | Skill不存在 |

---

### 2.5 更新Skill

**PUT** `/api/skill/{id}`

#### 路径参数
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | string | 是 | Skill ID (UUID) |

#### 请求
- **Content-Type**: `multipart/form-data`
- **Body**:
  - `file`: (File, 可选) 新的执行套件文件
  - `data`: (JSON, 必填) Skill更新请求

```json
{
  "name": "data-processor-v2",
  "description": "更新后的描述",
  "executionType": "AI",
  "category": "SYSTEM",
  "accessType": "PUBLIC",
  "isContainer": true,
  "allowAddInputParams": true,
  "allowAddOutputParams": true,
  "updatedBy": "admin",
  "inputParameters": [
    {
      "paramType": "String",
      "paramName": "inputFile",
      "defaultValue": "/default/input.xlsx",
      "description": "输入文件路径",
      "required": true
    }
  ],
  "outputParameters": [
    {
      "paramType": "Object",
      "paramName": "result",
      "description": "处理结果",
      "required": true
    }
  ]
}
```

#### 响应
- **Status**: `200 OK`
- **Body**: SkillResponse

#### 错误码
| 状态码 | 说明 |
|--------|------|
| 400 | 参数验证失败 |
| 404 | Skill不存在 |
| 409 | 新名称已被使用 |

---

### 2.6 删除Skill

**DELETE** `/api/skill/{id}`

#### 路径参数
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | string | 是 | Skill ID (UUID) |

#### 响应
- **Status**: `204 No Content`

#### 错误码
| 状态码 | 说明 |
|--------|------|
| 404 | Skill不存在 |

---

### 2.7 发布Skill

**POST** `/api/skill/{id}/publish`

#### 路径参数
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | string | 是 | Skill ID (UUID) |

#### 响应
- **Status**: `200 OK`
- **Body**: SkillResponse (status=PUBLISHED)

#### 错误码
| 状态码 | 说明 |
|--------|------|
| 404 | Skill不存在 |

---

### 2.8 取消发布Skill

**POST** `/api/skill/{id}/unpublish`

#### 路径参数
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | string | 是 | Skill ID (UUID) |

#### 响应
- **Status**: `200 OK`
- **Body**: SkillResponse (status=DRAFT)

#### 错误码
| 状态码 | 说明 |
|--------|------|
| 404 | Skill不存在 |

---

### 2.9 复制Skill

**POST** `/api/skill/{id}/copy`

#### 路径参数
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | string | 是 | 源Skill ID (UUID) |

#### 响应
- **Status**: `201 Created`
- **Body**: SkillResponse (新Skill, status=DRAFT, name="原名称 (副本)")

#### 说明
- 复制后的Skill名称自动添加 " (副本)" 后缀
- 复制后的Skill状态始终为 DRAFT
- 同时复制所有参数和访问控制

#### 错误码
| 状态码 | 说明 |
|--------|------|
| 404 | 源Skill不存在 |

---

### 2.10 下载执行套件

**GET** `/api/skill/{id}/download`

#### 路径参数
| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| id | string | 是 | Skill ID (UUID) |

#### 响应
- **Status**: `200 OK`
- **Content-Type**: `application/octet-stream`
- **Headers**:
  - `Content-Disposition`: `attachment; filename*=UTF-8''原始文件名`
- **Body**: 二进制文件流

#### 错误码
| 状态码 | 说明 |
|--------|------|
| 404 | Skill不存在或文件不存在 |
| 400 | Skill没有关联的执行套件 |

---

## 3. 数据模型

### 3.1 SkillCreateRequest

```typescript
interface SkillCreateRequest {
  name: string;                    // 必填, max:100
  description?: string;            // max:2000
  executionType: 'AUTOMATED' | 'AI';  // 必填
  category: 'SYSTEM' | 'USER';       // 必填
  accessType: 'PUBLIC' | 'PRIVATE' | 'WHITELIST' | 'PROJECT';  // 必填
  isContainer: boolean;             // 必填
  allowAddInputParams?: boolean;
  allowAddOutputParams?: boolean;
  createdBy?: string;
  inputParameters?: InputParameter[];
  outputParameters?: OutputParameter[];
  accessControls?: AccessControl[];
}

interface InputParameter {
  paramType?: string;      // max:50
  paramName?: string;      // max:100
  defaultValue?: string;   // max:1000
  description?: string;    // max:500
  required: boolean;       // 必填
}

interface OutputParameter {
  paramType?: string;      // max:50
  paramName?: string;      // max:100
  description?: string;    // max:500
  required: boolean;       // 必填
}

interface AccessControl {
  targetType: 'USER' | 'PROJECT';  // 必填
  targetId: string;                  // 必填
}
```

### 3.2 SkillUpdateRequest

```typescript
interface SkillUpdateRequest {
  name?: string;
  description?: string;
  executionType?: 'AUTOMATED' | 'AI';
  category?: 'SYSTEM' | 'USER';
  accessType?: 'PUBLIC' | 'PRIVATE' | 'WHITELIST' | 'PROJECT';
  isContainer?: boolean;
  allowAddInputParams?: boolean;
  allowAddOutputParams?: boolean;
  updatedBy?: string;
  inputParameters?: InputParameter[];
  outputParameters?: OutputParameter[];
}
```

### 3.3 SkillQueryRequest

```typescript
interface SkillQueryRequest {
  name?: string;           // 模糊匹配
  executionType?: string;
  category?: string;
  accessType?: string;
  status?: string;
  createdBy?: string;
  isContainer?: boolean;
}
```

### 3.4 SkillResponse

```typescript
interface SkillResponse {
  id: string;
  name: string;
  description?: string;
  suitePath?: string;
  suiteFilename?: string;
  executionType: 'AUTOMATED' | 'AI';
  category: 'SYSTEM' | 'USER';
  accessType: 'PUBLIC' | 'PRIVATE' | 'WHITELIST' | 'PROJECT';
  isContainer: boolean;
  allowAddInputParams: boolean;
  allowAddOutputParams: boolean;
  status: 'PUBLISHED' | 'DRAFT';
  createdBy: string;
  createdAt: string;       // ISO 8601 datetime
  updatedBy: string;
  updatedAt: string;       // ISO 8601 datetime
  inputParamCount?: number;
  outputParamCount?: number;
  inputParameters?: InputParameterResponse[];
  outputParameters?: OutputParameterResponse[];
  accessControls?: AccessControlResponse[];
}

interface InputParameterResponse {
  id: string;
  paramOrder: number;
  paramType: string;
  paramName: string;
  defaultValue?: string;
  description?: string;
  required: boolean;
}

interface OutputParameterResponse {
  id: string;
  paramOrder: number;
  paramType: string;
  paramName: string;
  description?: string;
  required: boolean;
}

interface AccessControlResponse {
  id: string;
  targetType: string;
  targetId: string;
}
```

---

## 4. 前端API封装

### 4.1 skill.js

```javascript
import { get, post, del, upload, uploadPut } from '@/utils/request'

const BASE_URL = 'http://localhost:8080/api/skill'

// 获取Skill列表（分页）
export async function getSkillList(params = {}) {
  const { page = 1, size = 10, name = '', category = '', status = '' } = params
  const queryParams = { page, size }
  if (name) queryParams.name = name
  if (category) queryParams.category = category
  if (status) queryParams.status = status
  return get(`${BASE_URL}/search`, queryParams)
}

// 获取Skill详情
export async function getSkillDetail(id) {
  return get(`${BASE_URL}/${id}`)
}

// 创建Skill（带文件上传）
export async function createSkill(data, file = null) {
  if (file) {
    const formData = new FormData()
    formData.append('file', file)
    formData.append('data', new Blob([JSON.stringify(data)], { type: 'application/json' }))
    return upload(`${BASE_URL}`, formData)
  }
  return post(`${BASE_URL}`, data)
}

// 更新Skill（支持文件上传）
export async function updateSkill(id, data, file = null) {
  const formData = new FormData()
  if (file) {
    formData.append('file', file)
  }
  formData.append('data', new Blob([JSON.stringify(data)], { type: 'application/json' }))
  return uploadPut(`${BASE_URL}/${id}`, formData)
}

// 删除Skill
export async function deleteSkill(id) {
  return del(`${BASE_URL}/${id}`)
}

// 发布Skill
export async function publishSkill(id) {
  return post(`${BASE_URL}/${id}/publish`)
}

// 取消发布Skill
export async function unpublishSkill(id) {
  return post(`${BASE_URL}/${id}/unpublish`)
}

// 复制Skill
export async function copySkill(id) {
  return post(`${BASE_URL}/${id}/copy`)
}

// 下载执行套件URL
export function getDownloadSuiteUrl(id) {
  return `${BASE_URL}/${id}/download`
}
```

---

## 5. Swagger/OpenAPI 文档

后端已集成Swagger文档，启动应用后访问:
- **Swagger UI**: `http://localhost:8080/swagger-ui.html`
- **OpenAPI JSON**: `http://localhost:8080/v3/api-docs`
