# 工作流执行框架设计文档

## 第五章：实时状态推送

### 5.1 推送架构总览

实时状态推送模块通过 WebSocket 将工作流执行过程中的状态变化实时推送到前端，提供良好的用户体验。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     WebSocket 实时推送架构                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐                  │
│   │  浏览器客户端  │   │  浏览器客户端  │   │  浏览器客户端  │                  │
│   │  (用户A)      │   │  (用户B)      │   │  (用户C)      │                  │
│   └──────┬───────┘   └──────┬───────┘   └──────┬───────┘                  │
│          │                  │                  │                           │
│          └──────────────────┼──────────────────┘                           │
│                             │ WebSocket 连接                                │
│                             ▼                                               │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    WebSocket Server (Spring WebSocket)               │  │
│   │  ┌─────────────────────────────────────────────────────────────┐   │  │
│   │  │  WebSocketHandler                                             │   │  │
│   │  │  - 连接管理 (ConnectionManager)                                │   │  │
│   │  │  - 会话管理 (SessionManager)                                   │   │  │
│   │  │  - 订阅管理 (SubscriptionManager)                              │   │  │
│   │  └─────────────────────────────────────────────────────────────┘   │  │
│   │                             │                                       │  │
│   │                             ▼                                       │  │
│   │  ┌─────────────────────────────────────────────────────────────┐   │  │
│   │  │  StatusEventPublisher                                         │   │  │
│   │  │  - 接收执行引擎状态变更                                        │   │  │
│   │  │  - 接收 Kafka 状态事件                                         │   │  │
│   │  │  - 转换为 WebSocket 消息推送                                   │   │  │
│   │  └─────────────────────────────────────────────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                    状态事件来源                                      │  │
│   │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │  │
│   │  │ExecutionEngine│  │ StatusEvent   │  │ NodeExecutor          │   │  │
│   │  │ (引擎状态变更) │  │ KafkaConsumer │  │ (节点执行状态)        │   │  │
│   │  └───────────────┘  └───────────────┘  └───────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 5.2 推送消息类型定义

#### 5.2.1 消息类型枚举

```java
/**
 * WebSocket 消息类型
 */
public enum WebSocketMessageType {

    // 连接相关
    CONNECTED("连接成功"),
    PONG("心跳响应"),

    // 订阅相关
    SUBSCRIBE_SUCCESS("订阅成功"),
    UNSUBSCRIBE_SUCCESS("取消订阅成功"),
    SUBSCRIBE_ERROR("订阅失败"),

    // 工作流状态
    WORKFLOW_STATUS_CHANGED("工作流状态变更"),
    WORKFLOW_EXECUTION_COMPLETED("工作流执行完成"),
    WORKFLOW_PROGRESS_UPDATED("工作流进度更新"),

    // 节点状态
    NODE_STATUS_CHANGED("节点状态变更"),
    NODE_EXECUTION_COMPLETED("节点执行完成"),
    NODE_EXECUTION_LOG("节点执行日志"),

    // 异步任务
    ASYNC_TASK_STATUS("异步任务状态"),

    // 错误
    ERROR("错误消息");

    private final String description;

    WebSocketMessageType(String description) {
        this.description = description;
    }

    public String getDescription() {
        return description;
    }
}
```

#### 5.2.2 消息结构定义

```java
/**
 * WebSocket 消息基础结构
 */
@Data
@Builder
public class WebSocketMessage<T> {

    /** 消息类型 */
    private String type;

    /** 时间戳 */
    private Long timestamp;

    /** 消息体 */
    private T payload;

    public static <T> WebSocketMessage<T> of(String type, T payload) {
        return WebSocketMessage.<T>builder()
            .type(type)
            .timestamp(System.currentTimeMillis())
            .payload(payload)
            .build();
    }
}
```

#### 5.2.3 各类型消息载荷定义

##### 5.2.3.1 工作流状态变更

```java
/**
 * 工作流状态变更载荷
 */
@Data
@Builder
public class WorkflowStatusPayload {

    /** 执行记录ID */
    private Long executionId;

    /** 执行UUID */
    private String executionUuid;

    /** 工作流ID */
    private Long workflowId;

    /** 工作流名称 */
    private String workflowName;

    /** 之前状态 */
    private String previousStatus;

    /** 当前状态 */
    private String currentStatus;

    /** 执行进度 (0-100) */
    private Integer progress;

    /** 触发人 */
    private String triggeredBy;
}
```

**消息示例：**

```json
{
  "type": "WORKFLOW_STATUS_CHANGED",
  "timestamp": 1709234567890,
  "payload": {
    "executionId": 12345,
    "executionUuid": "exec-uuid-001",
    "workflowId": 100,
    "workflowName": "AI应用测试工作流",
    "previousStatus": "PENDING",
    "currentStatus": "RUNNING",
    "progress": 0,
    "triggeredBy": "zhangsan"
  }
}
```

##### 5.2.3.2 节点状态变更

```java
/**
 * 节点状态变更载荷
 */
@Data
@Builder
public class NodeStatusPayload {

    /** 执行记录ID */
    private Long executionId;

    /** 执行UUID */
    private String executionUuid;

    /** 节点UUID */
    private String nodeUuid;

    /** 节点名称 */
    private String nodeName;

    /** 节点类型 */
    private String nodeType;

    /** 之前状态 */
    private String previousStatus;

    /** 当前状态 */
    private String currentStatus;

    /** 开始时间 */
    private Long startTime;

    /** 执行进度 (0-100) */
    private Integer progress;
}
```

**消息示例：**

```json
{
  "type": "NODE_STATUS_CHANGED",
  "timestamp": 1709234568123,
  "payload": {
    "executionId": 12345,
    "executionUuid": "exec-uuid-001",
    "nodeUuid": "skill-node-1",
    "nodeName": "文本清洗",
    "nodeType": "skill",
    "previousStatus": "PENDING",
    "currentStatus": "RUNNING",
    "startTime": 1709234568123,
    "progress": 0
  }
}
```

##### 5.2.3.3 节点执行完成

```java
/**
 * 节点执行完成载荷
 */
@Data
@Builder
public class NodeCompletedPayload {

    /** 执行记录ID */
    private Long executionId;

    /** 执行UUID */
    private String executionUuid;

    /** 节点UUID */
    private String nodeUuid;

    /** 节点名称 */
    private String nodeName;

    /** 节点类型 */
    private String nodeType;

    /** 执行状态 */
    private String status;

    /** 执行耗时（毫秒） */
    private Long durationMs;

    /** 输出预览（仅包含关键字段） */
    private Map<String, Object> outputPreview;

    /** 错误信息（失败时） */
    private String errorMessage;
}
```

**消息示例：**

```json
{
  "type": "NODE_EXECUTION_COMPLETED",
  "timestamp": 1709234570623,
  "payload": {
    "executionId": 12345,
    "executionUuid": "exec-uuid-001",
    "nodeUuid": "skill-node-1",
    "nodeName": "文本清洗",
    "nodeType": "skill",
    "status": "SUCCESS",
    "durationMs": 2500,
    "outputPreview": {
      "output_file": "/data/cleaned.xlsx"
    },
    "errorMessage": null
  }
}
```

##### 5.2.3.4 节点执行日志

```java
/**
 * 节点执行日志载荷
 */
@Data
@Builder
public class NodeLogPayload {

    /** 执行记录ID */
    private Long executionId;

    /** 执行UUID */
    private String executionUuid;

    /** 节点UUID */
    private String nodeUuid;

    /** 节点名称 */
    private String nodeName;

    /** 日志级别: DEBUG / INFO / WARN / ERROR */
    private String logLevel;

    /** 日志消息 */
    private String message;

    /** 执行进度 (0-100) */
    private Integer progress;

    /** 时间戳 */
    private Long timestamp;
}
```

**消息示例：**

```json
{
  "type": "NODE_EXECUTION_LOG",
  "timestamp": 1709234569000,
  "payload": {
    "executionId": 12345,
    "executionUuid": "exec-uuid-001",
    "nodeUuid": "skill-node-1",
    "nodeName": "文本清洗",
    "logLevel": "INFO",
    "message": "开始处理文件: /data/input.xlsx",
    "progress": 25,
    "timestamp": 1709234569000
  }
}
```

##### 5.2.3.5 工作流执行完成

```java
/**
 * 工作流执行完成载荷
 */
@Data
@Builder
public class WorkflowCompletedPayload {

    /** 执行记录ID */
    private Long executionId;

    /** 执行UUID */
    private String executionUuid;

    /** 工作流ID */
    private Long workflowId;

    /** 工作流名称 */
    private String workflowName;

    /** 执行状态 */
    private String status;

    /** 执行进度 */
    private Integer progress;

    /** 开始时间 */
    private Long startTime;

    /** 结束时间 */
    private Long endTime;

    /** 执行耗时（毫秒） */
    private Long durationMs;

    /** 节点统计 */
    private NodeSummary nodeSummary;

    /** 输出预览 */
    private Map<String, Object> outputPreview;

    /**
     * 节点统计内部类
     */
    @Data
    @Builder
    public static class NodeSummary {
        private int total;
        private int success;
        private int failed;
        private int skipped;
    }
}
```

**消息示例：**

```json
{
  "type": "WORKFLOW_EXECUTION_COMPLETED",
  "timestamp": 1709234590000,
  "payload": {
    "executionId": 12345,
    "executionUuid": "exec-uuid-001",
    "workflowId": 100,
    "workflowName": "AI应用测试工作流",
    "status": "SUCCESS",
    "progress": 100,
    "startTime": 1709234567890,
    "endTime": 1709234590000,
    "durationMs": 22110,
    "nodeSummary": {
      "total": 5,
      "success": 5,
      "failed": 0,
      "skipped": 0
    },
    "outputPreview": {
      "final_result": "/data/report.xlsx"
    }
  }
}
```

##### 5.2.3.6 异步任务状态

```java
/**
 * 异步任务状态载荷
 */
@Data
@Builder
public class AsyncTaskStatusPayload {

    /** 执行记录ID */
    private Long executionId;

    /** 执行UUID */
    private String executionUuid;

    /** 异步节点UUID */
    private String asyncNodeUuid;

    /** 异步节点名称 */
    private String asyncNodeName;

    /** 任务统计信息 */
    private TaskInfo taskInfo;

    /**
     * 任务统计信息
     */
    @Data
    @Builder
    public static class TaskInfo {
        private int totalTasks;
        private int completedTasks;
        private int runningTasks;
        private int pendingTasks;
        private int failedTasks;
    }
}
```

**消息示例：**

```json
{
  "type": "ASYNC_TASK_STATUS",
  "timestamp": 1709234570000,
  "payload": {
    "executionId": 12345,
    "executionUuid": "exec-uuid-001",
    "asyncNodeUuid": "async-node-1",
    "asyncNodeName": "批量处理",
    "taskInfo": {
      "totalTasks": 10,
      "completedTasks": 3,
      "runningTasks": 2,
      "pendingTasks": 5,
      "failedTasks": 0
    }
  }
}
```

---

### 5.3 WebSocket 服务端实现

#### 5.3.1 WebSocket 配置

```java
/**
 * WebSocket 配置
 */
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    private final WorkflowWebSocketHandler webSocketHandler;
    private final WebSocketInterceptor webSocketInterceptor;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(webSocketHandler, "/ws/workflow")
            .addInterceptors(webSocketInterceptor)
            .setAllowedOrigins("*");
    }
}
```

#### 5.3.2 WebSocket 处理器

```java
/**
 * 工作流 WebSocket 处理器
 */
@Component
@Slf4j
public class WorkflowWebSocketHandler extends TextWebSocketHandler {

    private final ObjectMapper objectMapper;
    private final SubscriptionManager subscriptionManager;

    /**
     * WebSocket 会话映射
     * key: sessionId
     * value: WebSocketSession
     */
    private final ConcurrentHashMap<String, WebSocketSession> sessions =
        new ConcurrentHashMap<>();

    /**
     * 用户会话映射
     * key: userId
     * value: Set<sessionId>
     */
    private final ConcurrentHashMap<String, Set<String>> userSessions =
        new ConcurrentHashMap<>();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        String sessionId = session.getId();
        String userId = getUserId(session);

        sessions.put(sessionId, session);

        // 记录用户会话
        userSessions.computeIfAbsent(userId, k -> ConcurrentHashMap.newKeySet())
            .add(sessionId);

        log.info("WebSocket 连接建立: sessionId={}, userId={}", sessionId, userId);

        // 发送连接成功消息
        sendMessage(session, WebSocketMessage.of("CONNECTED",
            Map.of("sessionId", sessionId)));
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        String sessionId = session.getId();
        String userId = getUserId(session);

        sessions.remove(sessionId);

        // 移除用户会话记录
        Set<String> userSessionSet = userSessions.get(userId);
        if (userSessionSet != null) {
            userSessionSet.remove(sessionId);
            if (userSessionSet.isEmpty()) {
                userSessions.remove(userId);
            }
        }

        // 清理订阅
        subscriptionManager.clearSubscriptions(sessionId);

        log.info("WebSocket 连接关闭: sessionId={}, userId={}, status={}",
                 sessionId, userId, status);
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        try {
            WebSocketMessage<?> wsMessage = objectMapper.readValue(
                message.getPayload(),
                new TypeReference<WebSocketMessage<?>>() {}
            );

            switch (wsMessage.getType()) {
                case "PING":
                    handlePing(session);
                    break;

                case "SUBSCRIBE_EXECUTION":
                    handleSubscribeExecution(session, wsMessage);
                    break;

                case "UNSUBSCRIBE_EXECUTION":
                    handleUnsubscribeExecution(session, wsMessage);
                    break;

                case "SUBSCRIBE_WORKFLOW":
                    handleSubscribeWorkflow(session, wsMessage);
                    break;

                default:
                    log.warn("未知消息类型: type={}", wsMessage.getType());
            }

        } catch (Exception e) {
            log.error("处理 WebSocket 消息失败: sessionId={}", session.getId(), e);
            sendError(session, "消息处理失败: " + e.getMessage());
        }
    }

    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) {
        log.error("WebSocket 传输错误: sessionId={}", session.getId(), exception);
    }

    /**
     * 处理心跳
     */
    private void handlePing(WebSocketSession session) {
        sendMessage(session, WebSocketMessage.of("PONG", Map.of("timestamp",
            System.currentTimeMillis())));
    }

    /**
     * 处理执行订阅
     */
    private void handleSubscribeExecution(WebSocketSession session,
                                           WebSocketMessage<?> message) {
        @SuppressWarnings("unchecked")
        Map<String, Object> payload = (Map<String, Object>) message.getPayload();
        String executionUuid = (String) payload.get("executionUuid");

        subscriptionManager.subscribe(session.getId(), executionUuid);

        sendMessage(session, WebSocketMessage.of("SUBSCRIBE_SUCCESS",
            Map.of("executionUuid", executionUuid)));

        log.info("订阅执行: sessionId={}, executionUuid={}",
                 session.getId(), executionUuid);
    }

    /**
     * 处理取消订阅
     */
    private void handleUnsubscribeExecution(WebSocketSession session,
                                             WebSocketMessage<?> message) {
        @SuppressWarnings("unchecked")
        Map<String, Object> payload = (Map<String, Object>) message.getPayload();
        String executionUuid = (String) payload.get("executionUuid");

        subscriptionManager.unsubscribe(session.getId(), executionUuid);

        sendMessage(session, WebSocketMessage.of("UNSUBSCRIBE_SUCCESS",
            Map.of("executionUuid", executionUuid)));

        log.info("取消订阅: sessionId={}, executionUuid={}",
                 session.getId(), executionUuid);
    }

    /**
     * 处理工作流订阅（订阅该工作流的所有执行）
     */
    private void handleSubscribeWorkflow(WebSocketSession session,
                                          WebSocketMessage<?> message) {
        @SuppressWarnings("unchecked")
        Map<String, Object> payload = (Map<String, Object>) message.getPayload();
        Long workflowId = Long.valueOf(payload.get("workflowId").toString());

        subscriptionManager.subscribeWorkflow(session.getId(), workflowId);

        sendMessage(session, WebSocketMessage.of("SUBSCRIBE_SUCCESS",
            Map.of("workflowId", workflowId)));

        log.info("订阅工作流: sessionId={}, workflowId={}",
                 session.getId(), workflowId);
    }

    /**
     * 向指定会话发送消息
     */
    public void sendMessage(WebSocketSession session, WebSocketMessage<?> message) {
        if (session == null || !session.isOpen()) {
            return;
        }

        try {
            String json = objectMapper.writeValueAsString(message);
            session.sendMessage(new TextMessage(json));
        } catch (Exception e) {
            log.error("发送 WebSocket 消息失败: sessionId={}", session.getId(), e);
        }
    }

    /**
     * 向指定用户发送消息
     */
    public void sendMessageToUser(String userId, WebSocketMessage<?> message) {
        Set<String> sessionIds = userSessions.get(userId);
        if (sessionIds != null) {
            for (String sessionId : sessionIds) {
                WebSocketSession session = sessions.get(sessionId);
                sendMessage(session, message);
            }
        }
    }

    /**
     * 广播消息到订阅了指定执行的所有会话
     */
    public void broadcastToExecution(String executionUuid, WebSocketMessage<?> message) {
        Set<String> sessionIds = subscriptionManager.getSubscribers(executionUuid);
        if (sessionIds != null) {
            for (String sessionId : sessionIds) {
                WebSocketSession session = sessions.get(sessionId);
                sendMessage(session, message);
            }
        }
    }

    /**
     * 广播消息到订阅了指定工作流的所有会话
     */
    public void broadcastToWorkflow(Long workflowId, WebSocketMessage<?> message) {
        Set<String> sessionIds = subscriptionManager.getWorkflowSubscribers(workflowId);
        if (sessionIds != null) {
            for (String sessionId : sessionIds) {
                WebSocketSession session = sessions.get(sessionId);
                sendMessage(session, message);
            }
        }
    }

    /**
     * 发送错误消息
     */
    private void sendError(WebSocketSession session, String errorMessage) {
        sendMessage(session, WebSocketMessage.of("ERROR",
            Map.of("message", errorMessage)));
    }

    /**
     * 获取用户ID
     */
    private String getUserId(WebSocketSession session) {
        // 从 session 属性中获取用户ID（由拦截器设置）
        return (String) session.getAttributes().get("userId");
    }
}
```

#### 5.3.3 订阅管理器

```java
/**
 * 订阅管理器
 * 管理 WebSocket 会话的订阅关系
 */
@Component
@Slf4j
public class SubscriptionManager {

    /**
     * 执行订阅映射
     * key: executionUuid
     * value: Set<sessionId>
     */
    private final ConcurrentHashMap<String, Set<String>> executionSubscriptions =
        new ConcurrentHashMap<>();

    /**
     * 工作流订阅映射
     * key: workflowId
     * value: Set<sessionId>
     */
    private final ConcurrentHashMap<Long, Set<String>> workflowSubscriptions =
        new ConcurrentHashMap<>();

    /**
     * 会话订阅映射（反向索引）
     * key: sessionId
     * value: 订阅信息
     */
    private final ConcurrentHashMap<String, SessionSubscription> sessionSubscriptions =
        new ConcurrentHashMap<>();

    /**
     * 订阅执行
     */
    public void subscribe(String sessionId, String executionUuid) {
        executionSubscriptions
            .computeIfAbsent(executionUuid, k -> ConcurrentHashMap.newKeySet())
            .add(sessionId);

        // 更新会话订阅索引
        sessionSubscriptions
            .computeIfAbsent(sessionId, k -> new SessionSubscription())
            .addExecution(executionUuid);
    }

    /**
     * 取消订阅执行
     */
    public void unsubscribe(String sessionId, String executionUuid) {
        Set<String> subscribers = executionSubscriptions.get(executionUuid);
        if (subscribers != null) {
            subscribers.remove(sessionId);
            if (subscribers.isEmpty()) {
                executionSubscriptions.remove(executionUuid);
            }
        }

        SessionSubscription subscription = sessionSubscriptions.get(sessionId);
        if (subscription != null) {
            subscription.removeExecution(executionUuid);
        }
    }

    /**
     * 订阅工作流
     */
    public void subscribeWorkflow(String sessionId, Long workflowId) {
        workflowSubscriptions
            .computeIfAbsent(workflowId, k -> ConcurrentHashMap.newKeySet())
            .add(sessionId);

        sessionSubscriptions
            .computeIfAbsent(sessionId, k -> new SessionSubscription())
            .addWorkflow(workflowId);
    }

    /**
     * 获取执行的订阅者
     */
    public Set<String> getSubscribers(String executionUuid) {
        return executionSubscriptions.get(executionUuid);
    }

    /**
     * 获取工作流的订阅者
     */
    public Set<String> getWorkflowSubscribers(Long workflowId) {
        return workflowSubscriptions.get(workflowId);
    }

    /**
     * 清理会话的所有订阅
     */
    public void clearSubscriptions(String sessionId) {
        SessionSubscription subscription = sessionSubscriptions.remove(sessionId);
        if (subscription != null) {
            // 清理执行订阅
            for (String executionUuid : subscription.getExecutions()) {
                Set<String> subscribers = executionSubscriptions.get(executionUuid);
                if (subscribers != null) {
                    subscribers.remove(sessionId);
                    if (subscribers.isEmpty()) {
                        executionSubscriptions.remove(executionUuid);
                    }
                }
            }

            // 清理工作流订阅
            for (Long workflowId : subscription.getWorkflows()) {
                Set<String> subscribers = workflowSubscriptions.get(workflowId);
                if (subscribers != null) {
                    subscribers.remove(sessionId);
                    if (subscribers.isEmpty()) {
                        workflowSubscriptions.remove(workflowId);
                    }
                }
            }
        }
    }

    /**
     * 会话订阅信息
     */
    @Data
    private static class SessionSubscription {
        private final Set<String> executions = ConcurrentHashMap.newKeySet();
        private final Set<Long> workflows = ConcurrentHashMap.newKeySet();

        public void addExecution(String executionUuid) {
            executions.add(executionUuid);
        }

        public void removeExecution(String executionUuid) {
            executions.remove(executionUuid);
        }

        public void addWorkflow(Long workflowId) {
            workflows.add(workflowId);
        }
    }
}
```

#### 5.3.4 WebSocket 拦截器

```java
/**
 * WebSocket 拦截器
 * 用于认证和提取用户信息
 */
@Component
public class WebSocketInterceptor implements HandshakeInterceptor {

    private final JwtTokenProvider jwtTokenProvider;

    @Override
    public boolean beforeHandshake(ServerHttpRequest request,
                                    ServerHttpResponse response,
                                    WebSocketHandler wsHandler,
                                    Map<String, Object> attributes) {

        // 从请求中获取 token
        String token = extractToken(request);

        if (token == null) {
            log.warn("WebSocket 握手失败: 缺少认证 token");
            return false;
        }

        try {
            // 验证 token 并获取用户信息
            String userId = jwtTokenProvider.getUserIdFromToken(token);

            if (userId == null) {
                log.warn("WebSocket 握手失败: 无效的 token");
                return false;
            }

            // 将用户信息存入 session 属性
            attributes.put("userId", userId);
            attributes.put("token", token);

            log.info("WebSocket 握手成功: userId={}", userId);
            return true;

        } catch (Exception e) {
            log.error("WebSocket 握手失败: token 验证异常", e);
            return false;
        }
    }

    @Override
    public void afterHandshake(ServerHttpRequest request,
                                ServerHttpResponse response,
                                WebSocketHandler wsHandler,
                                Exception exception) {
        // 握手后处理（可选）
    }

    /**
     * 从请求中提取 token
     */
    private String extractToken(ServerHttpRequest request) {
        // 从 URL 参数中获取
        String query = request.getURI().getQuery();
        if (query != null) {
            for (String param : query.split("&")) {
                String[] pair = param.split("=");
                if (pair.length == 2 && "token".equals(pair[0])) {
                    return pair[1];
                }
            }
        }

        // 从 Header 中获取
        List<String> authHeaders = request.getHeaders().get("Authorization");
        if (authHeaders != null && !authHeaders.isEmpty()) {
            String authHeader = authHeaders.get(0);
            if (authHeader.startsWith("Bearer ")) {
                return authHeader.substring(7);
            }
        }

        return null;
    }
}
```

---

### 5.4 状态事件发布服务

#### 5.4.1 状态事件发布器

```java
/**
 * 状态事件发布器
 * 统一管理状态变更的 WebSocket 推送
 */
@Service
@Slf4j
public class StatusEventPublisher {

    private final WorkflowWebSocketHandler webSocketHandler;

    public StatusEventPublisher(WorkflowWebSocketHandler webSocketHandler) {
        this.webSocketHandler = webSocketHandler;
    }

    /**
     * 发布工作流状态变更
     */
    public void publishWorkflowStatusChanged(Long executionId,
                                              String executionUuid,
                                              Long workflowId,
                                              String workflowName,
                                              String previousStatus,
                                              String currentStatus,
                                              Integer progress) {

        WorkflowStatusPayload payload = WorkflowStatusPayload.builder()
            .executionId(executionId)
            .executionUuid(executionUuid)
            .workflowId(workflowId)
            .workflowName(workflowName)
            .previousStatus(previousStatus)
            .currentStatus(currentStatus)
            .progress(progress)
            .build();

        WebSocketMessage<WorkflowStatusPayload> message = WebSocketMessage.<WorkflowStatusPayload>builder()
            .type("WORKFLOW_STATUS_CHANGED")
            .timestamp(System.currentTimeMillis())
            .payload(payload)
            .build();

        webSocketHandler.broadcastToExecution(executionUuid, message);

        log.debug("发布工作流状态变更: executionUuid={}, status={}",
                  executionUuid, currentStatus);
    }

    /**
     * 发布节点状态变更
     */
    public void publishNodeStatusChanged(Long executionId,
                                          String executionUuid,
                                          String nodeUuid,
                                          String nodeName,
                                          String nodeType,
                                          String previousStatus,
                                          String currentStatus,
                                          Integer progress) {

        NodeStatusPayload payload = NodeStatusPayload.builder()
            .executionId(executionId)
            .executionUuid(executionUuid)
            .nodeUuid(nodeUuid)
            .nodeName(nodeName)
            .nodeType(nodeType)
            .previousStatus(previousStatus)
            .currentStatus(currentStatus)
            .progress(progress)
            .startTime(currentStatus.equals("RUNNING") ?
                       System.currentTimeMillis() : null)
            .build();

        WebSocketMessage<NodeStatusPayload> message = WebSocketMessage.<NodeStatusPayload>builder()
            .type("NODE_STATUS_CHANGED")
            .timestamp(System.currentTimeMillis())
            .payload(payload)
            .build();

        webSocketHandler.broadcastToExecution(executionUuid, message);

        log.debug("发布节点状态变更: executionUuid={}, nodeUuid={}, status={}",
                  executionUuid, nodeUuid, currentStatus);
    }

    /**
     * 发布节点执行完成
     */
    public void publishNodeCompleted(Long executionId,
                                      String executionUuid,
                                      String nodeUuid,
                                      String nodeName,
                                      String nodeType,
                                      String status,
                                      Long durationMs,
                                      Map<String, Object> outputPreview,
                                      String errorMessage) {

        NodeCompletedPayload payload = NodeCompletedPayload.builder()
            .executionId(executionId)
            .executionUuid(executionUuid)
            .nodeUuid(nodeUuid)
            .nodeName(nodeName)
            .nodeType(nodeType)
            .status(status)
            .durationMs(durationMs)
            .outputPreview(outputPreview)
            .errorMessage(errorMessage)
            .build();

        WebSocketMessage<NodeCompletedPayload> message = WebSocketMessage.<NodeCompletedPayload>builder()
            .type("NODE_EXECUTION_COMPLETED")
            .timestamp(System.currentTimeMillis())
            .payload(payload)
            .build();

        webSocketHandler.broadcastToExecution(executionUuid, message);

        log.debug("发布节点执行完成: executionUuid={}, nodeUuid={}, status={}",
                  executionUuid, nodeUuid, status);
    }

    /**
     * 发布节点执行日志
     */
    public void publishNodeLog(Long executionId,
                                String executionUuid,
                                String nodeUuid,
                                String nodeName,
                                String logLevel,
                                String message,
                                Integer progress) {

        NodeLogPayload payload = NodeLogPayload.builder()
            .executionId(executionId)
            .executionUuid(executionUuid)
            .nodeUuid(nodeUuid)
            .nodeName(nodeName)
            .logLevel(logLevel)
            .message(message)
            .progress(progress)
            .timestamp(System.currentTimeMillis())
            .build();

        WebSocketMessage<NodeLogPayload> wsMessage = WebSocketMessage.<NodeLogPayload>builder()
            .type("NODE_EXECUTION_LOG")
            .timestamp(System.currentTimeMillis())
            .payload(payload)
            .build();

        webSocketHandler.broadcastToExecution(executionUuid, wsMessage);
    }

    /**
     * 发布工作流执行完成
     */
    public void publishWorkflowCompleted(Long executionId,
                                          String executionUuid,
                                          Long workflowId,
                                          String workflowName,
                                          String status,
                                          Integer progress,
                                          Long startTime,
                                          Long endTime,
                                          Long durationMs,
                                          int totalNodes,
                                          int successNodes,
                                          int failedNodes,
                                          int skippedNodes,
                                          Map<String, Object> outputPreview) {

        WorkflowCompletedPayload.NodeSummary nodeSummary =
            WorkflowCompletedPayload.NodeSummary.builder()
                .total(totalNodes)
                .success(successNodes)
                .failed(failedNodes)
                .skipped(skippedNodes)
                .build();

        WorkflowCompletedPayload payload = WorkflowCompletedPayload.builder()
            .executionId(executionId)
            .executionUuid(executionUuid)
            .workflowId(workflowId)
            .workflowName(workflowName)
            .status(status)
            .progress(progress)
            .startTime(startTime)
            .endTime(endTime)
            .durationMs(durationMs)
            .nodeSummary(nodeSummary)
            .outputPreview(outputPreview)
            .build();

        WebSocketMessage<WorkflowCompletedPayload> message =
            WebSocketMessage.<WorkflowCompletedPayload>builder()
                .type("WORKFLOW_EXECUTION_COMPLETED")
                .timestamp(System.currentTimeMillis())
                .payload(payload)
                .build();

        webSocketHandler.broadcastToExecution(executionUuid, message);

        log.info("发布工作流执行完成: executionUuid={}, status={}",
                 executionUuid, status);
    }

    /**
     * 发布异步任务状态
     */
    public void publishAsyncTaskStatus(String executionUuid,
                                        String asyncNodeUuid,
                                        String asyncNodeName,
                                        int totalTasks,
                                        int completedTasks,
                                        int runningTasks,
                                        int pendingTasks,
                                        int failedTasks) {

        AsyncTaskStatusPayload.TaskInfo taskInfo =
            AsyncTaskStatusPayload.TaskInfo.builder()
                .totalTasks(totalTasks)
                .completedTasks(completedTasks)
                .runningTasks(runningTasks)
                .pendingTasks(pendingTasks)
                .failedTasks(failedTasks)
                .build();

        AsyncTaskStatusPayload payload = AsyncTaskStatusPayload.builder()
            .executionUuid(executionUuid)
            .asyncNodeUuid(asyncNodeUuid)
            .asyncNodeName(asyncNodeName)
            .taskInfo(taskInfo)
            .build();

        WebSocketMessage<AsyncTaskStatusPayload> message =
            WebSocketMessage.<AsyncTaskStatusPayload>builder()
                .type("ASYNC_TASK_STATUS")
                .timestamp(System.currentTimeMillis())
                .payload(payload)
                .build();

        webSocketHandler.broadcastToExecution(executionUuid, message);
    }
}
```

#### 5.4.2 与执行引擎集成

```java
/**
 * 执行引擎扩展 - 集成状态推送
 */
@Service
public class ExecutionEngineWithPush extends ExecutionEngine {

    private final StatusEventPublisher statusEventPublisher;

    /**
     * 执行单个节点（重写，添加状态推送）
     */
    @Override
    protected void executeNode(WorkflowNode node, ExecutionContext context) {
        String nodeUuid = node.getUuid();
        String nodeName = node.getName();
        String nodeType = node.getType();
        String executionUuid = context.getExecutionUuid();
        Long executionId = context.getExecutionId();

        // 发布节点状态变更：RUNNING
        statusEventPublisher.publishNodeStatusChanged(
            executionId, executionUuid, nodeUuid, nodeName, nodeType,
            "PENDING", "RUNNING", 0
        );

        long startTime = System.currentTimeMillis();

        try {
            // 执行节点（调用父类方法）
            super.executeNode(node, context);

            long durationMs = System.currentTimeMillis() - startTime;

            // 发布节点执行完成
            statusEventPublisher.publishNodeCompleted(
                executionId, executionUuid, nodeUuid, nodeName, nodeType,
                "SUCCESS", durationMs,
                context.getNodeOutputs(nodeUuid),
                null
            );

        } catch (Exception e) {
            long durationMs = System.currentTimeMillis() - startTime;

            // 发布节点执行完成（失败）
            statusEventPublisher.publishNodeCompleted(
                executionId, executionUuid, nodeUuid, nodeName, nodeType,
                "FAILED", durationMs,
                null,
                e.getMessage()
            );

            throw e;
        }
    }
}
```

---

### 5.5 前端 WebSocket 客户端

#### 5.5.1 客户端连接管理

```typescript
/**
 * WebSocket 客户端管理类
 */
export class WorkflowWebSocketClient {

  private ws: WebSocket | null = null;
  private url: string;
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;
  private reconnectInterval = 3000;
  private handlers: Map<string, Set<(payload: any) => void>> = new Map();
  private pingInterval: NodeJS.Timeout | null = null;

  constructor(baseUrl: string, token: string) {
    this.url = `${baseUrl}/ws/workflow?token=${token}`;
  }

  /**
   * 连接 WebSocket
   */
  connect(): Promise<void> {
    return new Promise((resolve, reject) => {
      try {
        this.ws = new WebSocket(this.url);

        this.ws.onopen = () => {
          console.log('WebSocket 连接成功');
          this.reconnectAttempts = 0;
          this.startPing();
          resolve();
        };

        this.ws.onmessage = (event) => {
          this.handleMessage(event.data);
        };

        this.ws.onerror = (error) => {
          console.error('WebSocket 错误:', error);
          reject(error);
        };

        this.ws.onclose = () => {
          console.log('WebSocket 连接关闭');
          this.stopPing();
          this.attemptReconnect();
        };

      } catch (error) {
        reject(error);
      }
    });
  }

  /**
   * 断开连接
   */
  disconnect() {
    this.stopPing();
    if (this.ws) {
      this.ws.close();
      this.ws = null;
    }
  }

  /**
   * 订阅执行状态
   */
  subscribeExecution(executionUuid: string) {
    this.send({
      type: 'SUBSCRIBE_EXECUTION',
      payload: { executionUuid }
    });
  }

  /**
   * 取消订阅
   */
  unsubscribeExecution(executionUuid: string) {
    this.send({
      type: 'UNSUBSCRIBE_EXECUTION',
      payload: { executionUuid }
    });
  }

  /**
   * 注册消息处理器
   */
  on(messageType: string, handler: (payload: any) => void) {
    if (!this.handlers.has(messageType)) {
      this.handlers.set(messageType, new Set());
    }
    this.handlers.get(messageType)!.add(handler);
  }

  /**
   * 移除消息处理器
   */
  off(messageType: string, handler: (payload: any) => void) {
    const handlers = this.handlers.get(messageType);
    if (handlers) {
      handlers.delete(handler);
    }
  }

  /**
   * 发送消息
   */
  private send(message: any) {
    if (this.ws && this.ws.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify(message));
    }
  }

  /**
   * 处理接收的消息
   */
  private handleMessage(data: string) {
    try {
      const message = JSON.parse(data);
      const handlers = this.handlers.get(message.type);

      if (handlers) {
        handlers.forEach(handler => handler(message.payload));
      }

      // 触发通配符处理器
      const wildcardHandlers = this.handlers.get('*');
      if (wildcardHandlers) {
        wildcardHandlers.forEach(handler => handler(message));
      }

    } catch (error) {
      console.error('解析 WebSocket 消息失败:', error);
    }
  }

  /**
   * 开始心跳
   */
  private startPing() {
    this.pingInterval = setInterval(() => {
      this.send({ type: 'PING', payload: {} });
    }, 30000);
  }

  /**
   * 停止心跳
   */
  private stopPing() {
    if (this.pingInterval) {
      clearInterval(this.pingInterval);
      this.pingInterval = null;
    }
  }

  /**
   * 尝试重连
   */
  private attemptReconnect() {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.reconnectAttempts++;
      console.log(`尝试重连 (${this.reconnectAttempts}/${this.maxReconnectAttempts})...`);

      setTimeout(() => {
        this.connect().catch(err => {
          console.error('重连失败:', err);
        });
      }, this.reconnectInterval);
    }
  }
}
```

#### 5.5.2 React Hook 封装

```typescript
/**
 * 工作流 WebSocket Hook
 */
import { useEffect, useRef, useCallback, useState } from 'react';
import { WorkflowWebSocketClient } from './ws-client';

interface UseWorkflowWebSocketOptions {
  baseUrl: string;
  token: string;
  autoConnect?: boolean;
}

interface ExecutionStatus {
  executionUuid: string;
  status: string;
  progress: number;
  nodes: Map<string, NodeStatus>;
  logs: LogEntry[];
}

interface NodeStatus {
  nodeUuid: string;
  nodeName: string;
  status: string;
  progress: number;
  startTime?: number;
  durationMs?: number;
}

interface LogEntry {
  timestamp: number;
  nodeUuid: string;
  nodeName: string;
  level: string;
  message: string;
}

export function useWorkflowWebSocket(options: UseWorkflowWebSocketOptions) {
  const { baseUrl, token, autoConnect = true } = options;

  const clientRef = useRef<WorkflowWebSocketClient | null>(null);
  const [connected, setConnected] = useState(false);
  const [executionStatus, setExecutionStatus] = useState<ExecutionStatus | null>(null);

  // 连接
  const connect = useCallback(async () => {
    if (!clientRef.current) {
      clientRef.current = new WorkflowWebSocketClient(baseUrl, token);
    }

    await clientRef.current.connect();
    setConnected(true);

    // 注册处理器
    clientRef.current.on('WORKFLOW_STATUS_CHANGED', handleWorkflowStatus);
    clientRef.current.on('NODE_STATUS_CHANGED', handleNodeStatus);
    clientRef.current.on('NODE_EXECUTION_COMPLETED', handleNodeCompleted);
    clientRef.current.on('NODE_EXECUTION_LOG', handleNodeLog);
    clientRef.current.on('WORKFLOW_EXECUTION_COMPLETED', handleWorkflowCompleted);

  }, [baseUrl, token]);

  // 断开连接
  const disconnect = useCallback(() => {
    if (clientRef.current) {
      clientRef.current.disconnect();
      clientRef.current = null;
    }
    setConnected(false);
  }, []);

  // 订阅执行
  const subscribeExecution = useCallback((executionUuid: string) => {
    if (clientRef.current) {
      clientRef.current.subscribeExecution(executionUuid);
      setExecutionStatus({
        executionUuid,
        status: 'PENDING',
        progress: 0,
        nodes: new Map(),
        logs: []
      });
    }
  }, []);

  // 取消订阅
  const unsubscribeExecution = useCallback((executionUuid: string) => {
    if (clientRef.current) {
      clientRef.current.unsubscribeExecution(executionUuid);
    }
    setExecutionStatus(null);
  }, []);

  // 处理工作流状态变更
  const handleWorkflowStatus = useCallback((payload: any) => {
    setExecutionStatus(prev => {
      if (!prev) return null;
      return {
        ...prev,
        status: payload.currentStatus,
        progress: payload.progress
      };
    });
  }, []);

  // 处理节点状态变更
  const handleNodeStatus = useCallback((payload: any) => {
    setExecutionStatus(prev => {
      if (!prev) return null;

      const newNodes = new Map(prev.nodes);
      newNodes.set(payload.nodeUuid, {
        nodeUuid: payload.nodeUuid,
        nodeName: payload.nodeName,
        status: payload.currentStatus,
        progress: payload.progress || 0,
        startTime: payload.startTime
      });

      return { ...prev, nodes: newNodes };
    });
  }, []);

  // 处理节点执行完成
  const handleNodeCompleted = useCallback((payload: any) => {
    setExecutionStatus(prev => {
      if (!prev) return null;

      const newNodes = new Map(prev.nodes);
      const existing = newNodes.get(payload.nodeUuid);

      if (existing) {
        newNodes.set(payload.nodeUuid, {
          ...existing,
          status: payload.status,
          durationMs: payload.durationMs
        });
      }

      return { ...prev, nodes: newNodes };
    });
  }, []);

  // 处理节点日志
  const handleNodeLog = useCallback((payload: any) => {
    setExecutionStatus(prev => {
      if (!prev) return null;

      return {
        ...prev,
        logs: [
          ...prev.logs,
          {
            timestamp: payload.timestamp,
            nodeUuid: payload.nodeUuid,
            nodeName: payload.nodeName,
            level: payload.logLevel,
            message: payload.message
          }
        ]
      };
    });
  }, []);

  // 处理工作流完成
  const handleWorkflowCompleted = useCallback((payload: any) => {
    setExecutionStatus(prev => {
      if (!prev) return null;
      return {
        ...prev,
        status: payload.status,
        progress: 100
      };
    });
  }, []);

  // 自动连接
  useEffect(() => {
    if (autoConnect) {
      connect();
    }

    return () => {
      disconnect();
    };
  }, [autoConnect, connect, disconnect]);

  return {
    connected,
    executionStatus,
    connect,
    disconnect,
    subscribeExecution,
    unsubscribeExecution
  };
}
```

---

### 5.6 推送流程图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          实时状态推送流程                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  1. 前端发起 WebSocket 连接                                                      │
│     │                                                                           │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  WebSocketInterceptor.beforeHandshake()                                   │  │
│  │  - 验证 token                                                             │  │
│  │  - 提取 userId                                                            │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                           │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  WebSocketHandler.afterConnectionEstablished()                            │  │
│  │  - 记录 session                                                           │  │
│  │  - 发送 CONNECTED 消息                                                    │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                           │
│     │  2. 前端订阅执行                                                          │
│     │     发送 SUBSCRIBE_EXECUTION { executionUuid: "xxx" }                    │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  WebSocketHandler.handleTextMessage()                                     │  │
│  │  - 解析消息类型                                                           │  │
│  │  - 调用 SubscriptionManager.subscribe()                                   │  │
│  │  - 发送 SUBSCRIBE_SUCCESS                                                 │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                           │
│     │  3. 工作流执行，状态变更                                                   │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  ExecutionEngine / NodeExecutor                                           │  │
│  │  - 节点状态变更时调用 StatusEventPublisher                                 │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                           │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  StatusEventPublisher.publishXxx()                                        │  │
│  │  - 构建消息载荷                                                           │  │
│  │  - 调用 WebSocketHandler.broadcastToExecution()                           │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                           │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  WebSocketHandler.broadcastToExecution()                                  │  │
│  │  - 获取订阅该执行的所有 sessionId                                          │  │
│  │  - 遍历发送 WebSocket 消息                                                │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                           │
│     │  4. 前端接收消息                                                          │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  前端 WebSocket 客户端                                                     │  │
│  │  - 接收消息                                                               │  │
│  │  - 根据 type 调用对应处理器                                                │  │
│  │  - 更新 UI 状态                                                           │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### 5.7 性能优化建议

#### 5.7.1 消息批量推送

```java
/**
 * 消息批量推送器
 * 将高频消息合并后批量推送
 */
@Component
public class BatchMessagePublisher {

    private final WorkflowWebSocketHandler webSocketHandler;

    // 消息缓冲区
    private final ConcurrentHashMap<String, List<WebSocketMessage<?>>> messageBuffer =
        new ConcurrentHashMap<>();

    // 批量大小
    private static final int BATCH_SIZE = 10;

    // 批量间隔（毫秒）
    private static final long BATCH_INTERVAL = 100;

    /**
     * 添加消息到缓冲区
     */
    public void addMessage(String executionUuid, WebSocketMessage<?> message) {
        messageBuffer.computeIfAbsent(executionUuid, k ->
            Collections.synchronizedList(new ArrayList<>())
        ).add(message);

        // 检查是否达到批量大小
        List<WebSocketMessage<?>> messages = messageBuffer.get(executionUuid);
        if (messages.size() >= BATCH_SIZE) {
            flush(executionUuid);
        }
    }

    /**
     * 刷新缓冲区
     */
    private void flush(String executionUuid) {
        List<WebSocketMessage<?>> messages = messageBuffer.remove(executionUuid);
        if (messages != null && !messages.isEmpty()) {
            // 合并消息
            WebSocketMessage<?> batchMessage = WebSocketMessage.of("BATCH_UPDATE",
                Map.of("messages", messages));

            webSocketHandler.broadcastToExecution(executionUuid, batchMessage);
        }
    }

    /**
     * 定时刷新
     */
    @Scheduled(fixedRate = BATCH_INTERVAL)
    public void scheduledFlush() {
        messageBuffer.keySet().forEach(this::flush);
    }
}
```

#### 5.7.2 连接数限制

```java
/**
 * WebSocket 连接限制配置
 */
@Configuration
public class WebSocketLimitConfig {

    // 单用户最大连接数
    private static final int MAX_CONNECTIONS_PER_USER = 5;

    // 全局最大连接数
    private static final int MAX_TOTAL_CONNECTIONS = 10000;

    /**
     * 检查是否允许连接
     */
    public boolean allowConnection(String userId, int currentUserConnections) {
        return currentUserConnections < MAX_CONNECTIONS_PER_USER;
    }
}
```

---

*第五章完成，请确认后继续生成第六章。*
