# 工作流执行框架设计文档

## 第三章：节点执行器设计

### 3.1 执行器体系总览

节点执行器是执行框架的核心扩展点，每种节点类型都有对应的执行器实现。执行器体系采用策略模式，通过统一的接口支持不同类型节点的执行。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          节点执行器体系结构                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                         ┌─────────────────────┐                             │
│                         │   <<interface>>     │                             │
│                         │   NodeExecutor      │                             │
│                         ├─────────────────────┤                             │
│                         │ + getNodeType()     │                             │
│                         │ + execute()         │                             │
│                         │ + validate()        │                             │
│                         └──────────┬──────────┘                             │
│                                    │                                        │
│          ┌─────────────────────────┼─────────────────────────┐             │
│          │                         │                         │             │
│          ▼                         ▼                         ▼             │
│   ┌─────────────┐           ┌─────────────┐           ┌─────────────┐     │
│   │BaseNodeExec │           │LogicNodeExec│           │SkillNodeExec│     │
│   │  (基础节点)  │           │  (逻辑节点)  │           │ (技能节点)   │     │
│   └──────┬──────┘           └──────┬──────┘           └──────┬──────┘     │
│          │                         │                         │             │
│    ┌─────┴─────┐           ┌───────┼───────┐         ┌──────┴──────┐     │
│    │           │           │       │       │         │             │     │
│    ▼           ▼           ▼       ▼       ▼         ▼             ▼     │
│ ┌────┐    ┌────┐      ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐     ┌────┐    │
│ │开始│    │结束│      │条件│ │循环│ │批处│ │异步│ │收集│     │技能│    │
│ │    │    │    │      │    │ │    │ │理  │ │    │ │    │     │    │    │
│ └────┘    └────┘      └────┘ └────┘ └────┘ └────┘ └────┘     └────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.1.1 节点类型与执行器映射表

| 节点类型 | 类型代码 | 分类 | 执行器类 | 核心职责 |
|----------|----------|------|----------|----------|
| 开始节点 | start | BASIC | StartNodeExecutor | 初始化参数，写入上下文 |
| 结束节点 | end | BASIC | EndNodeExecutor | 输出结果，完成执行 |
| 简单条件 | condition_simple | LOGIC | SimpleConditionExecutor | 二分支判断 |
| 多路条件 | condition_multi | LOGIC | MultiConditionExecutor | 多分支判断 |
| 循环节点 | loop | LOGIC | LoopNodeExecutor | 顺序遍历执行 |
| 批处理节点 | batch | LOGIC | BatchNodeExecutor | 并发批量执行 |
| 异步处理节点 | async | LOGIC | AsyncNodeExecutor | 异步触发子工作流 |
| 结果收集节点 | collect | LOGIC | CollectNodeExecutor | 收集异步任务结果 |
| 技能节点 | skill | EXECUTION | SkillNodeExecutor | 执行Skill库中的技能 |

---

### 3.2 基础节点执行器

#### 3.2.1 开始节点执行器

```java
/**
 * 开始节点执行器
 * 负责初始化工作流参数，将输入数据写入执行上下文
 */
@Component
@Slf4j
public class StartNodeExecutor implements NodeExecutor {

    @Override
    public String getNodeType() {
        return "start";
    }

    @Override
    public NodeExecutionResult execute(WorkflowNode node,
                                        Map<String, Object> inputs,
                                        ExecutionContext context) {
        log.info("执行开始节点: nodeUuid={}", node.getUuid());

        Map<String, Object> outputs = new HashMap<>();

        // 获取开始节点定义的输出参数
        OutputConfig outputConfig = node.getOutputConfig();
        if (outputConfig != null && outputConfig.getParameters() != null) {

            for (ParameterConfig param : outputConfig.getParameters()) {
                String paramName = param.getName();

                // 从输入数据中获取参数值
                Object value = inputs.get(paramName);

                // 如果输入中没有，使用默认值
                if (value == null && param.getDefaultValue() != null) {
                    value = param.getDefaultValue();
                }

                // 必填参数校验
                if (param.isRequired() && value == null) {
                    return NodeExecutionResult.failure(
                        "必填参数缺失: " + paramName
                    );
                }

                outputs.put(paramName, value);
            }
        }

        log.info("开始节点执行完成: nodeUuid={}, outputs={}",
                 node.getUuid(), outputs.keySet());

        return NodeExecutionResult.success(outputs);
    }

    @Override
    public ValidationResult validate(WorkflowNode node) {
        // 验证开始节点配置
        if (node.getOutputConfig() == null) {
            return ValidationResult.failure("开始节点必须定义输出参数");
        }
        return ValidationResult.success();
    }
}
```

#### 3.2.2 结束节点执行器

```java
/**
 * 结束节点执行器
 * 负责收集最终输出结果，标记工作流完成
 */
@Component
@Slf4j
public class EndNodeExecutor implements NodeExecutor {

    private final StateManager stateManager;

    @Override
    public String getNodeType() {
        return "end";
    }

    @Override
    public NodeExecutionResult execute(WorkflowNode node,
                                        Map<String, Object> inputs,
                                        ExecutionContext context) {
        log.info("执行结束节点: nodeUuid={}", node.getUuid());

        Map<String, Object> outputs = new HashMap<>();

        // 获取结束节点定义的输出参数
        OutputConfig outputConfig = node.getOutputConfig();
        if (outputConfig != null && outputConfig.getParameters() != null) {

            for (ParameterConfig param : outputConfig.getParameters()) {
                String paramName = param.getName();

                // 从输入中获取参数值（已经由参数解析器解析好）
                Object value = inputs.get(paramName);
                outputs.put(paramName, value);
            }
        }

        // 更新工作流最终输出
        stateManager.updateWorkflowOutput(
            context.getExecutionId(),
            outputs
        );

        log.info("结束节点执行完成: nodeUuid={}, outputs={}",
                 node.getUuid(), outputs.keySet());

        return NodeExecutionResult.success(outputs);
    }
}
```

---

### 3.3 逻辑节点执行器

#### 3.3.1 简单条件节点执行器

```java
/**
 * 简单条件节点执行器
 * 实现二分支条件判断（是/否）
 */
@Component
@Slf4j
public class SimpleConditionExecutor implements NodeExecutor {

    private final ConditionEvaluator conditionEvaluator;

    @Override
    public String getNodeType() {
        return "condition_simple";
    }

    @Override
    public NodeExecutionResult execute(WorkflowNode node,
                                        Map<String, Object> inputs,
                                        ExecutionContext context) {
        log.info("执行简单条件节点: nodeUuid={}, nodeName={}",
                 node.getUuid(), node.getName());

        // 获取条件配置
        ConditionConfig conditionConfig = parseConditionConfig(node.getConditions());
        Expression expression = conditionConfig.getExpression();

        // 评估条件表达式
        boolean result = conditionEvaluator.evaluate(expression, context);

        log.info("条件评估结果: nodeUuid={}, result={}", node.getUuid(), result);

        // 确定下一步要执行的分支
        List<String> nextNodes = new ArrayList<>();
        String branchLabel = result ? "true" : "false";

        // 从连线中获取对应分支的目标节点
        for (WorkflowConnection conn : context.getDefinition().getConnections()) {
            if (conn.getSourceNodeUuid().equals(node.getUuid())) {
                if (branchLabel.equals(conn.getBranchLabel())) {
                    nextNodes.add(conn.getTargetNodeUuid());
                }
            }
        }

        Map<String, Object> outputs = new HashMap<>();
        outputs.put("condition_result", result);
        outputs.put("branch", branchLabel);

        return NodeExecutionResult.builder()
            .success(true)
            .outputs(outputs)
            .nextNodes(nextNodes)
            .build();
    }
}
```

#### 3.3.2 多路条件节点执行器

```java
/**
 * 多路条件节点执行器
 * 实现多分支条件判断（类似 switch-case）
 */
@Component
@Slf4j
public class MultiConditionExecutor implements NodeExecutor {

    private final ConditionEvaluator conditionEvaluator;

    @Override
    public String getNodeType() {
        return "condition_multi";
    }

    @Override
    public NodeExecutionResult execute(WorkflowNode node,
                                        Map<String, Object> inputs,
                                        ExecutionContext context) {
        log.info("执行多路条件节点: nodeUuid={}, nodeName={}",
                 node.getUuid(), node.getName());

        // 获取条件配置
        MultiConditionConfig config = parseMultiConditionConfig(node.getConditions());

        // 按优先级排序评估各分支条件
        List<CaseConfig> sortedCases = config.getCases().stream()
            .sorted(Comparator.comparingInt(CaseConfig::getPriority))
            .collect(Collectors.toList());

        String matchedCaseId = null;

        for (CaseConfig caseConfig : sortedCases) {
            boolean matched = conditionEvaluator.evaluate(caseConfig.getExpression(), context);

            if (matched) {
                matchedCaseId = caseConfig.getId();
                log.info("条件分支匹配: caseId={}, label={}",
                         caseConfig.getId(), caseConfig.getLabel());
                break;
            }
        }

        // 如果没有匹配的分支，使用默认分支
        if (matchedCaseId == null && config.getDefaultCase() != null) {
            matchedCaseId = config.getDefaultCase().getId();
            log.info("使用默认分支: caseId={}", matchedCaseId);
        }

        // 获取对应分支的目标节点
        List<String> nextNodes = new ArrayList<>();
        for (WorkflowConnection conn : context.getDefinition().getConnections()) {
            if (conn.getSourceNodeUuid().equals(node.getUuid())) {
                if (matchedCaseId != null && matchedCaseId.equals(conn.getBranchLabel())) {
                    nextNodes.add(conn.getTargetNodeUuid());
                }
            }
        }

        Map<String, Object> outputs = new HashMap<>();
        outputs.put("matched_case", matchedCaseId);

        return NodeExecutionResult.builder()
            .success(true)
            .outputs(outputs)
            .nextNodes(nextNodes)
            .build();
    }
}
```

#### 3.3.3 条件表达式评估器

```java
/**
 * 条件表达式评估器
 * 支持多种比较运算符
 */
@Component
public class ConditionEvaluator {

    /**
     * 评估条件表达式
     */
    public boolean evaluate(Expression expression, ExecutionContext context) {
        Object leftValue = resolveOperand(expression.getLeftOperand(), context);
        Object rightValue = resolveOperand(expression.getRightOperand(), context);
        String operator = expression.getOperator();

        return compare(leftValue, rightValue, operator);
    }

    /**
     * 解析操作数
     */
    private Object resolveOperand(Object operand, ExecutionContext context) {
        if (operand == null) {
            return null;
        }

        String strOperand = operand.toString();

        // 如果是引用表达式 ${xxx.yyy}
        if (strOperand.startsWith("${") && strOperand.endsWith("}")) {
            String ref = strOperand.substring(2, strOperand.length() - 1);
            String[] parts = ref.split("\\.", 2);

            if (parts.length == 2) {
                return context.getNodeOutput(parts[0], parts[1]);
            }
        }

        return operand;
    }

    /**
     * 执行比较
     */
    private boolean compare(Object left, Object right, String operator) {
        switch (operator.toLowerCase()) {
            case "equals":
            case "==":
                return Objects.equals(left, right);

            case "notequals":
            case "!=":
                return !Objects.equals(left, right);

            case "greaterthan":
            case ">":
                return toNumber(left) > toNumber(right);

            case "greaterthanorequals":
            case ">=":
                return toNumber(left) >= toNumber(right);

            case "lessthan":
            case "<":
                return toNumber(left) < toNumber(right);

            case "lessthanorequals":
            case "<=":
                return toNumber(left) <= toNumber(right);

            case "contains":
                return left != null && right != null &&
                       left.toString().contains(right.toString());

            case "startswith":
                return left != null && right != null &&
                       left.toString().startsWith(right.toString());

            case "endswith":
                return left != null && right != null &&
                       left.toString().endsWith(right.toString());

            case "isnull":
                return left == null;

            case "isnotnull":
                return left != null;

            case "isempty":
                return left == null || left.toString().isEmpty();

            case "isnotempty":
                return left != null && !left.toString().isEmpty();

            case "between":
                // right 应该是包含两个元素的数组 [min, max]
                if (right instanceof List) {
                    List<?> range = (List<?>) right;
                    if (range.size() >= 2) {
                        double value = toNumber(left);
                        double min = toNumber(range.get(0));
                        double max = toNumber(range.get(1));
                        return value >= min && value <= max;
                    }
                }
                return false;

            case "in":
                // right 应该是数组，检查 left 是否在其中
                if (right instanceof Collection) {
                    return ((Collection<?>) right).contains(left);
                }
                return false;

            default:
                throw new IllegalArgumentException("未知的比较运算符: " + operator);
        }
    }

    /**
     * 转换为数值
     */
    private double toNumber(Object value) {
        if (value == null) {
            return 0;
        }
        if (value instanceof Number) {
            return ((Number) value).doubleValue();
        }
        return Double.parseDouble(value.toString());
    }
}
```

---

### 3.4 循环节点执行器

#### 3.4.1 循环节点执行器实现

```java
/**
 * 循环节点执行器
 * 支持三种循环类型：计数循环、数组遍历、条件循环
 */
@Component
@Slf4j
public class LoopNodeExecutor implements NodeExecutor {

    private final ExecutionEngine executionEngine;
    private final ParameterResolver parameterResolver;

    @Override
    public String getNodeType() {
        return "loop";
    }

    @Override
    public NodeExecutionResult execute(WorkflowNode node,
                                        Map<String, Object> inputs,
                                        ExecutionContext context) {
        log.info("执行循环节点: nodeUuid={}, nodeName={}",
                 node.getUuid(), node.getName());

        LoopConfig loopConfig = parseLoopConfig(node.getLoopConfig());

        // 获取循环体节点
        List<WorkflowNode> bodyNodes = getLoopBodyNodes(node, context);

        if (bodyNodes.isEmpty()) {
            log.warn("循环体为空: nodeUuid={}", node.getUuid());
            return NodeExecutionResult.success();
        }

        // 根据循环类型执行
        List<Object> collectedOutputs = new ArrayList<>();

        switch (loopConfig.getType()) {
            case COUNT:
                collectedOutputs = executeCountLoop(node, loopConfig, bodyNodes, context);
                break;

            case ARRAY:
                collectedOutputs = executeArrayLoop(node, loopConfig, bodyNodes, context);
                break;

            case CONDITION:
                collectedOutputs = executeConditionLoop(node, loopConfig, bodyNodes, context);
                break;

            default:
                return NodeExecutionResult.failure("未知的循环类型: " + loopConfig.getType());
        }

        // 构建输出
        Map<String, Object> outputs = new HashMap<>();
        outputs.put("loop_results", collectedOutputs);
        outputs.put("iteration_count", collectedOutputs.size());

        log.info("循环节点执行完成: nodeUuid={}, iterations={}",
                 node.getUuid(), collectedOutputs.size());

        return NodeExecutionResult.success(outputs);
    }

    /**
     * 计数循环
     */
    private List<Object> executeCountLoop(WorkflowNode node,
                                           LoopConfig config,
                                           List<WorkflowNode> bodyNodes,
                                           ExecutionContext context) {
        int times = config.getTimes();
        List<Object> results = new ArrayList<>();

        log.info("开始计数循环: times={}", times);

        for (int i = 0; i < times; i++) {
            // 进入循环上下文
            context.enterLoop(node.getUuid(), i, i);

            // 执行循环体
            Object iterationResult = executeLoopBody(bodyNodes, context);
            results.add(iterationResult);

            // 退出循环上下文
            context.exitLoop();

            log.debug("循环迭代完成: index={}/{}", i + 1, times);
        }

        return results;
    }

    /**
     * 数组遍历循环
     */
    private List<Object> executeArrayLoop(WorkflowNode node,
                                           LoopConfig config,
                                           List<WorkflowNode> bodyNodes,
                                           ExecutionContext context) {
        // 解析数组源
        List<?> items = resolveArraySource(config.getArraySource(), context);

        if (items == null || items.isEmpty()) {
            log.warn("数组为空，跳过循环: nodeUuid={}", node.getUuid());
            return Collections.emptyList();
        }

        List<Object> results = new ArrayList<>();

        log.info("开始数组遍历循环: itemsCount={}", items.size());

        for (int i = 0; i < items.size(); i++) {
            Object currentItem = items.get(i);

            // 进入循环上下文
            context.enterLoop(node.getUuid(), currentItem, i);

            // 设置当前元素到上下文变量
            context.setGlobalVariable("current_item", currentItem);
            context.setGlobalVariable("current_index", i);

            // 执行循环体
            Object iterationResult = executeLoopBody(bodyNodes, context);
            results.add(iterationResult);

            // 退出循环上下文
            context.exitLoop();

            log.debug("循环迭代完成: index={}/{}, item={}",
                     i + 1, items.size(), currentItem);
        }

        return results;
    }

    /**
     * 条件循环（类似 while）
     */
    private List<Object> executeConditionLoop(WorkflowNode node,
                                               LoopConfig config,
                                               List<WorkflowNode> bodyNodes,
                                               ExecutionContext context) {
        List<Object> results = new ArrayList<>();
        int maxIterations = config.getMaxIterations() > 0 ?
                            config.getMaxIterations() : 10000; // 防止无限循环

        int iteration = 0;
        boolean conditionMet = true;

        log.info("开始条件循环: maxIterations={}", maxIterations);

        while (conditionMet && iteration < maxIterations) {
            // 评估循环条件
            conditionMet = evaluateLoopCondition(config.getCondition(), context);

            if (!conditionMet) {
                log.debug("循环条件不满足，退出循环: iteration={}", iteration);
                break;
            }

            // 进入循环上下文
            context.enterLoop(node.getUuid(), iteration, iteration);

            // 执行循环体
            Object iterationResult = executeLoopBody(bodyNodes, context);
            results.add(iterationResult);

            // 退出循环上下文
            context.exitLoop();

            iteration++;
            log.debug("条件循环迭代完成: iteration={}", iteration);
        }

        if (iteration >= maxIterations) {
            log.warn("条件循环达到最大迭代次数限制: maxIterations={}", maxIterations);
        }

        return results;
    }

    /**
     * 执行循环体
     */
    private Object executeLoopBody(List<WorkflowNode> bodyNodes,
                                    ExecutionContext context) {
        Map<String, Object> bodyOutputs = new HashMap<>();

        for (WorkflowNode bodyNode : bodyNodes) {
            try {
                // 解析输入参数
                Map<String, Object> inputs = parameterResolver.resolveInputs(
                    bodyNode, context);

                // 获取执行器
                NodeExecutor executor = executorRegistry.getExecutor(bodyNode.getType());

                // 执行节点
                NodeExecutionResult result = executor.execute(bodyNode, inputs, context);

                if (result.isSuccess()) {
                    bodyOutputs.putAll(result.getOutputs());
                    // 存储节点输出到上下文
                    context.setNodeOutputs(bodyNode.getUuid(), result.getOutputs());
                } else {
                    // 循环体内节点执行失败处理
                    handleBodyNodeFailure(bodyNode, result, context);
                }

            } catch (Exception e) {
                log.error("循环体节点执行异常: nodeUuid={}", bodyNode.getUuid(), e);
                throw new ExecutionException("循环体执行失败", e);
            }
        }

        return bodyOutputs;
    }

    /**
     * 获取循环体节点
     */
    private List<WorkflowNode> getLoopBodyNodes(WorkflowNode loopNode,
                                                 ExecutionContext context) {
        return context.getDefinition().getNodes().stream()
            .filter(n -> loopNode.getUuid().equals(n.getParentNodeUuid()))
            .sorted(Comparator.comparing(WorkflowNode::getPositionY))
            .collect(Collectors.toList());
    }

    /**
     * 解析数组源
     */
    private List<?> resolveArraySource(ArraySource arraySource, ExecutionContext context) {
        if (arraySource == null) {
            return Collections.emptyList();
        }

        Object value;
        if ("reference".equals(arraySource.getType())) {
            // 解析引用
            String ref = arraySource.getValue();
            // ... 解析逻辑
            value = resolveReference(ref, context);
        } else {
            value = arraySource.getValue();
        }

        if (value instanceof List) {
            return (List<?>) value;
        }

        if (value instanceof Object[]) {
            return Arrays.asList((Object[]) value);
        }

        // 单个值包装成列表
        return Collections.singletonList(value);
    }
}
```

---

### 3.5 批处理节点执行器

#### 3.5.1 批处理节点执行器实现

```java
/**
 * 批处理节点执行器
 * 并发执行子工作流，等待全部完成后合并结果
 */
@Component
@Slf4j
public class BatchNodeExecutor implements NodeExecutor {

    private final AsyncTaskManager asyncTaskManager;
    private final ParameterResolver parameterResolver;

    @Override
    public String getNodeType() {
        return "batch";
    }

    @Override
    public NodeExecutionResult execute(WorkflowNode node,
                                        Map<String, Object> inputs,
                                        ExecutionContext context) {
        log.info("执行批处理节点: nodeUuid={}, nodeName={}",
                 node.getUuid(), node.getName());

        BatchConfig batchConfig = parseBatchConfig(node.getBatchConfig());

        // 解析要处理的数组
        List<?> items = resolveArraySource(batchConfig.getArraySource(), context);

        if (items == null || items.isEmpty()) {
            log.warn("批处理数组为空: nodeUuid={}", node.getUuid());
            return NodeExecutionResult.success(Map.of(
                "batch_results", Collections.emptyList(),
                "total_count", 0,
                "success_count", 0,
                "failed_count", 0
            ));
        }

        int concurrency = batchConfig.getConcurrency();
        String failureStrategy = batchConfig.getFailureStrategy();
        String outputMode = batchConfig.getOutputMode();

        log.info("开始批处理: itemsCount={}, concurrency={}, failureStrategy={}",
                 items.size(), concurrency, failureStrategy);

        // 创建批处理任务
        List<BatchTask> tasks = createBatchTasks(node, items, context);

        // 并发执行
        List<BatchTaskResult> results = executeBatchTasks(
            tasks, concurrency, failureStrategy, context
        );

        // 处理输出
        Map<String, Object> outputs = processBatchResults(results, outputMode);

        log.info("批处理节点执行完成: nodeUuid={}, total={}, success={}, failed={}",
                 node.getUuid(), results.size(),
                 results.stream().filter(BatchTaskResult::isSuccess).count(),
                 results.stream().filter(r -> !r.isSuccess()).count());

        return NodeExecutionResult.success(outputs);
    }

    /**
     * 创建批处理任务
     */
    private List<BatchTask> createBatchTasks(WorkflowNode node,
                                              List<?> items,
                                              ExecutionContext context) {
        List<BatchTask> tasks = new ArrayList<>();

        // 获取批处理体节点
        List<WorkflowNode> bodyNodes = getBatchBodyNodes(node, context);

        for (int i = 0; i < items.size(); i++) {
            BatchTask task = BatchTask.builder()
                .taskId(UUID.randomUUID().toString())
                .batchNodeUuid(node.getUuid())
                .index(i)
                .item(items.get(i))
                .bodyNodes(bodyNodes)
                .status(BatchTaskStatus.PENDING)
                .build();

            tasks.add(task);
        }

        return tasks;
    }

    /**
     * 并发执行批处理任务
     */
    private List<BatchTaskResult> executeBatchTasks(List<BatchTask> tasks,
                                                     int concurrency,
                                                     String failureStrategy,
                                                     ExecutionContext context) {
        List<BatchTaskResult> results = Collections.synchronizedList(new ArrayList<>());
        AtomicInteger failedCount = new AtomicInteger(0);
        AtomicBoolean shouldStop = new AtomicBoolean(false);

        // 使用信号量控制并发
        Semaphore semaphore = new Semaphore(concurrency);

        List<CompletableFuture<Void>> futures = new ArrayList<>();

        for (BatchTask task : tasks) {
            // 检查是否应该停止
            if (shouldStop.get() && "STOP_ALL".equals(failureStrategy)) {
                task.setStatus(BatchTaskStatus.SKIPPED);
                results.add(BatchTaskResult.skipped(task, "因前置任务失败而跳过"));
                continue;
            }

            try {
                semaphore.acquire();

                CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
                    try {
                        // 设置当前元素到上下文
                        context.setGlobalVariable("current_item", task.getItem());
                        context.setGlobalVariable("current_index", task.getIndex());

                        // 执行批处理体
                        BatchTaskResult result = executeBatchTask(task, context);
                        results.add(result);

                        if (!result.isSuccess()) {
                            failedCount.incrementAndGet();

                            if ("STOP_ALL".equals(failureStrategy)) {
                                shouldStop.set(true);
                            }
                        }

                    } finally {
                        semaphore.release();
                    }
                });

                futures.add(future);

            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }

        // 等待所有任务完成
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();

        return results;
    }

    /**
     * 执行单个批处理任务
     */
    private BatchTaskResult executeBatchTask(BatchTask task,
                                              ExecutionContext context) {
        task.setStatus(BatchTaskStatus.RUNNING);
        task.setStartTime(LocalDateTime.now());

        try {
            Map<String, Object> taskOutputs = new HashMap<>();

            for (WorkflowNode bodyNode : task.getBodyNodes()) {
                Map<String, Object> inputs = parameterResolver.resolveInputs(
                    bodyNode, context);

                NodeExecutor executor = executorRegistry.getExecutor(bodyNode.getType());
                NodeExecutionResult result = executor.execute(bodyNode, inputs, context);

                if (!result.isSuccess()) {
                    return BatchTaskResult.failure(task, result.getErrorMessage());
                }

                taskOutputs.putAll(result.getOutputs());
            }

            task.setStatus(BatchTaskStatus.SUCCESS);
            return BatchTaskResult.success(task, taskOutputs);

        } catch (Exception e) {
            task.setStatus(BatchTaskStatus.FAILED);
            return BatchTaskResult.failure(task, e.getMessage());

        } finally {
            task.setEndTime(LocalDateTime.now());
        }
    }

    /**
     * 处理批处理结果
     */
    private Map<String, Object> processBatchResults(List<BatchTaskResult> results,
                                                     String outputMode) {
        List<Object> successOutputs = results.stream()
            .filter(BatchTaskResult::isSuccess)
            .map(BatchTaskResult::getOutput)
            .collect(Collectors.toList());

        Map<String, Object> outputs = new HashMap<>();

        if ("MERGE".equals(outputMode)) {
            outputs.put("batch_results", successOutputs);
        } else {
            // SEPARATE - 保持独立
            outputs.put("batch_results", results);
        }

        outputs.put("total_count", results.size());
        outputs.put("success_count", successOutputs.size());
        outputs.put("failed_count", results.size() - successOutputs.size());

        return outputs;
    }
}
```

#### 3.5.2 批处理相关类定义

```java
/**
 * 批处理任务
 */
@Data
@Builder
public class BatchTask {
    private String taskId;
    private String batchNodeUuid;
    private int index;
    private Object item;
    private List<WorkflowNode> bodyNodes;
    private BatchTaskStatus status;
    private LocalDateTime startTime;
    private LocalDateTime endTime;
}

/**
 * 批处理任务状态
 */
public enum BatchTaskStatus {
    PENDING,
    RUNNING,
    SUCCESS,
    FAILED,
    SKIPPED
}

/**
 * 批处理任务结果
 */
@Data
@Builder
public class BatchTaskResult {
    private BatchTask task;
    private boolean success;
    private Map<String, Object> output;
    private String errorMessage;

    public static BatchTaskResult success(BatchTask task, Map<String, Object> output) {
        return BatchTaskResult.builder()
            .task(task)
            .success(true)
            .output(output)
            .build();
    }

    public static BatchTaskResult failure(BatchTask task, String errorMessage) {
        return BatchTaskResult.builder()
            .task(task)
            .success(false)
            .errorMessage(errorMessage)
            .build();
    }

    public static BatchTaskResult skipped(BatchTask task, String reason) {
        return BatchTaskResult.builder()
            .task(task)
            .success(true)
            .errorMessage(reason)
            .build();
    }
}
```

---

### 3.6 异步处理节点执行器

#### 3.6.1 异步处理节点执行器实现

```java
/**
 * 异步处理节点执行器
 * 异步触发子工作流，不阻塞主流程
 */
@Component
@Slf4j
public class AsyncNodeExecutor implements NodeExecutor {

    private final AsyncTaskManager asyncTaskManager;
    private final KafkaTaskProducer kafkaTaskProducer;

    @Override
    public String getNodeType() {
        return "async";
    }

    @Override
    public NodeExecutionResult execute(WorkflowNode node,
                                        Map<String, Object> inputs,
                                        ExecutionContext context) {
        log.info("执行异步处理节点: nodeUuid={}, nodeName={}",
                 node.getUuid(), node.getName());

        AsyncConfig asyncConfig = parseAsyncConfig(node.getAsyncConfig());

        List<AsyncTaskInfo> tasks;
        int taskCount;

        if ("SINGLE".equals(asyncConfig.getType())) {
            // 单次执行
            tasks = Collections.singletonList(
                createAsyncTask(node, null, 0, inputs, context)
            );
            taskCount = 1;
        } else {
            // 数组遍历
            List<?> items = resolveArraySource(asyncConfig.getArraySource(), context);
            tasks = new ArrayList<>();

            for (int i = 0; i < items.size(); i++) {
                tasks.add(createAsyncTask(node, items.get(i), i, inputs, context));
            }

            taskCount = items.size();
        }

        // 注册异步任务到上下文
        context.registerAsyncTasks(node.getUuid(), tasks);

        // 触发异步执行（通过 Kafka）
        triggerAsyncTasks(tasks, asyncConfig.getConcurrency());

        log.info("异步任务已触发: nodeUuid={}, taskCount={}", node.getUuid(), taskCount);

        // 立即返回，不等待
        return NodeExecutionResult.success(Map.of(
            "task_count", taskCount,
            "async_triggered", true,
            "async_node_uuid", node.getUuid()
        ));
    }

    /**
     * 创建异步任务
     */
    private AsyncTaskInfo createAsyncTask(WorkflowNode node,
                                           Object item,
                                           int index,
                                           Map<String, Object> inputs,
                                           ExecutionContext context) {
        Map<String, Object> taskInput = new HashMap<>(inputs);

        // 设置循环变量
        if (item != null) {
            taskInput.put("current_item", item);
            taskInput.put("current_index", index);
        }

        return AsyncTaskInfo.builder()
            .taskId(UUID.randomUUID().toString())
            .executionId(context.getExecutionId())
            .asyncNodeUuid(node.getUuid())
            .loopIndex(index)
            .input(taskInput)
            .status(AsyncTaskStatus.PENDING)
            .build();
    }

    /**
     * 触发异步任务
     */
    private void triggerAsyncTasks(List<AsyncTaskInfo> tasks, int concurrency) {
        // 使用信号量控制并发
        Semaphore semaphore = new Semaphore(concurrency);

        for (AsyncTaskInfo task : tasks) {
            try {
                semaphore.acquire();

                // 更新状态
                task.setStatus(AsyncTaskStatus.RUNNING);
                task.setStartTime(LocalDateTime.now());

                // 发送任务到 Kafka
                kafkaTaskProducer.sendAsyncTaskCommand(task);

            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                task.setStatus(AsyncTaskStatus.FAILED);
                task.setErrorMessage("任务被中断");
            } finally {
                semaphore.release();
            }
        }
    }
}
```

---

### 3.7 结果收集节点执行器

```java
/**
 * 结果收集节点执行器
 * 等待并收集异步任务的输出结果
 */
@Component
@Slf4j
public class CollectNodeExecutor implements NodeExecutor {

    private final AsyncTaskManager asyncTaskManager;
    private final KafkaResultConsumer kafkaResultConsumer;

    @Override
    public String getNodeType() {
        return "collect";
    }

    @Override
    public NodeExecutionResult execute(WorkflowNode node,
                                        Map<String, Object> inputs,
                                        ExecutionContext context) {
        log.info("执行结果收集节点: nodeUuid={}, nodeName={}",
                 node.getUuid(), node.getName());

        CollectConfig collectConfig = parseCollectConfig(node.getCollectConfig());

        // 获取关联的异步任务
        String asyncNodeUuid = collectConfig.getAsyncNodeId();
        List<AsyncTaskInfo> tasks = context.getAsyncTasks(asyncNodeUuid);

        if (tasks == null || tasks.isEmpty()) {
            log.warn("没有找到关联的异步任务: asyncNodeUuid={}", asyncNodeUuid);
            return NodeExecutionResult.success(Map.of(
                "async_results", Collections.emptyList(),
                "total_count", 0,
                "success_count", 0,
                "failed_count", 0
            ));
        }

        log.info("等待异步任务完成: taskCount={}, waitStrategy={}",
                 tasks.size(), collectConfig.getWaitStrategy());

        // 等待任务完成
        List<AsyncTaskResult> results = waitForCompletion(
            tasks,
            collectConfig.getWaitStrategy(),
            collectConfig.getWaitCount(),
            collectConfig.getTimeout()
        );

        // 处理结果
        Map<String, Object> outputs = processResults(
            results,
            collectConfig.getOutputMode(),
            collectConfig.getFailureHandling()
        );

        log.info("结果收集完成: nodeUuid={}, total={}, success={}, failed={}",
                 node.getUuid(), results.size(),
                 results.stream().filter(AsyncTaskResult::isSuccess).count(),
                 results.stream().filter(r -> !r.isSuccess()).count());

        return NodeExecutionResult.success(outputs);
    }

    /**
     * 等待任务完成
     */
    private List<AsyncTaskResult> waitForCompletion(List<AsyncTaskInfo> tasks,
                                                     String waitStrategy,
                                                     Integer waitCount,
                                                     long timeout) {
        List<AsyncTaskResult> results = new ArrayList<>();
        long startTime = System.currentTimeMillis();

        int targetCount;
        switch (waitStrategy) {
            case "FIRST":
                targetCount = 1;
                break;
            case "N":
                targetCount = waitCount != null ? waitCount : tasks.size();
                break;
            case "ALL":
            default:
                targetCount = tasks.size();
                break;
        }

        // 等待结果
        while (results.size() < targetCount) {
            // 检查超时
            if (System.currentTimeMillis() - startTime > timeout) {
                log.warn("等待异步任务超时: completed={}, target={}",
                         results.size(), targetCount);
                break;
            }

            // 检查每个任务的状态
            for (AsyncTaskInfo task : tasks) {
                if (task.getStatus() == AsyncTaskStatus.SUCCESS ||
                    task.getStatus() == AsyncTaskStatus.FAILED) {

                    // 避免重复添加
                    boolean alreadyAdded = results.stream()
                        .anyMatch(r -> r.getTaskId().equals(task.getTaskId()));

                    if (!alreadyAdded) {
                        results.add(AsyncTaskResult.from(task));
                    }
                }
            }

            // 短暂等待
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }

        return results;
    }

    /**
     * 处理结果
     */
    private Map<String, Object> processResults(List<AsyncTaskResult> results,
                                                String outputMode,
                                                String failureHandling) {
        List<AsyncTaskResult> successResults = results.stream()
            .filter(AsyncTaskResult::isSuccess)
            .collect(Collectors.toList());

        List<AsyncTaskResult> failedResults = results.stream()
            .filter(r -> !r.isSuccess())
            .collect(Collectors.toList());

        // 检查失败处理策略
        if (!failedResults.isEmpty() && "FAIL_FAST".equals(failureHandling)) {
            throw new ExecutionException(
                "异步任务存在失败，快速失败: " +
                failedResults.stream()
                    .map(r -> r.getTaskId() + ": " + r.getErrorMessage())
                    .collect(Collectors.joining(", "))
            );
        }

        Map<String, Object> outputs = new HashMap<>();

        // 根据输出模式处理
        switch (outputMode) {
            case "FIRST_SUCCESS":
                if (!successResults.isEmpty()) {
                    outputs.put("async_results", successResults.get(0).getOutput());
                }
                break;

            case "SEPARATE":
                outputs.put("async_results", results);
                break;

            case "MERGE":
            default:
                outputs.put("async_results",
                    successResults.stream()
                        .map(AsyncTaskResult::getOutput)
                        .collect(Collectors.toList())
                );
                break;
        }

        outputs.put("total_count", results.size());
        outputs.put("success_count", successResults.size());
        outputs.put("failed_count", failedResults.size());

        return outputs;
    }
}
```

---

### 3.8 技能节点执行器

#### 3.8.1 技能节点执行器架构

技能节点执行器是执行框架中最重要的执行器之一，负责调用 Skill 库中的技能执行测试任务。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        技能执行器架构                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                         ┌─────────────────────┐                             │
│                         │   SkillNodeExecutor │                             │
│                         │   (技能执行入口)     │                             │
│                         └──────────┬──────────┘                             │
│                                    │                                        │
│              ┌─────────────────────┼─────────────────────┐                  │
│              ▼                                           ▼                  │
│   ┌─────────────────────┐                   ┌─────────────────────┐        │
│   │  ExecutionTypeRouter │                   │  LocationRouter     │        │
│   │  (执行方式路由)       │                   │  (执行位置路由)      │        │
│   └──────────┬──────────┘                   └──────────┬──────────┘        │
│              │                                           │                  │
│       ┌──────┴──────┐                             ┌──────┴──────┐           │
│       ▼             ▼                             ▼             ▼           │
│ ┌──────────┐  ┌──────────┐               ┌──────────┐  ┌──────────┐       │
│ │ AI Agent │  │自动化脚本 │               │ Client端  │  │ Service端 │       │
│ │ Executor │  │ Executor │               │Executor  │  │ Executor  │       │
│ └──────────┘  └──────────┘               └──────────┘  └──────────┘       │
│       │             │                           │             │            │
│       └──────┬──────┘                           └──────┬──────┘            │
│              │                                         │                   │
│              ▼                                         ▼                   │
│   ┌─────────────────────────────────────────────────────────────┐         │
│   │                    SkillExecutorRegistry                     │         │
│   │  管理所有组合执行器：                                         │         │
│   │  - AI_AGENT + CLIENT   → AIAgentClientExecutor              │         │
│   │  - AI_AGENT + SERVICE  → AIAgentServiceExecutor             │         │
│   │  - AUTOMATED + CLIENT  → AutomatedClientExecutor            │         │
│   │  - AUTOMATED + SERVICE → AutomatedServiceExecutor           │         │
│   └─────────────────────────────────────────────────────────────┘         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 3.8.2 执行方式 + 执行位置 组合矩阵

| 执行方式 | Client端执行机 | Service端执行机 |
|----------|----------------|-----------------|
| AI Agent执行 | AIAgentClientExecutor | AIAgentServiceExecutor |
| 自动化脚本执行 | AutomatedClientExecutor | AutomatedServiceExecutor |

#### 3.8.3 技能节点执行器实现

```java
/**
 * 技能节点执行器
 * 根据执行方式和执行位置分发到对应的执行器
 */
@Component
@Slf4j
public class SkillNodeExecutor implements NodeExecutor {

    private final SkillService skillService;
    private final SkillExecutorRegistry executorRegistry;
    private final ParameterResolver parameterResolver;

    @Override
    public String getNodeType() {
        return "skill";
    }

    @Override
    public NodeExecutionResult execute(WorkflowNode node,
                                        Map<String, Object> inputs,
                                        ExecutionContext context) {
        log.info("执行技能节点: nodeUuid={}, nodeName={}, skillId={}",
                 node.getUuid(), node.getName(), node.getSkillId());

        // 1. 获取 Skill 定义
        SkillEntity skill = skillService.getById(node.getSkillId());

        if (skill == null) {
            return NodeExecutionResult.failure(
                "Skill 不存在: " + node.getSkillId()
            );
        }

        // 检查 Skill 状态
        if (!"PUBLISHED".equals(skill.getStatus())) {
            return NodeExecutionResult.failure(
                "Skill 未发布或已禁用: " + skill.getName()
            );
        }

        // 2. 参数校验与转换
        Map<String, Object> resolvedInputs = resolveAndValidateInputs(skill, inputs, context);

        // 3. 获取执行配置
        ExecutionType execType = ExecutionType.valueOf(skill.getExecutionType());
        ExecutionLocation execLocation = getExecutionLocation(node, skill);

        log.info("技能执行配置: skill={}, execType={}, execLocation={}",
                 skill.getName(), execType, execLocation);

        // 4. 获取对应执行器
        SkillExecutor executor = executorRegistry.getExecutor(execType, execLocation);

        // 5. 执行
        long startTime = System.currentTimeMillis();
        SkillExecutionResult result;

        try {
            result = executor.execute(skill, resolvedInputs, context);
        } catch (Exception e) {
            log.error("技能执行异常: skill={}, error={}", skill.getName(), e.getMessage(), e);
            return handleExecutionFailure(node, e, context);
        }

        long duration = System.currentTimeMillis() - startTime;

        // 6. 处理结果
        if (!result.isSuccess()) {
            return handleExecutionFailure(node, result, context);
        }

        // 7. 映射输出参数
        Map<String, Object> outputs = mapOutputs(skill, result.getOutputs());

        log.info("技能节点执行完成: nodeUuid={}, skill={}, duration={}ms",
                 node.getUuid(), skill.getName(), duration);

        return NodeExecutionResult.builder()
            .success(true)
            .outputs(outputs)
            .logs(result.getLogs())
            .durationMs(duration)
            .build();
    }

    /**
     * 解析并验证输入参数
     */
    private Map<String, Object> resolveAndValidateInputs(SkillEntity skill,
                                                          Map<String, Object> inputs,
                                                          ExecutionContext context) {
        Map<String, Object> resolved = new HashMap<>();

        // 获取 Skill 输入参数定义
        List<SkillParameter> paramDefs = skill.getInputParameters();

        for (SkillParameter paramDef : paramDefs) {
            String paramName = paramDef.getName();
            Object value = inputs.get(paramName);

            // 必填校验
            if (paramDef.isRequired() && value == null) {
                throw new ParameterValidationException(
                    "必填参数缺失: " + paramName
                );
            }

            // 类型转换
            if (value != null) {
                value = convertType(value, paramDef.getType());
            }

            // 使用默认值
            if (value == null && paramDef.getDefaultValue() != null) {
                value = paramDef.getDefaultValue();
            }

            resolved.put(paramName, value);
        }

        return resolved;
    }

    /**
     * 获取执行位置
     * 优先级：节点配置 > Skill 配置 > 默认值(SERVICE)
     */
    private ExecutionLocation getExecutionLocation(WorkflowNode node, SkillEntity skill) {
        // 节点级别配置优先
        if (node.getExecutionLocation() != null) {
            return ExecutionLocation.valueOf(node.getExecutionLocation());
        }

        // Skill 级别配置
        if (skill.getDefaultExecutionLocation() != null) {
            return ExecutionLocation.valueOf(skill.getDefaultExecutionLocation());
        }

        // 默认在 Service 端执行
        return ExecutionLocation.SERVICE;
    }

    /**
     * 映射输出参数
     */
    private Map<String, Object> mapOutputs(SkillEntity skill,
                                            Map<String, Object> rawOutputs) {
        Map<String, Object> outputs = new HashMap<>();

        List<SkillParameter> outputDefs = skill.getOutputParameters();

        for (SkillParameter outputDef : outputDefs) {
            String paramName = outputDef.getName();
            Object value = rawOutputs.get(paramName);
            outputs.put(paramName, value);
        }

        return outputs;
    }

    /**
     * 处理执行失败
     */
    private NodeExecutionResult handleExecutionFailure(WorkflowNode node,
                                                        Object error,
                                                        ExecutionContext context) {
        String errorMessage;
        String errorStack = null;

        if (error instanceof Exception) {
            errorMessage = ((Exception) error).getMessage();
            StringWriter sw = new StringWriter();
            ((Exception) error).printStackTrace(new PrintWriter(sw));
            errorStack = sw.toString();
        } else if (error instanceof SkillExecutionResult) {
            SkillExecutionResult result = (SkillExecutionResult) error;
            errorMessage = result.getErrorMessage();
            errorStack = result.getErrorStack();
        } else {
            errorMessage = error.toString();
        }

        log.error("技能节点执行失败: nodeUuid={}, error={}",
                  node.getUuid(), errorMessage);

        return NodeExecutionResult.builder()
            .success(false)
            .errorMessage(errorMessage)
            .errorStack(errorStack)
            .build();
    }
}
```

#### 3.8.4 技能执行器接口

```java
/**
 * 技能执行器接口
 */
public interface SkillExecutor {

    /**
     * 获取执行方式
     */
    ExecutionType getExecutionType();

    /**
     * 获取执行位置
     */
    ExecutionLocation getExecutionLocation();

    /**
     * 执行技能
     *
     * @param skill   技能定义
     * @param inputs  输入参数
     * @param context 执行上下文
     * @return 执行结果
     */
    SkillExecutionResult execute(SkillEntity skill,
                                  Map<String, Object> inputs,
                                  ExecutionContext context);
}

/**
 * 执行方式枚举
 */
public enum ExecutionType {
    AI_AGENT,       // AI Agent 执行
    AUTOMATED       // 自动化脚本执行
}

/**
 * 执行位置枚举
 */
public enum ExecutionLocation {
    CLIENT,         // Client 端执行机
    SERVICE         // Service 端执行机
}
```

#### 3.8.5 技能执行结果

```java
/**
 * 技能执行结果
 */
@Data
@Builder
public class SkillExecutionResult {

    /** 是否成功 */
    private boolean success;

    /** 输出参数 */
    @Builder.Default
    private Map<String, Object> outputs = new HashMap<>();

    /** 执行日志 */
    private String logs;

    /** 执行耗时（毫秒） */
    private Long durationMs;

    /** 错误信息 */
    private String errorMessage;

    /** 错误堆栈 */
    private String errorStack;

    /**
     * 创建成功结果
     */
    public static SkillExecutionResult success(Map<String, Object> outputs) {
        return SkillExecutionResult.builder()
            .success(true)
            .outputs(outputs)
            .build();
    }

    /**
     * 创建失败结果
     */
    public static SkillExecutionResult failure(String errorMessage) {
        return SkillExecutionResult.builder()
            .success(false)
            .errorMessage(errorMessage)
            .build();
    }
}
```

#### 3.8.6 技能执行器注册表

```java
/**
 * 技能执行器注册表
 * 管理所有技能执行器的注册和分发
 */
@Component
@Slf4j
public class SkillExecutorRegistry {

    private final Map<String, SkillExecutor> executorMap = new HashMap<>();

    @Autowired
    public SkillExecutorRegistry(List<SkillExecutor> executors) {
        for (SkillExecutor executor : executors) {
            String key = buildKey(executor.getExecutionType(),
                                  executor.getExecutionLocation());
            executorMap.put(key, executor);
            log.info("注册技能执行器: key={}, executor={}",
                     key, executor.getClass().getSimpleName());
        }
    }

    /**
     * 获取执行器
     */
    public SkillExecutor getExecutor(ExecutionType type, ExecutionLocation location) {
        String key = buildKey(type, location);
        SkillExecutor executor = executorMap.get(key);

        if (executor == null) {
            throw new IllegalArgumentException(
                "未找到执行器: " + type + " + " + location
            );
        }

        return executor;
    }

    private String buildKey(ExecutionType type, ExecutionLocation location) {
        return type.name() + "_" + location.name();
    }
}
```

---

### 3.9 执行器配置类定义

```java
/**
 * 循环配置
 */
@Data
public class LoopConfig {
    private LoopType type;
    private Integer times;
    private ArraySource arraySource;
    private Expression condition;
    private Integer maxIterations;
}

public enum LoopType {
    COUNT,      // 计数循环
    ARRAY,      // 数组遍历
    CONDITION   // 条件循环
}

/**
 * 批处理配置
 */
@Data
public class BatchConfig {
    private ArraySource arraySource;
    private int concurrency = 5;
    private String failureStrategy = "CONTINUE_OTHERS"; // STOP_ALL | CONTINUE_OTHERS
    private String outputMode = "MERGE"; // MERGE | SEPARATE
}

/**
 * 异步处理配置
 */
@Data
public class AsyncConfig {
    private String type = "ARRAY"; // ARRAY | SINGLE
    private ArraySource arraySource;
    private int concurrency = 10;
    private String triggerMode = "FIRE_AND_FORGET"; // FIRE_AND_FORGET | CALLBACK
    private Long callbackWorkflowId;
    private long timeout = 3600000; // 默认1小时
}

/**
 * 结果收集配置
 */
@Data
public class CollectConfig {
    private String asyncNodeId;
    private String waitStrategy = "ALL"; // ALL | FIRST | N
    private Integer waitCount;
    private long timeout = 300000; // 默认5分钟
    private String outputMode = "MERGE"; // MERGE | SEPARATE | FIRST_SUCCESS
    private String failureHandling = "PARTIAL_SUCCESS"; // FAIL_FAST | PARTIAL_SUCCESS | IGNORE_FAILURES
}

/**
 * 数组源配置
 */
@Data
public class ArraySource {
    private String type; // reference | literal
    private Object value;
}

/**
 * 条件表达式
 */
@Data
public class Expression {
    private Object leftOperand;
    private String operator;
    private Object rightOperand;
}
```

---

*第三章完成，请确认后继续生成第四章。*
