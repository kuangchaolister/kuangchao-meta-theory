flowchart TD
    IN["输入源<br/>1.用户原始Query<br/>2.外部intent_contract<br/>(来自JY记忆 / TJ推荐变体)"] --> MOD1

    MOD1["模块1：意图解析与契约生成层<br/>意图拆解、歧义消解、约束抽取<br/>输出标准化intent_contract<br/>携带：目标、约束、排除项、可信度阈值"] --> MOD2

    MOD2["模块2：搜索路由调度层<br/>根据intent_contract选择数据源集合<br/>多源调度、并发配额、熔断限流控制"] --> MOD3

    MOD3["模块3：多源原始信息采集层<br/>网页/知识库/API等多源原始数据获取<br/>保留原始来源溯源标记"] --> MOD4

    MOD4["模块4：信息提纯与可信度过滤层<br/>去重、噪声剔除、可信度打分<br/>污染标记；不可信内容打上隔离标记"] --> TMP_POOL

    TMP_POOL[("搜索临时结果池<br/>本次调用生命周期，不自动持久化")] --> MOD5

    MOD5["模块5：结果重排与契约适配网关<br/>依照intent_contract约束重排序<br/>裁剪过量信息；对齐输出契约格式"] --> OUT

    OUT["输出<br/>①提纯结构化搜索结果 → 上层模型<br/>②intent_contract → 供给JY记忆 / TJ推荐变体消费"]

    %% 故障降级分支（挂载到采集层）
    MOD3 -.-> note_err["故障降级：数据源超时/不可用<br/>不强行编造结果；返回可用子集+可信度告警"]

    %% 与记忆变体交互备注（挂载到输出端）
    OUT -.-> note_link["双向互通：JY输出意图驱动SS搜索<br/>SS结果送入JY仅进临时工作区<br/>不直接写入长期记忆骨架"]

    %% 样式美化
    classDef inputFill fill:#f9f,stroke:#333,stroke-width:2px;
    classDef stateFill fill:#bbf,stroke:#333,stroke-width:2px;
    classDef moduleFill fill:#fff,stroke:#333,stroke-width:1px;
    classDef outFill fill:#afa,stroke:#333,stroke-width:2px;
    classDef alertFill fill:#fdd,stroke:#d00,stroke-width:2px;

    class IN inputFill;
    class TMP_POOL stateFill;
    class MOD1,MOD2,MOD3,MOD4,MOD5 moduleFill;
    class OUT outFill;
    class note_err alertFill;