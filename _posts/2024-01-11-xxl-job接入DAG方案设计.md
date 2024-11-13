## **需求：**

平台与设备交互的一系列操作需要定制流程模板，并为其提供超时处理。

考虑是否在xxl-job统一调度平台增加工作流能力，提供通用化的工作流调度功能。

## **DAG工作流**

- 任务以有向边相连。
- 无环的，即不存在任务的循环依赖。
- 一个任务可能依赖于其他任务的输出结果或状态。
- 工作流描述了任务与任务之间的依赖关系，通过有向无环图（DAG）来描述该关系。

![Untitled](../images/xxl-job%E6%8E%A5%E5%85%A5DAG%20dbc64737eeb246069d60d63fa0c33220/Untitled.png)

### **特点：**

1. **有向无环性质：** 保证了任务的执行顺序不会形成循环，避免了死锁和无限循环等问题。
2. **任务依赖：** 每个任务都有其依赖的前驱任务，只有在前驱任务成功执行完成后，后继任务才能被触发执行。
3. **灵活性：** 可以通过定义任务之间的依赖关系来设计灵活的工作流程，支持复杂的任务编排和调度。

## **xxl-job现有架构**

![Untitled](../images/xxl-job%E6%8E%A5%E5%85%A5DAG%20dbc64737eeb246069d60d63fa0c33220/Untitled%201.png)

## **PowerJob**

### **工作流相关概念**

### **任务类型**

- Normal Job：无嵌套的任务
- WorkFlow Job：存在嵌套和流转的任务（工作流）

（工作流中的任务不能作为普通任务单独调度）

### **Workflow工作流**

```java
public class SaveWorkflowRequest implements Serializable {

    private Long id;

    /**
     * 工作流名称
     */
    private String wfName;
    /**
     * 工作流描述
     */
    private String wfDescription;

    /**
     * 所属应用ID（OpenClient不需要用户填写，自动填充）
     */
    private Long appId;

    /* ************************** 定时参数 ************************** */
    /**
     * 时间表达式类型，仅支持 CRON 和 API
     */
    private TimeExpressionType timeExpressionType;
    /**
     * 时间表达式，CRON/NULL/LONG/LONG
     */
    private String timeExpression;

    /**
     * 最大同时运行的工作流个数，默认 1
     */
    private Integer maxWfInstanceNum = 1;

    /**
     * ENABLE / DISABLE
     */
    private boolean enable = true;

    /**
     * 工作流整体失败的报警
     */
    private List<Long> notifyUserIds = Lists.newLinkedList();

    /** 点线表示法*/
    private PEWorkflowDAG dag;

    private LifeCycle lifeCycle;

}

/**
 ** DAG
 **/
public class PEWorkflowDAG implements Serializable {

    /**
     * Nodes of DAG diagram.
     */
    private List<Node> nodes;
    /**
     * Edges of DAG diagram.
     */
    private List<Edge> edges;

    /**
     * Point.
     */
    public static class Node implements Serializable {
        /**
         * node id
         *
         * @since 20210128
         */
        private Long nodeId;
        /* Instance running param, which is not required by DAG. */

        /**
         * note type
         *
         * @see WorkflowNodeType
         * @since 20210316
         */
        private Integer nodeType;
        /**
         * job id or workflow id (if this Node type is a nested workflow)
         *
         * @see WorkflowNodeType#NESTED_WORKFLOW
         */
        private Long jobId;
        /**
         * node name
         */
        private String nodeName;

        @JsonSerialize(using = ToStringSerializer.class)
        private Long instanceId;
        /**
         * for decision node, it is JavaScript code
         */
        private String nodeParams;

        private Integer status;
        /**
         * for decision node, it only be can "true" or "false"
         */
        private String result;
        /**
         * instanceId will be null if disable .
         */
        private Boolean enable;
        /**
         * mark node which disable by control node.
         */
        private Boolean disableByControlNode;

        private Boolean skipWhenFailed;

        private String startTime;

        private String finishedTime;

        public Node(Long nodeId) {
            this.nodeId = nodeId;
            this.nodeType = WorkflowNodeType.JOB.getCode();
        }

        public Node(Long nodeId, Integer nodeType) {
            this.nodeId = nodeId;
            this.nodeType = nodeType;
        }
    }

    /**
     * Edge formed by two node ids.
     */
    public static class Edge implements Serializable {

        private Long from;

        private Long to;
        /**
         * property,support for complex flow control
         * for decision node , it can be "true" or "false"
         */
        private String property;

        private Boolean enable;

        public Edge(long from, long to) {
            this.from = from;
            this.to = to;
        }

        public Edge(long from, long to, String property) {
            this.from = from;
            this.to = to;
            this.property = property;
        }
    }

    public PEWorkflowDAG(@Nonnull List<Node> nodes, @Nullable List<Edge> edges) {
        this.nodes = nodes;
        this.edges = edges == null ? Lists.newLinkedList() : edges;
    }
}
```

- WorkflowContext：工作流上下文，用于传递上下文参数（后覆盖前，配置限制长度）
- DAG：任务依赖关系
    - Node类型
        - JobNode：工作节点，需要实际调度的实例
        - ControlNode：js代码，通过WorkflowContext中取值判断to Node是否enable
        - NestedWorkflowNode：内嵌工作流节点
    - Edge
        - property：方法返回参数，用于更新上下文
        - enable：控制to Node 是否执行
        
        ![Untitled](../images/xxl-job%E6%8E%A5%E5%85%A5DAG%20dbc64737eeb246069d60d63fa0c33220/Untitled%202.png)
        

### **工作流相关功能**

### **WorkFlow工作流管理**

- 保存/修改工作流信息
- 深度复制工作流
- 获取工作流信息
- 删除工作流
- 启用/禁用工作流
- 立即执行工作流
- 新增/保存工作流节点

### **WorkFlow工作流实例管理**

- 停止工作流实例
- 获取工作流实例

### **WorkFlow工作流调度执行（Server端）**

### **创建工作流实例/开始任务**

### **定时调度的整体流程入口：**

![Untitled](../images/xxl-job%E6%8E%A5%E5%85%A5DAG%20dbc64737eeb246069d60d63fa0c33220/Untitled%203.png)

### **序列图**

![Untitled](../images/xxl-job%E6%8E%A5%E5%85%A5DAG%20dbc64737eeb246069d60d63fa0c33220/Untitled%204.png)

### **任务流转**

### **任务实例执行完成触发workflow特殊处理**

![Untitled](../images/xxl-job%E6%8E%A5%E5%85%A5DAG%20dbc64737eeb246069d60d63fa0c33220/Untitled%205.png)

![Untitled](../images/xxl-job%E6%8E%A5%E5%85%A5DAG%20dbc64737eeb246069d60d63fa0c33220/Untitled%206.png)

### **时序图**

![Untitled](../images/xxl-job%E6%8E%A5%E5%85%A5DAG%20dbc64737eeb246069d60d63fa0c33220/Untitled%207.png)

### **任务执行（Task Execution）**

执行器会根据任务定义的执行逻辑执行具体的任务，任务完成后上报任务实例给Server，本次预研不关注实际执行。

## **xxl-job接入DAG**

### **与PowerJob相比预期支持的能力**

| **功能块** | **功能** | **powerjob** | **xxljob** |
| --- | --- | --- | --- |
| **工作流管理API** | **保存/修改工作流信息** | **√** | **√** |
|  | **深度复制工作流** | **√** | **√** |
|  | **获取工作流信息** | **√** | **√** |
|  | **删除工作流** | **√** | **√** |
|  | **启用/禁用工作流** | **√** | **√** |
|  | **立即执行工作流** | **√** | **√** |
|  | **新增/保存工作流节点** | **√** | **√** |
| **工作流实例管理API** | **停止工作流实例** | **√** | **√** |
|  | **获取工作流实例** | **√** | **√** |
| **实际流转控制** | **工作流实例并行度控制** | √ | √ |
|  | **单任务超时控制** | × | **√** |
|  | **工作流超时控制** | × | × |
|  | **工作流失败告警** | **√** | **√** |
| **运维和使用** | **工作流日志聚合** | **√** | √ |
|  | **可视化工作流** | **√** | × |
|  | **在线编辑工作流** | √ | × |
|  |  |  |  |
|  |  |  |  |
|  |  |  |  |

### **整体流程**

![Untitled](../images/xxl-job%E6%8E%A5%E5%85%A5DAG%20dbc64737eeb246069d60d63fa0c33220/Untitled%208.png)

### **整体设计**

### **逻辑改动点**

![Untitled](../images/xxl-job%E6%8E%A5%E5%85%A5DAG%20dbc64737eeb246069d60d63fa0c33220/Untitled%209.png)

### **新增概念**

### **任务类型：工作流WorkFlow**

```java
public class Workflow {

    private Long id;

    private String wfName;

    private String wfDescription;

    /**
     * 所属应用appname
     */
    private String appName;

    /**
     * 工作流的DAG图信息（点线式DAG的json）
     */
    @Lob
    @Column
    private String peDAG;

    /* ************************** 定时参数 ************************** */
    /**
     * 时间表达式类型（CRON/API）
     */
    private Integer timeExpressionType;
    /**
     * 时间表达式，CRON/NULL/LONG/LONG
     */
    private String timeExpression;

    /**
     * 1 正常运行，2 停止（不再调度）
     */
    private Integer status;
    /**
     * 下一次调度时间
     */
    private Long nextTriggerTime;
}
```

### **任务属性：上下文参数WorkFlowContext**

```java
public class WorkflowContext {
    /**
     * 工作流实例 ID
     */
    private final Long  wfInstanceId;
    /**
     * 当前工作流上下文数据
     * 这里的 data 实际上等价于 {@link TaskContext} 中的 instanceParams
     */
    private final Map<String, String> data = Maps.newHashMap();
}
```

### **调度与执行**

依赖xxl-job原生的调度和执行，增加对Workflow的处理逻辑。

### **子任务触发方式**

通过现有的回调线程逻辑，依赖父任务、父任务输出更新的上下文、控制节点，判断子任务是否触发。

### **参数传递与上下游关系**

通过WorkFlowContext传递上下文参数。

通过ControlNode控制节点判断DAG中的边是否enable，来判断是否继续执行。

## **失败处理与容错性**

失败重试：元任务

阻塞策略：阻塞只能按元任务维度，如何配置元任务的阻塞策略是否需要规范。

## **扩展性与性能优化**

1. 增加的额外逻辑带来的调度延迟（可忽略，待观察）
2. 数据库锁性能瓶颈/负载均衡（未来是否优化）

## **版本计划**

### **1.0版本**

| **功能块** | **功能** | **xxljob** |
| --- | --- | --- |
| **工作流管理API** | **保存/修改工作流信息** | **√** |
|  | **深度复制工作流** | **√** |
|  | **获取工作流信息** | **√** |
|  | **删除工作流** | **√** |
|  | **启用/禁用工作流** | **√** |
|  | **立即执行工作流** | **√** |
|  | **新增/保存工作流节点** | **√** |
| **工作流实例管理API** | **停止工作流实例** | **√** |
|  | **获取工作流实例** | **√** |
|  | **单任务超时控制** | **√** |

### **2.0版本**

| **功能块** | **功能** | **xxljob** |
| --- | --- | --- |
| **运维和使用** | **工作流日志聚合** | √ |
| **实际流转控制** | **工作流超时控制** | √ |
|  | **工作流失败告警** | √ |
|  | **可视化工作流** | √ |
| **工作流实例管理** | **支持内联类型的工作流** | √ |
|  |  |  |

### **待定**

| **功能块** | **功能** | **xxljob** |
| --- | --- | --- |
|  |  |  |
| **运维和使用** | **在线编辑工作流** | √ |
| **实际流转控制** | **工作流实例并行度控制** | √ |
|  |  |  |

## **模板接入**

![Untitled](../images/xxl-job%E6%8E%A5%E5%85%A5DAG%20dbc64737eeb246069d60d63fa0c33220/Untitled%2010.png)