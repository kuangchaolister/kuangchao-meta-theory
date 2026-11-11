flowchart TD
    IN["输入源<br/>1.用户行为事件流<br/>2.外部intent_contract<br/>(来自JY记忆 / SS搜索变体)"] --> MOD1

    %% 外部状态注入
    STATE_MEM[("读取外部偏好状态<br/>来自JY六层记忆骨架库，只读访问")] -.-> MOD1

    MOD1["模块1：行为意图解析层<br/>行为序列解析、偏好信号提取<br/>输出标准化intent_contract<br/>携带偏好权重、过滤黑名单、多样性约束"] --> MOD2

    MOD2["模块2：候选池生成调度层<br/>基于intent_contract召回多组候选物料<br/>多路候选源并行召回，配额管控"] --> MOD3

    MOD3["模块3：候选过滤与风控闸门<br/>黑名单过滤、安全规则、多样性约束裁剪<br/>剔除冲突、低质量候选"] --> CAND_POOL

    CAND_POOL[("推荐候选中间池")] --> MOD4

    MOD4["模块4：重排与演化叠权层<br/>结合历史偏好惯性做打分重排<br/>异源信号抑制旧偏好；防止推荐固化内卷"] --> MOD5

    MOD5["模块5：推荐输出契约网关<br/>输出最终推荐序列<br/>封装标准化intent_contract"] --> OUT

    OUT["输出网关"]
    OUT --> OUT1["输出①推荐物料序列交付上层业务"]
    OUT --> OUT2["输出②intent_contract<br/>回传给JY记忆、SS搜索变体形成闭环"]

    %% 故障降级分支（挂载到候选召回层）
    MOD2 -.-> note_tj_err["故障降级：候选召回失败<br/>输出保底通用集合，上报监控<br/>禁止编造虚假候选"]

    %% 样式美化
    classDef inputFill fill:#f9f,stroke:#333,stroke-width:2px;
    classDef stateFill fill:#bbf,stroke:#333,stroke-width:2px;
    classDef moduleFill fill:#fff,stroke:#333,stroke-width:1px;
    classDef outFill fill:#afa,stroke:#333,stroke-width:2px;
    classDef alertFill fill:#fdd,stroke:#d00,stroke-width:2px;

    class IN inputFill;
    class STATE_MEM,CAND_POOL stateFill;
    class MOD1,MOD2,MOD3,MOD4,MOD5 moduleFill;
    class OUT,OUT1,OUT2 outFill;
    class note_tj_err alertFill;