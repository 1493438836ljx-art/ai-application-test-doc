# 工作流执行框架设计文档

## 第四章：基于 Kafka 的分布式执行

### 4.1 分布式执行架构总览

本章描述如何通过 Kafka 消息队列实现 Service 端与 Client 执行机之间的通信，支持跨网络分区的任务执行。

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     基于 Kafka 的分布式执行架构                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│   ┌─────────────────────────────────────────────────────────────────────────┐  │
│   │                            Service 端                                    │  │
│   │                                                                          │  │
│   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────┐  │  │
│   │  │WorkflowEngine│  │SkillExecutor │  │    KafkaMessageProducer      │  │  │
│   │  │  (调度执行)   │  │  (任务下发)   │  │    (发送任务到 Kafka)         │  │  │
│   │  └──────────────┘  └──────────────┘  └──────────────────────────────┘  │  │
│   │                                                │                        │  │
│   │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────┐  │  │
│   │  │StateManager  │  │WebSocketPush │  │    KafkaResultConsumer       │  │  │
│   │  │  (状态管理)   │  │  (状态推送)   │  │    (接收执行结果)             │  │  │
│   │  └──────────────┘  └──────────────┘  └──────────────────────────────┘  │  │
│   │                                                │                        │  │
│   └────────────────────────────────────────────────┼────────────────────────┘  │
│                                                    │                           │
│                                                    ▼                           │
│   ┌─────────────────────────────────────────────────────────────────────────┐  │
│   │                           Kafka Cluster                                  │  │
│   │                                                                          │  │
│   │   ┌─────────────────────┐      ┌─────────────────────┐                  │  │
│   │   │ task-client-commands │      │ task-client-results │                  │  │
│   │   │ (任务命令 Topic)     │      │ (执行结果 Topic)     │                  │  │
│   │   │                     │      │                     │                  │  │
│   │   │  Partition 0        │      │  Partition 0        │                  │  │
│   │   │  Partition 1        │      │  Partition 1        │                  │  │
│   │   │  Partition N        │      │  Partition N        │                  │  │
│   │   └─────────────────────┘      └─────────────────────┘                  │  │
│   │                                                                          │  │
│   │   ┌─────────────────────┐      ┌─────────────────────┐                  │  │
│   │   │ heartbeat-client    │      │ task-status-events  │                  │  │
│   │   │ (Client 心跳 Topic)  │      │ (状态事件 Topic)     │                  │  │
│   │   └─────────────────────┘      └─────────────────────┘                  │  │
│   │                                                                          │  │
│   └─────────────────────────────────────────────────────────────────────────┘  │
│                                                    │                           │
│                                                    ▼                           │
│   ┌─────────────────────────────────────────────────────────────────────────┐  │
│   │                        Client 执行机集群                                  │  │
│   │                                                                          │  │
│   │   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐               │  │
│   │   │  Client-1    │   │  Client-2    │   │  Client-N    │               │  │
│   │   │ ┌──────────┐ │   │ ┌──────────┐ │   │ ┌──────────┐ │               │  │
│   │   │ │Consumer   │ │   │ │Consumer   │ │   │ │Consumer   │ │               │  │
│   │   │ │(消费任务) │ │   │ │(消费任务) │ │   │ │(消费任务) │ │               │  │
│   │   │ └──────────┘ │   │ └──────────┘ │   │ └──────────┘ │               │  │
│   │   │ ┌──────────┐ │   │ ┌──────────┐ │   │ ┌──────────┐ │               │  │
│   │   │ │Executor   │ │   │ │Executor   │ │   │ │Executor   │ │               │  │
│   │   │ │(执行套件) │ │   │ │(执行套件) │ │   │ │(执行套件) │ │               │  │
│   │   │ └──────────┘ │   │ └──────────┘ │   │ └──────────┘ │               │  │
│   │   │ ┌──────────┐ │   │ ┌──────────┐ │   │ ┌──────────┐ │               │  │
│   │   │ │Producer   │ │   │ │Producer   │ │   │ │Producer   │ │               │  │
│   │   │ │(回传结果) │ │   │ │(回传结果) │ │   │ │(回传结果) │ │               │  │
│   │   │ └──────────┘ │   │ └──────────┘ │   │ └──────────┘ │               │  │
│   │   └──────────────┘   └──────────────┘   └──────────────┘               │  │
│   │                                                                          │  │
│   └─────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### 4.2 Kafka Topic 设计

#### 4.2.1 Topic 列表

| Topic 名称 | 分区数 | 生产者 | 消费者 | 用途 |
|------------|--------|--------|--------|------|
| task-client-commands | N | Service | Client | 任务命令下发 |
| task-client-results | N | Client | Service | 执行结果回传 |
| heartbeat-client | 1 | Client | Service | Client 心跳上报 |
| task-status-events | N | Client | Service | 任务状态实时事件 |

#### 4.2.2 Topic 配置建议

```yaml
# Kafka Topic 配置
kafka:
  topics:
    task-client-commands:
      partitions: 10
      replication-factor: 3
      config:
        retention.ms: 86400000      # 保留1天
        cleanup.policy: delete
        compression.type: lz4

    task-client-results:
      partitions: 10
      replication-factor: 3
      config:
        retention.ms: 86400000
        cleanup.policy: delete
        compression.type: lz4

    heartbeat-client:
      partitions: 1
      replication-factor: 3
      config:
        retention.ms: 3600000       # 保留1小时
        cleanup.policy: compact     # 压缩模式，保留最新状态

    task-status-events:
      partitions: 10
      replication-factor: 3
      config:
        retention.ms: 604800000     # 保留7天
        cleanup.policy: delete
```

#### 4.2.3 分区策略

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          分区策略设计                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  task-client-commands (任务命令)                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  分区键: clientId                                                    │   │
│  │  策略: 同一 Client 的任务发送到同一分区，保证顺序性                    │   │
│  │                                                                     │   │
│  │  Partition 0 ← Client-1 的所有任务                                  │   │
│  │  Partition 1 ← Client-2 的所有任务                                  │   │
│  │  Partition 2 ← Client-3 的所有任务                                  │   │
│  │  ...                                                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  task-client-results (执行结果)                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  分区键: executionId                                                 │   │
│  │  策略: 同一工作流执行的所有结果发送到同一分区，便于聚合处理            │   │
│  │                                                                     │   │
│  │  Partition 0 ← Execution-1 的所有结果                               │   │
│  │  Partition 1 ← Execution-2 的所有结果                               │   │
│  │  ...                                                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  heartbeat-client (心跳)                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  分区键: clientId                                                    │   │
│  │  策略: 单分区，使用 compact 模式保留最新心跳                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  task-status-events (状态事件)                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  分区键: executionId                                                 │   │
│  │  策略: 同一工作流的状态事件发送到同一分区                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 4.3 消息协议定义

#### 4.3.1 消息基础结构

```java
/**
 * Kafka 消息基础结构
 */
@Data
@Builder
public class KafkaMessage<T> {

    /** 消息ID */
    private String messageId;

    /** 消息类型 */
    private String messageType;

    /** 时间戳 */
    private Long timestamp;

    /** 消息版本 */
    @Builder.Default
    private String version = "1.0";

    /** 消息体 */
    private T payload;
}
```

#### 4.3.2 任务命令消息

```java
/**
 * 任务命令消息
 */
@Data
@Builder
public class TaskCommandMessage {

    private String messageId;
    private String messageType = "TASK_COMMAND";
    private Long timestamp;
    private TaskCommandPayload payload;
}

/**
 * 任务命令载荷
 */
@Data
@Builder
public class TaskCommandPayload {

    /** 任务ID */
    private String taskId;

    /** 执行记录ID */
    private Long executionId;

    /** 工作流ID */
    private Long workflowId;

    /** 节点UUID */
    private String nodeUuid;

    /** Skill ID */
    private String skillId;

    /** Skill 名称 */
    private String skillName;

    /** 执行类型: AI_AGENT / AUTOMATED */
    private String executionType;

    /** 执行套件路径 */
    private String suitePath;

    /** 输入参数 */
    private Map<String, Object> inputs;

    /** 超时时间（毫秒） */
    private Long timeout;

    /** 重试次数 */
    private Integer retryCount;

    /** 优先级: HIGH / NORMAL / LOW */
    private String priority;

    /** 回调 Topic */
    private String callbackTopic;

    /** 循环索引（批处理/循环场景） */
    private Integer loopIndex;

    /** 当前元素（批处理/循环场景） */
    private Object currentItem;
}
```

**消息示例：**

```json
{
  "messageId": "msg-uuid-001",
  "messageType": "TASK_COMMAND",
  "timestamp": 1709234567890,
  "version": "1.0",
  "payload": {
    "taskId": "task-uuid-001",
    "executionId": 12345,
    "workflowId": 100,
    "nodeUuid": "skill-node-1",
    "skillId": "skill-uuid-001",
    "skillName": "文本清洗",
    "executionType": "AUTOMATED",
    "suitePath": "/suites/text_clean.py",
    "inputs": {
      "input_file": "/data/input.xlsx",
      "cols": "A,B,C"
    },
    "timeout": 300000,
    "retryCount": 3,
    "priority": "NORMAL",
    "callbackTopic": "task-client-results",
    "loopIndex": 0,
    "currentItem": null
  }
}
```

#### 4.3.3 执行结果消息

```java
/**
 * 执行结果消息
 */
@Data
@Builder
public class TaskResultMessage {

    private String messageId;
    private String messageType = "TASK_RESULT";
    private Long timestamp;
    private TaskResultPayload payload;
}

/**
 * 执行结果载荷
 */
@Data
@Builder
public class TaskResultPayload {

    /** 任务ID */
    private String taskId;

    /** 执行记录ID */
    private Long executionId;

    /** 节点UUID */
    private String nodeUuid;

    /** 执行的 Client ID */
    private String clientId;

    /** 执行状态: SUCCESS / FAILED / TIMEOUT */
    private String status;

    /** 输出结果 */
    private Map<String, Object> outputs;

    /** 执行日志 */
    private String logs;

    /** 执行耗时（毫秒） */
    private Long durationMs;

    /** 错误信息 */
    private String errorMessage;

    /** 错误堆栈 */
    private String errorStack;

    /** 重试次数 */
    private Integer retryCount;
}
```

**消息示例：**

```json
{
  "messageId": "msg-uuid-002",
  "messageType": "TASK_RESULT",
  "timestamp": 1709234570000,
  "payload": {
    "taskId": "task-uuid-001",
    "executionId": 12345,
    "nodeUuid": "skill-node-1",
    "clientId": "client-machine-001",
    "status": "SUCCESS",
    "outputs": {
      "output_file": "/data/output.xlsx"
    },
    "logs": "执行开始...\n处理完成",
    "durationMs": 2500,
    "errorMessage": null,
    "errorStack": null,
    "retryCount": 0
  }
}
```

#### 4.3.4 心跳消息

```java
/**
 * Client 心跳消息
 */
@Data
@Builder
public class HeartbeatMessage {

    private String messageId;
    private String messageType = "HEARTBEAT";
    private Long timestamp;
    private HeartbeatPayload payload;
}

/**
 * 心跳载荷
 */
@Data
@Builder
public class HeartbeatPayload {

    /** Client ID */
    private String clientId;

    /** Client 名称 */
    private String clientName;

    /** Client 状态: IDLE / BUSY / OFFLINE */
    private String status;

    /** 正在执行的任务数 */
    private Integer runningTasks;

    /** 最大并发数 */
    private Integer maxConcurrency;

    /** CPU 使用率 (%) */
    private Double cpuUsage;

    /** 内存使用率 (%) */
    private Double memoryUsage;

    /** 支持的执行类型 */
    private List<String> supportedExecutionTypes;

    /** Client 版本 */
    private String version;

    /** 标签（用于任务匹配） */
    private Map<String, String> labels;
}
```

**消息示例：**

```json
{
  "messageId": "msg-uuid-003",
  "messageType": "HEARTBEAT",
  "timestamp": 1709234580000,
  "payload": {
    "clientId": "client-machine-001",
    "clientName": "测试执行机-001",
    "status": "IDLE",
    "runningTasks": 0,
    "maxConcurrency": 5,
    "cpuUsage": 35.5,
    "memoryUsage": 62.3,
    "supportedExecutionTypes": ["AUTOMATED", "AI_AGENT"],
    "version": "1.0.0",
    "labels": {
      "env": "test",
      "region": "cn-east"
    }
  }
}
```

#### 4.3.5 状态事件消息

```java
/**
 * 状态事件消息
 */
@Data
@Builder
public class StatusEventMessage {

    private String messageId;
    private String messageType = "STATUS_EVENT";
    private Long timestamp;
    private StatusEventPayload payload;
}

/**
 * 状态事件载荷
 */
@Data
@Builder
public class StatusEventPayload {

    /** 任务ID */
    private String taskId;

    /** 执行记录ID */
    private Long executionId;

    /** 节点UUID */
    private String nodeUuid;

    /** 节点名称 */
    private String nodeName;

    /** Client ID */
    private String clientId;

    /** 事件类型 */
    private String eventType;

    /** 事件数据 */
    private Map<String, Object> eventData;
}
```

**事件类型定义：**

| 事件类型 | 说明 | eventData 示例 |
|----------|------|----------------|
| NODE_STATUS_CHANGED | 节点状态变更 | `{previousStatus, currentStatus, progress, message}` |
| NODE_LOG | 节点执行日志 | `{logLevel, message, progress}` |
| ASYNC_TASK_PROGRESS | 异步任务进度 | `{totalTasks, completedTasks, runningTasks, failedTasks}` |

**消息示例：**

```json
{
  "messageId": "msg-uuid-004",
  "messageType": "STATUS_EVENT",
  "timestamp": 1709234568123,
  "payload": {
    "taskId": "task-uuid-001",
    "executionId": 12345,
    "nodeUuid": "skill-node-1",
    "nodeName": "文本清洗",
    "clientId": "client-machine-001",
    "eventType": "NODE_STATUS_CHANGED",
    "eventData": {
      "previousStatus": "PENDING",
      "currentStatus": "RUNNING",
      "progress": 0,
      "message": "开始执行文本清洗"
    }
  }
}
```

---

### 4.4 Service 端 Kafka 组件

#### 4.4.1 Kafka 生产者配置

```java
/**
 * Kafka 生产者配置
 */
@Configuration
public class KafkaProducerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> config = new HashMap<>();

        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                   StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                   StringSerializer.class);

        // 可靠性配置
        config.put(ProducerConfig.ACKS_CONFIG, "all");
        config.put(ProducerConfig.RETRIES_CONFIG, 3);
        config.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 1);

        // 性能配置
        config.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        config.put(ProducerConfig.LINGER_MS_CONFIG, 5);
        config.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);

        // 压缩配置
        config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");

        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

#### 4.4.2 Kafka 任务生产者

```java
/**
 * Kafka 任务生产者
 * 负责发送任务命令到 Kafka
 */
@Component
@Slf4j
public class KafkaTaskProducer {

    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    private static final String TASK_COMMAND_TOPIC = "task-client-commands";

    /**
     * 发送任务命令到指定 Client
     *
     * @param command  任务命令
     * @param clientId 目标 Client ID
     */
    public void sendTaskCommand(TaskCommandMessage command, String clientId) {
        String partitionKey = clientId; // 使用 clientId 作为分区键

        try {
            String messageJson = objectMapper.writeValueAsString(command);

            CompletableFuture<SendResult<String, String>> future =
                kafkaTemplate.send(TASK_COMMAND_TOPIC, partitionKey, messageJson);

            future.whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("发送任务命令失败: taskId={}, clientId={}",
                              command.getPayload().getTaskId(), clientId, ex);
                    handleSendFailure(command, ex);
                } else {
                    log.info("任务命令已发送: taskId={}, clientId={}, partition={}, offset={}",
                             command.getPayload().getTaskId(),
                             clientId,
                             result.getRecordMetadata().partition(),
                             result.getRecordMetadata().offset());
                }
            });

        } catch (JsonProcessingException e) {
            log.error("序列化任务命令失败: taskId={}", command.getPayload().getTaskId(), e);
            throw new KafkaMessageException("序列化任务命令失败", e);
        }
    }

    /**
     * 批量发送任务命令
     */
    public void sendTaskCommands(List<TaskCommandMessage> commands, String clientId) {
        for (TaskCommandMessage command : commands) {
            sendTaskCommand(command, clientId);
        }
    }

    /**
     * 发送失败处理
     */
    private void handleSendFailure(TaskCommandMessage command, Throwable ex) {
        // 可以实现重试逻辑或将失败任务写入数据库等待后续处理
        // TODO: 实现失败处理逻辑
    }
}
```

#### 4.4.3 Kafka 消费者配置

```java
/**
 * Kafka 消费者配置
 */
@Configuration
@EnableKafka
public class KafkaConsumerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> config = new HashMap<>();

        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                   StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                   StringDeserializer.class);
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "latest");
        config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
        config.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);
        config.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000);

        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, String>
            kafkaListenerContainerFactory() {

        ConcurrentKafkaListenerContainerFactory<String, String> factory =
            new ConcurrentKafkaListenerContainerFactory<>();

        factory.setConsumerFactory(consumerFactory());
        factory.setConcurrency(3);
        factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);

        return factory;
    }
}
```

#### 4.4.4 Kafka 结果消费者

```java
/**
 * Kafka 结果消费者
 * 消费 Client 端回传的执行结果
 */
@Component
@Slf4j
public class KafkaResultConsumer {

    private final ConcurrentHashMap<String, CompletableFuture<SkillExecutionResult>>
        pendingTasks = new ConcurrentHashMap<>();

    private final ObjectMapper objectMapper;
    private final StateManager stateManager;
    private final WebSocketManager webSocketManager;

    /**
     * 消费执行结果
     */
    @KafkaListener(
        topics = "task-client-results",
        groupId = "workflow-engine-result-consumer",
        concurrency = "3"
    )
    public void consumeTaskResult(ConsumerRecord<String, String> record,
                                    Acknowledgment acknowledgment) {
        log.debug("收到执行结果: partition={}, offset={}",
                  record.partition(), record.offset());

        try {
            TaskResultMessage result = objectMapper.readValue(
                record.value(),
                TaskResultMessage.class
            );

            String taskId = result.getPayload().getTaskId();

            // 1. 完成等待中的 Future
            CompletableFuture<SkillExecutionResult> future = pendingTasks.get(taskId);
            if (future != null) {
                SkillExecutionResult executionResult = convertToExecutionResult(result);
                future.complete(executionResult);
                pendingTasks.remove(taskId);
            }

            // 2. 更新数据库状态
            updateExecutionState(result);

            // 3. 手动提交偏移量
            acknowledgment.acknowledge();

        } catch (Exception e) {
            log.error("处理执行结果失败: partition={}, offset={}",
                      record.partition(), record.offset(), e);
            // 不提交偏移量，等待重试
        }
    }

    /**
     * 注册待处理任务
     */
    public CompletableFuture<SkillExecutionResult> registerPendingTask(
            String taskId, long timeoutMs) {

        CompletableFuture<SkillExecutionResult> future = new CompletableFuture<>();
        pendingTasks.put(taskId, future);

        // 超时自动移除
        scheduler.schedule(() -> {
            CompletableFuture<SkillExecutionResult> removed = pendingTasks.remove(taskId);
            if (removed != null && !removed.isDone()) {
                removed.completeExceptionally(new TimeoutException("任务等待超时"));
            }
        }, timeoutMs, TimeUnit.MILLISECONDS);

        return future;
    }

    /**
     * 移除待处理任务
     */
    public void removePendingTask(String taskId) {
        pendingTasks.remove(taskId);
    }

    /**
     * 转换为执行结果
     */
    private SkillExecutionResult convertToExecutionResult(TaskResultMessage message) {
        TaskResultPayload payload = message.getPayload();

        return SkillExecutionResult.builder()
            .success("SUCCESS".equals(payload.getStatus()))
            .outputs(payload.getOutputs())
            .logs(payload.getLogs())
            .durationMs(payload.getDurationMs())
            .errorMessage(payload.getErrorMessage())
            .errorStack(payload.getErrorStack())
            .build();
    }

    /**
     * 更新执行状态
     */
    private void updateExecutionState(TaskResultMessage result) {
        TaskResultPayload payload = result.getPayload();

        // 更新节点执行状态
        NodeExecutionStatus status = "SUCCESS".equals(payload.getStatus()) ?
            NodeExecutionStatus.SUCCESS : NodeExecutionStatus.FAILED;

        stateManager.updateNodeStatus(
            payload.getExecutionId(),
            payload.getNodeUuid(),
            status,
            NodeExecutionResult.builder()
                .success(status == NodeExecutionStatus.SUCCESS)
                .outputs(payload.getOutputs())
                .errorMessage(payload.getErrorMessage())
                .durationMs(payload.getDurationMs())
                .build()
        );

        // 推送 WebSocket 状态
        webSocketManager.pushNodeStatus(
            payload.getExecutionId().toString(),
            payload.getNodeUuid(),
            payload.getNodeUuid(),
            status
        );
    }
}
```

#### 4.4.5 心跳消费者

```java
/**
 * Client 心跳消费者
 * 维护 Client 在线状态
 */
@Component
@Slf4j
public class HeartbeatConsumer {

    private final ClientRegistry clientRegistry;
    private final ObjectMapper objectMapper;

    /**
     * 消费心跳消息
     */
    @KafkaListener(
        topics = "heartbeat-client",
        groupId = "workflow-engine-heartbeat-consumer"
    )
    public void consumeHeartbeat(ConsumerRecord<String, String> record) {
        try {
            HeartbeatMessage heartbeat = objectMapper.readValue(
                record.value(),
                HeartbeatMessage.class
            );

            HeartbeatPayload payload = heartbeat.getPayload();

            // 更新 Client 状态
            ClientInfo clientInfo = ClientInfo.builder()
                .clientId(payload.getClientId())
                .clientName(payload.getClientName())
                .status(ClientStatus.valueOf(payload.getStatus()))
                .runningTasks(payload.getRunningTasks())
                .maxConcurrency(payload.getMaxConcurrency())
                .cpuUsage(payload.getCpuUsage())
                .memoryUsage(payload.getMemoryUsage())
                .supportedExecutionTypes(payload.getSupportedExecutionTypes())
                .version(payload.getVersion())
                .labels(payload.getLabels())
                .lastHeartbeat(LocalDateTime.now())
                .build();

            clientRegistry.registerOrUpdate(clientInfo);

            log.debug("更新 Client 心跳: clientId={}, status={}",
                      payload.getClientId(), payload.getStatus());

        } catch (Exception e) {
            log.error("处理心跳消息失败", e);
        }
    }
}
```

#### 4.4.6 状态事件消费者

```java
/**
 * 状态事件消费者
 * 消费 Client 端推送的状态事件，转发到 WebSocket
 */
@Component
@Slf4j
public class StatusEventConsumer {

    private final StatusEventPublisher statusEventPublisher;
    private final ObjectMapper objectMapper;

    @KafkaListener(
        topics = "task-status-events",
        groupId = "workflow-status-consumer",
        concurrency = "3"
    )
    public void consumeStatusEvent(ConsumerRecord<String, String> record) {
        log.debug("收到状态事件: partition={}, offset={}",
                  record.partition(), record.offset());

        try {
            StatusEventMessage event = objectMapper.readValue(
                record.value(),
                StatusEventMessage.class
            );

            StatusEventPayload payload = event.getPayload();

            switch (payload.getEventType()) {
                case "NODE_STATUS_CHANGED":
                    handleNodeStatusChanged(payload);
                    break;

                case "NODE_LOG":
                    handleNodeLog(payload);
                    break;

                case "ASYNC_TASK_PROGRESS":
                    handleAsyncTaskProgress(payload);
                    break;

                default:
                    log.warn("未知状态事件类型: {}", payload.getEventType());
            }

        } catch (Exception e) {
            log.error("处理状态事件失败", e);
        }
    }

    private void handleNodeStatusChanged(StatusEventPayload payload) {
        Map<String, Object> data = payload.getEventData();

        statusEventPublisher.publishNodeStatusChanged(
            payload.getExecutionId(),
            payload.getExecutionId().toString(),
            payload.getNodeUuid(),
            payload.getNodeName(),
            (String) data.get("previousStatus"),
            (String) data.get("currentStatus")
        );
    }

    private void handleNodeLog(StatusEventPayload payload) {
        Map<String, Object> data = payload.getEventData();

        statusEventPublisher.publishNodeLog(
            payload.getExecutionId(),
            payload.getExecutionId().toString(),
            payload.getNodeUuid(),
            payload.getNodeName(),
            (String) data.get("logLevel"),
            (String) data.get("message"),
            (Integer) data.get("progress")
        );
    }

    private void handleAsyncTaskProgress(StatusEventPayload payload) {
        Map<String, Object> data = payload.getEventData();

        AsyncTaskStatusInfo taskInfo = AsyncTaskStatusInfo.builder()
            .totalTasks((Integer) data.get("totalTasks"))
            .completedTasks((Integer) data.get("completedTasks"))
            .runningTasks((Integer) data.get("runningTasks"))
            .failedTasks((Integer) data.get("failedTasks"))
            .build();

        statusEventPublisher.publishAsyncTaskStatus(
            payload.getExecutionId().toString(),
            payload.getNodeUuid(),
            taskInfo
        );
    }
}
```

---

### 4.5 Client 注册表

#### 4.5.1 Client 信息模型

```java
/**
 * Client 信息
 */
@Data
@Builder
public class ClientInfo {

    /** Client ID */
    private String clientId;

    /** Client 名称 */
    private String clientName;

    /** 状态 */
    private ClientStatus status;

    /** 正在执行的任务数 */
    private Integer runningTasks;

    /** 最大并发数 */
    private Integer maxConcurrency;

    /** CPU 使用率 */
    private Double cpuUsage;

    /** 内存使用率 */
    private Double memoryUsage;

    /** 支持的执行类型 */
    private List<String> supportedExecutionTypes;

    /** 版本号 */
    private String version;

    /** 标签 */
    private Map<String, String> labels;

    /** 最后心跳时间 */
    private LocalDateTime lastHeartbeat;

    /**
     * 判断是否可用
     */
    public boolean isAvailable() {
        return status == ClientStatus.IDLE ||
               (status == ClientStatus.BUSY && runningTasks < maxConcurrency);
    }

    /**
     * 判断是否支持指定执行类型
     */
    public boolean supportsExecutionType(String executionType) {
        return supportedExecutionTypes != null &&
               supportedExecutionTypes.contains(executionType);
    }
}

/**
 * Client 状态
 */
public enum ClientStatus {
    IDLE,       // 空闲
    BUSY,       // 忙碌（有任务在执行）
    OFFLINE     // 离线
}
```

#### 4.5.2 Client 注册表实现

```java
/**
 * Client 注册表
 * 管理所有在线的 Client 执行机
 */
@Component
@Slf4j
public class ClientRegistry {

    /**
     * Client 信息缓存
     * key: clientId
     * value: ClientInfo
     */
    private final ConcurrentHashMap<String, ClientInfo> clientCache =
        new ConcurrentHashMap<>();

    /**
     * 心跳超时时间（毫秒）
     */
    @Value("${workflow.client.heartbeat-timeout:30000}")
    private long heartbeatTimeout;

    /**
     * 注册或更新 Client
     */
    public void registerOrUpdate(ClientInfo clientInfo) {
        clientCache.put(clientInfo.getClientId(), clientInfo);
        log.debug("Client 注册/更新: clientId={}, status={}",
                  clientInfo.getClientId(), clientInfo.getStatus());
    }

    /**
     * 移除 Client
     */
    public void remove(String clientId) {
        clientCache.remove(clientId);
        log.info("Client 已移除: clientId={}", clientId);
    }

    /**
     * 获取 Client 信息
     */
    public ClientInfo get(String clientId) {
        return clientCache.get(clientId);
    }

    /**
     * 获取所有 Client
     */
    public List<ClientInfo> getAll() {
        return new ArrayList<>(clientCache.values());
    }

    /**
     * 获取所有可用的 Client
     */
    public List<ClientInfo> getAvailableClients() {
        return clientCache.values().stream()
            .filter(this::isClientAlive)
            .filter(ClientInfo::isAvailable)
            .collect(Collectors.toList());
    }

    /**
     * 根据条件选择最优 Client
     */
    public ClientInfo selectBestClient(String executionType,
                                        Map<String, String> labelSelectors) {
        return clientCache.values().stream()
            // 过滤：必须存活
            .filter(this::isClientAlive)
            // 过滤：必须可用
            .filter(ClientInfo::isAvailable)
            // 过滤：支持执行类型
            .filter(c -> c.supportsExecutionType(executionType))
            // 过滤：标签匹配
            .filter(c -> matchesLabels(c, labelSelectors))
            // 排序：优先选择负载最低的
            .sorted(Comparator.comparingInt(ClientInfo::getRunningTasks))
            .findFirst()
            .orElse(null);
    }

    /**
     * 判断 Client 是否存活（心跳未超时）
     */
    private boolean isClientAlive(ClientInfo client) {
        if (client.getLastHeartbeat() == null) {
            return false;
        }

        long secondsSinceLastHeartbeat = Duration.between(
            client.getLastHeartbeat(),
            LocalDateTime.now()
        ).getSeconds() * 1000;

        return secondsSinceLastHeartbeat < heartbeatTimeout;
    }

    /**
     * 标签匹配
     */
    private boolean matchesLabels(ClientInfo client, Map<String, String> selectors) {
        if (selectors == null || selectors.isEmpty()) {
            return true;
        }

        Map<String, String> labels = client.getLabels();
        if (labels == null) {
            return false;
        }

        for (Map.Entry<String, String> selector : selectors.entrySet()) {
            String labelValue = labels.get(selector.getKey());
            if (!selector.getValue().equals(labelValue)) {
                return false;
            }
        }

        return true;
    }

    /**
     * 定时清理过期 Client
     */
    @Scheduled(fixedRate = 60000) // 每分钟执行一次
    public void cleanupExpiredClients() {
        List<String> expiredClients = clientCache.entrySet().stream()
            .filter(e -> !isClientAlive(e.getValue()))
            .map(Map.Entry::getKey)
            .collect(Collectors.toList());

        for (String clientId : expiredClients) {
            remove(clientId);
            log.warn("Client 心跳超时，已移除: clientId={}", clientId);
        }
    }

    /**
     * 获取 Client 统计信息
     */
    public ClientStatistics getStatistics() {
        List<ClientInfo> allClients = getAll();
        List<ClientInfo> availableClients = getAvailableClients();

        return ClientStatistics.builder()
            .totalClients(allClients.size())
            .availableClients(availableClients.size())
            .busyClients((int) allClients.stream()
                .filter(c -> c.getStatus() == ClientStatus.BUSY)
                .count())
            .offlineClients((int) allClients.stream()
                .filter(c -> c.getStatus() == ClientStatus.OFFLINE)
                .count())
            .totalRunningTasks(allClients.stream()
                .mapToInt(ClientInfo::getRunningTasks)
                .sum())
            .totalMaxConcurrency(allClients.stream()
                .mapToInt(ClientInfo::getMaxConcurrency)
                .sum())
            .build();
    }
}

/**
 * Client 统计信息
 */
@Data
@Builder
public class ClientStatistics {
    private int totalClients;
    private int availableClients;
    private int busyClients;
    private int offlineClients;
    private int totalRunningTasks;
    private int totalMaxConcurrency;
}
```

---

### 4.6 Client 端执行机架构

#### 4.6.1 Client 端架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Client 执行机架构                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Kafka Consumer                                │   │
│  │  - 订阅 task-client-commands (按 clientId 消费者组)                  │   │
│  │  - 接收任务命令                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Task Dispatcher                               │   │
│  │  - 任务解析与分发                                                    │   │
│  │  - 并发控制 (Semaphore)                                              │   │
│  │  - 任务队列管理                                                      │   │
│  │  - 优先级调度                                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                    ┌───────────────┴───────────────┐                       │
│                    ▼                               ▼                       │
│  ┌──────────────────────────┐    ┌──────────────────────────────────┐     │
│  │    AutomatedExecutor     │    │       AIAgentExecutor            │     │
│  │    (自动化脚本执行器)     │    │       (AI Agent 执行器)           │     │
│  │                          │    │                                  │     │
│  │  ┌────────────────────┐  │    │  ┌────────────────────────────┐  │     │
│  │  │ Python Executor    │  │    │  │ LLM Client                 │  │     │
│  │  │ Shell Executor     │  │    │  │ Agent Orchestrator         │  │     │
│  │  │ Robot Framework    │  │    │  │ Context Manager            │  │     │
│  │  │ Selenium/Playwright│  │    │  └────────────────────────────┘  │     │
│  │  └────────────────────┘  │    │                                  │     │
│  └──────────────────────────┘    └──────────────────────────────────┘     │
│                    │                               │                       │
│                    └───────────────┬───────────────┘                       │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Result Collector                              │   │
│  │  - 收集执行结果                                                      │   │
│  │  - 收集执行日志                                                      │   │
│  │  - 错误处理                                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Kafka Producer                                │   │
│  │  - 发送执行结果到 task-client-results                                │   │
│  │  - 发送状态事件到 task-status-events                                 │   │
│  │  - 发送心跳到 heartbeat-client                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        Heartbeat Sender                              │   │
│  │  - 定时发送心跳（默认每 10 秒）                                       │   │
│  │  - 上报状态、资源使用情况                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 4.6.2 Client 端核心类

```java
/**
 * Client 执行机启动器
 */
@SpringBootApplication
public class ClientExecutorApplication {

    public static void main(String[] args) {
        SpringApplication.run(ClientExecutorApplication.class, args);
    }
}

/**
 * Client 配置
 */
@Configuration
public class ClientConfig {

    @Value("${client.id}")
    private String clientId;

    @Value("${client.name}")
    private String clientName;

    @Value("${client.max-concurrency:5}")
    private int maxConcurrency;

    @Value("${client.heartbeat-interval:10000}")
    private long heartbeatInterval;

    // ... getters
}

/**
 * 任务分发器
 */
@Component
@Slf4j
public class TaskDispatcher {

    private final ClientConfig clientConfig;
    private final AutomatedTaskExecutor automatedExecutor;
    private final AIAgentTaskExecutor aiAgentExecutor;
    private final KafkaResultProducer resultProducer;
    private final StatusEventProducer statusEventProducer;

    // 并发控制信号量
    private final Semaphore semaphore;

    @Autowired
    public TaskDispatcher(ClientConfig clientConfig,
                          AutomatedTaskExecutor automatedExecutor,
                          AIAgentTaskExecutor aiAgentExecutor,
                          KafkaResultProducer resultProducer,
                          StatusEventProducer statusEventProducer) {
        this.clientConfig = clientConfig;
        this.automatedExecutor = automatedExecutor;
        this.aiAgentExecutor = aiAgentExecutor;
        this.resultProducer = resultProducer;
        this.statusEventProducer = statusEventProducer;
        this.semaphore = new Semaphore(clientConfig.getMaxConcurrency());
    }

    /**
     * 消费并处理任务
     */
    @KafkaListener(
        topics = "task-client-commands",
        groupId = "${client.id}-consumer-group",
        concurrency = "1"
    )
    public void dispatchTask(ConsumerRecord<String, String> record,
                              Acknowledgment acknowledgment) {
        log.info("收到任务: partition={}, offset={}",
                 record.partition(), record.offset());

        TaskCommandMessage command;
        try {
            command = parseTaskCommand(record.value());
        } catch (Exception e) {
            log.error("解析任务命令失败", e);
            acknowledgment.acknowledge();
            return;
        }

        // 异步处理任务
        CompletableFuture.runAsync(() -> processTask(command));

        // 立即提交偏移量（任务已入队）
        acknowledgment.acknowledge();
    }

    /**
     * 处理任务
     */
    private void processTask(TaskCommandMessage command) {
        TaskCommandPayload payload = command.getPayload();
        String taskId = payload.getTaskId();

        try {
            // 获取执行许可
            semaphore.acquire();

            // 发送状态事件：开始执行
            sendStatusEvent(taskId, payload, "NODE_STATUS_CHANGED",
                Map.of("previousStatus", "PENDING",
                       "currentStatus", "RUNNING",
                       "message", "开始执行任务"));

            // 执行任务
            TaskExecutionResult result = executeTask(payload);

            // 发送结果
            sendResult(taskId, payload, result);

            // 发送状态事件：执行完成
            sendStatusEvent(taskId, payload, "NODE_STATUS_CHANGED",
                Map.of("previousStatus", "RUNNING",
                       "currentStatus", result.isSuccess() ? "SUCCESS" : "FAILED",
                       "message", result.isSuccess() ? "执行成功" : "执行失败"));

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            sendResult(taskId, payload,
                TaskExecutionResult.failure("任务被中断"));

        } catch (Exception e) {
            log.error("任务执行异常: taskId={}", taskId, e);
            sendResult(taskId, payload,
                TaskExecutionResult.failure(e.getMessage()));

        } finally {
            semaphore.release();
        }
    }

    /**
     * 执行任务
     */
    private TaskExecutionResult executeTask(TaskCommandPayload payload) {
        String executionType = payload.getExecutionType();

        if ("AUTOMATED".equals(executionType)) {
            return automatedExecutor.execute(payload);
        } else if ("AI_AGENT".equals(executionType)) {
            return aiAgentExecutor.execute(payload);
        } else {
            return TaskExecutionResult.failure("不支持的执行类型: " + executionType);
        }
    }

    /**
     * 发送执行结果
     */
    private void sendResult(String taskId, TaskCommandPayload payload,
                            TaskExecutionResult result) {
        TaskResultMessage message = TaskResultMessage.builder()
            .messageId(UUID.randomUUID().toString())
            .messageType("TASK_RESULT")
            .timestamp(System.currentTimeMillis())
            .payload(TaskResultPayload.builder()
                .taskId(taskId)
                .executionId(payload.getExecutionId())
                .nodeUuid(payload.getNodeUuid())
                .clientId(clientConfig.getClientId())
                .status(result.isSuccess() ? "SUCCESS" : "FAILED")
                .outputs(result.getOutputs())
                .logs(result.getLogs())
                .durationMs(result.getDurationMs())
                .errorMessage(result.getErrorMessage())
                .errorStack(result.getErrorStack())
                .build())
            .build();

        resultProducer.sendResult(message);
    }

    /**
     * 发送状态事件
     */
    private void sendStatusEvent(String taskId, TaskCommandPayload payload,
                                  String eventType, Map<String, Object> eventData) {
        StatusEventMessage message = StatusEventMessage.builder()
            .messageId(UUID.randomUUID().toString())
            .messageType("STATUS_EVENT")
            .timestamp(System.currentTimeMillis())
            .payload(StatusEventPayload.builder()
                .taskId(taskId)
                .executionId(payload.getExecutionId())
                .nodeUuid(payload.getNodeUuid())
                .nodeName(payload.getSkillName())
                .clientId(clientConfig.getClientId())
                .eventType(eventType)
                .eventData(eventData)
                .build())
            .build();

        statusEventProducer.sendStatusEvent(message);
    }
}
```

#### 4.6.3 心跳发送器

```java
/**
 * 心跳发送器
 */
@Component
@Slf4j
public class HeartbeatSender {

    private final ClientConfig clientConfig;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;
    private final SystemMetricsCollector metricsCollector;

    private static final String HEARTBEAT_TOPIC = "heartbeat-client";

    /**
     * 定时发送心跳
     */
    @Scheduled(fixedRateString = "${client.heartbeat-interval:10000}")
    public void sendHeartbeat() {
        try {
            // 收集系统指标
            SystemMetrics metrics = metricsCollector.collect();

            HeartbeatMessage message = HeartbeatMessage.builder()
                .messageId(UUID.randomUUID().toString())
                .messageType("HEARTBEAT")
                .timestamp(System.currentTimeMillis())
                .payload(HeartbeatPayload.builder()
                    .clientId(clientConfig.getClientId())
                    .clientName(clientConfig.getClientName())
                    .status(determineStatus())
                    .runningTasks(getRunningTasks())
                    .maxConcurrency(clientConfig.getMaxConcurrency())
                    .cpuUsage(metrics.getCpuUsage())
                    .memoryUsage(metrics.getMemoryUsage())
                    .supportedExecutionTypes(getSupportedExecutionTypes())
                    .version(clientConfig.getVersion())
                    .labels(clientConfig.getLabels())
                    .build())
                .build();

            String messageJson = objectMapper.writeValueAsString(message);
            kafkaTemplate.send(HEARTBEAT_TOPIC, clientConfig.getClientId(), messageJson);

            log.debug("心跳已发送: clientId={}", clientConfig.getClientId());

        } catch (Exception e) {
            log.error("发送心跳失败", e);
        }
    }

    private String determineStatus() {
        int runningTasks = getRunningTasks();
        if (runningTasks == 0) {
            return "IDLE";
        } else if (runningTasks < clientConfig.getMaxConcurrency()) {
            return "BUSY";
        } else {
            return "BUSY";
        }
    }

    private int getRunningTasks() {
        // 获取当前正在执行的任务数
        // TODO: 实现任务计数
        return 0;
    }

    private List<String> getSupportedExecutionTypes() {
        return Arrays.asList("AUTOMATED", "AI_AGENT");
    }
}

/**
 * 系统指标收集器
 */
@Component
public class SystemMetricsCollector {

    public SystemMetrics collect() {
        Runtime runtime = Runtime.getRuntime();

        long maxMemory = runtime.maxMemory();
        long totalMemory = runtime.totalMemory();
        long freeMemory = runtime.freeMemory();
        long usedMemory = totalMemory - freeMemory;

        double memoryUsage = (double) usedMemory / maxMemory * 100;

        // CPU 使用率需要通过操作系统 MBean 获取
        double cpuUsage = getCpuUsage();

        return SystemMetrics.builder()
            .cpuUsage(cpuUsage)
            .memoryUsage(memoryUsage)
            .build();
    }

    private double getCpuUsage() {
        // 使用 OperatingSystemMXBean 获取 CPU 使用率
        try {
            com.sun.management.OperatingSystemMXBean osBean =
                (com.sun.management.OperatingSystemMXBean)
                    ManagementFactory.getOperatingSystemMXBean();
            return osBean.getProcessCpuLoad() * 100;
        } catch (Exception e) {
            return 0.0;
        }
    }
}
```

---

### 4.7 执行流程图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          分布式执行完整流程                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  1. 工作流触发执行                                                               │
│     │                                                                           │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Service 端                                                               │  │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  WorkflowScheduler.submitExecution()                                │  │  │
│  │  │  - 创建执行记录                                                     │  │  │
│  │  │  - 初始化执行上下文                                                 │  │  │
│  │  └────────────────────────────────────────────────────────────────────┘  │  │
│  │                                    │                                      │  │
│  │                                    ▼                                      │  │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  ExecutionEngine.execute()                                          │  │  │
│  │  │  - 构建执行图 (DAG)                                                 │  │  │
│  │  │  - 拓扑排序调度                                                     │  │  │
│  │  └────────────────────────────────────────────────────────────────────┘  │  │
│  │                                    │                                      │  │
│  │                                    ▼                                      │  │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  SkillNodeExecutor.execute() (Client 端执行)                        │  │  │
│  │  │  - 选择可用 Client                                                  │  │  │
│  │  │  - 构建任务命令                                                     │  │  │
│  │  │  - 发送到 Kafka                                                     │  │  │
│  │  └────────────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                    │                                           │
│                                    │ Kafka (task-client-commands)              │
│                                    ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Client 端                                                                │  │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  TaskDispatcher.dispatchTask()                                      │  │  │
│  │  │  - 消费任务命令                                                     │  │  │
│  │  │  - 获取执行许可 (并发控制)                                          │  │  │
│  │  │  - 分发到对应执行器                                                 │  │  │
│  │  └────────────────────────────────────────────────────────────────────┘  │  │
│  │                                    │                                      │  │
│  │          ┌─────────────────────────┴─────────────────────────┐          │  │
│  │          ▼                                                   ▼          │  │
│  │  ┌────────────────────┐                          ┌────────────────────┐  │  │
│  │  │ AutomatedExecutor  │                          │ AIAgentExecutor    │  │  │
│  │  │ - 执行自动化脚本    │                          │ - 执行 AI Agent    │  │  │
│  │  │ - 收集结果         │                          │ - 收集结果         │  │  │
│  │  └────────────────────┘                          └────────────────────┘  │  │
│  │          │                                                   │          │  │
│  │          └─────────────────────────┬─────────────────────────┘          │  │
│  │                                    ▼                                      │  │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  ResultCollector                                                    │  │  │
│  │  │  - 收集执行结果                                                     │  │  │
│  │  │  - 发送到 Kafka (task-client-results)                               │  │  │
│  │  │  - 发送状态事件 (task-status-events)                                │  │  │
│  │  └────────────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                    │                                           │
│                                    │ Kafka (task-client-results)               │
│                                    ▼                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  Service 端                                                               │  │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  KafkaResultConsumer.consumeTaskResult()                            │  │  │
│  │  │  - 消费执行结果                                                     │  │  │
│  │  │  - 完成 Future                                                      │  │  │
│  │  │  - 更新数据库状态                                                   │  │  │
│  │  │  - 推送 WebSocket 状态                                              │  │  │
│  │  └────────────────────────────────────────────────────────────────────┘  │  │
│  │                                    │                                      │  │
│  │                                    ▼                                      │  │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  StateManager.updateNodeStatus()                                    │  │  │
│  │  │  - 更新节点执行状态                                                 │  │  │
│  │  │  - 更新工作流进度                                                   │  │  │
│  │  └────────────────────────────────────────────────────────────────────┘  │  │
│  │                                    │                                      │  │
│  │                                    ▼                                      │  │
│  │  ┌────────────────────────────────────────────────────────────────────┐  │  │
│  │  │  WebSocketManager.pushNodeStatus()                                  │  │  │
│  │  │  - 推送节点状态变更到前端                                           │  │  │
│  │  └────────────────────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

*第四章完成，请确认后继续生成第五章。*
