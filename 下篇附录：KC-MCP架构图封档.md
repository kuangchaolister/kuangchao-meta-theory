# 通用映像转换中间层 v3.1【封档固化版｜修复 Mermaid 图表】

> 文档状态：**封档固化 v3.1**
> 变更规则：
>
> 1. 🔒骨架强制部分：Schema、接口、契约、校验规则封档，后续修改必须走正式架构评审 + 更新版本号；
> 2. ✅取舍表征部分：算法、阈值、二选一方案不改动本文本体，全部通过【附录 B‑ADR 决策记录】记录落地选择，本文不修改；
> 3. 元理论引用：评审依据来自《狂潮・元理论 v2.4 + errata01 勘误页》，仅用于评审复盘，**不编译进业务运行代码**；
> 4. 不移植 REA 源码；仅参考业界 MCP 编排项目：可插拔适配器、证据账本、提供者生命周期管控思想。

> ⚠️图表使用重要提示：三张 Mermaid**必须分开单张复制粘贴到 mermaid.live 渲染，禁止三张一次性粘贴**，否则会解析报错。

> DP 修复变更汇总
>
> 1. 修复 Mermaid 语法 BUG：`subgraph XXX["标题"]` → `subgraph "标题"`，移除非法写法，删除不兼容的`direction TB`，兼容绝大多数 Mermaid 渲染器；
> 2. Mermaid‑1 顶层总图：补齐`bind_target`会话绑定入口，补充 fallback 适配器触发链路，完善动作执行完整双向流向；
> 3. Mermaid‑2 模块拆解图：新增完整 Provider 运行时、映像内存缓存子模块，全部骨架模块补齐无遗漏；
> 4. Mermaid‑3 完整时序：全覆盖 5 个 MCP 工具（`bind_target`/`observe_target`/`read_snapshot`/`perform_abstract_action`/`list_checkpoints`），增加异步`notification`推送完整时序，标注 MVP‑1 阶段`list_checkpoints`返回空数组约束。

---

## Mermaid‑1：整体系统数据流总图【顶层架构图】

预览

查看代码

```
flowchart LR
    subgraph "上层 Agent / MCP-Client（外部）"
        A["Agent 大模型业务逻辑层"]
    end
    subgraph "通用映像转换中间层 v3.1（MCP-Server）"
        B["MCP 接入模块<br />骨架强制<br />JSON-RPC / Notification"]
        C["Target 会话路由器<br />骨架强制<br />target_id 绑定与一致性校验"]
        D["Grounding 提供者注册表<br />骨架强制<br />usable 唯一启用条件"]
        E["Provider 运行时<br />骨架强制<br />spawn/terminate/独立超时"]
        subgraph "可插拔适配器集合 IAdapter（骨架强制）"
            AD1["WindowsUIA 适配器"]
            AD2["HarmonyAccessibility 适配器"]
            AD3["BrowserDOM 适配器"]
            AD_FB["MockFallbackAdapter 兜底适配器"]
        end
        F["映像组装与可信度标记模块（骨架强制）<br />RawEntity 转 Canonical<br />domain_payload key 校验<br />坐标系校验 / 冲突标记 / occluded 计算"]
        subgraph "存储层 系统工作区与证据账本（骨架强制）"
            MEM["映像内存缓存<br />snapshot_id 版本隔离"]
            DISK["磁盘证据账本<br />快照 / ActionLog / checkpoint<br />source_tag 外部或同源"]
        end
        NOTIFY["Notification 事件输出<br />provider-diagnostic / load-water-level"]
    end
    subgraph "异构外部系统（被管控对象）"
        S1["Windows 桌面 GUI"]
        S2["鸿蒙移动端"]
        S3["浏览器网页 DOM"]
        S_MOCK["Mock 模拟目标系统"]
    end
    A -->|"bind_target 会话绑定"| B
    A -->|"MCP JSON-RPC 请求"| B
    B --> C
    C --> D
    D --> E
    E -.->|"spawn 实例"| AD1
    E -.->|"spawn 实例"| AD2
    E -.->|"spawn 实例"| AD3
    E -.->|"spawn 实例"| AD_FB
    AD1 --> S1
    AD2 --> S2
    AD3 --> S3
    AD_FB --> S_MOCK
    AD1 -->|"RawEntity 数组"| F
    AD2 -->|"RawEntity 数组"| F
    AD3 -->|"RawEntity 数组"| F
    AD_FB -->|"RawEntity 数组"| F
    F -->|"Canonical 快照"| MEM
    F -->|"Canonical 快照"| DISK
    MEM -.->|"read_snapshot 按需读取<br />校验 target_id 一致"| B
    DISK -.->|"read_snapshot 按需读取<br />校验 target_id 一致"| B
    B -->|"返回 snapshot_id 与摘要"| A
    A -->|"AbstractAction"| B
    B --> C
    C --> D
    D --> E
    E -.->|"校验 action_type 后调用 perform_action"| AD1
    E -.->|"校验 action_type 后调用 perform_action"| AD2
    E -.->|"校验 action_type 后调用 perform_action"| AD3
    E -.->|"校验 action_type 后调用 perform_action"| AD_FB
    AD1 -->|"执行指令"| S1
    AD2 -->|"执行指令"| S2
    AD3 -->|"执行指令"| S3
    AD_FB -->|"执行指令"| S_MOCK
    AD1 -->|"ActionResult"| B
    AD2 -->|"ActionResult"| B
    AD3 -->|"ActionResult"| B
    AD_FB -->|"ActionResult"| B
    B -->|"ActionResult"| A
    E -->|"诊断与负载水位"| NOTIFY
    NOTIFY -.->|"MCP 异步事件"| A
    D -.->|"结构化全部不可用才降级"| AD_FB
    AD_FB -.->|"fallback_triggered 置真"| F
    classDef lock fill:#e6f7ff,stroke:#1890ff,stroke-width:2px
    classDef ext fill:#f5f5f5,stroke:#888888
    classDef fallback fill:#fff7e6,stroke:#faad14
    class B,C,D,E,F,MEM,DISK lock
    class A,S1,S2,S3,S_MOCK ext
    class AD_FB fallback
```



## Mermaid‑2：模块内部拆解图

预览

查看代码

```
flowchart TD
    subgraph "IAdapter 适配器接口（骨架强制）"
        IA_probe["probe 返回 usable 与 confidence"]
        IA_collect["collect_entities 返回 RawEntity 数组"]
        IA_perform["perform_action 返回 ActionResult"]
        IA_diag["get_diagnostics 返回 AdapterDiagnostic"]
        IA_destroy["destroy 独立超时兜底"]
    end
    subgraph "Provider 运行时（骨架强制）"
        PR_spawn["spawn_adapter 全局默认 deadline"]
        PR_term["terminate 强制回收"]
        PR_destroy["destroy 独立超时后强制 kill"]
        PR_iso["stdout/stderr 隔离"]
        PR_diag["get_diag 诊断采集"]
    end
    subgraph "映像组装与可信度标记模块（骨架强制）"
        AS_in["输入 RawEntity 数组"]
        AS_ck1["domain_payload key 属于 source_providers"]
        AS_ck2["坐标系不同跳过 IoU"]
        AS_proc["possible_duplicate 与 conflict_with 标记"]
        AS_occl["GUI 才计算 occluded，非 GUI 跳过"]
        AS_out["输出 CanonicalSystemImage v1"]
        AS_in --> AS_ck1 --> AS_ck2 --> AS_proc --> AS_occl --> AS_out
    end
    subgraph "证据账本 系统工作区（骨架强制）"
        L_write["write_snapshot 带 source_tag"]
        L_read["read_snapshot 校验 target_id"]
        L_chk["list_checkpoints"]
        L_actionlog["ActionLog 字段锁死，存储载体取舍"]
    end
    subgraph "映像内存缓存（骨架强制）"
        CA_put["cache_put 含 snapshot_id"]
        CA_get["cache_get 含 snapshot_id"]
        CA_rule["禁止仅 target_id 做 key"]
    end
    subgraph "内部预留 API（不对外 MCP 暴露，取舍表征）"
        HYP["write_hypothesis_snapshot 生成同源快照"]
        GATE["异源校验闸门"]
    end
    IA_collect --> AS_in
    IA_perform --> L_actionlog
    AS_out --> L_write
    AS_out --> CA_put
    L_write --> L_read
    L_write --> L_chk
    HYP --> GATE
    GATE --> L_write
    PR_spawn --> IA_probe
    PR_destroy --> IA_destroy
    classDef lock fill:#e6f7ff,stroke:#1890ff,stroke-width:2px
    classDef fallback fill:#fff7e6,stroke:#faad14
    class IA_probe,IA_collect,IA_perform,IA_diag,IA_destroy,PR_spawn,PR_term,PR_destroy,PR_iso,PR_diag,AS_ck1,AS_ck2,AS_proc,AS_occl,AS_out,L_write,L_read,L_chk,L_actionlog,CA_put,CA_get,CA_rule lock
    class HYP,GATE fallback
```



## Mermaid‑3：完整业务时序图（全部 MCP 工具全覆盖）

预览

查看代码

```
sequenceDiagram
    participant Agent as Agent 上层业务
    participant MCP as MCP 接入模块
    participant Router as Target 会话路由器
    participant Reg as Grounding 注册表
    participant Runtime as Provider 运行时
    participant AD as 适配器集合
    participant Assembler as 映像组装模块
    participant Ledger as 证据账本
    Note over Agent,Ledger: 会话绑定链路
    Agent->>MCP: bind_target(target_id)
    MCP->>Router: 绑定会话与 target_id
    Router->>Agent: 绑定成功，后续以绑定值为权威真值
    Note over Agent,Ledger: 观察链路
    Agent->>MCP: observe_target(target_id)
    MCP->>Router: 转发请求
    Router->>Router: 校验 target_id 与会话绑定一致
    Router->>Reg: 请求筛选可用适配器
    Reg->>AD: probe(target_id)
    Reg->>Runtime: spawn 实例化 usable 为真的适配器
    Runtime->>AD: collect_entities(target_id)
    AD->>Assembler: RawEntity 数组
    Assembler->>Assembler: domain‑payload 校验 / 坐标系校验 / 冲突标记 / occluded 规则
    Assembler->>Ledger: write_snapshot 带 source_tag="external"
    Ledger->>MCP: 返回 {snapshot_id,confidence_summary,entity_count,fallback_triggered}
    MCP->>Agent: 返回摘要，不含完整实体
    Note over Agent,Ledger: 读取完整快照链路
    Agent->>MCP: read_snapshot(snapshot_id)
    MCP->>Router: 转发请求
    Router->>Router: 校验快照 target_id 与会话绑定一致
    Router->>Ledger: 读取完整 Canonical 快照
    Ledger->>MCP: 返回完整 CanonicalSystemImage
    MCP->>Agent: 返回完整映像
    Note over Agent,Ledger: 执行动作链路
    Agent->>MCP: perform_abstract_action(target_id, action)
    MCP->>Router: 转发请求
    Router->>Router: 校验 target_id 一致 && action_type ∈ abstract_actions
    Router->>Reg: 获取可用适配器
    Reg->>Runtime: 调用适配器
    Runtime->>AD: perform_action(target_id, action)
    AD->>Ledger: 写入 ActionLog
    AD->>MCP: 返回 ActionResult
    MCP->>Agent: 返回 ActionResult，suggest_observe 为真
    Note over Agent,Ledger: checkpoint 查询链路（MVP‑1 返回空数组）
    Agent->>MCP: list_checkpoints(target_id)
    MCP->>Router: 校验 target_id 一致
    Router->>Ledger: 查询 checkpoint
    Ledger->>MCP: 返回 checkpoint 列表或空数组
    MCP->>Agent: 返回结果
    Note over Runtime,Agent: 异步 notification 推送链路
    Runtime->>MCP: provider‑diagnostic 或 load‑water‑level
    MCP-->>Agent: 异步事件推送，不阻塞主调用
```



---

# 文字版完整架构总览（Wiki 可直接粘贴，封档文档配套文本描述）

## 1、整体分层（自上而下）

1. **上层：Agent / MCP‑Client（外部业务层，不在本中间层管控范围）**
   - 职责：大模型推理、业务闭环逻辑（看‑执行‑校验）；
   - 约束：不直接访问底层系统 API；只调用中间层暴露的 5 个 MCP 工具。
2. **第一层：MCP 接入模块【入口网关】🔒骨架强制**
   - 接收 MCP JSON‑RPC 请求；分发到 Target 会话路由器；
   - 向外输出 MCP Notification 事件；
   - 🔒禁止：直接调用适配器、直接读写磁盘账本；大块 Canonical 映像**禁止塞入 RPC 报文**，只能返回 snapshot_id 摘要。
3. **第二层：Target 会话路由器【会话绑定与校验】🔒骨架强制**
   - 维护会话 ↔ target_id 绑定关系；绑定 target_id 为权威真值；
   - 🔒强制校验：工具入参 target_id 必须与会话绑定 target_id 一致，否则返回结构化错误；
   - 职责：路由请求，**不做适配器选择，不执行业务逻辑**。
4. **第三层：Grounding 提供者注册表【适配器调度】🔒骨架强制**
   - 维护全部可插拔适配器清单、优先级配置；
   - 调用各适配器`probe()`；`usable`布尔是启用适配器唯一判定条件；probe.confidence 只代表探测可信度，**不控制是否启用适配器**；
   - 结构化适配器全部`usable=false`才切换兜底 fallback 适配器。
5. **第四层：Provider 运行时【适配器生命周期沙盒】🔒骨架强制**
   - `spawn_adapter()`实例适配器；
   - 统一 deadline 总超时；`destroy()`独立超时，超时强制 kill 适配器实例；
   - stderr/stdout 隔离，适配器异常输出禁止污染 MCP RPC 流；
   - 职责：资源管控，**不修改业务映像数据**。
6. **第五层：可插拔适配器集合 IAdapter【外部系统适配层】🔒骨架强制**

> 所有适配器必须实现 IAdapter 接口；新增外部系统只写适配器，上层全部模块不用改动。

- `probe(target_id)` 判断本适配器是否可以处理该 target；
- `collect_entities(target_id)`输出`RawEntity[]`，**禁止自己组装 CanonicalSystemImage**；
- `perform_action()`把系统无关抽象动作原语翻译成外部系统真实指令，返回`ActionResult`；
- `get_diagnostics()`输出`AdapterDiagnostic`；
- `destroy()`释放句柄、连接资源，带独立超时兜底。
7. **第六层：映像组装 & 可信度标记模块【归一化心脏】🔒骨架强制**
   - 输入来自多个适配器的`RawEntity[]`；
   - 🔒骨架强制校验清单：
     1. `domain_payload`所有顶层 key 必须属于`source_providers`；非法 key 返回错误；
     2. 实体 bounds 仅当`coordinate_system`完全相同时才做 IoU 重叠比对；坐标系不同，跳过 IoU，仅依靠`entity_kind+semantic_label`匹配；
     3. 标记`possible_duplicate`疑似重复实体；标记`conflict_with[]`属性冲突实体；二者允许同时为 true；
     4. z_index=null 的非 GUI/DOM 实体**禁止计算 occluded**，visibility 只允许`visible｜invisible`；
   - 输出：完整`CanonicalSystemImage(v1)`标准快照。
8. **第七层：存储层｜系统工作区‑证据账本🔒骨架强制**
   - 内存缓存：携带`snapshot_id`版本标识，**禁止仅以 target_id 做 key，规避并发覆盖**；
   - 磁盘账本：持久化 Canonical 快照、ActionLog、checkpoint 标记；
   - MVP‑1：JSON 文件 + 串行写锁（仅 Demo，禁止生产）；MVP‑2 升级 sqlite；
   - 对外只提供读快照、写快照、列出 checkpoint；**不会主动向外推送大批量完整映像**。

> 内部预留 API（不对外 MCP 暴露）✅取舍表征
> `write_hypothesis_snapshot()`：写入`source_tag=agent_generated`同源假设快照；配合异源校验闸门，防止同源回流漂移；落地 MVP‑2/MVP‑3 记入 ADR。

9. **底层：异构外部系统（Windows/macOS/ 鸿蒙 / 浏览器 / IoT/Mock 模拟系统）**
   - 被适配器调用；不属于中间层代码本体。

## 2、对外 MCP 工具全集【🔒骨架强制】

1. `bind_target(target_id)`
2. `observe_target(target_id)` → 返回摘要`snapshot_id`，**不返回完整实体**
3. `read_snapshot(snapshot_id)` → 读取完整 Canonical 映像，校验快照 target_id 与会话绑定 target_id，不一致返回错误。
4. `perform_abstract_action(target_id,action)` → 执行系统无关抽象动作原语；校验`action_type`属于目标实体`abstract_actions`数组；返回`ActionResult`。
5. `list_checkpoints(target_id)` → MVP‑1 返回空数组；MVP‑2 完整实现 checkpoint。

> MCP Notification（异步推送，🔒骨架强制）
>
> - `provider‑diagnostic`：载荷 `AdapterDiagnostic`
> - `load‑water‑level`：载荷 `LoadWaterLevelNotification`

## 封档固化声明

```
# 通用映像转换中间层 v3.1｜封档固化
【版本】v3.1
【封档日期】2026‑11‑10
【引用元理论】狂潮・元理论 v2.4 + errata01勘误页（仅评审复盘，不编译运行）

🔒【骨架强制】：Schema、接口契约、模块职责、校验规则、字段语义，本版封档。
任何修改骨架强制内容，必须执行正式架构评审，并且递增主/次版本号，禁止就地修改本文。

✅【取舍表征】：匹配算法、IoU阈值、编辑距离、超时毫秒、存储载体选择、内部API落地阶段。
本文本体不填写具体参数、不选定二选一方案；所有落地决策，全部记录在【附录B ADR决策记录模板】，随工程版本归档，不改动本封档文档正文。

禁止行为：
1. 禁止把取舍表征的落地选择直接改写进本封档v3.1文档；
2. 禁止直接删除骨架强制字段、校验逻辑；变更必须走评审升级版本号；
3. 元理论仅作为评审审计工具，不能直接当做业务运行时代码硬断言。
```

## 配套附录清单

- 附录 A：狂潮元理论 v2.4 审计检查表（评审复盘使用）
- 附录 B：取舍表征 ADR 决策记录模板（工程落地填写归档）
