# 工作流执行框架设计文档

## 第二章：执行引擎核心组件

### 2.1 组件总览

执行引擎的核心组件负责工作流的调度、编排和状态管理。各组件协同工作，确保工作流按照预定义的拓扑结构正确执行。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          执行引擎核心组件关系图                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│     ┌──────────────────────────────────────────────────────────────────┐   │
│     │                      WorkflowScheduler                            │   │
│     │                        (执行调度器)                                │   │
│     │  ┌─────────────────────────────────────────────────────────────┐ │   │
│     │  │ 职责：接收执行请求 → 创建执行记录 → 初始化上下文 → 提交执行   │ │   │
│     │  └─────────────────────────────────────────────────────────────┘ │   │
│     └──────────────────────────┬───────────────────────────────────────┘   │
│                                │                                            │
│                                │ 提交执行任务                                │
│                                ▼                                            │
│     ┌──────────────────────────────────────────────────────────────────┐   │
│     │                       ExecutionEngine                             │   │
│     │                        (执行引擎)                                 │   │
│     │  ┌─────────────────────────────────────────────────────────────┐ │   │
│     │  │ 职责：解析工作流 → 构建执行图 → 拓扑排序 → 调度节点执行      │ │   │
│     │  └─────────────────────────────────────────────────────────────┘ │   │
│     └──────────────────────────┬───────────────────────────────────────┘   │
│                                │                                            │
│          ┌─────────────────────┼─────────────────────┐                     │
│          ▼                     ▼                     ▼                     │
│   ┌─────────────┐       ┌─────────────┐       ┌─────────────┐             │
│   │ExecutionContext│     │StateManager │       │NodeExecutor │             │
│   │  (执行上下文)  │     │ (状态管理器) │     │  Registry   │             │
│   └─────────────┘       └─────────────┘       └─────────────┘             │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 2.2 WorkflowScheduler（执行调度器）

#### 2.2.1 职责定义

WorkflowScheduler 是执行流程的入口点，负责：
- 接收工作流执行请求
- 创建执行记录（workflow_execution）
- 初始化执行上下文
- 提交执行任务到线程池

#### 2.2.2 类设计

```java
/**
 * 工作流执行调度器
 * 负责接收执行请求并调度执行资源
 */
@Service
@Slf4j
public class WorkflowScheduler {

    private final WorkflowRepository workflowRepository;
    private final WorkflowExecutionRepository executionRepository;
    private final WorkflowNodeRepository nodeRepository;
    private final WorkflowConnectionRepository connectionRepository;
    private final ExecutionEngine executionEngine;
    private final StateManager stateManager;
    private final WebSocketManager webSocketManager;

    // 执行线程池
    private final ThreadPoolTaskExecutor workflowExecutor;

    /**
     * 提交工作流执行请求
     *
     * @param workflowId 工作流ID
     * @param inputData  输入参数
     * @param triggeredBy 触发人
     * @param triggerType 触发类型
     * @return 执行UUID
     */
    @Transactional
    public String submitExecution(Long workflowId,
                                   Map<String, Object> inputData,
                                   String triggeredBy,
                                   String triggerType) {

        // 1. 查询工作流定义
        Workflow workflow = workflowRepository.findById(workflowId)
            .orElseThrow(() -> new WorkflowNotFoundException(workflowId));

        // 2. 检查工作流状态
        if (!"PUBLISHED".equals(workflow.getStatus())) {
            throw new WorkflowNotPublishedException(workflowId);
        }

        // 3. 创建执行记录
        String executionUuid = UUID.randomUUID().toString();
        WorkflowExecution execution = WorkflowExecution.builder()
            .workflowId(workflowId)
            .executionUuid(executionUuid)
            .status(ExecutionStatus.PENDING.name())
            .triggerType(triggerType)
            .triggeredBy(triggeredBy)
            .inputData(inputData)
            .progress(0)
            .build();

        executionRepository.save(execution);

        // 4. 初始化节点执行记录
        initializeNodeExecutions(execution.getId(), workflowId);

        // 5. 提交异步执行任务
        workflowExecutor.submit(() -> executeWorkflowAsync(execution, workflow));

        log.info("工作流执行已提交: workflowId={}, executionUuid={}",
                 workflowId, executionUuid);

        return executionUuid;
    }

    /**
     * 异步执行工作流
     */
    private void executeWorkflowAsync(WorkflowExecution execution, Workflow workflow) {
        String executionUuid = execution.getExecutionUuid();

        try {
            // 1. 更新状态为 RUNNING
            updateExecutionStatus(execution, ExecutionStatus.RUNNING);

            // 2. 构建工作流定义
            WorkflowDefinition definition = buildWorkflowDefinition(workflow);

            // 3. 创建执行上下文
            ExecutionContext context = createExecutionContext(execution, definition);

            // 4. 调用执行引擎执行
            executionEngine.execute(definition, context);

            // 5. 更新最终状态
            determineFinalStatus(execution, context);

        } catch (Exception e) {
            log.error("工作流执行异常: executionUuid={}", executionUuid, e);
            handleExecutionError(execution, e);
        }
    }

    /**
     * 构建工作流定义
     */
    private WorkflowDefinition buildWorkflowDefinition(Workflow workflow) {
        // 查询所有节点
        List<WorkflowNode> nodes = nodeRepository.findByWorkflowId(workflow.getId());

        // 查询所有连线
        List<WorkflowConnection> connections =
            connectionRepository.findByWorkflowId(workflow.getId());

        return WorkflowDefinition.builder()
            .workflowId(workflow.getId())
            .workflowName(workflow.getName())
            .nodes(nodes)
            .connections(connections)
            .build();
    }

    /**
     * 创建执行上下文
     */
    private ExecutionContext createExecutionContext(WorkflowExecution execution,
                                                     WorkflowDefinition definition) {
        return ExecutionContext.builder()
            .executionId(execution.getId())
            .executionUuid(execution.getExecutionUuid())
            .workflowId(execution.getWorkflowId())
            .triggeredBy(execution.getTriggeredBy())
            .inputData(execution.getInputData())
            .definition(definition)
            .build();
    }

    /**
     * 更新执行状态
     */
    private void updateExecutionStatus(WorkflowExecution execution,
                                        ExecutionStatus status) {
        execution.setStatus(status.name());

        if (status == ExecutionStatus.RUNNING) {
            execution.setStartTime(LocalDateTime.now());
        } else if (status.isTerminal()) {
            execution.setEndTime(LocalDateTime.now());
            if (execution.getStartTime() != null) {
                long duration = Duration.between(execution.getStartTime(),
                                                  execution.getEndTime()).toMillis();
                execution.setDurationMs(duration);
            }
        }

        executionRepository.save(execution);

        // 推送状态变更
        webSocketManager.pushWorkflowStatus(
            execution.getExecutionUuid(),
            status.name(),
            execution.getProgress()
        );
    }
}
```

#### 2.2.3 执行状态枚举

```java
/**
 * 执行状态枚举
 */
public enum ExecutionStatus {

    PENDING("待执行", false),
    RUNNING("执行中", false),
    SUCCESS("成功", true),
    FAILED("失败", true),
    PARTIAL_SUCCESS("部分成功", true),
    ABORTED("已中止", true),
    TIMEOUT("超时", true);

    private final String description;
    private final boolean terminal;

    ExecutionStatus(String description, boolean terminal) {
        this.description = description;
        this.terminal = terminal;
    }

    public boolean isTerminal() {
        return terminal;
    }

    public String getDescription() {
        return description;
    }
}
```

#### 2.2.4 线程池配置

```java
/**
 * 执行线程池配置
 */
@Configuration
public class ExecutorConfig {

    @Bean("workflowExecutor")
    public ThreadPoolTaskExecutor workflowExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

        // 核心线程数
        executor.setCorePoolSize(10);
        // 最大线程数
        executor.setMaxPoolSize(50);
        // 队列容量
        executor.setQueueCapacity(500);
        // 线程名前缀
        executor.setThreadNamePrefix("workflow-exec-");
        // 拒绝策略：由调用线程执行
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // 线程空闲时间
        executor.setKeepAliveSeconds(60);
        // 允许核心线程超时
        executor.setAllowCoreThreadTimeOut(true);

        executor.initialize();
        return executor;
    }
}
```

---

### 2.3 ExecutionEngine（执行引擎）

#### 2.3.1 职责定义

ExecutionEngine 是执行流程的核心编排器，负责：
- 解析工作流定义
- 构建执行图（DAG）
- 按拓扑顺序调度节点执行
- 管理并行执行
- 处理执行异常

#### 2.3.2 执行图（ExecutionGraph）

```java
/**
 * 执行图 - 有向无环图（DAG）表示
 */
@Data
public class ExecutionGraph {

    // 所有节点
    private Map<String, WorkflowNode> nodes;

    // 节点入度（前置节点数量）
    private Map<String, Integer> inDegree;

    // 节点出度（后续节点）
    private Map<String, List<String>> outDegree;

    // 起始节点（入度为0）
    private List<String> startNodes;

    // 结束节点（出度为0）
    private List<String> endNodes;

    /**
     * 获取节点的所有前置节点
     */
    public List<String> getPredecessors(String nodeUuid) {
        return nodes.values().stream()
            .filter(node -> hasConnectionTo(node.getUuid(), nodeUuid))
            .map(WorkflowNode::getUuid)
            .collect(Collectors.toList());
    }

    /**
     * 获取节点的所有后续节点
     */
    public List<String> getSuccessors(String nodeUuid) {
        return outDegree.getOrDefault(nodeUuid, Collections.emptyList());
    }

    /**
     * 判断是否存在从source到target的连线
     */
    private boolean hasConnectionTo(String sourceUuid, String targetUuid) {
        return outDegree.getOrDefault(sourceUuid, Collections.emptyList())
            .contains(targetUuid);
    }
}
```

#### 2.3.3 执行引擎实现

```java
/**
 * 工作流执行引擎
 * 负责流程编排和节点调度
 */
@Service
@Slf4j
public class ExecutionEngine {

    private final NodeExecutorRegistry executorRegistry;
    private final StateManager stateManager;
    private final WebSocketManager webSocketManager;
    private final ParameterResolver parameterResolver;

    /**
     * 执行工作流
     *
     * @param definition 工作流定义
     * @param context    执行上下文
     */
    public void execute(WorkflowDefinition definition, ExecutionContext context) {
        log.info("开始执行工作流: workflowId={}, executionUuid={}",
                 definition.getWorkflowId(), context.getExecutionUuid());

        try {
            // 1. 构建执行图
            ExecutionGraph graph = buildExecutionGraph(definition);
            context.setExecutionGraph(graph);

            // 2. 获取起始节点
            List<String> startNodes = graph.getStartNodes();

            if (startNodes.isEmpty()) {
                throw new ExecutionException("工作流没有起始节点");
            }

            // 3. 执行节点
            executeFromNodes(graph, startNodes, context);

            // 4. 检查是否所有节点都执行完成
            verifyAllNodesCompleted(context);

        } catch (Exception e) {
            log.error("工作流执行异常: executionUuid={}", context.getExecutionUuid(), e);
            throw new ExecutionException("工作流执行失败", e);
        }
    }

    /**
     * 构建执行图
     */
    private ExecutionGraph buildExecutionGraph(WorkflowDefinition definition) {
        ExecutionGraph graph = new ExecutionGraph();
        graph.setNodes(new HashMap<>());
        graph.setInDegree(new HashMap<>());
        graph.setOutDegree(new HashMap<>());

        // 初始化所有节点
        for (WorkflowNode node : definition.getNodes()) {
            String nodeUuid = node.getUuid();
            graph.getNodes().put(nodeUuid, node);
            graph.getInDegree().put(nodeUuid, 0);
            graph.getOutDegree().put(nodeUuid, new ArrayList<>());
        }

        // 处理连线，构建入度和出度
        for (WorkflowConnection conn : definition.getConnections()) {
            String sourceUuid = conn.getSourceNodeUuid();
            String targetUuid = conn.getTargetNodeUuid();

            // 增加目标节点入度
            graph.getInDegree().merge(targetUuid, 1, Integer::sum);

            // 增加源节点出度
            graph.getOutDegree().get(sourceUuid).add(targetUuid);
        }

        // 找出起始节点（入度为0）
        List<String> startNodes = graph.getInDegree().entrySet().stream()
            .filter(e -> e.getValue() == 0)
            .map(Map.Entry::getKey)
            .collect(Collectors.toList());
        graph.setStartNodes(startNodes);

        // 找出结束节点（出度为0）
        List<String> endNodes = graph.getOutDegree().entrySet().stream()
            .filter(e -> e.getValue().isEmpty())
            .map(Map.Entry::getKey)
            .collect(Collectors.toList());
        graph.setEndNodes(endNodes);

        // 验证DAG（检测环）
        validateDAG(graph);

        return graph;
    }

    /**
     * 验证DAG - 检测是否存在环
     */
    private void validateDAG(ExecutionGraph graph) {
        // 使用Kahn算法检测环
        Map<String, Integer> inDegreeCopy = new HashMap<>(graph.getInDegree());
        Queue<String> queue = new LinkedList<>();
        int visitedCount = 0;

        // 将入度为0的节点加入队列
        inDegreeCopy.entrySet().stream()
            .filter(e -> e.getValue() == 0)
            .forEach(e -> queue.offer(e.getKey()));

        while (!queue.isEmpty()) {
            String nodeUuid = queue.poll();
            visitedCount++;

            for (String successor : graph.getSuccessors(nodeUuid)) {
                int newInDegree = inDegreeCopy.get(successor) - 1;
                inDegreeCopy.put(successor, newInDegree);
                if (newInDegree == 0) {
                    queue.offer(successor);
                }
            }
        }

        // 如果访问的节点数不等于总节点数，说明存在环
        if (visitedCount != graph.getNodes().size()) {
            throw new CyclicDependencyException("工作流存在循环依赖");
        }
    }

    /**
     * 从指定节点开始执行
     */
    private void executeFromNodes(ExecutionGraph graph,
                                   List<String> nodeUuids,
                                   ExecutionContext context) {

        // 待执行节点队列
        Queue<String> pendingNodes = new LinkedList<>(nodeUuids);

        // 正在执行的节点计数器
        AtomicInteger runningCount = new AtomicInteger(0);

        // 已完成节点集合
        Set<String> completedNodes = ConcurrentHashMap.newKeySet();

        // 执行锁
        Object lock = new Object();

        while (!pendingNodes.isEmpty() || runningCount.get() > 0) {

            // 获取可执行节点（所有前置节点已完成）
            List<String> executableNodes = new ArrayList<>();
            Iterator<String> iterator = pendingNodes.iterator();

            while (iterator.hasNext()) {
                String nodeUuid = iterator.next();
                List<String> predecessors = graph.getPredecessors(nodeUuid);

                // 检查所有前置节点是否完成
                boolean allPredecessorsCompleted = predecessors.isEmpty() ||
                    completedNodes.containsAll(predecessors);

                if (allPredecessorsCompleted) {
                    executableNodes.add(nodeUuid);
                    iterator.remove();
                }
            }

            if (executableNodes.isEmpty() && runningCount.get() == 0 && !pendingNodes.isEmpty()) {
                // 没有可执行节点，也没有正在执行的节点，但还有待执行节点
                // 说明存在死锁或环
                throw new ExecutionException("执行死锁：无法继续执行剩余节点");
            }

            // 并行执行可执行节点
            List<CompletableFuture<Void>> futures = new ArrayList<>();

            for (String nodeUuid : executableNodes) {
                runningCount.incrementAndGet();

                CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
                    try {
                        executeNode(graph.getNodes().get(nodeUuid), context);
                        completedNodes.add(nodeUuid);

                        // 将后续节点加入待执行队列
                        List<String> successors = graph.getSuccessors(nodeUuid);
                        synchronized (lock) {
                            pendingNodes.addAll(successors);
                        }

                    } catch (Exception e) {
                        handleNodeExecutionError(nodeUuid, context, e);
                    } finally {
                        runningCount.decrementAndGet();
                    }
                });

                futures.add(future);
            }

            // 等待当前批次完成
            if (!futures.isEmpty()) {
                CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).join();
            }

            // 短暂等待，避免忙等待
            if (pendingNodes.isEmpty() && runningCount.get() > 0) {
                try {
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }
    }

    /**
     * 执行单个节点
     */
    private void executeNode(WorkflowNode node, ExecutionContext context) {
        String nodeUuid = node.getUuid();
        String nodeName = node.getName();

        log.info("开始执行节点: nodeUuid={}, nodeName={}", nodeUuid, nodeName);

        // 1. 更新节点状态为 RUNNING
        stateManager.updateNodeStatus(context.getExecutionId(), nodeUuid,
                                       NodeExecutionStatus.RUNNING);
        webSocketManager.pushNodeStatus(context.getExecutionUuid(), nodeUuid,
                                         nodeName, NodeExecutionStatus.RUNNING);

        try {
            // 2. 解析输入参数
            Map<String, Object> inputs = parameterResolver.resolveInputs(node, context);

            // 3. 获取节点执行器
            NodeExecutor executor = executorRegistry.getExecutor(node.getType());

            // 4. 执行节点
            NodeExecutionResult result = executor.execute(node, inputs, context);

            // 5. 处理执行结果
            if (result.isSuccess()) {
                // 存储节点输出
                context.setNodeOutputs(nodeUuid, result.getOutputs());

                // 更新节点状态为 SUCCESS
                stateManager.updateNodeStatus(context.getExecutionId(), nodeUuid,
                                               NodeExecutionStatus.SUCCESS, result);
                webSocketManager.pushNodeStatus(context.getExecutionUuid(), nodeUuid,
                                                 nodeName, NodeExecutionStatus.SUCCESS);

                log.info("节点执行成功: nodeUuid={}, nodeName={}", nodeUuid, nodeName);
            } else {
                // 处理执行失败
                handleNodeFailure(node, context, result);
            }

        } catch (Exception e) {
            log.error("节点执行异常: nodeUuid={}, nodeName={}", nodeUuid, nodeName, e);
            handleNodeExecutionError(nodeUuid, context, e);
        }
    }

    /**
     * 验证所有节点是否执行完成
     */
    private void verifyAllNodesCompleted(ExecutionContext context) {
        ExecutionGraph graph = context.getExecutionGraph();

        for (String nodeUuid : graph.getNodes().keySet()) {
            NodeExecutionStatus status = stateManager.getNodeStatus(
                context.getExecutionId(), nodeUuid);

            if (status == null || status == NodeExecutionStatus.PENDING) {
                log.warn("节点未执行: nodeUuid={}", nodeUuid);
            }
        }
    }
}
```

---

### 2.4 ExecutionContext（执行上下文）

#### 2.4.1 职责定义

ExecutionContext 是执行过程中的状态容器，负责：
- 存储执行基本信息
- 管理变量池（节点输出）
- 计算和维护前置节点关系
- 追踪异步任务

#### 2.4.2 执行上下文实现

```java
/**
 * 执行上下文 - 管理工作流执行过程中的所有状态和数据
 */
@Data
@Builder
public class ExecutionContext {

    // ==================== 基本信息 ====================

    /** 执行记录ID */
    private Long executionId;

    /** 执行UUID */
    private String executionUuid;

    /** 工作流ID */
    private Long workflowId;

    /** 触发人 */
    private String triggeredBy;

    /** 输入参数 */
    private Map<String, Object> inputData;

    /** 工作流定义 */
    private WorkflowDefinition definition;

    /** 执行图 */
    private ExecutionGraph executionGraph;

    // ==================== 变量存储 ====================

    /**
     * 节点输出缓存
     * key: nodeUuid
     * value: {paramName: value}
     */
    @Builder.Default
    private Map<String, Map<String, Object>> nodeOutputs = new ConcurrentHashMap<>();

    /**
     * 全局变量池
     * key: 变量名
     * value: 变量值
     */
    @Builder.Default
    private Map<String, Object> globalVariables = new ConcurrentHashMap<>();

    // ==================== 执行状态 ====================

    /** 各节点执行状态 */
    @Builder.Default
    private Map<String, NodeExecutionStatus> nodeStates = new ConcurrentHashMap<>();

    /** 当前执行节点UUID */
    private String currentNodeUuid;

    // ==================== 前置节点计算缓存 ====================

    /** 节点前置节点缓存 */
    @Builder.Default
    private Map<String, Set<String>> predecessorCache = new ConcurrentHashMap<>();

    // ==================== 异步任务追踪 ====================

    /**
     * 异步任务信息
     * key: async节点UUID
     * value: 子任务列表
     */
    @Builder.Default
    private Map<String, List<AsyncTaskInfo>> asyncTasks = new ConcurrentHashMap<>();

    // ==================== 循环上下文 ====================

    /** 当前循环上下文栈 */
    @Builder.Default
    private Deque<LoopContext> loopContextStack = new ConcurrentLinkedDeque<>();

    // ==================== 工具方法 ====================

    /**
     * 获取节点输出参数
     *
     * @param nodeUuid  节点UUID
     * @param paramName 参数名
     * @return 参数值
     */
    public Object getNodeOutput(String nodeUuid, String paramName) {
        Map<String, Object> outputs = nodeOutputs.get(nodeUuid);
        return outputs != null ? outputs.get(paramName) : null;
    }

    /**
     * 获取节点所有输出
     *
     * @param nodeUuid 节点UUID
     * @return 输出Map
     */
    public Map<String, Object> getNodeOutputs(String nodeUuid) {
        return nodeOutputs.getOrDefault(nodeUuid, Collections.emptyMap());
    }

    /**
     * 设置节点输出
     *
     * @param nodeUuid 节点UUID
     * @param outputs  输出Map
     */
    public void setNodeOutputs(String nodeUuid, Map<String, Object> outputs) {
        nodeOutputs.put(nodeUuid, new ConcurrentHashMap<>(outputs));
    }

    /**
     * 获取全局变量
     *
     * @param varName 变量名
     * @return 变量值
     */
    public Object getGlobalVariable(String varName) {
        return globalVariables.get(varName);
    }

    /**
     * 设置全局变量
     *
     * @param varName  变量名
     * @param value    变量值
     */
    public void setGlobalVariable(String varName, Object value) {
        globalVariables.put(varName, value);
    }

    /**
     * 获取节点的前置节点集合
     *
     * @param nodeUuid 节点UUID
     * @return 前置节点UUID集合
     */
    public Set<String> getPredecessors(String nodeUuid) {
        return predecessorCache.computeIfAbsent(nodeUuid,
            uuid -> calculatePredecessors(uuid));
    }

    /**
     * 计算节点的前置节点（递归计算）
     */
    private Set<String> calculatePredecessors(String nodeUuid) {
        Set<String> predecessors = new HashSet<>();
        calculatePredecessorsRecursive(nodeUuid, predecessors);
        return predecessors;
    }

    private void calculatePredecessorsRecursive(String nodeUuid, Set<String> visited) {
        List<String> directPredecessors = executionGraph.getPredecessors(nodeUuid);

        for (String predUuid : directPredecessors) {
            if (!visited.contains(predUuid)) {
                visited.add(predUuid);
                calculatePredecessorsRecursive(predUuid, visited);
            }
        }
    }

    /**
     * 判断当前节点是否可以引用目标节点的输出
     * 规则：只能引用前置节点的输出
     *
     * @param currentNodeUuid 当前节点UUID
     * @param targetNodeUuid  目标节点UUID
     * @return 是否可引用
     */
    public boolean canReferenceNode(String currentNodeUuid, String targetNodeUuid) {
        return getPredecessors(currentNodeUuid).contains(targetNodeUuid);
    }

    /**
     * 获取当前可引用的所有节点UUID
     *
     * @param currentNodeUuid 当前节点UUID
     * @return 可引用的节点UUID列表
     */
    public List<String> getReferenceableNodes(String currentNodeUuid) {
        return new ArrayList<>(getPredecessors(currentNodeUuid));
    }

    // ==================== 异步任务管理 ====================

    /**
     * 注册异步任务
     *
     * @param asyncNodeUuid 异步节点UUID
     * @param tasks         任务列表
     */
    public void registerAsyncTasks(String asyncNodeUuid, List<AsyncTaskInfo> tasks) {
        asyncTasks.put(asyncNodeUuid, new CopyOnWriteArrayList<>(tasks));
    }

    /**
     * 获取异步任务列表
     *
     * @param asyncNodeUuid 异步节点UUID
     * @return 任务列表
     */
    public List<AsyncTaskInfo> getAsyncTasks(String asyncNodeUuid) {
        return asyncTasks.getOrDefault(asyncNodeUuid, Collections.emptyList());
    }

    /**
     * 更新异步任务状态
     *
     * @param asyncNodeUuid 异步节点UUID
     * @param taskId        任务ID
     * @param status        新状态
     * @param output        输出结果
     */
    public void updateAsyncTaskStatus(String asyncNodeUuid,
                                       String taskId,
                                       AsyncTaskStatus status,
                                       Map<String, Object> output) {
        List<AsyncTaskInfo> tasks = asyncTasks.get(asyncNodeUuid);
        if (tasks != null) {
            tasks.stream()
                .filter(t -> t.getTaskId().equals(taskId))
                .findFirst()
                .ifPresent(task -> {
                    task.setStatus(status);
                    task.setOutput(output);
                });
        }
    }

    // ==================== 循环上下文管理 ====================

    /**
     * 进入循环
     *
     * @param loopNodeUuid 循环节点UUID
     * @param currentItem  当前元素
     * @param currentIndex 当前索引
     */
    public void enterLoop(String loopNodeUuid, Object currentItem, int currentIndex) {
        LoopContext loopContext = new LoopContext(loopNodeUuid, currentItem, currentIndex);
        loopContextStack.push(loopContext);
    }

    /**
     * 退出循环
     */
    public void exitLoop() {
        if (!loopContextStack.isEmpty()) {
            loopContextStack.pop();
        }
    }

    /**
     * 获取当前循环上下文
     *
     * @return 循环上下文，如果不在循环中则返回null
     */
    public LoopContext getCurrentLoopContext() {
        return loopContextStack.peek();
    }

    /**
     * 获取当前循环元素
     */
    public Object getCurrentLoopItem() {
        LoopContext context = getCurrentLoopContext();
        return context != null ? context.getCurrentItem() : null;
    }

    /**
     * 获取当前循环索引
     */
    public Integer getCurrentLoopIndex() {
        LoopContext context = getCurrentLoopContext();
        return context != null ? context.getCurrentIndex() : null;
    }

    /**
     * 循环上下文内部类
     */
    @Data
    @AllArgsConstructor
    public static class LoopContext {
        private String loopNodeUuid;
        private Object currentItem;
        private int currentIndex;
    }
}
```

#### 2.4.3 异步任务信息

```java
/**
 * 异步任务信息
 */
@Data
@Builder
public class AsyncTaskInfo {

    /** 任务ID */
    private String taskId;

    /** 关联的执行记录ID */
    private Long executionId;

    /** 异步节点UUID */
    private String asyncNodeUuid;

    /** 循环索引 */
    private int loopIndex;

    /** 输入数据 */
    private Map<String, Object> input;

    /** 任务状态 */
    @Builder.Default
    private AsyncTaskStatus status = AsyncTaskStatus.PENDING;

    /** 输出结果 */
    private Map<String, Object> output;

    /** 开始时间 */
    private LocalDateTime startTime;

    /** 结束时间 */
    private LocalDateTime endTime;

    /** 错误信息 */
    private String errorMessage;

    /** 执行的Client ID */
    private String clientId;
}

/**
 * 异步任务状态
 */
public enum AsyncTaskStatus {
    PENDING,        // 待执行
    RUNNING,        // 执行中
    SUCCESS,        // 成功
    FAILED,         // 失败
    TIMEOUT,        // 超时
    CANCELLED       // 已取消
}
```

---

### 2.5 StateManager（状态管理器）

#### 2.5.1 职责定义

StateManager 负责执行状态的持久化和管理：
- 更新工作流执行状态
- 更新节点执行状态
- 记录执行日志
- 查询执行历史

#### 2.5.2 状态管理器实现

```java
/**
 * 状态管理器
 * 负责执行状态的持久化和管理
 */
@Service
@Slf4j
public class StateManager {

    private final WorkflowExecutionRepository executionRepository;
    private final WorkflowNodeExecutionRepository nodeExecutionRepository;
    private final ExecutionLogRepository logRepository;

    /**
     * 更新工作流执行状态
     */
    @Transactional
    public void updateWorkflowStatus(Long executionId,
                                      ExecutionStatus status,
                                      Integer progress,
                                      Map<String, Object> outputData) {

        WorkflowExecution execution = executionRepository.findById(executionId)
            .orElseThrow(() -> new ExecutionNotFoundException(executionId));

        execution.setStatus(status.name());
        execution.setProgress(progress);

        if (outputData != null) {
            execution.setOutputData(outputData);
        }

        if (status == ExecutionStatus.RUNNING && execution.getStartTime() == null) {
            execution.setStartTime(LocalDateTime.now());
        }

        if (status.isTerminal()) {
            execution.setEndTime(LocalDateTime.now());
            if (execution.getStartTime() != null) {
                execution.setDurationMs(
                    Duration.between(execution.getStartTime(),
                                     execution.getEndTime()).toMillis()
                );
            }
        }

        executionRepository.save(execution);
    }

    /**
     * 更新节点执行状态
     */
    @Transactional
    public void updateNodeStatus(Long executionId,
                                  String nodeUuid,
                                  NodeExecutionStatus status) {
        updateNodeStatus(executionId, nodeUuid, status, null);
    }

    /**
     * 更新节点执行状态（带结果）
     */
    @Transactional
    public void updateNodeStatus(Long executionId,
                                  String nodeUuid,
                                  NodeExecutionStatus status,
                                  NodeExecutionResult result) {

        WorkflowNodeExecution nodeExecution = nodeExecutionRepository
            .findByExecutionIdAndNodeUuid(executionId, nodeUuid)
            .orElseThrow(() -> new NodeExecutionNotFoundException(executionId, nodeUuid));

        nodeExecution.setStatus(status.name());

        if (status == NodeExecutionStatus.RUNNING && nodeExecution.getStartTime() == null) {
            nodeExecution.setStartTime(LocalDateTime.now());
        }

        if (status.isTerminal()) {
            nodeExecution.setEndTime(LocalDateTime.now());
            if (nodeExecution.getStartTime() != null) {
                nodeExecution.setDurationMs(
                    Duration.between(nodeExecution.getStartTime(),
                                     nodeExecution.getEndTime()).toMillis()
                );
            }
        }

        if (result != null) {
            nodeExecution.setOutputData(result.getOutputs());
            if (result.getErrorMessage() != null) {
                nodeExecution.setErrorMessage(result.getErrorMessage());
            }
        }

        nodeExecutionRepository.save(nodeExecution);

        // 更新工作流进度
        updateWorkflowProgress(executionId);
    }

    /**
     * 更新工作流执行进度
     */
    private void updateWorkflowProgress(Long executionId) {
        // 统计所有节点执行状态
        List<WorkflowNodeExecution> nodeExecutions =
            nodeExecutionRepository.findByExecutionId(executionId);

        int total = nodeExecutions.size();
        long completed = nodeExecutions.stream()
            .filter(ne -> NodeExecutionStatus.valueOf(ne.getStatus()).isTerminal())
            .count();

        int progress = (int) ((completed * 100) / total);

        executionRepository.updateProgress(executionId, progress);
    }

    /**
     * 记录执行日志
     */
    public void logExecution(Long executionId,
                             String nodeUuid,
                             String logLevel,
                             String message) {

        ExecutionLog log = ExecutionLog.builder()
            .executionId(executionId)
            .nodeUuid(nodeUuid)
            .logLevel(logLevel)
            .message(message)
            .timestamp(LocalDateTime.now())
            .build();

        logRepository.save(log);
    }

    /**
     * 获取节点执行状态
     */
    public NodeExecutionStatus getNodeStatus(Long executionId, String nodeUuid) {
        return nodeExecutionRepository
            .findByExecutionIdAndNodeUuid(executionId, nodeUuid)
            .map(ne -> NodeExecutionStatus.valueOf(ne.getStatus()))
            .orElse(null);
    }

    /**
     * 获取所有节点执行状态
     */
    public Map<String, NodeExecutionStatus> getAllNodeStatuses(Long executionId) {
        List<WorkflowNodeExecution> executions =
            nodeExecutionRepository.findByExecutionId(executionId);

        return executions.stream()
            .collect(Collectors.toMap(
                WorkflowNodeExecution::getNodeUuid,
                ne -> NodeExecutionStatus.valueOf(ne.getStatus())
            ));
    }
}
```

#### 2.5.3 节点执行状态枚举

```java
/**
 * 节点执行状态枚举
 */
public enum NodeExecutionStatus {

    PENDING("待执行", false),
    RUNNING("执行中", false),
    SUCCESS("成功", true),
    FAILED("失败", true),
    SKIPPED("已跳过", true),
    TIMEOUT("超时", true);

    private final String description;
    private final boolean terminal;

    NodeExecutionStatus(String description, boolean terminal) {
        this.description = description;
        this.terminal = terminal;
    }

    public boolean isTerminal() {
        return terminal;
    }

    public String getDescription() {
        return description;
    }
}
```

---

### 2.6 NodeExecutorRegistry（节点执行器注册表）

#### 2.6.1 职责定义

NodeExecutorRegistry 负责：
- 注册和管理所有节点执行器
- 根据节点类型分发到对应执行器
- 支持执行器的动态扩展

#### 2.6.2 执行器注册表实现

```java
/**
 * 节点执行器注册表
 * 管理所有节点执行器的注册和分发
 */
@Component
@Slf4j
public class NodeExecutorRegistry {

    /**
     * 执行器映射表
     * key: 节点类型
     * value: 执行器实例
     */
    private final Map<String, NodeExecutor> executorMap = new ConcurrentHashMap<>();

    /**
     * 构造函数 - 自动注入所有 NodeExecutor 实现
     */
    @Autowired
    public NodeExecutorRegistry(List<NodeExecutor> executors) {
        for (NodeExecutor executor : executors) {
            register(executor);
        }
        log.info("已注册 {} 个节点执行器", executorMap.size());
    }

    /**
     * 注册执行器
     */
    public void register(NodeExecutor executor) {
        String nodeType = executor.getNodeType();

        if (executorMap.containsKey(nodeType)) {
            log.warn("覆盖已存在的执行器: nodeType={}", nodeType);
        }

        executorMap.put(nodeType, executor);
        log.info("注册节点执行器: nodeType={}, executor={}",
                 nodeType, executor.getClass().getSimpleName());
    }

    /**
     * 获取执行器
     *
     * @param nodeType 节点类型
     * @return 执行器实例
     * @throws ExecutorNotFoundException 执行器未找到异常
     */
    public NodeExecutor getExecutor(String nodeType) {
        NodeExecutor executor = executorMap.get(nodeType);

        if (executor == null) {
            throw new ExecutorNotFoundException(
                "未找到节点类型对应的执行器: " + nodeType);
        }

        return executor;
    }

    /**
     * 检查执行器是否存在
     */
    public boolean hasExecutor(String nodeType) {
        return executorMap.containsKey(nodeType);
    }

    /**
     * 获取所有已注册的节点类型
     */
    public Set<String> getRegisteredNodeTypes() {
        return Collections.unmodifiableSet(executorMap.keySet());
    }
}
```

#### 2.6.3 节点执行器接口

```java
/**
 * 节点执行器接口
 * 所有节点执行器必须实现此接口
 */
public interface NodeExecutor {

    /**
     * 获取支持的节点类型
     *
     * @return 节点类型标识
     */
    String getNodeType();

    /**
     * 执行节点
     *
     * @param node    节点定义
     * @param inputs  解析后的输入参数
     * @param context 执行上下文
     * @return 执行结果
     */
    NodeExecutionResult execute(WorkflowNode node,
                                 Map<String, Object> inputs,
                                 ExecutionContext context);

    /**
     * 验证节点配置（可选实现）
     *
     * @param node 节点定义
     * @return 验证结果
     */
    default ValidationResult validate(WorkflowNode node) {
        return ValidationResult.success();
    }
}
```

#### 2.6.4 执行结果类

```java
/**
 * 节点执行结果
 */
@Data
@Builder
public class NodeExecutionResult {

    /** 是否成功 */
    private boolean success;

    /** 输出参数 */
    @Builder.Default
    private Map<String, Object> outputs = new HashMap<>();

    /** 错误信息 */
    private String errorMessage;

    /** 错误堆栈 */
    private String errorStack;

    /** 执行日志 */
    private String logs;

    /** 执行耗时（毫秒） */
    private Long durationMs;

    /** 跳过原因（当节点被跳过时） */
    private String skipReason;

    /** 下一步要执行的节点（用于条件分支） */
    private List<String> nextNodes;

    /**
     * 创建成功结果
     */
    public static NodeExecutionResult success(Map<String, Object> outputs) {
        return NodeExecutionResult.builder()
            .success(true)
            .outputs(outputs)
            .build();
    }

    /**
     * 创建成功结果（无输出）
     */
    public static NodeExecutionResult success() {
        return NodeExecutionResult.builder()
            .success(true)
            .build();
    }

    /**
     * 创建失败结果
     */
    public static NodeExecutionResult failure(String errorMessage) {
        return NodeExecutionResult.builder()
            .success(false)
            .errorMessage(errorMessage)
            .build();
    }

    /**
     * 创建失败结果（带异常）
     */
    public static NodeExecutionResult failure(String errorMessage, Exception e) {
        StringWriter sw = new StringWriter();
        e.printStackTrace(new PrintWriter(sw));

        return NodeExecutionResult.builder()
            .success(false)
            .errorMessage(errorMessage)
            .errorStack(sw.toString())
            .build();
    }

    /**
     * 创建跳过结果
     */
    public static NodeExecutionResult skipped(String reason) {
        return NodeExecutionResult.builder()
            .success(true)
            .skipReason(reason)
            .build();
    }
}
```

---

### 2.7 参数解析器

#### 2.7.1 职责定义

ParameterResolver 负责：
- 解析节点输入参数配置
- 从执行上下文中获取引用值
- 支持表达式求值

#### 2.7.2 参数解析器实现

```java
/**
 * 参数解析器
 * 解析节点输入参数配置，从上下文中获取值
 */
@Service
@Slf4j
public class ParameterResolver {

    /**
     * 解析节点输入参数
     *
     * @param node    节点定义
     * @param context 执行上下文
     * @return 解析后的参数Map
     */
    public Map<String, Object> resolveInputs(WorkflowNode node,
                                              ExecutionContext context) {
        Map<String, Object> inputs = new HashMap<>();

        InputConfig inputConfig = node.getInputConfig();
        if (inputConfig == null || inputConfig.getParameters() == null) {
            return inputs;
        }

        for (ParameterConfig param : inputConfig.getParameters()) {
            Object value = resolveParameterValue(param, node.getUuid(), context);
            inputs.put(param.getName(), value);
        }

        return inputs;
    }

    /**
     * 解析单个参数值
     */
    private Object resolveParameterValue(ParameterConfig param,
                                          String currentNodeUuid,
                                          ExecutionContext context) {
        ValueSource valueSource = param.getValueSource();

        if (valueSource == null) {
            // 使用默认值
            return param.getDefaultValue();
        }

        if ("literal".equals(valueSource.getType())) {
            // 字面值
            return convertValue(valueSource.getValue(), param.getType());
        }

        if ("reference".equals(valueSource.getType())) {
            // 引用其他节点输出
            return resolveReference(valueSource.getValue(), currentNodeUuid, context);
        }

        if ("expression".equals(valueSource.getType())) {
            // 表达式求值
            return evaluateExpression(valueSource.getValue(), context);
        }

        return null;
    }

    /**
     * 解析引用表达式
     * 格式: ${节点名称.输出参数名} 或 ${节点UUID.输出参数名}
     */
    private Object resolveReference(String reference,
                                     String currentNodeUuid,
                                     ExecutionContext context) {
        // 解析引用格式: ${xxx.yyy}
        String refContent = reference;
        if (reference.startsWith("${") && reference.endsWith("}")) {
            refContent = reference.substring(2, reference.length() - 1);
        }

        // 分割节点和参数
        String[] parts = refContent.split("\\.", 2);
        if (parts.length != 2) {
            throw new ParameterResolveException("无效的引用格式: " + reference);
        }

        String nodeIdentifier = parts[0];
        String paramName = parts[1];

        // 查找节点UUID（可能是节点名称或UUID）
        String targetNodeUuid = resolveNodeIdentifier(nodeIdentifier, context);

        // 验证是否可引用（只能引用前置节点）
        if (!context.canReferenceNode(currentNodeUuid, targetNodeUuid)) {
            throw new ParameterResolveException(
                "无法引用非前置节点的输出: " + nodeIdentifier +
                "，当前节点: " + currentNodeUuid);
        }

        // 获取节点输出
        Object value = context.getNodeOutput(targetNodeUuid, paramName);

        if (value == null) {
            log.warn("引用的参数值为空: node={}, param={}", targetNodeUuid, paramName);
        }

        return value;
    }

    /**
     * 解析节点标识符（名称或UUID）
     */
    private String resolveNodeIdentifier(String identifier, ExecutionContext context) {
        // 首先尝试直接匹配UUID
        if (context.getExecutionGraph().getNodes().containsKey(identifier)) {
            return identifier;
        }

        // 尝试通过节点名称查找
        for (WorkflowNode node : context.getDefinition().getNodes()) {
            if (node.getName().equals(identifier)) {
                return node.getUuid();
            }
        }

        throw new ParameterResolveException("未找到节点: " + identifier);
    }

    /**
     * 类型转换
     */
    private Object convertValue(Object value, String targetType) {
        if (value == null) {
            return null;
        }

        if (targetType == null) {
            return value;
        }

        String strValue = value.toString();

        switch (targetType.toLowerCase()) {
            case "string":
                return strValue;
            case "integer":
            case "int":
                return Integer.parseInt(strValue);
            case "long":
                return Long.parseLong(strValue);
            case "double":
            case "number":
                return Double.parseDouble(strValue);
            case "boolean":
            case "bool":
                return Boolean.parseBoolean(strValue);
            default:
                return value;
        }
    }

    /**
     * 表达式求值
     */
    private Object evaluateExpression(String expression, ExecutionContext context) {
        // 实现表达式求值逻辑
        // 支持: 算术运算、条件表达式、字符串操作等
        // 可以使用 Spring Expression Language (SpEL) 或其他表达式引擎

        // TODO: 实现表达式引擎
        return expression;
    }
}
```

---

### 2.8 组件交互流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          执行引擎组件交互流程                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 前端发起执行请求                                                         │
│     │                                                                       │
│     ▼                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  WorkflowScheduler.submitExecution()                                  │  │
│  │  ├── 查询工作流定义                                                   │  │
│  │  ├── 创建执行记录 (workflow_execution)                                │  │
│  │  ├── 初始化节点执行记录 (workflow_node_execution)                     │  │
│  │  └── 提交到线程池执行                                                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│     │                                                                       │
│     ▼                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  ExecutionEngine.execute()                                            │  │
│  │  ├── buildExecutionGraph() - 构建DAG执行图                            │  │
│  │  ├── validateDAG() - 检测循环依赖                                     │  │
│  │  └── executeFromNodes() - 按拓扑顺序执行                              │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│     │                                                                       │
│     │  ┌─────────────────────────────────────────────────────────────┐   │
│     │  │  对每个可执行节点:                                            │   │
│     │  │                                                              │   │
│     │  │  ParameterResolver.resolveInputs()  ──┐                     │   │
│     │  │      │                                │                     │   │
│     │  │      ├── 解析字面值                    │                     │   │
│     │  │      ├── 解析引用 (从ExecutionContext)│                     │   │
│     │  │      └── 解析表达式                    │                     │   │
│     │  │                                        │                     │   │
│     │  │  NodeExecutorRegistry.getExecutor()  ◄─┘                    │   │
│     │  │      │                                                      │   │
│     │  │      ▼                                                      │   │
│     │  │  NodeExecutor.execute()                                     │   │
│     │  │      │                                                      │   │
│     │  │      ├── StateManager.updateNodeStatus()                    │   │
│     │  │      └── WebSocketManager.pushNodeStatus()                  │   │
│     │  │                                                             │   │
│     │  └─────────────────────────────────────────────────────────────┘   │
│     │                                                                       │
│     ▼                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  执行完成                                                              │  │
│  │  ├── StateManager.updateWorkflowStatus() - 更新最终状态              │  │
│  │  └── WebSocketManager.pushWorkflowStatus() - 推送完成通知            │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*第二章完成，请确认后继续生成第三章。*
