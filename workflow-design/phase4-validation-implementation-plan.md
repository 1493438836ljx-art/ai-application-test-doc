# Phase 4：验证逻辑实现 - 实施计划

> 版本：1.0.0
> 日期：2026-03-31
> 依赖：Phase 2 服务层重构完成

---

## 一、目标

实现工作流配置层的完整验证逻辑，包括：
- 结构验证（开始/结束节点、孤立节点）
- 参数引用验证（引用解析、前置节点校验）
- 循环依赖检测（有向图环检测）
- Skill 兼容性检查

---

## 二、验证器架构

```
WorkflowValidator（协调器）
    ├── StructureValidator（结构验证）
    ├── ParameterReferenceValidator（参数引用验证）
    ├── CyclicDependencyValidator（循环依赖检测）
    └── SkillCompatibilityChecker（Skill 兼容性检查）
```

---

## 三、任务清单

### 3.1 WorkflowValidator（协调器）

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `WorkflowValidator.java` | 验证协调器 | 待完成 |

#### 接口定义

```java
@Service
@RequiredArgsConstructor
public class WorkflowValidator {

    private final StructureValidator structureValidator;
    private final ParameterReferenceValidator paramRefValidator;
    private final CyclicDependencyValidator cyclicValidator;
    private final SkillCompatibilityChecker skillChecker;

    /**
     * 执行完整验证
     */
    public ValidationResult validate(Long workflowId) {
        ValidationResult result = new ValidationResult();
        result.setValid(true);

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

    /**
     * 快速验证（仅结构和循环）
     */
    public ValidationResult quickValidate(Long workflowId) {
        ValidationResult result = new ValidationResult();
        result.setValid(true);

        result.merge(structureValidator.validate(workflowId));
        result.merge(cyclicValidator.validate(workflowId));

        return result;
    }
}
```

---

### 3.2 StructureValidator（结构验证器）

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `StructureValidator.java` | 结构验证器 | 待完成 |

#### 验证规则

| 规则 | 错误码 | 说明 |
|------|--------|------|
| WF_STR_001 | START_NODE_MISSING | 缺少开始节点 |
| WF_STR_002 | START_NODE_DUPLICATE | 存在多个开始节点 |
| WF_STR_003 | END_NODE_MISSING | 缺少结束节点 |
| WF_STR_004 | END_NODE_DUPLICATE | 存在多个结束节点 |
| WF_STR_005 | ORPHAN_NODE | 存在孤立节点（无入边也无出边） |
| WF_STR_006 | START_NODE_HAS_INPUT | 开始节点不能有入边 |
| WF_STR_007 | END_NODE_HAS_OUTPUT | 结束节点不能有出边 |

#### 核心实现

```java
@Service
@RequiredArgsConstructor
public class StructureValidator {

    private final WorkflowNodeMapper nodeMapper;
    private final WorkflowConnectionMapper connectionMapper;

    public ValidationResult validate(Long workflowId) {
        ValidationResult result = new ValidationResult();
        result.setValid(true);

        List<WorkflowNodeEntity> nodes = nodeMapper.selectByWorkflowId(workflowId);
        List<WorkflowConnectionEntity> connections = connectionMapper.selectByWorkflowId(workflowId);

        // 构建节点映射
        Map<String, WorkflowNodeEntity> nodeMap = nodes.stream()
                .collect(Collectors.toMap(WorkflowNodeEntity::getNodeUuid, n -> n));

        // 构建入边和出边映射
        Set<String> hasInput = new HashSet<>();
        Set<String> hasOutput = new HashSet<>();
        for (WorkflowConnectionEntity conn : connections) {
            hasInput.add(conn.getTargetNodeUuid());
            hasOutput.add(conn.getSourceNodeUuid());
        }

        // 1. 验证开始节点
        List<WorkflowNodeEntity> startNodes = nodes.stream()
                .filter(n -> NodeType.START.name().equals(n.getNodeType()))
                .collect(Collectors.toList());

        if (startNodes.isEmpty()) {
            result.addError("WF_STR_001", "缺少开始节点", null, null);
        } else if (startNodes.size() > 1) {
            result.addError("WF_STR_002", "存在多个开始节点", null, null);
        } else {
            String startUuid = startNodes.get(0).getNodeUuid();
            if (hasInput.contains(startUuid)) {
                result.addError("WF_STR_006", "开始节点不能有入边", startUuid, null);
            }
        }

        // 2. 验证结束节点
        List<WorkflowNodeEntity> endNodes = nodes.stream()
                .filter(n -> NodeType.END.name().equals(n.getNodeType()))
                .collect(Collectors.toList());

        if (endNodes.isEmpty()) {
            result.addError("WF_STR_003", "缺少结束节点", null, null);
        } else if (endNodes.size() > 1) {
            result.addError("WF_STR_004", "存在多个结束节点", null, null);
        } else {
            String endUuid = endNodes.get(0).getNodeUuid();
            if (hasOutput.contains(endUuid)) {
                result.addError("WF_STR_007", "结束节点不能有出边", endUuid, null);
            }
        }

        // 3. 验证孤立节点
        for (WorkflowNodeEntity node : nodes) {
            String uuid = node.getNodeUuid();
            // 开始和结束节点不检查孤立
            if (NodeType.START.name().equals(node.getNodeType())
                    || NodeType.END.name().equals(node.getNodeType())) {
                continue;
            }
            if (!hasInput.contains(uuid) && !hasOutput.contains(uuid)) {
                result.addWarning("WF_STR_005", "节点未连接到工作流", uuid, null);
            }
        }

        return result;
    }
}
```

---

### 3.3 ParameterReferenceValidator（参数引用验证器）

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `ParameterReferenceValidator.java` | 参数引用验证器 | 待完成 |

#### 参数引用语法

```
${节点名称.参数名称}
${节点名称.输出.参数名称}
```

#### 验证规则

| 规则 | 错误码 | 说明 |
|------|--------|------|
| WF_PARAM_001 | INVALID_REFERENCE_SYNTAX | 参数引用语法错误 |
| WF_PARAM_002 | NODE_NOT_FOUND | 引用的节点不存在 |
| WF_PARAM_003 | NODE_NOT_PREDECESSOR | 引用的节点不是前置节点 |
| WF_PARAM_004 | PARAM_NOT_FOUND | 引用的参数不存在于目标节点 |
| WF_PARAM_005 | PARAM_TYPE_MISMATCH | 参数类型不匹配 |

#### 核心实现

```java
@Service
@RequiredArgsConstructor
public class ParameterReferenceValidator {

    private final WorkflowNodeMapper nodeMapper;
    private final WorkflowConnectionMapper connectionMapper;

    // 参数引用正则：${节点名.参数名} 或 ${节点名.输出.参数名}
    private static final Pattern PARAM_REF_PATTERN =
            Pattern.compile("\\$\\{([^}]+)\\}");

    public ValidationResult validate(Long workflowId) {
        ValidationResult result = new ValidationResult();
        result.setValid(true);

        List<WorkflowNodeEntity> nodes = nodeMapper.selectByWorkflowId(workflowId);
        List<WorkflowConnectionEntity> connections = connectionMapper.selectByWorkflowId(workflowId);

        // 构建节点名称映射
        Map<String, WorkflowNodeEntity> nameToNode = nodes.stream()
                .collect(Collectors.toMap(WorkflowNodeEntity::getNodeName, n -> n, (a, b) -> a));

        // 构建节点 UUID 映射
        Map<String, WorkflowNodeEntity> uuidToNode = nodes.stream()
                .collect(Collectors.toMap(WorkflowNodeEntity::getNodeUuid, n -> n));

        // 构建前置节点映射
        Map<String, Set<String>> predecessorsMap = buildPredecessorsMap(nodes, connections);

        // 遍历每个节点的输入参数
        for (WorkflowNodeEntity node : nodes) {
            if (node.getInputParams() == null) continue;

            Set<String> predecessors = predecessorsMap.getOrDefault(node.getNodeUuid(), Collections.emptySet());

            // 解析参数引用
            Set<String> references = extractParamReferences(node.getInputParams());

            for (String ref : references) {
                // 解析引用：节点名.参数名
                String[] parts = ref.split("\\.");
                if (parts.length < 2) {
                    result.addError("WF_PARAM_001", "参数引用语法错误: " + ref,
                            node.getNodeUuid(), "inputParams");
                    continue;
                }

                String refNodeName = parts[0];
                String refParamName = parts.length == 2 ? parts[1] : parts[2];

                // 检查节点是否存在
                WorkflowNodeEntity refNode = nameToNode.get(refNodeName);
                if (refNode == null) {
                    result.addError("WF_PARAM_002", "引用的节点不存在: " + refNodeName,
                            node.getNodeUuid(), "inputParams");
                    continue;
                }

                // 检查是否为前置节点
                if (!predecessors.contains(refNode.getNodeUuid())) {
                    result.addError("WF_PARAM_003", "引用的节点不是前置节点: " + refNodeName,
                            node.getNodeUuid(), "inputParams");
                    continue;
                }

                // 检查参数是否存在
                if (!isParamExists(refNode, refParamName)) {
                    result.addWarning("WF_PARAM_004", "引用的参数可能不存在: " + ref,
                            node.getNodeUuid(), "inputParams");
                }
            }
        }

        return result;
    }

    /**
     * 构建前置节点映射（BFS）
     */
    private Map<String, Set<String>> buildPredecessorsMap(
            List<WorkflowNodeEntity> nodes,
            List<WorkflowConnectionEntity> connections) {

        Map<String, Set<String>> predecessorsMap = new HashMap<>();

        // 构建邻接表（反向）
        Map<String, List<String>> reverseAdj = new HashMap<>();
        for (WorkflowConnectionEntity conn : connections) {
            reverseAdj.computeIfAbsent(conn.getTargetNodeUuid(), k -> new ArrayList<>())
                    .add(conn.getSourceNodeUuid());
        }

        // BFS 计算每个节点的所有前置节点
        for (WorkflowNodeEntity node : nodes) {
            Set<String> predecessors = new HashSet<>();
            Queue<String> queue = new LinkedList<>();
            queue.add(node.getNodeUuid());

            while (!queue.isEmpty()) {
                String current = queue.poll();
                List<String> sources = reverseAdj.get(current);
                if (sources != null) {
                    for (String source : sources) {
                        if (predecessors.add(source)) {
                            queue.add(source);
                        }
                    }
                }
            }
            predecessorsMap.put(node.getNodeUuid(), predecessors);
        }

        return predecessorsMap;
    }

    /**
     * 提取参数引用
     */
    private Set<String> extractParamReferences(String inputParams) {
        Set<String> references = new HashSet<>();
        Matcher matcher = PARAM_REF_PATTERN.matcher(inputParams);
        while (matcher.find()) {
            references.add(matcher.group(1));
        }
        return references;
    }

    /**
     * 检查参数是否存在
     */
    private boolean isParamExists(WorkflowNodeEntity node, String paramName) {
        String outputParams = node.getOutputParams();
        if (outputParams == null) return false;

        try {
            List<Map> params = objectMapper.readValue(outputParams, List.class);
            return params.stream().anyMatch(p -> paramName.equals(p.get("name")));
        } catch (Exception e) {
            return false;
        }
    }

    @Autowired
    private ObjectMapper objectMapper;
}
```

---

### 3.4 CyclicDependencyValidator（循环依赖检测器）

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `CyclicDependencyValidator.java` | 循环依赖检测器 | 待完成 |

#### 算法说明

使用 **Kahn 算法**（拓扑排序）检测有向图中是否存在环：
1. 计算所有节点的入度
2. 将入度为 0 的节点加入队列
3. 依次移除节点并更新相邻节点入度
4. 如果最终存在入度不为 0 的节点，则存在环

#### 验证规则

| 规则 | 错误码 | 说明 |
|------|--------|------|
| WF_CYCLE_001 | CYCLIC_DEPENDENCY | 工作流存在循环依赖 |

#### 核心实现

```java
@Service
@RequiredArgsConstructor
public class CyclicDependencyValidator {

    private final WorkflowNodeMapper nodeMapper;
    private final WorkflowConnectionMapper connectionMapper;

    public ValidationResult validate(Long workflowId) {
        ValidationResult result = new ValidationResult();
        result.setValid(true);

        List<WorkflowNodeEntity> nodes = nodeMapper.selectByWorkflowId(workflowId);
        List<WorkflowConnectionEntity> connections = connectionMapper.selectByWorkflowId(workflowId);

        // 构建邻接表和入度表
        Map<String, List<String>> adj = new HashMap<>();
        Map<String, Integer> inDegree = new HashMap<>();

        // 初始化
        for (WorkflowNodeEntity node : nodes) {
            adj.put(node.getNodeUuid(), new ArrayList<>());
            inDegree.put(node.getNodeUuid(), 0);
        }

        // 构建边
        for (WorkflowConnectionEntity conn : connections) {
            adj.get(conn.getSourceNodeUuid()).add(conn.getTargetNodeUuid());
            inDegree.merge(conn.getTargetNodeUuid(), 1, Integer::sum);
        }

        // Kahn 算法
        Queue<String> queue = new LinkedList<>();
        for (Map.Entry<String, Integer> entry : inDegree.entrySet()) {
            if (entry.getValue() == 0) {
                queue.add(entry.getKey());
            }
        }

        int processedCount = 0;
        while (!queue.isEmpty()) {
            String current = queue.poll();
            processedCount++;

            for (String next : adj.get(current)) {
                int newDegree = inDegree.get(next) - 1;
                inDegree.put(next, newDegree);
                if (newDegree == 0) {
                    queue.add(next);
                }
            }
        }

        // 如果处理的节点数小于总节点数，说明存在环
        if (processedCount < nodes.size()) {
            // 找出参与循环的节点
            List<String> cycleNodes = nodes.stream()
                    .filter(n -> inDegree.get(n.getNodeUuid()) > 0)
                    .map(WorkflowNodeEntity::getNodeName)
                    .collect(Collectors.toList());

            result.addError("WF_CYCLE_001",
                    "工作流存在循环依赖，涉及节点: " + String.join(", ", cycleNodes),
                    null, null);
        }

        return result;
    }

    /**
     * 获取节点拓扑排序
     */
    public List<String> getTopologicalOrder(Long workflowId) {
        List<WorkflowNodeEntity> nodes = nodeMapper.selectByWorkflowId(workflowId);
        List<WorkflowConnectionEntity> connections = connectionMapper.selectByWorkflowId(workflowId);

        Map<String, List<String>> adj = new HashMap<>();
        Map<String, Integer> inDegree = new HashMap<>();
        Map<String, String> uuidToName = new HashMap<>();

        for (WorkflowNodeEntity node : nodes) {
            adj.put(node.getNodeUuid(), new ArrayList<>());
            inDegree.put(node.getNodeUuid(), 0);
            uuidToName.put(node.getNodeUuid(), node.getNodeName());
        }

        for (WorkflowConnectionEntity conn : connections) {
            adj.get(conn.getSourceNodeUuid()).add(conn.getTargetNodeUuid());
            inDegree.merge(conn.getTargetNodeUuid(), 1, Integer::sum);
        }

        Queue<String> queue = new LinkedList<>();
        for (Map.Entry<String, Integer> entry : inDegree.entrySet()) {
            if (entry.getValue() == 0) {
                queue.add(entry.getKey());
            }
        }

        List<String> order = new ArrayList<>();
        while (!queue.isEmpty()) {
            String current = queue.poll();
            order.add(uuidToName.get(current));

            for (String next : adj.get(current)) {
                int newDegree = inDegree.get(next) - 1;
                inDegree.put(next, newDegree);
                if (newDegree == 0) {
                    queue.add(next);
                }
            }
        }

        return order;
    }
}
```

---

### 3.5 SkillCompatibilityChecker（Skill 兼容性检查器）

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `SkillCompatibilityChecker.java` | Skill 兼容性检查器 | 待完成 |

#### 验证规则

| 规则 | 错误码 | 说明 | 兼容性状态 |
|------|--------|------|------------|
| WF_SKILL_001 | SKILL_NOT_FOUND | 引用的 Skill 不存在 | INVALID |
| WF_SKILL_002 | SKILL_DISABLED | Skill 已被禁用 | INCOMPATIBLE |
| WF_SKILL_003 | MISSING_REQUIRED_PARAM | 缺少必填参数 | NEEDS_UPDATE |
| WF_SKILL_004 | PARAM_TYPE_MISMATCH | 参数类型不匹配 | NEEDS_UPDATE |
| WF_SKILL_005 | SKILL_VERSION_CHANGED | Skill 版本已变更 | COMPATIBLE（警告） |

#### 核心实现

```java
@Service
@RequiredArgsConstructor
public class SkillCompatibilityChecker {

    private final WorkflowNodeMapper nodeMapper;
    private final SkillService skillService;  // 假设存在

    public ValidationResult check(Long workflowId) {
        ValidationResult result = new ValidationResult();
        result.setValid(true);

        List<WorkflowNodeEntity> nodes = nodeMapper.selectByWorkflowId(workflowId);

        for (WorkflowNodeEntity node : nodes) {
            // 只检查 Skill 节点
            if (!NodeType.SKILL.name().equals(node.getNodeType())) {
                continue;
            }

            if (node.getSkillId() == null) {
                result.addError("WF_SKILL_001", "Skill 节点未配置 Skill",
                        node.getNodeUuid(), "skillId");
                updateCompatibilityStatus(node, CompatibilityStatus.INVALID);
                continue;
            }

            // 获取 Skill 信息
            SkillInfo skill = skillService.getSkillById(node.getSkillId());
            if (skill == null) {
                result.addError("WF_SKILL_001", "引用的 Skill 不存在: " + node.getSkillId(),
                        node.getNodeUuid(), "skillId");
                updateCompatibilityStatus(node, CompatibilityStatus.INVALID);
                continue;
            }

            if (!skill.isEnabled()) {
                result.addError("WF_SKILL_002", "Skill 已被禁用: " + skill.getName(),
                        node.getNodeUuid(), "skillId");
                updateCompatibilityStatus(node, CompatibilityStatus.INCOMPATIBLE);
                continue;
            }

            // 检查参数配置
            List<ValidationError> paramErrors = checkParameters(node, skill);
            result.getErrors().addAll(paramErrors);

            // 检查版本变更
            if (isVersionChanged(node, skill)) {
                result.addWarning("WF_SKILL_005", "Skill 版本已变更，建议重新配置",
                        node.getNodeUuid(), "skillId");
            }

            // 更新兼容性状态
            CompatibilityStatus status = paramErrors.isEmpty()
                    ? CompatibilityStatus.COMPATIBLE
                    : CompatibilityStatus.NEEDS_UPDATE;
            updateCompatibilityStatus(node, status);
        }

        return result;
    }

    /**
     * 检查单个节点的兼容性
     */
    public CompatibilityStatus checkNode(WorkflowNodeEntity node) {
        if (!NodeType.SKILL.name().equals(node.getNodeType())) {
            return CompatibilityStatus.COMPATIBLE;
        }

        if (node.getSkillId() == null) {
            return CompatibilityStatus.INVALID;
        }

        SkillInfo skill = skillService.getSkillById(node.getSkillId());
        if (skill == null || !skill.isEnabled()) {
            return CompatibilityStatus.INCOMPATIBLE;
        }

        List<ValidationError> errors = checkParameters(node, skill);
        return errors.isEmpty() ? CompatibilityStatus.COMPATIBLE : CompatibilityStatus.NEEDS_UPDATE;
    }

    /**
     * 检查参数配置
     */
    private List<ValidationError> checkParameters(WorkflowNodeEntity node, SkillInfo skill) {
        List<ValidationError> errors = new ArrayList<>();

        // 解析节点配置的参数
        Map<String, Object> configuredParams = parseInputParams(node.getInputParams());

        // 获取 Skill 定义的参数
        List<SkillParam> skillParams = skill.getInputParams();

        for (SkillParam param : skillParams) {
            // 检查必填参数
            if (param.isRequired() && !configuredParams.containsKey(param.getName())) {
                errors.add(buildError("WF_SKILL_003",
                        "缺少必填参数: " + param.getName(),
                        node.getNodeUuid(), "inputParams"));
                continue;
            }

            // 检查类型匹配
            if (configuredParams.containsKey(param.getName())) {
                Object value = configuredParams.get(param.getName());
                if (!isTypeMatch(value, param.getType())) {
                    errors.add(buildError("WF_SKILL_004",
                            "参数类型不匹配: " + param.getName() + "，期望: " + param.getType(),
                            node.getNodeUuid(), "inputParams"));
                }
            }
        }

        return errors;
    }

    /**
     * 更新节点兼容性状态
     */
    private void updateCompatibilityStatus(WorkflowNodeEntity node, CompatibilityStatus status) {
        node.setCompatibilityStatus(status.name());
        nodeMapper.updateById(node);
    }

    /**
     * 检查 Skill 版本是否变更
     */
    private boolean isVersionChanged(WorkflowNodeEntity node, SkillInfo skill) {
        if (node.getSkillSnapshot() == null) {
            return true;
        }
        try {
            Map<String, Object> snapshot = objectMapper.readValue(
                    node.getSkillSnapshot(), Map.class);
            String snapshotVersion = (String) snapshot.get("version");
            return !skill.getVersion().equals(snapshotVersion);
        } catch (Exception e) {
            return true;
        }
    }

    private Map<String, Object> parseInputParams(String inputParams) {
        if (inputParams == null) return Collections.emptyMap();
        try {
            return objectMapper.readValue(inputParams, Map.class);
        } catch (Exception e) {
            return Collections.emptyMap();
        }
    }

    private boolean isTypeMatch(Object value, String expectedType) {
        // 简化实现，实际需要更复杂的类型检查
        switch (expectedType.toLowerCase()) {
            case "string":
                return value instanceof String;
            case "number":
            case "integer":
                return value instanceof Number;
            case "boolean":
                return value instanceof Boolean;
            case "array":
                return value instanceof List;
            case "object":
                return value instanceof Map;
            default:
                return true;
        }
    }

    private ValidationError buildError(String code, String message, String nodeUuid, String field) {
        ValidationError error = new ValidationError();
        error.setCode(code);
        error.setMessage(message);
        error.setNodeUuid(nodeUuid);
        error.setField(field);
        return error;
    }

    @Autowired
    private ObjectMapper objectMapper;
}
```

---

## 四、实施步骤

### Step 1：创建 ValidationResult 增强
1. 添加 `addError` 和 `addWarning` 方法
2. 添加 `merge` 方法用于合并结果

### Step 2：实现 StructureValidator
1. 验证开始/结束节点
2. 验证孤立节点
3. 验证边约束

### Step 3：实现 ParameterReferenceValidator
1. 解析参数引用语法
2. 构建前置节点映射
3. 验证引用有效性

### Step 4：实现 CyclicDependencyValidator
1. 实现 Kahn 算法
2. 检测循环依赖
3. 提供拓扑排序方法

### Step 5：实现 SkillCompatibilityChecker
1. 检查 Skill 存在性
2. 检查参数配置
3. 更新兼容性状态

### Step 6：实现 WorkflowValidator
1. 组装所有验证器
2. 实现完整验证流程
3. 实现快速验证流程

---

## 五、代码结构

```
com.example.demo.workflow.service.impl.validation/
├── WorkflowValidator.java
├── StructureValidator.java
├── ParameterReferenceValidator.java
├── CyclicDependencyValidator.java
└── SkillCompatibilityChecker.java
```

---

## 六、ValidationResult 增强

```java
@Data
public class ValidationResult {
    private boolean valid;
    private List<ValidationError> errors = new ArrayList<>();
    private List<ValidationWarning> warnings = new ArrayList<>();

    public void addError(String code, String message, String nodeUuid, String field) {
        ValidationError error = new ValidationError();
        error.setCode(code);
        error.setMessage(message);
        error.setNodeUuid(nodeUuid);
        error.setField(field);
        errors.add(error);
        this.valid = false;
    }

    public void addWarning(String code, String message, String nodeUuid, String field) {
        ValidationWarning warning = new ValidationWarning();
        warning.setCode(code);
        warning.setMessage(message);
        warning.setNodeUuid(nodeUuid);
        warnings.add(warning);
    }

    public void merge(ValidationResult other) {
        if (other == null) return;
        if (!other.isValid()) {
            this.valid = false;
        }
        this.errors.addAll(other.getErrors());
        this.warnings.addAll(other.getWarnings());
    }

    @Data
    public static class ValidationError {
        private String code;
        private String message;
        private String nodeUuid;
        private String field;
    }

    @Data
    public static class ValidationWarning {
        private String code;
        private String message;
        private String nodeUuid;
    }
}
```

---

## 七、验收标准

- [ ] 所有验证器类编译通过
- [ ] 结构验证测试通过
- [ ] 参数引用验证测试通过
- [ ] 循环依赖检测测试通过
- [ ] Skill 兼容性检查测试通过
- [ ] 集成测试通过

---

## 八、单元测试用例

### 8.1 StructureValidator 测试

| 测试场景 | 预期结果 |
|----------|----------|
| 正常工作流 | 验证通过 |
| 缺少开始节点 | WF_STR_001 错误 |
| 多个开始节点 | WF_STR_002 错误 |
| 缺少结束节点 | WF_STR_003 错误 |
| 存在孤立节点 | WF_STR_005 警告 |
| 开始节点有入边 | WF_STR_006 错误 |

### 8.2 CyclicDependencyValidator 测试

| 测试场景 | 预期结果 |
|----------|----------|
| 无环工作流 | 验证通过 |
| 简单环 A→B→C→A | WF_CYCLE_001 错误 |
| 复杂环 | WF_CYCLE_001 错误 |

### 8.3 SkillCompatibilityChecker 测试

| 测试场景 | 预期结果 |
|----------|----------|
| Skill 不存在 | INVALID 状态 |
| Skill 已禁用 | INCOMPATIBLE 状态 |
| 缺少必填参数 | NEEDS_UPDATE 状态 |
| 参数类型不匹配 | NEEDS_UPDATE 状态 |
| 完全兼容 | COMPATIBLE 状态 |

---

## 九、注意事项

1. **性能优化**：前置节点计算可缓存
2. **错误信息**：提供清晰的错误描述
3. **状态更新**：兼容性状态需持久化
4. **并发安全**：验证过程需考虑并发

---

## 十、预计工作量

| 类型 | 数量 | 预计时间 |
|------|------|----------|
| 验证器 | 5 | 6h |
| 单元测试 | 15+ | 4h |
| 集成测试 | 5 | 2h |
| **总计** | **25** | **12h** |

---

## 十一、下一阶段

Phase 4 完成后，进入 **Phase 5：事件集成**
