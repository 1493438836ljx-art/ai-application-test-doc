# Phase 5：事件集成 - 实施计划

> 版本：1.0.0
> 日期：2026-03-31
> 依赖：Phase 4 验证逻辑实现完成

---

## 一、目标

实现 Skill 变更事件监听和响应机制，包括：
- Skill 变更事件定义
- SkillChangeListener 事件监听器
- 与 SkillService 的集成
- 兼容性状态自动更新

---

## 二、事件架构

```
SkillService（Skill 模块）
    │
    ├── 更新 Skill
    │
    └── 发布 SkillChangeEvent
            │
            ▼
    SkillChangeListener（工作流模块）
            │
            ├── 查找使用该 Skill 的节点
            │
            └── 执行兼容性检查
                    │
                    └── 更新节点 compatibilityStatus
```

---

## 三、任务清单

### 3.1 SkillChangeEvent（事件类）

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `SkillChangeEvent.java` | Skill 变更事件 | 待完成 |

#### 事件定义

```java
/**
 * Skill 变更事件
 */
@Getter
public class SkillChangeEvent extends ApplicationEvent {

    /**
     * 变更的 Skill ID
     */
    private final String skillId;

    /**
     * 变更类型
     */
    private final ChangeType changeType;

    /**
     * 变更前的 Skill 快照（可选）
     */
    private final SkillSnapshot oldSnapshot;

    /**
     * 变更后的 Skill 快照（可选）
     */
    private final SkillSnapshot newSnapshot;

    /**
     * 变更时间
     */
    private final LocalDateTime changedAt;

    /**
     * 变更来源
     */
    private final String changedBy;

    public SkillChangeEvent(Object source, String skillId, ChangeType changeType,
                            SkillSnapshot oldSnapshot, SkillSnapshot newSnapshot,
                            String changedBy) {
        super(source);
        this.skillId = skillId;
        this.changeType = changeType;
        this.oldSnapshot = oldSnapshot;
        this.newSnapshot = newSnapshot;
        this.changedAt = LocalDateTime.now();
        this.changedBy = changedBy;
    }

    /**
     * 变更类型枚举
     */
    public enum ChangeType {
        /**
         * 新增 Skill
         */
        CREATED,

        /**
         * 更新 Skill（参数、版本等）
         */
        UPDATED,

        /**
         * 禁用 Skill
         */
        DISABLED,

        /**
         * 启用 Skill
         */
        ENABLED,

        /**
         * 删除 Skill
         */
        DELETED,

        /**
         * 版本升级
         */
        VERSION_UPGRADED
    }

    /**
     * Skill 快照
     */
    @Data
    @Builder
    public static class SkillSnapshot {
        private String skillId;
        private String name;
        private String version;
        private boolean enabled;
        private List<ParamDefinition> inputParams;
        private List<ParamDefinition> outputParams;

        @Data
        @Builder
        public static class ParamDefinition {
            private String name;
            private String type;
            private boolean required;
            private Object defaultValue;
        }
    }
}
```

---

### 3.2 SkillChangeListener（事件监听器）

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `SkillChangeListener.java` | Skill 变更监听器 | 待完成 |

#### 监听器实现

```java
/**
 * Skill 变更事件监听器
 * 当 Skill 发生变更时，自动检查相关节点的兼容性
 */
@Component
@Slf4j
@RequiredArgsConstructor
public class SkillChangeListener {

    private final WorkflowNodeMapper nodeMapper;
    private final SkillCompatibilityChecker skillChecker;
    private final SkillService skillService;

    /**
     * 监听 Skill 变更事件
     */
    @EventListener
    @Async("workflowTaskExecutor")
    public void onSkillChange(SkillChangeEvent event) {
        String skillId = event.getSkillId();
        ChangeType changeType = event.getChangeType();

        log.info("收到 Skill 变更事件: skillId={}, changeType={}", skillId, changeType);

        try {
            // 根据变更类型处理
            switch (changeType) {
                case CREATED:
                    // 新增 Skill，无需处理
                    log.debug("Skill 新增，无需处理兼容性检查");
                    break;

                case UPDATED:
                case VERSION_UPGRADED:
                    // 更新或版本升级，检查兼容性
                    handleSkillUpdate(skillId, event);
                    break;

                case DISABLED:
                    // 禁用，标记所有相关节点为不兼容
                    handleSkillDisable(skillId);
                    break;

                case ENABLED:
                    // 启用，重新检查兼容性
                    handleSkillEnable(skillId);
                    break;

                case DELETED:
                    // 删除，标记所有相关节点为无效
                    handleSkillDelete(skillId);
                    break;

                default:
                    log.warn("未知的变更类型: {}", changeType);
            }
        } catch (Exception e) {
            log.error("处理 Skill 变更事件失败: skillId={}", skillId, e);
        }
    }

    /**
     * 处理 Skill 更新
     */
    private void handleSkillUpdate(String skillId, SkillChangeEvent event) {
        // 1. 查找所有使用该 Skill 的节点
        List<WorkflowNodeEntity> nodes = nodeMapper.findBySkillId(skillId);

        if (nodes.isEmpty()) {
            log.debug("没有节点使用该 Skill: {}", skillId);
            return;
        }

        log.info("开始检查 {} 个节点的兼容性", nodes.size());

        // 2. 逐个检查节点兼容性
        int updatedCount = 0;
        int needsUpdateCount = 0;
        int incompatibleCount = 0;

        for (WorkflowNodeEntity node : nodes) {
            try {
                // 执行兼容性检查
                CompatibilityStatus status = skillChecker.checkNode(node);

                // 更新节点状态
                node.setCompatibilityStatus(status.name());
                nodeMapper.updateById(node);

                // 统计
                switch (status) {
                    case COMPATIBLE:
                        updatedCount++;
                        break;
                    case NEEDS_UPDATE:
                        needsUpdateCount++;
                        break;
                    case INCOMPATIBLE:
                    case INVALID:
                        incompatibleCount++;
                        break;
                }

                log.debug("节点 {} 兼容性检查结果: {}", node.getNodeName(), status);

            } catch (Exception e) {
                log.error("检查节点兼容性失败: nodeId={}", node.getId(), e);
                node.setCompatibilityStatus(CompatibilityStatus.INVALID.name());
                nodeMapper.updateById(node);
                incompatibleCount++;
            }
        }

        log.info("兼容性检查完成: 兼容={}, 需更新={}, 不兼容={}",
                updatedCount, needsUpdateCount, incompatibleCount);

        // 3. 如果有不兼容的节点，发送通知（可选）
        if (incompatibleCount > 0 || needsUpdateCount > 0) {
            sendCompatibilityNotification(skillId, nodes, needsUpdateCount, incompatibleCount);
        }
    }

    /**
     * 处理 Skill 禁用
     */
    private void handleSkillDisable(String skillId) {
        List<WorkflowNodeEntity> nodes = nodeMapper.findBySkillId(skillId);

        for (WorkflowNodeEntity node : nodes) {
            node.setCompatibilityStatus(CompatibilityStatus.INCOMPATIBLE.name());
            nodeMapper.updateById(node);
        }

        log.info("Skill 已禁用，已标记 {} 个节点为不兼容", nodes.size());

        // 发送通知
        if (!nodes.isEmpty()) {
            sendDisableNotification(skillId, nodes);
        }
    }

    /**
     * 处理 Skill 启用
     */
    private void handleSkillEnable(String skillId) {
        List<WorkflowNodeEntity> nodes = nodeMapper.findBySkillId(skillId);

        int compatibleCount = 0;
        int needsUpdateCount = 0;

        for (WorkflowNodeEntity node : nodes) {
            CompatibilityStatus status = skillChecker.checkNode(node);
            node.setCompatibilityStatus(status.name());
            nodeMapper.updateById(node);

            if (status == CompatibilityStatus.COMPATIBLE) {
                compatibleCount++;
            } else if (status == CompatibilityStatus.NEEDS_UPDATE) {
                needsUpdateCount++;
            }
        }

        log.info("Skill 已启用，兼容性检查完成: 兼容={}, 需更新={}", compatibleCount, needsUpdateCount);
    }

    /**
     * 处理 Skill 删除
     */
    private void handleSkillDelete(String skillId) {
        List<WorkflowNodeEntity> nodes = nodeMapper.findBySkillId(skillId);

        for (WorkflowNodeEntity node : nodes) {
            node.setCompatibilityStatus(CompatibilityStatus.INVALID.name());
            nodeMapper.updateById(node);
        }

        log.info("Skill 已删除，已标记 {} 个节点为无效", nodes.size());

        // 发送通知
        if (!nodes.isEmpty()) {
            sendDeleteNotification(skillId, nodes);
        }
    }

    /**
     * 发送兼容性变更通知
     */
    private void sendCompatibilityNotification(String skillId,
                                                List<WorkflowNodeEntity> nodes,
                                                int needsUpdateCount,
                                                int incompatibleCount) {
        // TODO: 实现通知逻辑
        // 可以通过 WebSocket、邮件、站内信等方式通知相关用户
        log.info("发送兼容性变更通知: skillId={}, 需更新={}, 不兼容={}",
                skillId, needsUpdateCount, incompatibleCount);
    }

    /**
     * 发送禁用通知
     */
    private void sendDisableNotification(String skillId, List<WorkflowNodeEntity> nodes) {
        // TODO: 实现通知逻辑
        log.info("发送 Skill 禁用通知: skillId={}, 影响节点数={}", skillId, nodes.size());
    }

    /**
     * 发送删除通知
     */
    private void sendDeleteNotification(String skillId, List<WorkflowNodeEntity> nodes) {
        // TODO: 实现通知逻辑
        log.info("发送 Skill 删除通知: skillId={}, 影响节点数={}", skillId, nodes.size());
    }
}
```

---

### 3.3 SkillService 集成

需要在 Skill 模块的 SkillService 中发布事件。

#### SkillService 集成示例

```java
@Service
@RequiredArgsConstructor
public class SkillServiceImpl implements SkillService {

    private final ApplicationEventPublisher eventPublisher;
    private final SkillMapper skillMapper;

    /**
     * 更新 Skill
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void updateSkill(String skillId, SkillUpdateRequest request) {
        // 1. 获取变更前的快照
        SkillEntity oldSkill = skillMapper.selectById(skillId);
        SkillChangeEvent.SkillSnapshot oldSnapshot = buildSnapshot(oldSkill);

        // 2. 执行更新
        SkillEntity newSkill = doUpdate(skillId, request);

        // 3. 获取变更后的快照
        SkillChangeEvent.SkillSnapshot newSnapshot = buildSnapshot(newSkill);

        // 4. 确定变更类型
        ChangeType changeType = determineChangeType(oldSkill, newSkill);

        // 5. 发布事件
        SkillChangeEvent event = new SkillChangeEvent(
                this,
                skillId,
                changeType,
                oldSnapshot,
                newSnapshot,
                SecurityUtils.getCurrentUserId()
        );
        eventPublisher.publishEvent(event);

        log.info("Skill 更新完成，已发布变更事件: skillId={}", skillId);
    }

    /**
     * 禁用 Skill
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void disableSkill(String skillId) {
        SkillEntity skill = skillMapper.selectById(skillId);
        SkillChangeEvent.SkillSnapshot oldSnapshot = buildSnapshot(skill);

        skill.setEnabled(false);
        skillMapper.updateById(skill);

        SkillChangeEvent event = new SkillChangeEvent(
                this,
                skillId,
                ChangeType.DISABLED,
                oldSnapshot,
                null,
                SecurityUtils.getCurrentUserId()
        );
        eventPublisher.publishEvent(event);
    }

    /**
     * 启用 Skill
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void enableSkill(String skillId) {
        SkillEntity skill = skillMapper.selectById(skillId);
        SkillChangeEvent.SkillSnapshot oldSnapshot = buildSnapshot(skill);

        skill.setEnabled(true);
        skillMapper.updateById(skill);

        SkillChangeEvent event = new SkillChangeEvent(
                this,
                skillId,
                ChangeType.ENABLED,
                oldSnapshot,
                null,
                SecurityUtils.getCurrentUserId()
        );
        eventPublisher.publishEvent(event);
    }

    /**
     * 删除 Skill
     */
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void deleteSkill(String skillId) {
        SkillEntity skill = skillMapper.selectById(skillId);
        SkillChangeEvent.SkillSnapshot oldSnapshot = buildSnapshot(skill);

        skillMapper.deleteById(skillId);

        SkillChangeEvent event = new SkillChangeEvent(
                this,
                skillId,
                ChangeType.DELETED,
                oldSnapshot,
                null,
                SecurityUtils.getCurrentUserId()
        );
        eventPublisher.publishEvent(event);
    }

    /**
     * 构建快照
     */
    private SkillChangeEvent.SkillSnapshot buildSnapshot(SkillEntity skill) {
        if (skill == null) return null;

        return SkillChangeEvent.SkillSnapshot.builder()
                .skillId(skill.getId())
                .name(skill.getName())
                .version(skill.getVersion())
                .enabled(skill.isEnabled())
                .inputParams(parseParamDefinitions(skill.getInputParams()))
                .outputParams(parseParamDefinitions(skill.getOutputParams()))
                .build();
    }

    /**
     * 确定变更类型
     */
    private ChangeType determineChangeType(SkillEntity oldSkill, SkillEntity newSkill) {
        if (!oldSkill.getVersion().equals(newSkill.getVersion())) {
            return ChangeType.VERSION_UPGRADED;
        }
        return ChangeType.UPDATED;
    }

    private List<SkillChangeEvent.SkillSnapshot.ParamDefinition> parseParamDefinitions(String json) {
        // JSON 解析逻辑
        // ...
        return Collections.emptyList();
    }
}
```

---

### 3.4 异步配置

| 序号 | 文件 | 说明 | 状态 |
|------|------|------|------|
| 1 | `AsyncConfig.java` | 异步任务配置 | 待完成 |

#### 异步配置实现

```java
/**
 * 异步任务配置
 */
@Configuration
@EnableAsync
public class AsyncConfig {

    /**
     * 工作流任务执行器
     */
    @Bean("workflowTaskExecutor")
    public Executor workflowTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(2);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("workflow-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

---

### 3.5 Mapper 方法补充

需要在 `WorkflowNodeMapper` 中添加按 SkillId 查询的方法。

#### Mapper 接口补充

```java
public interface WorkflowNodeMapper extends BaseMapper<WorkflowNodeEntity> {

    /**
     * 根据 SkillId 查询节点列表
     */
    List<WorkflowNodeEntity> findBySkillId(@Param("skillId") String skillId);

    /**
     * 批量更新兼容性状态
     */
    int batchUpdateCompatibilityStatus(@Param("nodeIds") List<Long> nodeIds,
                                        @Param("status") String status);
}
```

#### Mapper XML 补充

```xml
<!-- WorkflowNodeMapper.xml -->

<select id="findBySkillId" resultType="com.example.demo.workflow.entity.WorkflowNodeEntity">
    SELECT * FROM workflow_node
    WHERE skill_id = #{skillId}
    AND deleted = 0
</select>

<update id="batchUpdateCompatibilityStatus">
    UPDATE workflow_node
    SET compatibility_status = #{status},
        updated_at = NOW()
    WHERE id IN
    <foreach collection="nodeIds" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</update>
```

---

## 四、实施步骤

### Step 1：创建事件类
1. 创建 `SkillChangeEvent` 类
2. 定义变更类型枚举
3. 定义快照结构

### Step 2：实现事件监听器
1. 创建 `SkillChangeListener` 类
2. 实现各变更类型的处理逻辑
3. 添加日志和异常处理

### Step 3：配置异步执行
1. 创建 `AsyncConfig` 配置类
2. 配置线程池参数
3. 启用异步支持

### Step 4：补充 Mapper 方法
1. 添加 `findBySkillId` 方法
2. 添加 `batchUpdateCompatibilityStatus` 方法

### Step 5：集成到 SkillService
1. 在 Skill 更新方法中发布事件
2. 在 Skill 禁用/启用方法中发布事件
3. 在 Skill 删除方法中发布事件

### Step 6：测试验证
1. 单元测试
2. 集成测试
3. 事件发布/监听测试

---

## 五、代码结构

```
com.example.demo.workflow/
├── event/
│   └── SkillChangeListener.java
│
├── config/
│   └── AsyncConfig.java
│
└── mapper/
    └── WorkflowNodeMapper.java（补充方法）

com.example.demo.skill/
├── event/
│   └── SkillChangeEvent.java
│
└── service/
    └── impl/
        └── SkillServiceImpl.java（集成事件发布）
```

---

## 六、事件流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        Skill 模块                                │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    SkillService                          │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐    │   │
│  │  │ update  │  │ disable │  │ enable  │  │ delete  │    │   │
│  │  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘    │   │
│  └───────┼────────────┼────────────┼────────────┼─────────┘   │
│          │            │            │            │              │
│          ▼            ▼            ▼            ▼              │
│  ┌─────────────────────────────────────────────────────┐      │
│  │              ApplicationEventPublisher               │      │
│  └─────────────────────────┬───────────────────────────┘      │
└────────────────────────────┼───────────────────────────────────┘
                             │
                             │ SkillChangeEvent
                             │
                             ▼
┌────────────────────────────────────────────────────────────────┐
│                       工作流模块                                │
│  ┌─────────────────────────────────────────────────────┐      │
│  │               SkillChangeListener                    │      │
│  │  ┌───────────────────────────────────────────────┐  │      │
│  │  │ @EventListener @Async                         │  │      │
│  │  │ onSkillChange(SkillChangeEvent event)         │  │      │
│  │  └───────────────────────────────────────────────┘  │      │
│  │                        │                            │      │
│  │          ┌─────────────┼─────────────┐             │      │
│  │          ▼             ▼             ▼             │      │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐        │      │
│  │   │ UPDATED  │  │ DISABLED │  │ DELETED  │        │      │
│  │   └────┬─────┘  └────┬─────┘  └────┬─────┘        │      │
│  │        │             │             │               │      │
│  │        ▼             ▼             ▼               │      │
│  │  ┌──────────────────────────────────────────┐     │      │
│  │  │      SkillCompatibilityChecker           │     │      │
│  │  └──────────────────────────────────────────┘     │      │
│  │                        │                            │      │
│  │                        ▼                            │      │
│  │  ┌──────────────────────────────────────────┐     │      │
│  │  │      更新节点 compatibilityStatus        │     │      │
│  │  └──────────────────────────────────────────┘     │      │
│  └─────────────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────────────┘
```

---

## 七、验收标准

- [ ] `SkillChangeEvent` 类编译通过
- [ ] `SkillChangeListener` 类编译通过
- [ ] 异步配置正确
- [ ] Mapper 方法补充完成
- [ ] SkillService 集成完成
- [ ] 事件发布/监听测试通过
- [ ] 兼容性状态自动更新测试通过

---

## 八、测试用例

### 8.1 事件发布测试

```java
@SpringBootTest
class SkillChangeEventTest {

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    @Autowired
    private SkillChangeListener listener;

    @Test
    void testSkillUpdateEvent() {
        // 准备测试数据
        SkillChangeEvent event = new SkillChangeEvent(
                this,
                "skill-001",
                ChangeType.UPDATED,
                null,
                null,
                "test-user"
        );

        // 发布事件
        eventPublisher.publishEvent(event);

        // 验证（异步，需要等待）
        await().atMost(5, TimeUnit.SECONDS)
                .untilAsserted(() -> {
                    // 验证兼容性状态更新
                });
    }
}
```

### 8.2 监听器测试

```java
@SpringBootTest
class SkillChangeListenerTest {

    @Autowired
    private SkillChangeListener listener;

    @Autowired
    private WorkflowNodeMapper nodeMapper;

    @MockBean
    private SkillCompatibilityChecker skillChecker;

    @Test
    void testHandleSkillDisable() {
        // 准备测试数据
        WorkflowNodeEntity node = new WorkflowNodeEntity();
        node.setSkillId("skill-001");
        node.setCompatibilityStatus(CompatibilityStatus.COMPATIBLE.name());
        nodeMapper.insert(node);

        // 执行禁用处理
        listener.onSkillChange(new SkillChangeEvent(
                this,
                "skill-001",
                ChangeType.DISABLED,
                null,
                null,
                "test-user"
        ));

        // 验证状态更新
        await().atMost(5, TimeUnit.SECONDS)
                .untilAsserted(() -> {
                    WorkflowNodeEntity updated = nodeMapper.selectById(node.getId());
                    assertThat(updated.getCompatibilityStatus())
                            .isEqualTo(CompatibilityStatus.INCOMPATIBLE.name());
                });
    }
}
```

---

## 九、注意事项

1. **异步执行**：事件监听使用 `@Async` 避免阻塞主流程
2. **事务边界**：事件发布在事务提交后执行
3. **幂等性**：处理逻辑需要保证幂等
4. **异常处理**：异常不影响 Skill 模块主流程
5. **性能考虑**：批量更新使用批量操作

---

## 十、预计工作量

| 类型 | 数量 | 预计时间 |
|------|------|----------|
| 事件类 | 1 | 1h |
| 监听器 | 1 | 2h |
| 异步配置 | 1 | 0.5h |
| Mapper 补充 | 2 | 0.5h |
| SkillService 集成 | 4 方法 | 1h |
| 测试 | 4 | 2h |
| **总计** | **13** | **7h** |

---

## 十一、实施完成

Phase 5 完成后，工作流配置层重构的 **5 个阶段全部完成**。

## 整体工作量汇总

| 阶段 | 主要内容 | 预计时间 |
|------|----------|----------|
| Phase 1 | 基础层更新（Entity、DTO、Mapper、枚举） | 5h |
| Phase 2 | 服务层重构（CRUD、批量操作） | 10h |
| Phase 3 | 控制器层重构（RESTful API） | 8h |
| Phase 4 | 验证逻辑实现（4 个验证器） | 12h |
| Phase 5 | 事件集成（Skill 变更监听） | 7h |
| **总计** | **44 个文件** | **42h** |

---

## 十二、后续建议

1. **文档完善**：补充 API 文档和使用说明
2. **前端对接**：与前端团队对接 API 接口
3. **性能测试**：对批量操作进行性能测试
4. **监控告警**：添加兼容性状态变更的监控
