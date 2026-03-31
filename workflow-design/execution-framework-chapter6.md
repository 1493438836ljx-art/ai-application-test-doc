# 工作流执行框架设计文档

## 第六章：错误处理与重试机制

### 6.1 错误处理架构总览

错误处理是工作流执行框架的重要组成部分，确保系统在异常情况下能够优雅降级、自动恢复或提供清晰的错误信息。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          错误处理架构                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        错误检测层                                    │  │
│   │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │  │
│   │  │ 节点执行异常   │  │ 超时检测       │  │ 资源不足/不可用       │   │  │
│   │  │ - 技能执行失败 │  │ - 节点超时     │  │ - Client 离线        │   │  │
│   │  │ - 参数错误    │  │ - 工作流超时   │  │ - 线程池满           │   │  │
│   │  │ - 网络异常    │  │               │  │ - 数据库连接失败      │   │  │
│   │  └───────────────┘  └───────────────┘  └───────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        错误分类层                                    │  │
│   │  ┌───────────────────────────────────────────────────────────────┐  │  │
│   │  │  可恢复错误 (Recoverable)                                      │  │  │
│   │  │  - 网络超时、临时资源不可用、外部服务限流                       │  │  │
│   │  │  → 策略：重试                                                 │  │  │
│   │  └───────────────────────────────────────────────────────────────┘  │  │
│   │  ┌───────────────────────────────────────────────────────────────┐  │  │
│   │  │  业务错误 (Business)                                           │  │  │
│   │  │  - 参数校验失败、业务规则不满足、数据格式错误                   │  │  │
│   │  │  → 策略：跳过/终止/执行错误分支                                │  │  │
│   │  └───────────────────────────────────────────────────────────────┘  │  │
│   │  ┌───────────────────────────────────────────────────────────────┐  │  │
│   │  │  系统错误 (System)                                             │  │  │
│   │  │  - 配置错误、依赖缺失、内部异常                                │  │  │
│   │  │  → 策略：终止并告警                                            │  │  │
│   │  └───────────────────────────────────────────────────────────────┘  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        错误处理层                                    │  │
│   │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │  │
│   │  │ 重试机制      │  │ 错误分支      │  │ 补偿机制              │   │  │
│   │  │ - 指数退避    │  │ - 错误处理节点│  │ - 回滚操作            │   │  │
│   │  │ - 最大重试次数│  │ - 降级处理    │  │ - 清理资源            │   │  │
│   │  └───────────────┘  └───────────────┘  └───────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        错误记录与告警                                │  │
│   │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │  │
│   │  │ 错误日志      │  │ 错误统计      │  │ 告警通知              │   │  │
│   │  │ - 详细堆栈    │  │ - 错误类型    │  │ - 邮件/钉钉/企微      │   │  │
│   │  │ - 上下文信息  │  │ - 错误频率    │  │ - 升级策略            │   │  │
│   │  └───────────────┘  └───────────────┘  └───────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 6.2 错误类型定义

#### 6.2.1 错误分类枚举

```java
/**
 * 错误类型枚举
 */
public enum ErrorType {

    /**
     * 可恢复错误 - 可通过重试解决
     */
    RECOVERABLE("可恢复错误"),

    /**
     * 业务错误 - 业务逻辑导致的错误
     */
    BUSINESS("业务错误"),

    /**
     * 系统错误 - 系统内部错误
     */
    SYSTEM("系统错误");

    private final String description;

    ErrorType(String description) {
        this.description = description;
    }

    public String getDescription() {
        return description;
    }
}
```

#### 6.2.2 错误代码定义

```java
/**
 * 错误代码枚举
 */
public enum ErrorCode {

    // ==================== 节点执行错误 (1xxx) ====================
    NODE_EXECUTION_FAILED(1001, "节点执行失败", ErrorType.BUSINESS),
    NODE_TIMEOUT(1002, "节点执行超时", ErrorType.RECOVERABLE),
    NODE_SKIPPED(1003, "节点被跳过", ErrorType.BUSINESS),
    NODE_INPUT_INVALID(1004, "节点输入参数无效", ErrorType.BUSINESS),
    NODE_OUTPUT_INVALID(1005, "节点输出参数无效", ErrorType.BUSINESS),

    // ==================== Skill 执行错误 (2xxx) ====================
    SKILL_NOT_FOUND(2001, "Skill 不存在", ErrorType.SYSTEM),
    SKILL_NOT_PUBLISHED(2002, "Skill 未发布", ErrorType.SYSTEM),
    SKILL_EXECUTION_FAILED(2003, "Skill 执行失败", ErrorType.BUSINESS),
    SKILL_TIMEOUT(2004, "Skill 执行超时", ErrorType.RECOVERABLE),
    SKILL_INCOMPATIBLE(2005, "Skill 版本不兼容", ErrorType.SYSTEM),

    // ==================== Client 执行机错误 (3xxx) ====================
    CLIENT_NOT_AVAILABLE(3001, "无可用的 Client 执行机", ErrorType.RECOVERABLE),
    CLIENT_OFFLINE(3002, "Client 执行机离线", ErrorType.RECOVERABLE),
    CLIENT_TIMEOUT(3003, "Client 响应超时", ErrorType.RECOVERABLE),
    CLIENT_EXECUTION_ERROR(3004, "Client 执行错误", ErrorType.BUSINESS),

    // ==================== 网络通信错误 (4xxx) ====================
    NETWORK_TIMEOUT(4001, "网络超时", ErrorType.RECOVERABLE),
    NETWORK_CONNECTION_FAILED(4002, "网络连接失败", ErrorType.RECOVERABLE),
    KAFKA_PUBLISH_FAILED(4003, "Kafka 消息发送失败", ErrorType.RECOVERABLE),
    KAFKA_CONSUME_FAILED(4004, "Kafka 消息消费失败", ErrorType.RECOVERABLE),

    // ==================== 工作流错误 (5xxx) ====================
    WORKFLOW_NOT_FOUND(5001, "工作流不存在", ErrorType.SYSTEM),
    WORKFLOW_NOT_PUBLISHED(5002, "工作流未发布", ErrorType.SYSTEM),
    WORKFLOW_CYCLIC_DEPENDENCY(5003, "工作流存在循环依赖", ErrorType.SYSTEM),
    WORKFLOW_EXECUTION_FAILED(5004, "工作流执行失败", ErrorType.BUSINESS),
    WORKFLOW_TIMEOUT(5005, "工作流执行超时", ErrorType.RECOVERABLE),

    // ==================== 系统错误 (9xxx) ====================
    INTERNAL_ERROR(9001, "系统内部错误", ErrorType.SYSTEM),
    CONFIGURATION_ERROR(9002, "配置错误", ErrorType.SYSTEM),
    RESOURCE_EXHAUSTED(9003, "资源不足", ErrorType.RECOVERABLE),
    DATABASE_ERROR(9004, "数据库错误", ErrorType.SYSTEM);

    private final int code;
    private final String message;
    private final ErrorType type;

    ErrorCode(int code, String message, ErrorType type) {
        this.code = code;
        this.message = message;
        this.type = type;
    }

    public int getCode() {
        return code;
    }

    public String getMessage() {
        return message;
    }

    public ErrorType getType() {
        return type;
    }
}
```

#### 6.2.3 执行异常类

```java
/**
 * 工作流执行异常基类
 */
public class WorkflowExecutionException extends RuntimeException {

    private final ErrorCode errorCode;
    private final ErrorType errorType;
    private final String nodeUuid;
    private final Map<String, Object> context;

    public WorkflowExecutionException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
        this.errorType = errorCode.getType();
        this.nodeUuid = null;
        this.context = new HashMap<>();
    }

    public WorkflowExecutionException(ErrorCode errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
        this.errorType = errorCode.getType();
        this.nodeUuid = null;
        this.context = new HashMap<>();
    }

    public WorkflowExecutionException(ErrorCode errorCode, String nodeUuid, String message) {
        super(message);
        this.errorCode = errorCode;
        this.errorType = errorCode.getType();
        this.nodeUuid = nodeUuid;
        this.context = new HashMap<>();
    }

    public WorkflowExecutionException(ErrorCode errorCode, Throwable cause) {
        super(errorCode.getMessage(), cause);
        this.errorCode = errorCode;
        this.errorType = errorCode.getType();
        this.nodeUuid = null;
        this.context = new HashMap<>();
    }

    public WorkflowExecutionException withContext(String key, Object value) {
        this.context.put(key, value);
        return this;
    }

    // Getters
    public ErrorCode getErrorCode() { return errorCode; }
    public ErrorType getErrorType() { return errorType; }
    public String getNodeUuid() { return nodeUuid; }
    public Map<String, Object> getContext() { return context; }
}

/**
 * 节点执行异常
 */
public class NodeExecutionException extends WorkflowExecutionException {
    public NodeExecutionException(String nodeUuid, String message) {
        super(ErrorCode.NODE_EXECUTION_FAILED, nodeUuid, message);
    }

    public NodeExecutionException(String nodeUuid, String message, Throwable cause) {
        super(ErrorCode.NODE_EXECUTION_FAILED, cause);
    }
}

/**
 * 节点超时异常
 */
public class NodeTimeoutException extends WorkflowExecutionException {
    public NodeTimeoutException(String nodeUuid, long timeoutMs) {
        super(ErrorCode.NODE_TIMEOUT, nodeUuid,
              String.format("节点执行超时: timeout=%dms", timeoutMs));
    }
}

/**
 * Skill 执行异常
 */
public class SkillExecutionException extends WorkflowExecutionException {
    public SkillExecutionException(String skillId, String message) {
        super(ErrorCode.SKILL_EXECUTION_FAILED, message);
        withContext("skillId", skillId);
    }

    public SkillExecutionException(String skillId, String message, Throwable cause) {
        super(ErrorCode.SKILL_EXECUTION_FAILED, cause);
        withContext("skillId", skillId);
    }
}

/**
 * Client 不可用异常
 */
public class ClientNotAvailableException extends WorkflowExecutionException {
    public ClientNotAvailableException() {
        super(ErrorCode.CLIENT_NOT_AVAILABLE);
    }

    public ClientNotAvailableException(String executionType) {
        super(ErrorCode.CLIENT_NOT_AVAILABLE,
              String.format("没有可用的 Client 支持执行类型: %s", executionType));
    }
}
```

---

### 6.3 节点级错误处理策略

#### 6.3.1 错误策略配置

```java
/**
 * 节点错误策略枚举
 */
public enum ErrorStrategy {

    /**
     * 终止工作流 - 节点失败时立即终止整个工作流
     */
    STOP("终止工作流"),

    /**
     * 跳过当前节点 - 忽略错误，继续执行后续节点
     */
    SKIP("跳过当前节点"),

    /**
     * 重试 - 自动重试执行
     */
    RETRY("重试执行"),

    /**
     * 执行错误分支 - 跳转到错误处理分支
     */
    ERROR_BRANCH("执行错误分支");

    private final String description;

    ErrorStrategy(String description) {
        this.description = description;
    }

    public String getDescription() {
        return description;
    }
}

/**
 * 节点错误处理配置
 */
@Data
@Builder
public class NodeErrorConfig {

    /**
     * 错误策略
     */
    private ErrorStrategy strategy;

    /**
     * 最大重试次数（RETRY 策略时有效）
     */
    @Builder.Default
    private int maxRetries = 3;

    /**
     * 重试间隔（毫秒）
     */
    @Builder.Default
    private long retryInterval = 1000;

    /**
     * 是否使用指数退避
     */
    @Builder.Default
    private boolean exponentialBackoff = true;

    /**
     * 指数退避倍数
     */
    @Builder.Default
    private double backoffMultiplier = 2.0;

    /**
     * 最大重试间隔（毫秒）
     */
    @Builder.Default
    private long maxRetryInterval = 60000;

    /**
     * 错误分支节点 UUID（ERROR_BRANCH 策略时有效）
     */
    private String errorBranchNodeUuid;

    /**
     * 跳过原因记录（SKIP 策略时有效）
     */
    private String skipReason;
}
```

#### 6.3.2 错误处理器

```java
/**
 * 节点错误处理器
 */
@Component
@Slf4j
public class NodeErrorHandler {

    private final StateManager stateManager;
    private final WebSocketManager webSocketManager;
    private final ErrorLogService errorLogService;

    /**
     * 处理节点执行错误
     *
     * @param node     节点定义
     * @param context  执行上下文
     * @param exception 异常
     * @return 错误处理结果
     */
    public ErrorHandleResult handleNodeError(WorkflowNode node,
                                              ExecutionContext context,
                                              Exception exception) {

        String nodeUuid = node.getUuid();
        NodeErrorConfig errorConfig = parseErrorConfig(node.getErrorStrategy(),
                                                        node.getErrorConfig());

        // 记录错误日志
        logError(node, context, exception);

        // 更新节点状态为失败
        stateManager.updateNodeStatus(context.getExecutionId(), nodeUuid,
                                       NodeExecutionStatus.FAILED);

        // 推送 WebSocket 状态
        webSocketManager.pushNodeStatus(context.getExecutionUuid(), nodeUuid,
                                         node.getName(), NodeExecutionStatus.FAILED);

        // 根据错误类型和策略处理
        ErrorType errorType = determineErrorType(exception);

        if (errorType == ErrorType.RECOVERABLE &&
            errorConfig.getStrategy() == ErrorStrategy.RETRY) {
            // 可恢复错误 + 重试策略
            return handleRetry(node, context, exception, errorConfig);
        }

        // 根据策略处理
        switch (errorConfig.getStrategy()) {
            case STOP:
                return handleStop(node, context, exception);

            case SKIP:
                return handleSkip(node, context, exception, errorConfig);

            case RETRY:
                return handleRetry(node, context, exception, errorConfig);

            case ERROR_BRANCH:
                return handleErrorBranch(node, context, exception, errorConfig);

            default:
                return handleStop(node, context, exception);
        }
    }

    /**
     * 处理终止策略
     */
    private ErrorHandleResult handleStop(WorkflowNode node,
                                          ExecutionContext context,
                                          Exception exception) {
        log.error("节点执行失败，终止工作流: nodeUuid={}, error={}",
                  node.getUuid(), exception.getMessage());

        // 更新工作流状态为失败
        stateManager.updateWorkflowStatus(context.getExecutionId(),
                                           ExecutionStatus.FAILED,
                                           null,
                                           Map.of("error", exception.getMessage()));

        return ErrorHandleResult.stop(exception.getMessage());
    }

    /**
     * 处理跳过策略
     */
    private ErrorHandleResult handleSkip(WorkflowNode node,
                                          ExecutionContext context,
                                          Exception exception,
                                          NodeErrorConfig errorConfig) {
        log.warn("节点执行失败，跳过继续: nodeUuid={}, error={}",
                 node.getUuid(), exception.getMessage());

        // 更新节点状态为跳过
        stateManager.updateNodeStatus(context.getExecutionId(), node.getUuid(),
                                       NodeExecutionStatus.SKIPPED);

        // 记录跳过原因
        String skipReason = errorConfig.getSkipReason() != null ?
            errorConfig.getSkipReason() : exception.getMessage();

        return ErrorHandleResult.skip(skipReason);
    }

    /**
     * 处理重试策略
     */
    private ErrorHandleResult handleRetry(WorkflowNode node,
                                           ExecutionContext context,
                                           Exception exception,
                                           NodeErrorConfig errorConfig) {

        String nodeUuid = node.getUuid();
        int currentRetryCount = getRetryCount(context, nodeUuid);

        if (currentRetryCount >= errorConfig.getMaxRetries()) {
            log.error("节点重试次数已达上限: nodeUuid={}, retries={}/{}",
                      nodeUuid, currentRetryCount, errorConfig.getMaxRetries());

            // 重试失败，根据默认策略处理
            return handleStop(node, context,
                new RuntimeException("重试次数已达上限: " + currentRetryCount));
        }

        // 计算重试间隔
        long retryInterval = calculateRetryInterval(errorConfig, currentRetryCount);

        log.info("节点执行失败，准备重试: nodeUuid={}, retry={}/{}, interval={}ms",
                 nodeUuid, currentRetryCount + 1, errorConfig.getMaxRetries(),
                 retryInterval);

        // 更新重试计数
        incrementRetryCount(context, nodeUuid);

        return ErrorHandleResult.retry(retryInterval);
    }

    /**
     * 处理错误分支策略
     */
    private ErrorHandleResult handleErrorBranch(WorkflowNode node,
                                                 ExecutionContext context,
                                                 Exception exception,
                                                 NodeErrorConfig errorConfig) {

        String errorBranchNodeUuid = errorConfig.getErrorBranchNodeUuid();

        if (errorBranchNodeUuid == null) {
            log.warn("未配置错误分支节点，使用终止策略: nodeUuid={}", node.getUuid());
            return handleStop(node, context, exception);
        }

        log.info("节点执行失败，跳转到错误分支: nodeUuid={}, errorBranch={}",
                 node.getUuid(), errorBranchNodeUuid);

        // 设置错误信息到上下文
        context.setGlobalVariable("_error_node_uuid", node.getUuid());
        context.setGlobalVariable("_error_message", exception.getMessage());

        return ErrorHandleResult.errorBranch(errorBranchNodeUuid);
    }

    /**
     * 确定错误类型
     */
    private ErrorType determineErrorType(Exception exception) {
        if (exception instanceof WorkflowExecutionException) {
            return ((WorkflowExecutionException) exception).getErrorType();
        }

        // 默认为业务错误
        return ErrorType.BUSINESS;
    }

    /**
     * 计算重试间隔（支持指数退避）
     */
    private long calculateRetryInterval(NodeErrorConfig config, int retryCount) {
        if (!config.isExponentialBackoff()) {
            return config.getRetryInterval();
        }

        long interval = (long) (config.getRetryInterval() *
                               Math.pow(config.getBackoffMultiplier(), retryCount));

        return Math.min(interval, config.getMaxRetryInterval());
    }

    /**
     * 获取重试次数
     */
    private int getRetryCount(ExecutionContext context, String nodeUuid) {
        Object count = context.getGlobalVariable("_retry_count_" + nodeUuid);
        return count != null ? (int) count : 0;
    }

    /**
     * 增加重试次数
     */
    private void incrementRetryCount(ExecutionContext context, String nodeUuid) {
        int currentCount = getRetryCount(context, nodeUuid);
        context.setGlobalVariable("_retry_count_" + nodeUuid, currentCount + 1);
    }

    /**
     * 记录错误日志
     */
    private void logError(WorkflowNode node, ExecutionContext context,
                          Exception exception) {
        ErrorLog errorLog = ErrorLog.builder()
            .executionId(context.getExecutionId())
            .nodeUuid(node.getUuid())
            .nodeName(node.getName())
            .errorType(determineErrorType(exception).name())
            .errorMessage(exception.getMessage())
            .errorStack(getStackTrace(exception))
            .timestamp(LocalDateTime.now())
            .build();

        errorLogService.save(errorLog);
    }

    /**
     * 获取异常堆栈
     */
    private String getStackTrace(Exception exception) {
        StringWriter sw = new StringWriter();
        exception.printStackTrace(new PrintWriter(sw));
        return sw.toString();
    }

    /**
     * 解析错误配置
     */
    private NodeErrorConfig parseErrorConfig(String strategy, String configJson) {
        // 从节点配置中解析错误处理配置
        // TODO: 实现 JSON 解析
        return NodeErrorConfig.builder()
            .strategy(ErrorStrategy.valueOf(strategy))
            .build();
    }
}

/**
 * 错误处理结果
 */
@Data
@Builder
public class ErrorHandleResult {

    /** 处理动作 */
    private ErrorHandleAction action;

    /** 错误消息 */
    private String errorMessage;

    /** 重试间隔（RETRY 时有效） */
    private Long retryIntervalMs;

    /** 错误分支节点 UUID（ERROR_BRANCH 时有效） */
    private String errorBranchNodeUuid;

    /** 跳过原因（SKIP 时有效） */
    private String skipReason;

    public static ErrorHandleResult stop(String errorMessage) {
        return ErrorHandleResult.builder()
            .action(ErrorHandleAction.STOP)
            .errorMessage(errorMessage)
            .build();
    }

    public static ErrorHandleResult skip(String reason) {
        return ErrorHandleResult.builder()
            .action(ErrorHandleAction.SKIP)
            .skipReason(reason)
            .build();
    }

    public static ErrorHandleResult retry(long intervalMs) {
        return ErrorHandleResult.builder()
            .action(ErrorHandleAction.RETRY)
            .retryIntervalMs(intervalMs)
            .build();
    }

    public static ErrorHandleResult errorBranch(String nodeUuid) {
        return ErrorHandleResult.builder()
            .action(ErrorHandleAction.ERROR_BRANCH)
            .errorBranchNodeUuid(nodeUuid)
            .build();
    }
}

/**
 * 错误处理动作
 */
public enum ErrorHandleAction {
    STOP,           // 终止工作流
    SKIP,           // 跳过继续
    RETRY,          // 重试执行
    ERROR_BRANCH    // 执行错误分支
}
```

---

### 6.4 重试机制实现

#### 6.4.1 重试执行器

```java
/**
 * 重试执行器
 * 封装节点执行的重试逻辑
 */
@Component
@Slf4j
public class RetryExecutor {

    private final NodeExecutorRegistry executorRegistry;
    private final ParameterResolver parameterResolver;
    private final NodeErrorHandler nodeErrorHandler;

    /**
     * 带重试的执行节点
     *
     * @param node    节点定义
     * @param context 执行上下文
     * @return 执行结果
     */
    public NodeExecutionResult executeWithRetry(WorkflowNode node,
                                                 ExecutionContext context) {

        NodeErrorConfig errorConfig = getErrorConfig(node);
        int maxRetries = errorConfig.getMaxRetries();
        int attempt = 0;
        Exception lastException = null;

        while (attempt <= maxRetries) {
            attempt++;

            try {
                // 解析输入参数
                Map<String, Object> inputs = parameterResolver.resolveInputs(node, context);

                // 获取执行器
                NodeExecutor executor = executorRegistry.getExecutor(node.getType());

                // 执行节点
                NodeExecutionResult result = executor.execute(node, inputs, context);

                if (result.isSuccess()) {
                    // 执行成功，返回结果
                    if (attempt > 1) {
                        log.info("节点重试成功: nodeUuid={}, attempts={}",
                                 node.getUuid(), attempt);
                    }
                    return result;
                }

                // 执行失败，记录异常
                lastException = new NodeExecutionException(
                    node.getUuid(),
                    result.getErrorMessage()
                );

            } catch (Exception e) {
                lastException = e;
                log.warn("节点执行异常: nodeUuid={}, attempt={}/{}, error={}",
                         node.getUuid(), attempt, maxRetries + 1, e.getMessage());
            }

            // 检查是否需要重试
            if (attempt <= maxRetries) {
                // 计算等待时间
                long waitTime = calculateWaitTime(errorConfig, attempt - 1);

                log.info("等待重试: nodeUuid={}, waitMs={}, nextAttempt={}",
                         node.getUuid(), waitTime, attempt + 1);

                // 等待
                try {
                    Thread.sleep(waitTime);
                } catch (InterruptedException ie) {
                    Thread.currentThread().interrupt();
                    throw new WorkflowExecutionException(ErrorCode.INTERNAL_ERROR, ie);
                }
            }
        }

        // 所有重试都失败
        log.error("节点执行最终失败: nodeUuid={}, attempts={}, lastError={}",
                  node.getUuid(), attempt, lastException.getMessage());

        // 根据错误策略处理
        ErrorHandleResult handleResult = nodeErrorHandler.handleNodeError(
            node, context, lastException
        );

        // 返回失败结果
        return NodeExecutionResult.failure(lastException.getMessage());
    }

    /**
     * 获取错误配置
     */
    private NodeErrorConfig getErrorConfig(WorkflowNode node) {
        // 从节点配置中获取
        if (node.getErrorConfig() != null) {
            return parseErrorConfig(node.getErrorConfig());
        }

        // 默认配置
        return NodeErrorConfig.builder()
            .strategy(ErrorStrategy.STOP)
            .maxRetries(3)
            .retryInterval(1000)
            .exponentialBackoff(true)
            .build();
    }

    /**
     * 计算等待时间
     */
    private long calculateWaitTime(NodeErrorConfig config, int retryCount) {
        if (!config.isExponentialBackoff()) {
            return config.getRetryInterval();
        }

        long interval = (long) (config.getRetryInterval() *
                               Math.pow(config.getBackoffMultiplier(), retryCount));

        // 添加随机抖动（避免重试风暴）
        double jitter = 0.5 + Math.random(); // 0.5 ~ 1.5
        interval = (long) (interval * jitter);

        return Math.min(interval, config.getMaxRetryInterval());
    }

    /**
     * 解析错误配置
     */
    private NodeErrorConfig parseErrorConfig(String json) {
        // TODO: 实现 JSON 解析
        return NodeErrorConfig.builder().build();
    }
}
```

#### 6.4.2 异步重试调度器

```java
/**
 * 异步重试调度器
 * 用于处理需要延迟重试的任务
 */
@Component
@Slf4j
public class AsyncRetryScheduler {

    private final ScheduledExecutorService scheduler =
        Executors.newScheduledThreadPool(5);

    private final ConcurrentHashMap<String, ScheduledFuture<?>> scheduledTasks =
        new ConcurrentHashMap<>();

    private final NodeExecutorRegistry executorRegistry;
    private final ParameterResolver parameterResolver;
    private final StateManager stateManager;

    /**
     * 调度延迟重试
     *
     * @param node      节点定义
     * @param context   执行上下文
     * @param delayMs   延迟时间
     * @param retryCount 当前重试次数
     */
    public void scheduleRetry(WorkflowNode node,
                               ExecutionContext context,
                               long delayMs,
                               int retryCount) {

        String taskId = context.getExecutionUuid() + "_" + node.getUuid() + "_retry";

        // 取消之前的重试任务
        cancelRetry(taskId);

        log.info("调度重试任务: taskId={}, delayMs={}, retryCount={}",
                 taskId, delayMs, retryCount);

        ScheduledFuture<?> future = scheduler.schedule(() -> {
            try {
                executeRetry(node, context, retryCount);
            } catch (Exception e) {
                log.error("重试任务执行失败: taskId={}", taskId, e);
            } finally {
                scheduledTasks.remove(taskId);
            }
        }, delayMs, TimeUnit.MILLISECONDS);

        scheduledTasks.put(taskId, future);
    }

    /**
     * 取消重试任务
     */
    public void cancelRetry(String taskId) {
        ScheduledFuture<?> future = scheduledTasks.remove(taskId);
        if (future != null && !future.isDone()) {
            future.cancel(false);
            log.info("取消重试任务: taskId={}", taskId);
        }
    }

    /**
     * 执行重试
     */
    private void executeRetry(WorkflowNode node,
                               ExecutionContext context,
                               int retryCount) {

        String nodeUuid = node.getUuid();

        log.info("执行重试: nodeUuid={}, retryCount={}", nodeUuid, retryCount);

        // 更新节点状态为重试中
        stateManager.updateNodeStatus(context.getExecutionId(), nodeUuid,
                                       NodeExecutionStatus.RUNNING);

        try {
            Map<String, Object> inputs = parameterResolver.resolveInputs(node, context);
            NodeExecutor executor = executorRegistry.getExecutor(node.getType());
            NodeExecutionResult result = executor.execute(node, inputs, context);

            if (result.isSuccess()) {
                // 重试成功
                context.setNodeOutputs(nodeUuid, result.getOutputs());
                stateManager.updateNodeStatus(context.getExecutionId(), nodeUuid,
                                               NodeExecutionStatus.SUCCESS);
                log.info("重试成功: nodeUuid={}", nodeUuid);
            } else {
                // 重试失败，继续调度
                scheduleNextRetry(node, context, retryCount);
            }

        } catch (Exception e) {
            log.error("重试执行异常: nodeUuid={}", nodeUuid, e);
            scheduleNextRetry(node, context, retryCount);
        }
    }

    /**
     * 调度下一次重试
     */
    private void scheduleNextRetry(WorkflowNode node,
                                    ExecutionContext context,
                                    int currentRetryCount) {

        NodeErrorConfig config = getErrorConfig(node);
        int nextRetryCount = currentRetryCount + 1;

        if (nextRetryCount > config.getMaxRetries()) {
            log.error("重试次数已达上限: nodeUuid={}", node.getUuid());
            stateManager.updateNodeStatus(context.getExecutionId(), node.getUuid(),
                                           NodeExecutionStatus.FAILED);
            return;
        }

        long delayMs = calculateRetryDelay(config, nextRetryCount);
        scheduleRetry(node, context, delayMs, nextRetryCount);
    }

    private long calculateRetryDelay(NodeErrorConfig config, int retryCount) {
        if (!config.isExponentialBackoff()) {
            return config.getRetryInterval();
        }
        return (long) (config.getRetryInterval() *
                       Math.pow(config.getBackoffMultiplier(), retryCount));
    }

    private NodeErrorConfig getErrorConfig(WorkflowNode node) {
        // TODO: 从节点配置获取
        return NodeErrorConfig.builder()
            .maxRetries(3)
            .retryInterval(1000)
            .exponentialBackoff(true)
            .build();
    }
}
```

---

### 6.5 工作流级错误处理

#### 6.5.1 工作流错误处理配置

```java
/**
 * 工作流错误处理配置
 */
@Data
@Builder
public class WorkflowErrorConfig {

    /**
     * 全局错误策略（节点未配置时使用）
     */
    @Builder.Default
    private ErrorStrategy defaultNodeStrategy = ErrorStrategy.STOP;

    /**
     * 默认重试次数
     */
    @Builder.Default
    private int defaultMaxRetries = 3;

    /**
     * 工作流超时时间（毫秒）
     */
    @Builder.Default
    private long workflowTimeout = 3600000; // 1小时

    /**
     * 是否允许部分成功
     */
    @Builder.Default
    private boolean allowPartialSuccess = false;

    /**
     * 失败时是否执行清理
     */
    @Builder.Default
    private boolean cleanupOnFailure = true;

    /**
     * 错误通知配置
     */
    private ErrorNotificationConfig notificationConfig;
}

/**
 * 错误通知配置
 */
@Data
@Builder
public class ErrorNotificationConfig {

    /** 是否启用通知 */
    private boolean enabled;

    /** 通知渠道: EMAIL / DINGTALK / WECOM */
    private List<String> channels;

    /** 通知接收人 */
    private List<String> recipients;

    /** 只在特定错误类型时通知 */
    private List<ErrorType> notifyOnErrorTypes;
}
```

#### 6.5.2 工作流错误处理器

```java
/**
 * 工作流错误处理器
 */
@Component
@Slf4j
public class WorkflowErrorHandler {

    private final StateManager stateManager;
    private final WebSocketManager webSocketManager;
    private final ErrorNotificationService notificationService;
    private final CompensationService compensationService;

    /**
     * 处理工作流执行错误
     */
    public void handleWorkflowError(Long executionId,
                                     String executionUuid,
                                     Exception exception,
                                     WorkflowErrorConfig errorConfig) {

        log.error("工作流执行失败: executionId={}, error={}",
                  executionId, exception.getMessage());

        // 1. 更新工作流状态
        ExecutionStatus finalStatus = determineFinalStatus(executionId, errorConfig);

        stateManager.updateWorkflowStatus(executionId, finalStatus, null,
            Map.of("errorMessage", exception.getMessage()));

        // 2. 推送 WebSocket 状态
        webSocketManager.pushWorkflowStatus(executionUuid,
            finalStatus.name(), 100);

        // 3. 执行清理/补偿
        if (errorConfig.isCleanupOnFailure()) {
            compensationService.executeCleanup(executionId);
        }

        // 4. 发送错误通知
        if (errorConfig.getNotificationConfig() != null &&
            errorConfig.getNotificationConfig().isEnabled()) {

            notificationService.sendErrorNotification(
                executionId, executionUuid, exception,
                errorConfig.getNotificationConfig()
            );
        }

        // 5. 记录错误日志
        logWorkflowError(executionId, exception);
    }

    /**
     * 确定最终状态
     */
    private ExecutionStatus determineFinalStatus(Long executionId,
                                                  WorkflowErrorConfig errorConfig) {

        if (errorConfig.isAllowPartialSuccess()) {
            // 检查是否有成功的节点
            int successCount = stateManager.countNodesByStatus(
                executionId, NodeExecutionStatus.SUCCESS);

            if (successCount > 0) {
                return ExecutionStatus.PARTIAL_SUCCESS;
            }
        }

        return ExecutionStatus.FAILED;
    }

    /**
     * 记录工作流错误日志
     */
    private void logWorkflowError(Long executionId, Exception exception) {
        // TODO: 实现详细的错误日志记录
    }
}
```

---

### 6.6 错误恢复与补偿

#### 6.6.1 补偿服务

```java
/**
 * 补偿服务
 * 负责在执行失败时执行清理和回滚操作
 */
@Service
@Slf4j
public class CompensationService {

    private final WorkflowExecutionRepository executionRepository;
    private final WorkflowNodeExecutionRepository nodeExecutionRepository;

    /**
     * 执行清理
     */
    public void executeCleanup(Long executionId) {
        log.info("开始执行清理: executionId={}", executionId);

        // 1. 获取所有已执行的节点
        List<WorkflowNodeExecution> executedNodes =
            nodeExecutionRepository.findByExecutionId(executionId);

        // 2. 按执行顺序逆序执行清理
        List<WorkflowNodeExecution> reverseOrder = executedNodes.stream()
            .sorted(Comparator.comparing(WorkflowNodeExecution::getStartTime).reversed())
            .collect(Collectors.toList());

        for (WorkflowNodeExecution nodeExecution : reverseOrder) {
            if (nodeExecution.getStatus().equals(NodeExecutionStatus.SUCCESS.name())) {
                executeNodeCleanup(nodeExecution);
            }
        }

        log.info("清理执行完成: executionId={}", executionId);
    }

    /**
     * 执行节点清理
     */
    private void executeNodeCleanup(WorkflowNodeExecution nodeExecution) {
        String nodeType = nodeExecution.getNodeType();
        Map<String, Object> outputs = nodeExecution.getOutputData();

        if (outputs == null) {
            return;
        }

        // 根据节点类型执行清理
        switch (nodeType) {
            case "skill":
                cleanupSkillOutputs(outputs);
                break;

            case "batch":
                cleanupBatchOutputs(outputs);
                break;

            default:
                log.debug("节点类型无需清理: nodeType={}", nodeType);
        }
    }

    /**
     * 清理 Skill 输出
     */
    private void cleanupSkillOutputs(Map<String, Object> outputs) {
        // 清理生成的临时文件
        outputs.values().stream()
            .filter(v -> v instanceof String)
            .map(v -> (String) v)
            .filter(path -> path.startsWith("/tmp/") || path.startsWith("/data/temp/"))
            .forEach(this::deleteTempFile);
    }

    /**
     * 清理批处理输出
     */
    private void cleanupBatchOutputs(Map<String, Object> outputs) {
        Object batchResults = outputs.get("batch_results");
        if (batchResults instanceof List) {
            ((List<?>) batchResults).forEach(result -> {
                if (result instanceof Map) {
                    cleanupSkillOutputs((Map<String, Object>) result);
                }
            });
        }
    }

    /**
     * 删除临时文件
     */
    private void deleteTempFile(String filePath) {
        try {
            Files.deleteIfExists(Paths.get(filePath));
            log.debug("删除临时文件: {}", filePath);
        } catch (IOException e) {
            log.warn("删除临时文件失败: {}", filePath, e);
        }
    }
}
```

#### 6.6.2 断点续执行

```java
/**
 * 断点续执行服务
 * 支持从中断点恢复工作流执行
 */
@Service
@Slf4j
public class ResumeService {

    private final WorkflowExecutionRepository executionRepository;
    private final WorkflowNodeExecutionRepository nodeExecutionRepository;
    private final ExecutionEngine executionEngine;
    private final ExecutionContextFactory contextFactory;

    /**
     * 从断点恢复执行
     *
     * @param executionUuid 执行UUID
     * @return 是否恢复成功
     */
    public boolean resumeFromCheckpoint(String executionUuid) {
        log.info("开始断点续执行: executionUuid={}", executionUuid);

        // 1. 获取执行记录
        WorkflowExecution execution = executionRepository
            .findByExecutionUuid(executionUuid)
            .orElseThrow(() -> new ExecutionNotFoundException(executionUuid));

        // 2. 检查是否可以恢复
        if (!canResume(execution)) {
            log.warn("执行记录不可恢复: executionUuid={}, status={}",
                     executionUuid, execution.getStatus());
            return false;
        }

        // 3. 获取已完成的节点
        Set<String> completedNodeUuids = nodeExecutionRepository
            .findByExecutionId(execution.getId()).stream()
            .filter(ne -> NodeExecutionStatus.SUCCESS.name().equals(ne.getStatus()))
            .map(WorkflowNodeExecution::getNodeUuid)
            .collect(Collectors.toSet());

        log.info("已完成节点数: {}", completedNodeUuids.size());

        // 4. 重建执行上下文
        ExecutionContext context = contextFactory.rebuildContext(
            execution, completedNodeUuids
        );

        // 5. 更新执行状态为运行中
        execution.setStatus(ExecutionStatus.RUNNING.name());
        executionRepository.save(execution);

        // 6. 继续执行
        try {
            executionEngine.resume(context, completedNodeUuids);
            return true;
        } catch (Exception e) {
            log.error("断点续执行失败: executionUuid={}", executionUuid, e);
            return false;
        }
    }

    /**
     * 检查是否可以恢复
     */
    private boolean canResume(WorkflowExecution execution) {
        String status = execution.getStatus();

        // 只有失败、中断、超时的执行可以恢复
        return ExecutionStatus.FAILED.name().equals(status) ||
               ExecutionStatus.ABORTED.name().equals(status) ||
               ExecutionStatus.TIMEOUT.name().equals(status);
    }
}

/**
 * 执行上下文工厂
 */
@Service
public class ExecutionContextFactory {

    private final WorkflowRepository workflowRepository;
    private final WorkflowNodeRepository nodeRepository;
    private final WorkflowConnectionRepository connectionRepository;

    /**
     * 重建执行上下文
     */
    public ExecutionContext rebuildContext(WorkflowExecution execution,
                                            Set<String> completedNodeUuids) {

        // 获取工作流定义
        Workflow workflow = workflowRepository.findById(execution.getWorkflowId())
            .orElseThrow();

        List<WorkflowNode> nodes = nodeRepository.findByWorkflowId(workflow.getId());
        List<WorkflowConnection> connections =
            connectionRepository.findByWorkflowId(workflow.getId());

        WorkflowDefinition definition = WorkflowDefinition.builder()
            .workflowId(workflow.getId())
            .workflowName(workflow.getName())
            .nodes(nodes)
            .connections(connections)
            .build();

        // 构建执行上下文
        ExecutionContext context = ExecutionContext.builder()
            .executionId(execution.getId())
            .executionUuid(execution.getExecutionUuid())
            .workflowId(execution.getWorkflowId())
            .definition(definition)
            .build();

        // 设置执行图
        ExecutionGraph graph = buildExecutionGraph(definition);
        context.setExecutionGraph(graph);

        // 恢复已完成的节点输出
        restoreNodeOutputs(context, completedNodeUuids);

        return context;
    }

    /**
     * 恢复节点输出
     */
    private void restoreNodeOutputs(ExecutionContext context,
                                     Set<String> completedNodeUuids) {
        // TODO: 从数据库恢复节点输出
    }

    private ExecutionGraph buildExecutionGraph(WorkflowDefinition definition) {
        // TODO: 构建执行图
        return new ExecutionGraph();
    }
}
```

---

### 6.7 错误日志与告警

#### 6.7.1 错误日志服务

```java
/**
 * 错误日志实体
 */
@Entity
@Table(name = "workflow_error_log")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class WorkflowErrorLog {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    /** 执行记录ID */
    private Long executionId;

    /** 节点UUID */
    private String nodeUuid;

    /** 节点名称 */
    private String nodeName;

    /** 错误类型 */
    private String errorType;

    /** 错误代码 */
    private Integer errorCode;

    /** 错误消息 */
    @Column(length = 2000)
    private String errorMessage;

    /** 错误堆栈 */
    @Column(columnDefinition = "TEXT")
    private String errorStack;

    /** 上下文信息 (JSON) */
    @Column(columnDefinition = "JSON")
    private String contextJson;

    /** 重试次数 */
    private Integer retryCount;

    /** 发生时间 */
    private LocalDateTime timestamp;
}

/**
 * 错误日志服务
 */
@Service
@Slf4j
public class ErrorLogService {

    private final WorkflowErrorLogRepository errorLogRepository;
    private final ObjectMapper objectMapper;

    /**
     * 保存错误日志
     */
    public void save(ErrorLog errorLog) {
        try {
            errorLogRepository.save(errorLog);
        } catch (Exception e) {
            log.error("保存错误日志失败", e);
        }
    }

    /**
     * 查询执行错误日志
     */
    public List<ErrorLog> findByExecutionId(Long executionId) {
        return errorLogRepository.findByExecutionId(executionId);
    }

    /**
     * 统计错误类型
     */
    public Map<String, Long> countByErrorType(LocalDateTime startTime,
                                               LocalDateTime endTime) {
        return errorLogRepository.countByErrorType(startTime, endTime);
    }

    /**
     * 获取高频错误
     */
    public List<Map<String, Object>> getFrequentErrors(int limit) {
        return errorLogRepository.findFrequentErrors(limit);
    }
}
```

#### 6.7.2 错误通知服务

```java
/**
 * 错误通知服务
 */
@Service
@Slf4j
public class ErrorNotificationService {

    private final EmailNotifier emailNotifier;
    private final DingTalkNotifier dingTalkNotifier;
    private final WeComNotifier weComNotifier;

    /**
     * 发送错误通知
     */
    public void sendErrorNotification(Long executionId,
                                       String executionUuid,
                                       Exception exception,
                                       ErrorNotificationConfig config) {

        if (config == null || !config.isEnabled()) {
            return;
        }

        // 构建通知内容
        String subject = String.format("工作流执行失败通知 - %s", executionUuid);
        String content = buildNotificationContent(executionId, executionUuid, exception);

        // 发送到各渠道
        for (String channel : config.getChannels()) {
            try {
                switch (channel.toLowerCase()) {
                    case "email":
                        emailNotifier.send(config.getRecipients(), subject, content);
                        break;

                    case "dingtalk":
                        dingTalkNotifier.send(config.getRecipients(), subject, content);
                        break;

                    case "wecom":
                        weComNotifier.send(config.getRecipients(), subject, content);
                        break;

                    default:
                        log.warn("未知的通知渠道: {}", channel);
                }
            } catch (Exception e) {
                log.error("发送通知失败: channel={}", channel, e);
            }
        }
    }

    /**
     * 构建通知内容
     */
    private String buildNotificationContent(Long executionId,
                                             String executionUuid,
                                             Exception exception) {
        StringBuilder sb = new StringBuilder();

        sb.append("## 工作流执行失败通知\n\n");
        sb.append("**执行ID**: ").append(executionId).append("\n");
        sb.append("**执行UUID**: ").append(executionUuid).append("\n");
        sb.append("**失败时间**: ").append(LocalDateTime.now()).append("\n");
        sb.append("**错误信息**: ").append(exception.getMessage()).append("\n");

        if (exception instanceof WorkflowExecutionException) {
            WorkflowExecutionException wee = (WorkflowExecutionException) exception;
            sb.append("**错误类型**: ").append(wee.getErrorType()).append("\n");
            sb.append("**错误代码**: ").append(wee.getErrorCode()).append("\n");

            if (wee.getNodeUuid() != null) {
                sb.append("**失败节点**: ").append(wee.getNodeUuid()).append("\n");
            }
        }

        return sb.toString();
    }
}

/**
 * 邮件通知器
 */
@Component
public class EmailNotifier {

    private final JavaMailSender mailSender;

    public void send(List<String> recipients, String subject, String content) {
        // TODO: 实现邮件发送
    }
}

/**
 * 钉钉通知器
 */
@Component
public class DingTalkNotifier {

    public void send(List<String> recipients, String subject, String content) {
        // TODO: 实现钉钉通知
    }
}

/**
 * 企业微信通知器
 */
@Component
public class WeComNotifier {

    public void send(List<String> recipients, String subject, String content) {
        // TODO: 实现企业微信通知
    }
}
```

---

### 6.8 错误处理流程图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          错误处理完整流程                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  1. 节点执行                                                                     │
│     │                                                                           │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  NodeExecutor.execute()                                                   │  │
│  │  - 执行节点逻辑                                                           │  │
│  │  - 返回执行结果                                                           │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                           │
│     │  执行成功?                                                                │
│     ├── 是 ──→ 返回成功结果                                                    │
│     │                                                                           │
│     │  否                                                                       │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  确定错误类型                                                              │  │
│  │  - RECOVERABLE (可恢复)                                                   │  │
│  │  - BUSINESS (业务错误)                                                    │  │
│  │  - SYSTEM (系统错误)                                                      │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                           │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  读取节点错误处理配置                                                      │  │
│  │  - errorStrategy: STOP / SKIP / RETRY / ERROR_BRANCH                     │  │
│  │  - maxRetries, retryInterval, exponentialBackoff                          │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                           │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  根据策略处理                                                              │  │
│  │                                                                          │  │
│  │  STOP:                                                                    │  │
│  │  ├── 更新节点状态为 FAILED                                                │  │
│  │  ├── 更新工作流状态为 FAILED                                              │  │
│  │  ├── 执行清理/补偿                                                        │  │
│  │  └── 发送错误通知                                                         │  │
│  │                                                                          │  │
│  │  SKIP:                                                                    │  │
│  │  ├── 更新节点状态为 SKIPPED                                               │  │
│  │  └── 继续执行后续节点                                                     │  │
│  │                                                                          │  │
│  │  RETRY:                                                                   │  │
│  │  ├── 检查重试次数                                                         │  │
│  │  ├── 未达上限: 等待后重试                                                 │  │
│  │  └── 达到上限: 按 STOP 处理                                               │  │
│  │                                                                          │  │
│  │  ERROR_BRANCH:                                                            │  │
│  │  ├── 设置错误上下文                                                       │  │
│  │  └── 跳转到错误处理分支                                                   │  │
│  │                                                                          │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                           │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  记录错误日志                                                              │  │
│  │  - 错误类型、错误代码                                                     │  │
│  │  - 错误消息、错误堆栈                                                     │  │
│  │  - 上下文信息                                                             │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

*第六章完成，请确认后继续生成第七章。*
