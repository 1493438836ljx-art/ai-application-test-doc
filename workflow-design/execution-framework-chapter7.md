# 工作流执行框架设计文档

## 第七章：并发控制与资源管理

### 7.1 并发控制架构总览

并发控制与资源管理是保证工作流执行框架稳定性和高性能的关键。本章描述如何有效管理并发执行、控制资源使用、防止系统过载。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          并发控制架构                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        入口控制层                                    │  │
│   │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │  │
│   │  │ 执行请求队列   │  │ 限流器        │  │ 优先级调度器          │   │  │
│   │  │ - FIFO 队列   │  │ - 令牌桶      │  │ - 优先级队列          │   │  │
│   │  │ - 有界队列    │  │ - 滑动窗口    │  │ - 公平调度            │   │  │
│   │  └───────────────┘  └───────────────┘  └───────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        线程池管理层                                  │  │
│   │  ┌───────────────────────────────────────────────────────────────┐  │  │
│   │  │  WorkflowExecutorPool (工作流执行线程池)                        │  │  │
│   │  │  - 核心线程数: 10                                              │  │  │
│   │  │  - 最大线程数: 50                                              │  │  │
│   │  │  - 队列容量: 500                                               │  │  │
│   │  └───────────────────────────────────────────────────────────────┘  │  │
│   │  ┌───────────────────────────────────────────────────────────────┐  │  │
│   │  │  NodeExecutorPool (节点执行线程池)                              │  │  │
│   │  │  - 核心线程数: 20                                              │  │  │
│   │  │  - 最大线程数: 100                                             │  │  │
│   │  │  - 队列容量: 1000                                              │  │  │
│   │  └───────────────────────────────────────────────────────────────┘  │  │
│   │  ┌───────────────────────────────────────────────────────────────┐  │  │
│   │  │  AsyncTaskPool (异步任务线程池)                                 │  │  │
│   │  │  - 核心线程数: 50                                              │  │  │
│   │  │  - 最大线程数: 200                                             │  │  │
│   │  │  - 队列容量: 2000                                              │  │  │
│   │  └───────────────────────────────────────────────────────────────┘  │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        资源控制层                                    │  │
│   │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │  │
│   │  │ 信号量控制    │  │ 资源配额      │  │ 背压机制              │   │  │
│   │  │ - 全局并发数  │  │ - 按用户配额  │  │ - 自适应限流          │   │  │
│   │  │ - 按类型控制  │  │ - 按工作流配额│  │ - 动态调整            │   │  │
│   │  └───────────────┘  └───────────────┘  └───────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │                        监控与告警层                                  │  │
│   │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │  │
│   │  │ 资源监控      │  │ 性能指标      │  │ 告警通知              │   │  │
│   │  │ - 线程池状态  │  │ - 吞吐量      │  │ - 资源不足告警        │   │  │
│   │  │ - 队列积压    │  │ - 延迟        │  │ - 死锁告警            │   │  │
│   │  │ - 内存使用    │  │ - 错误率      │  │ - 队列满告警          │   │  │
│   │  └───────────────┘  └───────────────┘  └───────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 7.2 线程池管理

#### 7.2.1 线程池配置类

```java
/**
 * 线程池配置属性
 */
@Data
@ConfigurationProperties(prefix = "workflow.executor")
public class ExecutorProperties {

    /**
     * 工作流执行线程池配置
     */
    private PoolConfig workflow = new PoolConfig(
        10,     // corePoolSize
        50,     // maxPoolSize
        500,    // queueCapacity
        60,     // keepAliveSeconds
        "workflow-exec-"
    );

    /**
     * 节点执行线程池配置
     */
    private PoolConfig node = new PoolConfig(
        20,     // corePoolSize
        100,    // maxPoolSize
        1000,   // queueCapacity
        60,     // keepAliveSeconds
        "node-exec-"
    );

    /**
     * 异步任务线程池配置
     */
    private PoolConfig asyncTask = new PoolConfig(
        50,     // corePoolSize
        200,    // maxPoolSize
        2000,   // queueCapacity
        60,     // keepAliveSeconds
        "async-task-"
    );

    /**
     * 调度线程池配置
     */
    private PoolConfig scheduled = new PoolConfig(
        5,      // corePoolSize
        20,     // maxPoolSize
        100,    // queueCapacity
        60,     // keepAliveSeconds
        "scheduled-"
    );

    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public static class PoolConfig {
        private int corePoolSize;
        private int maxPoolSize;
        private int queueCapacity;
        private int keepAliveSeconds;
        private String threadNamePrefix;
    }
}
```

#### 7.2.2 线程池配置实现

```java
/**
 * 线程池配置
 */
@Configuration
@EnableConfigurationProperties(ExecutorProperties.class)
public class ThreadPoolConfig {

    private final ExecutorProperties properties;

    /**
     * 工作流执行线程池
     */
    @Bean("workflowExecutor")
    public ThreadPoolTaskExecutor workflowExecutor() {
        ExecutorProperties.PoolConfig config = properties.getWorkflow();

        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(config.getCorePoolSize());
        executor.setMaxPoolSize(config.getMaxPoolSize());
        executor.setQueueCapacity(config.getQueueCapacity());
        executor.setKeepAliveSeconds(config.getKeepAliveSeconds());
        executor.setThreadNamePrefix(config.getThreadNamePrefix());
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setAllowCoreThreadTimeOut(true);
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);

        executor.initialize();
        return executor;
    }

    /**
     * 节点执行线程池
     */
    @Bean("nodeExecutor")
    public ThreadPoolTaskExecutor nodeExecutor() {
        ExecutorProperties.PoolConfig config = properties.getNode();

        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(config.getCorePoolSize());
        executor.setMaxPoolSize(config.getMaxPoolSize());
        executor.setQueueCapacity(config.getQueueCapacity());
        executor.setKeepAliveSeconds(config.getKeepAliveSeconds());
        executor.setThreadNamePrefix(config.getThreadNamePrefix());
        executor.setRejectedExecutionHandler(new CustomRejectionHandler());
        executor.setAllowCoreThreadTimeOut(true);
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);

        executor.initialize();
        return executor;
    }

    /**
     * 异步任务线程池
     */
    @Bean("asyncTaskExecutor")
    public ThreadPoolTaskExecutor asyncTaskExecutor() {
        ExecutorProperties.PoolConfig config = properties.getAsyncTask();

        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(config.getCorePoolSize());
        executor.setMaxPoolSize(config.getMaxPoolSize());
        executor.setQueueCapacity(config.getQueueCapacity());
        executor.setKeepAliveSeconds(config.getKeepAliveSeconds());
        executor.setThreadNamePrefix(config.getThreadNamePrefix());
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.setAllowCoreThreadTimeOut(true);
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(120);

        executor.initialize();
        return executor;
    }

    /**
     * 调度线程池
     */
    @Bean("scheduledExecutor")
    public ThreadPoolTaskScheduler scheduledExecutor() {
        ExecutorProperties.PoolConfig config = properties.getScheduled();

        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(config.getCorePoolSize());
        scheduler.setThreadNamePrefix(config.getThreadNamePrefix());
        scheduler.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.setAwaitTerminationSeconds(30);

        scheduler.initialize();
        return scheduler;
    }
}
```

#### 7.2.3 自定义拒绝策略

```java
/**
 * 自定义拒绝策略
 * 记录日志并尝试降级处理
 */
@Slf4j
public class CustomRejectionHandler implements RejectedExecutionHandler {

    private final AtomicLong rejectedCount = new AtomicLong(0);

    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        long count = rejectedCount.incrementAndGet();

        log.warn("任务被拒绝: poolName={}, activeCount={}, poolSize={}, queueSize={}, " +
                 "rejectedCount={}",
                 getPoolName(executor),
                 executor.getActiveCount(),
                 executor.getPoolSize(),
                 executor.getQueue().size(),
                 count);

        // 策略1: 尝试等待一段时间后重新提交
        if (count % 100 == 0) {
            // 每100次拒绝，等待一下
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }

        // 策略2: 由调用线程执行（降级）
        if (!executor.isShutdown()) {
            r.run();
        }
    }

    private String getPoolName(ThreadPoolExecutor executor) {
        ThreadFactory factory = executor.getThreadFactory();
        if (factory instanceof ThreadFactory) {
            return "unknown";
        }
        return executor.toString();
    }

    public long getRejectedCount() {
        return rejectedCount.get();
    }
}
```

#### 7.2.4 线程池监控

```java
/**
 * 线程池监控服务
 */
@Service
@Slf4j
public class ThreadPoolMonitorService {

    private final Map<String, ThreadPoolTaskExecutor> executors;

    @Autowired
    public ThreadPoolMonitorService(
            @Qualifier("workflowExecutor") ThreadPoolTaskExecutor workflowExecutor,
            @Qualifier("nodeExecutor") ThreadPoolTaskExecutor nodeExecutor,
            @Qualifier("asyncTaskExecutor") ThreadPoolTaskExecutor asyncTaskExecutor) {

        this.executors = Map.of(
            "workflow", workflowExecutor,
            "node", nodeExecutor,
            "asyncTask", asyncTaskExecutor
        );
    }

    /**
     * 获取线程池状态
     */
    public Map<String, ThreadPoolStatus> getAllPoolStatus() {
        Map<String, ThreadPoolStatus> statusMap = new HashMap<>();

        executors.forEach((name, executor) -> {
            ThreadPoolExecutor pool = executor.getThreadPoolExecutor();
            statusMap.put(name, ThreadPoolStatus.builder()
                .poolName(name)
                .corePoolSize(pool.getCorePoolSize())
                .maxPoolSize(pool.getMaximumPoolSize())
                .activeCount(pool.getActiveCount())
                .poolSize(pool.getPoolSize())
                .queueSize(pool.getQueue().size())
                .completedTaskCount(pool.getCompletedTaskCount())
                .taskCount(pool.getTaskCount())
                .build());
        });

        return statusMap;
    }

    /**
     * 检查线程池健康状态
     */
    public List<PoolHealthCheck> checkHealth() {
        List<PoolHealthCheck> checks = new ArrayList<>();

        executors.forEach((name, executor) -> {
            ThreadPoolExecutor pool = executor.getThreadPoolExecutor();

            double usageRate = (double) pool.getActiveCount() / pool.getMaximumPoolSize();
            double queueUsageRate = (double) pool.getQueue().size() /
                                    pool.getQueue().remainingCapacity() + pool.getQueue().size();

            PoolHealthStatus status;
            if (usageRate > 0.9 || queueUsageRate > 0.9) {
                status = PoolHealthStatus.CRITICAL;
            } else if (usageRate > 0.7 || queueUsageRate > 0.7) {
                status = PoolHealthStatus.WARNING;
            } else {
                status = PoolHealthStatus.HEALTHY;
            }

            checks.add(PoolHealthCheck.builder()
                .poolName(name)
                .status(status)
                .activeRate(usageRate)
                .queueUsageRate(queueUsageRate)
                .message(status.getMessage())
                .build());
        });

        return checks;
    }

    /**
     * 定时记录线程池状态
     */
    @Scheduled(fixedRate = 30000) // 每30秒
    public void logPoolStatus() {
        Map<String, ThreadPoolStatus> statusMap = getAllPoolStatus();

        statusMap.forEach((name, status) -> {
            log.info("线程池状态 [{}]: active={}/{}, queue={}, completed={}",
                     name,
                     status.getActiveCount(),
                     status.getMaxPoolSize(),
                     status.getQueueSize(),
                     status.getCompletedTaskCount());
        });
    }
}

/**
 * 线程池状态
 */
@Data
@Builder
public class ThreadPoolStatus {
    private String poolName;
    private int corePoolSize;
    private int maxPoolSize;
    private int activeCount;
    private int poolSize;
    private int queueSize;
    private long completedTaskCount;
    private long taskCount;
}

/**
 * 健康检查结果
 */
@Data
@Builder
public class PoolHealthCheck {
    private String poolName;
    private PoolHealthStatus status;
    private double activeRate;
    private double queueUsageRate;
    private String message;
}

/**
 * 健康状态
 */
public enum PoolHealthStatus {
    HEALTHY("健康"),
    WARNING("警告"),
    CRITICAL("严重");

    private final String message;

    PoolHealthStatus(String message) {
        this.message = message;
    }

    public String getMessage() {
        return message;
    }
}
```

---

### 7.3 信号量与限流

#### 7.3.1 并发控制服务

```java
/**
 * 并发控制服务
 * 使用信号量控制各类资源的并发访问
 */
@Service
@Slf4j
public class ConcurrencyControlService {

    /**
     * 全局工作流执行信号量
     */
    private final Semaphore globalWorkflowSemaphore;

    /**
     * 按执行类型的信号量
     */
    private final Map<String, Semaphore> executionTypeSemaphores;

    /**
     * 按用户的信号量
     */
    private final ConcurrentHashMap<String, Semaphore> userSemaphores;

    /**
     * 按工作流的信号量
     */
    private final ConcurrentHashMap<Long, Semaphore> workflowSemaphores;

    @Autowired
    public ConcurrencyControlService(ConcurrencyConfig config) {
        // 初始化全局信号量
        this.globalWorkflowSemaphore = new Semaphore(config.getGlobalMaxConcurrency());

        // 初始化执行类型信号量
        this.executionTypeSemaphores = new HashMap<>();
        config.getTypeLimits().forEach((type, limit) -> {
            executionTypeSemaphores.put(type, new Semaphore(limit));
        });

        this.userSemaphores = new ConcurrentHashMap<>();
        this.workflowSemaphores = new ConcurrentHashMap<>();
    }

    /**
     * 获取工作流执行许可
     *
     * @param context 执行上下文
     * @return 是否获取成功
     */
    public boolean acquireWorkflowPermit(ExecutionContext context) {
        String userId = context.getTriggeredBy();
        Long workflowId = context.getWorkflowId();

        // 1. 尝试获取全局许可
        if (!globalWorkflowSemaphore.tryAcquire()) {
            log.warn("全局并发数已达上限: workflowId={}", workflowId);
            return false;
        }

        // 2. 尝试获取用户级许可
        Semaphore userSemaphore = getUserSemaphore(userId);
        if (!userSemaphore.tryAcquire()) {
            globalWorkflowSemaphore.release();
            log.warn("用户并发数已达上限: userId={}, workflowId={}", userId, workflowId);
            return false;
        }

        // 3. 尝试获取工作流级许可
        Semaphore workflowSemaphore = getWorkflowSemaphore(workflowId);
        if (!workflowSemaphore.tryAcquire()) {
            userSemaphore.release();
            globalWorkflowSemaphore.release();
            log.warn("工作流并发数已达上限: workflowId={}", workflowId);
            return false;
        }

        log.debug("获取工作流执行许可成功: workflowId={}, userId={}", workflowId, userId);
        return true;
    }

    /**
     * 释放工作流执行许可
     */
    public void releaseWorkflowPermit(ExecutionContext context) {
        String userId = context.getTriggeredBy();
        Long workflowId = context.getWorkflowId();

        Semaphore workflowSemaphore = workflowSemaphores.get(workflowId);
        Semaphore userSemaphore = userSemaphores.get(userId);

        if (workflowSemaphore != null) {
            workflowSemaphore.release();
        }
        if (userSemaphore != null) {
            userSemaphore.release();
        }
        globalWorkflowSemaphore.release();

        log.debug("释放工作流执行许可: workflowId={}, userId={}", workflowId, userId);
    }

    /**
     * 获取节点执行许可
     *
     * @param nodeType 节点类型
     * @param executionType 执行类型
     * @return 许可对象（用于 try-with-resources）
     */
    public Permit acquireNodePermit(String nodeType, String executionType) {
        Semaphore typeSemaphore = executionTypeSemaphores.get(executionType);

        if (typeSemaphore != null) {
            try {
                boolean acquired = typeSemaphore.tryAcquire(10, TimeUnit.SECONDS);
                if (!acquired) {
                    throw new ResourceExhaustedException(
                        "执行类型并发数已达上限: " + executionType
                    );
                }
                return new Permit(typeSemaphore);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new WorkflowExecutionException(ErrorCode.INTERNAL_ERROR, e);
            }
        }

        return Permit.noOp();
    }

    /**
     * 获取用户信号量
     */
    private Semaphore getUserSemaphore(String userId) {
        return userSemaphores.computeIfAbsent(userId,
            k -> new Semaphore(getConfig().getUserMaxConcurrency()));
    }

    /**
     * 获取工作流信号量
     */
    private Semaphore getWorkflowSemaphore(Long workflowId) {
        return workflowSemaphores.computeIfAbsent(workflowId,
            k -> new Semaphore(getConfig().getWorkflowMaxConcurrency()));
    }

    /**
     * 获取当前并发状态
     */
    public ConcurrencyStatus getStatus() {
        return ConcurrencyStatus.builder()
            .globalAvailable(globalWorkflowSemaphore.availablePermits())
            .globalQueued(globalWorkflowSemaphore.getQueueLength())
            .byType(executionTypeSemaphores.entrySet().stream()
                .collect(Collectors.toMap(
                    Map.Entry::getKey,
                    e -> e.getValue().availablePermits()
                )))
            .byUser(userSemaphores.entrySet().stream()
                .collect(Collectors.toMap(
                    Map.Entry::getKey,
                    e -> e.getValue().availablePermits()
                )))
            .build();
    }

    private ConcurrencyConfig getConfig() {
        // TODO: 从配置获取
        return new ConcurrencyConfig();
    }
}

/**
 * 许可对象（AutoCloseable）
 */
@Data
public class Permit implements AutoCloseable {

    private final Semaphore semaphore;
    private final boolean acquired;

    public static Permit noOp() {
        return new Permit(null, false);
    }

    @Override
    public void close() {
        if (acquired && semaphore != null) {
            semaphore.release();
        }
    }
}

/**
 * 并发配置
 */
@Data
public class ConcurrencyConfig {
    private int globalMaxConcurrency = 100;
    private int userMaxConcurrency = 10;
    private int workflowMaxConcurrency = 5;
    private Map<String, Integer> typeLimits = Map.of(
        "AUTOMATED", 50,
        "AI_AGENT", 20
    );
}

/**
 * 并发状态
 */
@Data
@Builder
public class ConcurrencyStatus {
    private int globalAvailable;
    private int globalQueued;
    private Map<String, Integer> byType;
    private Map<String, Integer> byUser;
}
```

#### 7.3.2 限流器实现

```java
/**
 * 限流器接口
 */
public interface RateLimiter {
    boolean tryAcquire();
    boolean tryAcquire(int permits);
    boolean tryAcquire(long timeout, TimeUnit unit);
}

/**
 * 令牌桶限流器
 */
public class TokenBucketRateLimiter implements RateLimiter {

    private final long capacity;        // 桶容量
    private final long refillRate;      // 每秒补充令牌数
    private final AtomicLong tokens;    // 当前令牌数
    private final AtomicLong lastRefillTime;

    public TokenBucketRateLimiter(long capacity, long refillRate) {
        this.capacity = capacity;
        this.refillRate = refillRate;
        this.tokens = new AtomicLong(capacity);
        this.lastRefillTime = new AtomicLong(System.currentTimeMillis());
    }

    @Override
    public boolean tryAcquire() {
        return tryAcquire(1);
    }

    @Override
    public boolean tryAcquire(int permits) {
        refill();

        while (true) {
            long current = tokens.get();
            if (current < permits) {
                return false;
            }
            if (tokens.compareAndSet(current, current - permits)) {
                return true;
            }
        }
    }

    @Override
    public boolean tryAcquire(long timeout, TimeUnit unit) {
        long deadline = System.currentTimeMillis() + unit.toMillis(timeout);

        while (System.currentTimeMillis() < deadline) {
            if (tryAcquire()) {
                return true;
            }
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return false;
            }
        }
        return false;
    }

    private void refill() {
        long now = System.currentTimeMillis();
        long last = lastRefillTime.get();

        if (now > last) {
            long elapsed = now - last;
            long tokensToAdd = (elapsed * refillRate) / 1000;

            if (tokensToAdd > 0 && lastRefillTime.compareAndSet(last, now)) {
                while (true) {
                    long current = tokens.get();
                    long newTokens = Math.min(current + tokensToAdd, capacity);
                    if (tokens.compareAndSet(current, newTokens)) {
                        break;
                    }
                }
            }
        }
    }
}

/**
 * 滑动窗口限流器
 */
public class SlidingWindowRateLimiter implements RateLimiter {

    private final long windowSizeMs;
    private final int maxRequests;
    private final ConcurrentLinkedDeque<Long> timestamps;

    public SlidingWindowRateLimiter(long windowSizeMs, int maxRequests) {
        this.windowSizeMs = windowSizeMs;
        this.maxRequests = maxRequests;
        this.timestamps = new ConcurrentLinkedDeque<>();
    }

    @Override
    public boolean tryAcquire() {
        return tryAcquire(1);
    }

    @Override
    public boolean tryAcquire(int permits) {
        long now = System.currentTimeMillis();
        long windowStart = now - windowSizeMs;

        // 清理过期的时间戳
        while (!timestamps.isEmpty() && timestamps.peekFirst() < windowStart) {
            timestamps.pollFirst();
        }

        // 检查是否超过限制
        if (timestamps.size() + permits > maxRequests) {
            return false;
        }

        // 添加当前时间戳
        for (int i = 0; i < permits; i++) {
            timestamps.addLast(now);
        }

        return true;
    }

    @Override
    public boolean tryAcquire(long timeout, TimeUnit unit) {
        long deadline = System.currentTimeMillis() + unit.toMillis(timeout);

        while (System.currentTimeMillis() < deadline) {
            if (tryAcquire()) {
                return true;
            }
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return false;
            }
        }
        return false;
    }
}

/**
 * 限流器工厂
 */
@Component
public class RateLimiterFactory {

    /**
     * 创建执行请求限流器
     */
    public RateLimiter createExecutionRateLimiter() {
        // 每秒最多 100 个请求
        return new TokenBucketRateLimiter(100, 100);
    }

    /**
     * 创建 API 调用限流器
     */
    public RateLimiter createApiRateLimiter() {
        // 每分钟最多 1000 个请求
        return new SlidingWindowRateLimiter(60000, 1000);
    }
}
```

---

### 7.4 资源隔离

#### 7.4.1 资源配额管理

```java
/**
 * 资源配额配置
 */
@Data
@ConfigurationProperties(prefix = "workflow.quota")
public class ResourceQuotaConfig {

    /**
     * 默认用户配额
     */
    private QuotaConfig defaultUserQuota = new QuotaConfig(10, 100, 5);

    /**
     * 按用户的配额覆盖
     */
    private Map<String, QuotaConfig> userQuotas = new HashMap<>();

    /**
     * 默认工作流配额
     */
    private QuotaConfig defaultWorkflowQuota = new QuotaConfig(5, 50, 3);

    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public static class QuotaConfig {
        /** 最大并发执行数 */
        private int maxConcurrentExecutions;

        /** 最大历史记录数 */
        private int maxHistoryRecords;

        /** 每日最大执行次数 */
        private int dailyExecutionLimit;
    }
}

/**
 * 资源配额服务
 */
@Service
@Slf4j
public class ResourceQuotaService {

    private final ResourceQuotaConfig config;
    private final WorkflowExecutionRepository executionRepository;

    /**
     * 用户执行计数缓存
     * key: userId_date
     * value: count
     */
    private final ConcurrentHashMap<String, AtomicInteger> dailyExecutionCounts =
        new ConcurrentHashMap<>();

    /**
     * 检查是否可以执行
     */
    public QuotaCheckResult checkExecutionQuota(String userId, Long workflowId) {
        ResourceQuotaConfig.QuotaConfig userQuota = getUserQuota(userId);
        ResourceQuotaConfig.QuotaConfig workflowQuota = config.getDefaultWorkflowQuota();

        // 1. 检查每日执行次数
        int dailyCount = getDailyExecutionCount(userId);
        if (dailyCount >= userQuota.getDailyExecutionLimit()) {
            return QuotaCheckResult.denied("每日执行次数已达上限: " +
                userQuota.getDailyExecutionLimit());
        }

        // 2. 检查用户并发数
        int userRunningCount = getRunningExecutionCount(userId);
        if (userRunningCount >= userQuota.getMaxConcurrentExecutions()) {
            return QuotaCheckResult.denied("用户并发执行数已达上限: " +
                userQuota.getMaxConcurrentExecutions());
        }

        // 3. 检查工作流并发数
        int workflowRunningCount = getRunningExecutionCountByWorkflow(workflowId);
        if (workflowRunningCount >= workflowQuota.getMaxConcurrentExecutions()) {
            return QuotaCheckResult.denied("工作流并发执行数已达上限: " +
                workflowQuota.getMaxConcurrentExecutions());
        }

        return QuotaCheckResult.allowed();
    }

    /**
     * 记录执行开始
     */
    public void recordExecutionStart(String userId) {
        String key = getDailyKey(userId);
        dailyExecutionCounts.computeIfAbsent(key, k -> new AtomicInteger(0))
            .incrementAndGet();
    }

    /**
     * 获取用户配额
     */
    private ResourceQuotaConfig.QuotaConfig getUserQuota(String userId) {
        return config.getUserQuotas().getOrDefault(userId, config.getDefaultUserQuota());
    }

    /**
     * 获取每日执行次数
     */
    private int getDailyExecutionCount(String userId) {
        String key = getDailyKey(userId);
        AtomicInteger count = dailyExecutionCounts.get(key);
        return count != null ? count.get() : 0;
    }

    /**
     * 获取正在执行的次数
     */
    private int getRunningExecutionCount(String userId) {
        return (int) executionRepository.countByTriggeredByAndStatus(
            userId, ExecutionStatus.RUNNING.name()
        );
    }

    /**
     * 获取工作流正在执行的次数
     */
    private int getRunningExecutionCountByWorkflow(Long workflowId) {
        return (int) executionRepository.countByWorkflowIdAndStatus(
            workflowId, ExecutionStatus.RUNNING.name()
        );
    }

    /**
     * 获取每日 key
     */
    private String getDailyKey(String userId) {
        return userId + "_" + LocalDate.now().toString();
    }

    /**
     * 定时清理过期计数
     */
    @Scheduled(cron = "0 0 0 * * ?") // 每天凌晨
    public void cleanupExpiredCounts() {
        String today = LocalDate.now().toString();
        dailyExecutionCounts.keySet().removeIf(key -> !key.endsWith(today));
        log.info("清理过期每日执行计数完成");
    }
}

/**
 * 配额检查结果
 */
@Data
@Builder
public class QuotaCheckResult {
    private boolean allowed;
    private String reason;

    public static QuotaCheckResult allowed() {
        return QuotaCheckResult.builder().allowed(true).build();
    }

    public static QuotaCheckResult denied(String reason) {
        return QuotaCheckResult.builder().allowed(false).reason(reason).build();
    }
}
```

#### 7.4.2 执行隔离

```java
/**
 * 执行隔离服务
 * 确保不同用户的执行相互隔离
 */
@Service
@Slf4j
public class ExecutionIsolationService {

    /**
     * 用户执行上下文隔离
     * key: userId
     * value: 用户专属的执行上下文集合
     */
    private final ConcurrentHashMap<String, UserExecutionContext> userContexts =
        new ConcurrentHashMap<>();

    /**
     * 获取或创建用户执行上下文
     */
    public UserExecutionContext getOrCreateUserContext(String userId) {
        return userContexts.computeIfAbsent(userId,
            k -> new UserExecutionContext(userId));
    }

    /**
     * 在隔离上下文中执行
     */
    public <T> T executeInIsolation(String userId, Callable<T> task) {
        UserExecutionContext context = getOrCreateUserContext(userId);

        try {
            context.acquire();
            return task.call();
        } catch (Exception e) {
            log.error("隔离执行异常: userId={}", userId, e);
            throw new RuntimeException(e);
        } finally {
            context.release();
        }
    }

    /**
     * 用户执行上下文
     */
    @Data
    private static class UserExecutionContext {
        private final String userId;
        private final Semaphore semaphore;
        private final AtomicInteger activeExecutions;
        private final AtomicInteger totalExecutions;
        private final AtomicLong totalDuration;

        public UserExecutionContext(String userId) {
            this.userId = userId;
            this.semaphore = new Semaphore(10); // 用户最大并发
            this.activeExecutions = new AtomicInteger(0);
            this.totalExecutions = new AtomicInteger(0);
            this.totalDuration = new AtomicLong(0);
        }

        public void acquire() throws InterruptedException {
            semaphore.acquire();
            activeExecutions.incrementAndGet();
        }

        public void release() {
            activeExecutions.decrementAndGet();
            totalExecutions.incrementAndGet();
            semaphore.release();
        }
    }
}
```

---

### 7.5 背压机制

#### 7.5.1 自适应背压控制器

```java
/**
 * 背压控制器
 * 根据系统负载动态调整执行速率
 */
@Service
@Slf4j
public class BackpressureController {

    private final SystemMetricsCollector metricsCollector;

    // 背压状态
    private volatile BackpressureState currentState = BackpressureState.NORMAL;

    // 当前限流比例 (0.0 - 1.0)
    private volatile double throttleRatio = 0.0;

    // 上次调整时间
    private volatile long lastAdjustTime = System.currentTimeMillis();

    // 调整间隔
    private static final long ADJUST_INTERVAL = 5000; // 5秒

    /**
     * 检查是否接受执行请求
     */
    public boolean shouldAcceptRequest() {
        updateBackpressureState();

        if (currentState == BackpressureState.REJECT) {
            return false;
        }

        if (currentState == BackpressureState.THROTTLE) {
            // 按比例随机拒绝
            return Math.random() > throttleRatio;
        }

        return true;
    }

    /**
     * 获取当前状态
     */
    public BackpressureStatus getStatus() {
        return BackpressureStatus.builder()
            .state(currentState)
            .throttleRatio(throttleRatio)
            .systemMetrics(metricsCollector.getCurrentMetrics())
            .build();
    }

    /**
     * 更新背压状态
     */
    private synchronized void updateBackpressureState() {
        long now = System.currentTimeMillis();
        if (now - lastAdjustTime < ADJUST_INTERVAL) {
            return;
        }
        lastAdjustTime = now;

        SystemMetrics metrics = metricsCollector.getCurrentMetrics();

        // 计算负载分数 (0-100)
        double loadScore = calculateLoadScore(metrics);

        // 根据负载分数调整状态
        if (loadScore > 90) {
            currentState = BackpressureState.REJECT;
            throttleRatio = 1.0;
            log.warn("背压状态: REJECT, loadScore={}", loadScore);
        } else if (loadScore > 70) {
            currentState = BackpressureState.THROTTLE;
            throttleRatio = (loadScore - 70) / 30.0; // 0.0 - 0.67
            log.info("背压状态: THROTTLE, loadScore={}, throttleRatio={}",
                     loadScore, throttleRatio);
        } else {
            currentState = BackpressureState.NORMAL;
            throttleRatio = 0.0;
        }
    }

    /**
     * 计算负载分数
     */
    private double calculateLoadScore(SystemMetrics metrics) {
        // CPU 权重 40%
        double cpuScore = metrics.getCpuUsage() * 0.4;

        // 内存权重 30%
        double memoryScore = metrics.getMemoryUsage() * 0.3;

        // 线程池使用率权重 20%
        double threadPoolScore = metrics.getThreadPoolUsage() * 0.2;

        // 队列积压权重 10%
        double queueScore = metrics.getQueueBacklog() * 0.1;

        return cpuScore + memoryScore + threadPoolScore + queueScore;
    }
}

/**
 * 背压状态枚举
 */
public enum BackpressureState {
    NORMAL("正常"),
    THROTTLE("限流"),
    REJECT("拒绝");

    private final String description;

    BackpressureState(String description) {
        this.description = description;
    }

    public String getDescription() {
        return description;
    }
}

/**
 * 背压状态信息
 */
@Data
@Builder
public class BackpressureStatus {
    private BackpressureState state;
    private double throttleRatio;
    private SystemMetrics systemMetrics;
}

/**
 * 系统指标
 */
@Data
@Builder
public class SystemMetrics {
    private double cpuUsage;          // CPU 使用率 (0-100)
    private double memoryUsage;       // 内存使用率 (0-100)
    private double threadPoolUsage;   // 线程池使用率 (0-100)
    private double queueBacklog;      // 队列积压率 (0-100)
    private long timestamp;
}

/**
 * 系统指标收集器
 */
@Component
public class SystemMetricsCollector {

    private final ThreadPoolMonitorService threadPoolMonitor;

    public SystemMetrics getCurrentMetrics() {
        Runtime runtime = Runtime.getRuntime();

        // CPU 使用率
        double cpuUsage = getCpuUsage();

        // 内存使用率
        long maxMemory = runtime.maxMemory();
        long usedMemory = runtime.totalMemory() - runtime.freeMemory();
        double memoryUsage = (double) usedMemory / maxMemory * 100;

        // 线程池使用率
        double threadPoolUsage = getThreadPoolUsage();

        // 队列积压率
        double queueBacklog = getQueueBacklog();

        return SystemMetrics.builder()
            .cpuUsage(cpuUsage)
            .memoryUsage(memoryUsage)
            .threadPoolUsage(threadPoolUsage)
            .queueBacklog(queueBacklog)
            .timestamp(System.currentTimeMillis())
            .build();
    }

    private double getCpuUsage() {
        try {
            com.sun.management.OperatingSystemMXBean osBean =
                (com.sun.management.OperatingSystemMXBean)
                    ManagementFactory.getOperatingSystemMXBean();
            return osBean.getSystemCpuLoad() * 100;
        } catch (Exception e) {
            return 0;
        }
    }

    private double getThreadPoolUsage() {
        // TODO: 从线程池监控服务获取
        return 0;
    }

    private double getQueueBacklog() {
        // TODO: 从队列监控获取
        return 0;
    }
}
```

---

### 7.6 死锁检测与预防

#### 7.6.1 死锁检测器

```java
/**
 * 死锁检测器
 */
@Component
@Slf4j
public class DeadlockDetector {

    private final ThreadMXBean threadMXBean;

    @Autowired
    public DeadlockDetector() {
        this.threadMXBean = ManagementFactory.getThreadMXBean();
    }

    /**
     * 检测死锁
     */
    @Scheduled(fixedRate = 10000) // 每10秒检测一次
    public void detectDeadlock() {
        long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();

        if (deadlockedThreads != null && deadlockedThreads.length > 0) {
            ThreadInfo[] threadInfos = threadMXBean.getThreadInfo(deadlockedThreads);

            log.error("检测到死锁! 死锁线程数: {}", deadlockedThreads.length);

            for (ThreadInfo info : threadInfos) {
                log.error("死锁线程: id={}, name={}, state={}, lockName={}",
                          info.getThreadId(),
                          info.getThreadName(),
                          info.getThreadState(),
                          info.getLockName());

                // 打印堆栈
                StackTraceElement[] stack = info.getStackTrace();
                for (StackTraceElement element : stack) {
                    log.error("    at {}", element);
                }
            }

            // 发送告警
            sendDeadlockAlert(threadInfos);
        }
    }

    /**
     * 发送死锁告警
     */
    private void sendDeadlockAlert(ThreadInfo[] deadlockedThreads) {
        // TODO: 实现告警发送
    }

    /**
     * 获取线程状态快照
     */
    public ThreadDump getThreadDump() {
        ThreadInfo[] allThreads = threadMXBean.dumpAllThreads(true, true);

        List<ThreadSnapshot> snapshots = Arrays.stream(allThreads)
            .map(this::toThreadSnapshot)
            .collect(Collectors.toList());

        return ThreadDump.builder()
            .timestamp(System.currentTimeMillis())
            .threadCount(threadMXBean.getThreadCount())
            .peakThreadCount(threadMXBean.getPeakThreadCount())
            .daemonThreadCount(threadMXBean.getDaemonThreadCount())
            .threads(snapshots)
            .build();
    }

    private ThreadSnapshot toThreadSnapshot(ThreadInfo info) {
        return ThreadSnapshot.builder()
            .threadId(info.getThreadId())
            .threadName(info.getThreadName())
            .threadState(info.getThreadState().name())
            .blockedCount(info.getBlockedCount())
            .blockedTime(info.getBlockedTime())
            .waitedCount(info.getWaitedCount())
            .waitedTime(info.getWaitedTime())
            .lockName(info.getLockName())
            .lockOwnerId(info.getLockOwnerId())
            .lockOwnerName(info.getLockOwnerName())
            .stackTrace(Arrays.toString(info.getStackTrace()))
            .build();
    }
}

/**
 * 线程转储
 */
@Data
@Builder
public class ThreadDump {
    private long timestamp;
    private int threadCount;
    private int peakThreadCount;
    private int daemonThreadCount;
    private List<ThreadSnapshot> threads;
}

/**
 * 线程快照
 */
@Data
@Builder
public class ThreadSnapshot {
    private long threadId;
    private String threadName;
    private String threadState;
    private long blockedCount;
    private long blockedTime;
    private long waitedCount;
    private long waitedTime;
    private String lockName;
    private long lockOwnerId;
    private String lockOwnerName;
    private String stackTrace;
}
```

#### 7.6.2 死锁预防策略

```java
/**
 * 死锁预防服务
 */
@Service
@Slf4j
public class DeadlockPreventionService {

    /**
     * 全局锁顺序
     * 所有锁必须按照此顺序获取，避免循环等待
     */
    private static final List<String> LOCK_ORDER = Arrays.asList(
        "WORKFLOW",
        "EXECUTION",
        "NODE",
        "CONTEXT",
        "RESOURCE"
    );

    /**
     * 带超时的锁获取
     */
    public <T> T withLock(Lock lock, long timeout, TimeUnit unit, Callable<T> task) {
        boolean acquired = false;
        try {
            acquired = lock.tryLock(timeout, unit);
            if (!acquired) {
                throw new LockAcquisitionTimeoutException(
                    "获取锁超时: " + timeout + " " + unit
                );
            }
            return task.call();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException("获取锁被中断", e);
        } catch (Exception e) {
            throw new RuntimeException("执行任务失败", e);
        } finally {
            if (acquired) {
                lock.unlock();
            }
        }
    }

    /**
     * 按顺序获取多个锁
     */
    public void acquireLocksInOrder(Lock... locks) {
        // 按哈希值排序，确保所有线程以相同顺序获取锁
        List<Lock> sortedLocks = Arrays.stream(locks)
            .sorted(Comparator.comparingInt(Object::hashCode))
            .collect(Collectors.toList());

        for (Lock lock : sortedLocks) {
            lock.lock();
        }
    }

    /**
     * 释放多个锁
     */
    public void releaseLocks(Lock... locks) {
        for (int i = locks.length - 1; i >= 0; i--) {
            try {
                locks[i].unlock();
            } catch (Exception e) {
                log.warn("释放锁失败", e);
            }
        }
    }
}

/**
 * 锁获取超时异常
 */
public class LockAcquisitionTimeoutException extends RuntimeException {
    public LockAcquisitionTimeoutException(String message) {
        super(message);
    }
}
```

---

### 7.7 资源监控与告警

#### 7.7.1 资源监控服务

```java
/**
 * 资源监控服务
 */
@Service
@Slf4j
public class ResourceMonitorService {

    private final ThreadPoolMonitorService threadPoolMonitor;
    private final BackpressureController backpressureController;
    private final ConcurrencyControlService concurrencyControl;
    private final DeadlockDetector deadlockDetector;

    /**
     * 获取完整资源状态
     */
    public ResourceStatus getFullStatus() {
        return ResourceStatus.builder()
            .threadPools(threadPoolMonitor.getAllPoolStatus())
            .healthChecks(threadPoolMonitor.checkHealth())
            .backpressure(backpressureController.getStatus())
            .concurrency(concurrencyControl.getStatus())
            .systemMetrics(collectSystemMetrics())
            .timestamp(LocalDateTime.now())
            .build();
    }

    /**
     * 健康检查
     */
    public HealthCheckResult healthCheck() {
        List<String> issues = new ArrayList<>();

        // 检查线程池
        List<PoolHealthCheck> poolChecks = threadPoolMonitor.checkHealth();
        for (PoolHealthCheck check : poolChecks) {
            if (check.getStatus() == PoolHealthStatus.CRITICAL) {
                issues.add("线程池 [" + check.getPoolName() + "] 状态严重: " +
                           check.getMessage());
            }
        }

        // 检查背压状态
        BackpressureStatus bpStatus = backpressureController.getStatus();
        if (bpStatus.getState() == BackpressureState.REJECT) {
            issues.add("系统处于拒绝状态，负载过高");
        }

        // 检查内存
        SystemMetrics metrics = collectSystemMetrics();
        if (metrics.getMemoryUsage() > 90) {
            issues.add("内存使用率过高: " + metrics.getMemoryUsage() + "%");
        }

        HealthStatus status = issues.isEmpty() ? HealthStatus.HEALTHY :
            (issues.size() > 2 ? HealthStatus.UNHEALTHY : HealthStatus.DEGRADED);

        return HealthCheckResult.builder()
            .status(status)
            .issues(issues)
            .timestamp(LocalDateTime.now())
            .build();
    }

    private SystemMetrics collectSystemMetrics() {
        Runtime runtime = Runtime.getRuntime();

        return SystemMetrics.builder()
            .cpuUsage(getCpuUsage())
            .memoryUsage((double)(runtime.totalMemory() - runtime.freeMemory()) /
                         runtime.maxMemory() * 100)
            .threadPoolUsage(getThreadPoolUsage())
            .queueBacklog(getQueueBacklog())
            .timestamp(System.currentTimeMillis())
            .build();
    }

    private double getCpuUsage() {
        try {
            com.sun.management.OperatingSystemMXBean osBean =
                (com.sun.management.OperatingSystemMXBean)
                    ManagementFactory.getOperatingSystemMXBean();
            return osBean.getSystemCpuLoad() * 100;
        } catch (Exception e) {
            return 0;
        }
    }

    private double getThreadPoolUsage() {
        // 简化实现
        return 0;
    }

    private double getQueueBacklog() {
        // 简化实现
        return 0;
    }
}

/**
 * 资源状态
 */
@Data
@Builder
public class ResourceStatus {
    private Map<String, ThreadPoolStatus> threadPools;
    private List<PoolHealthCheck> healthChecks;
    private BackpressureStatus backpressure;
    private ConcurrencyStatus concurrency;
    private SystemMetrics systemMetrics;
    private LocalDateTime timestamp;
}

/**
 * 健康检查结果
 */
@Data
@Builder
public class HealthCheckResult {
    private HealthStatus status;
    private List<String> issues;
    private LocalDateTime timestamp;
}

/**
 * 健康状态
 */
public enum HealthStatus {
    HEALTHY("健康"),
    DEGRADED("降级"),
    UNHEALTHY("不健康");

    private final String description;

    HealthStatus(String description) {
        this.description = description;
    }
}
```

---

### 7.8 并发控制流程图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          并发控制完整流程                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  1. 执行请求到达                                                                 │
│     │                                                                           │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  背压控制器检查                                                            │  │
│  │  - 检查系统负载 (CPU, 内存, 线程池, 队列)                                  │  │
│  │  - 根据负载状态决定是否接受请求                                            │  │
│  │  - NORMAL: 接受                                                           │  │
│  │  - THROTTLE: 按比例随机接受                                               │  │
│  │  - REJECT: 拒绝                                                           │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                           │
│     │  接受请求                                                                 │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  资源配额检查                                                              │  │
│  │  - 检查用户每日执行次数                                                    │  │
│  │  - 检查用户当前并发数                                                      │  │
│  │  - 检查工作流当前并发数                                                    │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                           │
│     │  配额充足                                                                 │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  并发控制 - 获取许可                                                       │  │
│  │  - 获取全局信号量                                                          │  │
│  │  - 获取用户级信号量                                                        │  │
│  │  - 获取工作流级信号量                                                      │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                           │
│     │  获取成功                                                                 │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  提交到线程池执行                                                          │  │
│  │  - 选择合适的线程池                                                        │  │
│  │  - 提交执行任务                                                            │  │
│  │  - 处理拒绝策略                                                            │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│     │                                                                           │
│     │  执行完成                                                                 │
│     ▼                                                                           │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  释放资源                                                                  │  │
│  │  - 释放各级信号量                                                          │  │
│  │  - 更新执行计数                                                            │  │
│  │  - 更新资源监控指标                                                        │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  后台监控任务                                                              │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐  │  │
│  │  │ 线程池状态监控   │  │ 背压状态调整     │  │ 死锁检测               │  │  │
│  │  │ (30秒一次)      │  │ (5秒一次)       │  │ (10秒一次)             │  │  │
│  │  └─────────────────┘  └─────────────────┘  └─────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

*第七章完成。执行框架设计文档全部章节已生成完毕。*

---

## 文档总结

本执行框架设计文档共包含七个章节：

| 章节 | 标题 | 主要内容 |
|------|------|----------|
| 第一章 | 执行框架概述 | 设计目标、架构分层、核心组件、技术选型 |
| 第二章 | 执行引擎核心组件 | WorkflowScheduler、ExecutionEngine、ExecutionContext、StateManager |
| 第三章 | 节点执行器设计 | 基础节点、逻辑节点、技能节点执行器 |
| 第四章 | 基于 Kafka 的分布式执行 | Topic 设计、消息协议、Client 端架构 |
| 第五章 | 实时状态推送 | WebSocket 架构、消息类型、前端客户端 |
| 第六章 | 错误处理与重试机制 | 错误策略、重试机制、补偿服务 |
| 第七章 | 并发控制与资源管理 | 线程池管理、限流、背压机制、死锁检测 |

文档位置：`C:\Users\Administrator\Documents\workflow-design\`
