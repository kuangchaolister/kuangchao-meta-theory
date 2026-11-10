# 打碎神话之后：通用映像转换中间层（KC-MCP) —— 架构层面的突围构想【工程可执行 v3.1 终稿｜带取舍表征标记】

> 修订说明：完成 第 1、2、3 轮全部审查闭环。
> 📌**取舍表征标记说明【补充责任归属】**
> 本文属于架构设计文档，非最终详细实现说明书。
>
> - 🔒【骨架强制约束｜架构师负责设计，变更必须走架构评审】：Schema 字段结构、接口契约、模块职责、错误返回行为、字段语义约束。必须遵守，保障跨适配器、MCP 客户端互操作性，不允许随意改动。
> - ✅【取舍表征｜MC 承接团队承担落地责任，选定方案后必须输出 ADR 决策记录归档】：架构仅提供候选方案、利弊、风险提示；**文档不固化算法、阈值、二选一实现路线、具体数值**，落地抉择、参数调优交给工程实现团队。选定的方案、参数、生效时间记入附录 B ADR 决策记录，避免不同部署实例出现语义分歧。
>
> 重要区分：
> 下面「业务工程实现」章节：研发可直接参考，产出可运行 MCP 服务；
> 【附录 A 狂潮・元理论 v2.4 审计检查表】：仅用于架构评审、代码 Review 阶段，属于分析工具，**不会编译进业务运行代码**。
> 【附录 B｜取舍表征 ADR 决策记录模板】：所有✅取舍表征项落地必须填写归档。
> 不移植 REA 源码，仅参考业界 MCP 编排项目成熟思想：可插拔适配器、证据账本、提供者生命周期工程思想。

## 前言

当前各类 Agent 普遍现象：每对接一类异构外部系统（桌面 GUI、浏览器 DOM、移动端鸿蒙、IoT 设备、二进制逆向），就要重新手写一整套：会话管理、状态快照、超时管控、错误降级、证据存储。大量逻辑重复复制粘贴，每个场景重新踩同一批边界 Bug。

Codex 内部存在同类映像‑动作转换层，但有三重枷锁：强绑定 GPT 基座、场景锁死 Windows/macOS 桌面、内部策略黑盒不可修改。

本中间层目标：实现独立 MCP 服务，作为**大模型 Agent ↔ 所有异构外部系统的隔离桥接层**。
核心双向逻辑：

1. 现实系统 → Canonical 标准映像：各适配器把五花八门外部私有数据，统一转换成一套标准化带可信度标记的映像，存入证据账本；大模型只阅读这套统一映像，不需要感知底层是什么硬件 / 软件。
2. Canonical 标准映像 → 现实系统：大模型输出**系统无关抽象动作原语**；中间层翻译成对应外部系统原生指令执行。

> ⚠️狂潮元理论 v2.4 定位：**架构评审审计工具，不是业务代码库。不会在运行时做硬断言；只在设计、评审、复盘故障的时候逐条核验。完整本体参阅独立封档文档，本篇工程文档不嵌入内部数学假说编号。**

## 一、核心数据定义：Canonical 标准系统映像（整套体系心脏）

> 🔒【骨架强制约束】所有适配器输出，**必须归一化为这份 JSON 结构**。不同外部系统（GUI 控件、DOM、IoT 寄存器）全部映射到这套字段。
> 字段设计取舍：只保留跨领域通用公共字段；领域特有内容塞进`domain_payload`扩展对象，不污染顶层 schema。

```
{
  "$schema":"canonical‑system‑image/v1",
  "target_id":"string",
  "snapshot_id":"uuid",
  "timestamp":1760000000,
  "confidence_summary":"high|medium|low|conflict",
  "source_providers":["adapter‑win‑uia","adapter‑mock‑fallback"],
  "fallback_triggered":false,
  "entities":[
    {
      "entity_id":"ent‑0001",
      "entity_kind":"control|node|register|widget",
      "semantic_label":"确认按钮",
      "confidence":"high|medium|low|conflict",
      "conflict_with":["ent‑0002"],
      "possible_duplicate":false,
      "is_enabled":true,
      "bounds":{
        "x":100,"y":200,"w":90,"h":32,
        "coordinate_system":"screen_px|page_css|device_unit"
      },
      "z_index":5,
      "visibility":"visible|invisible|occluded",
      "text_content":"确认",
      "abstract_actions":["click","input_text"],
      "domain_payload":{
        "adapter‑win‑uia":{"native_control_id":"..."},
        "adapter‑mock‑fallback":{"bbox_confidence":0.83}
      }
    }
  ],
  "evidence_meta":{
    "checkpoint_flag":false,
    "conflict_note":""
  }
}
```

**字段语义约束补充 🔒骨架强制**

1. `fallback_triggered`：**仅全部结构化适配器 probe 返回不可用，完全切换兜底通路时置为 true**；部分适配器失效但仍存在可用结构化适配器，此字段为 false，降级信息依靠`source_providers`、单实体 confidence 表达。
2. `possible_duplicate` / `conflict_with` 🔒骨架强制：
   - `possible_duplicate=true`：检测到多条原始记录**疑似指向同一个物理实体**，但属性没有直接矛盾，需要上层业务做合并确认；
   - `conflict_with[]`：**同一逻辑实体，来自不同适配器采集到的属性互相矛盾**；
   - 两个标记**允许同时置为 true**，代表既疑似重复，同时属性还互相冲突；上层业务需要做区分处理：疑似重复倾向合并，属性冲突标记为不可信。
3. `bounds.coordinate_system`适配器输出必填，组装模块**仅记录，不做跨坐标系转换**；🔒骨架强制实体对齐规则：**仅当两条实体`coordinate_system`完全相同时，才允许做 bounds IoU 重叠比对；坐标系不一致时跳过 IoU 计算，仅依靠`entity_kind + semantic_label`做实体匹配**。
4. ✅上层 Agent/MCP 客户端**禁止直接读取`domain_payload`下任意适配器子 key**；所有业务推理只能使用顶层标准化字段。`domain_payload`仅供适配器内部反向解析动作使用。
   - 🔒骨架强制：映像组装模块必须做校验：`domain_payload`所有顶层 key 必须全部存在于`source_providers`数组，出现未声明适配器 key 直接返回结构化错误。
5. `z_index`：**🔒骨架强制**，非 GUI/DOM 领域填`null`，不可当作数值 0 处理。
6. `visibility`：🔒骨架强制，适配器只输出`visible｜invisible`；`occluded`由映像组装模块合并多实体信息后计算生成；**非 GUI/DOM 领域直接跳过 occluded 计算，visibility 仅允许`visible|invisible`两个取值**。
7. `timestamp`🔒骨架强制：固定为 Unix 时间戳**毫秒**，统一跨组件时序对齐。

> ✅【取舍表征｜交由 MC 承接团队选定实现，落地写入 ADR】
>
> 1. `confidence_summary`聚合规则：架构要求：存在任意`conflict`实体则顶层 summary=`conflict`；其余聚合逻辑（按实体最低置信度 / 加权统计）、置信阈值，候选多种方案，利弊风险文档备注，团队选定。
> 2. 实体对齐合并算法：
>    - 必须具备能力：多适配器 RawEntity 做实体匹配，识别疑似重复实体打上`possible_duplicate`标记；
>    - 候选匹配维度集合：`entity_kind`、同坐标系下 bounds IoU 重叠、semantic_label 字符串编辑距离；
>    - IoU 阈值、编辑距离阈值、是否允许部分维度不匹配，全部为可配置参数，架构不写死固定数值。
>    - 风险提示：阈值过宽产生误合并；阈值过严产生大量重复实体。
> 3. `entity_id`生成策略二选一候选：
>
> > 架构不强制二选一，实现团队选定，同时必须在对外文档写明该 ID 策略带来的业务约束，记入 ADR。
>
> - 路线 A：稳定 ID：`hash(entity_kind + semantic_label + 归一化bounds)`，追求跨快照同一物理实体尽量复用 ID；风险：界面微小变化 hash 漂移，ID 失效。
> - 路线 B：**快照局部临时 ID**：entity_id 仅在当前 snapshot 内部有效；上层 Agent 禁止跨快照复用 entity_id，必须依靠`semantic_label+bounds`重定位实体。风险：上层业务编码需要额外约束。
> 4. ✅【取舍表征｜交由 MC 承接团队选定实现】assembler 校验项：当`fallback_triggered=true`，校验`source_providers`不包含任何结构化适配器标识；属于质量增强校验，不阻塞 MVP‑1 运行。

### RawEntity（适配器原始输出，组装模块输入）🔒骨架强制

> RawEntity：适配器`collect_entities()`返回原始中间数据；**缺少 entity_id、confidence、conflict 标记、possible_duplicate**；映像组装模块统一补齐这些字段生成完整 Canonical 实体。

```
{
  "entity_kind":"control|node|register|widget",
  "semantic_label":"确认按钮",
  "is_enabled":true,
  "bounds":{
    "x":100,"y":200,"w":90,"h":32,
    "coordinate_system":"screen_px|page_css|device_unit"
  },
  "z_index":5,
  "visibility":"visible|invisible",
  "text_content":"确认",
  "abstract_actions":["click","input_text"],
  "domain_payload":{}
}
```

### 抽象动作原语（大模型输出，系统无关）🔒骨架强制

> 大模型不输出`mouse_click(x,y)`系统原生调用；输出抽象动作，适配器翻译成真实系统指令。

```
{
  "action_id":"act‑uuid‑01",
  "entity_id":"ent‑0001",
  "action_type":"click|input_text|scroll|read_value|write_value",
  "params":{"text":"hello world"}
}
```

> 🔒骨架强制：`action_id`由上层 Agent 生成，**必须全局 UUID 唯一**；中间层不做幂等校验、不复用 id，幂等逻辑全部由上层 Agent 保障。

### ActionResult（perform_action 返回结果）🔒骨架强制

```
{
  "success":true|false,
  "error_code":"",
  "error_message":"",
  "suggest_observe":true|false
}
```

> 🔒骨架强制设计取舍：中间层**不会自动调用 observe_target 触发状态采集**；「看‑执行‑校验闭环」属于上层 Agent 业务逻辑。`suggest_observe`仅作为提示，告诉 Agent 执行完成后建议重新拉取系统快照，不会隐式发起 MCP 调用。

### AdapterDiagnostic 适配器诊断结构 🔒骨架强制

```
{
  "adapter_name":"string",
  "level":"info|warn|error",
  "code":"string",
  "message":"string",
  "timestamp":1760000000
}
```

### ActionLog 动作执行日志结构 🔒骨架强制【字段锁死；存储位置取舍表征】

```
{
  "action_id":"string",
  "entity_id":"string",
  "snapshot_id":"string",
  "timestamp":1760000000,
  "result":"ActionResult完整对象"
}
```

> ✅【取舍表征｜交由 MC 承接团队选定实现】存储载体二选一：
>
> - 方案 A：扩展存入`snapshot.evidence_meta`扩展字段，复用现有快照存储；优点：不用新增表；缺点：快照体积膨胀。
> - 方案 B：独立 ActionLog 持久集合；优点：查询动作历史高效；缺点：新增一套存储。
>
> > ActionLog 内部字段结构🔒骨架强制不可改动，仅存储载体交给团队选择，选定记入 ADR。

### LoadWaterLevelNotification 负载水位通知载荷 🔒骨架强制

```
{
  "active_adapters":number,
  "queue_depth":number,
  "cpu_pct":number,
  "timestamp":1760000000
}
```

## 二、模块总览 + 每个模块对外职责与接口契约

表格

| 模块名称                    | 核心职责                                                                                                                             | 对外暴露接口（伪函数）                                                                                  | 禁止行为（越界黑名单）                                                   |
| ----------------------- | -------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| MCP 接入模块                | MCP‑JSON‑RPC 报文收发，对外暴露全部工具、notification 通知                                                                                       | `handle_rpc(request)`、`emit_notification(event)`                                             | 禁止直接调用适配器；禁止直接读写磁盘证据账本；大块映像数据不能塞进 rpc 报文                      |
| Target 会话路由器            | 维护 MCP 会话 ↔ target_id 绑定；给请求附加 target 上下文向下路由                                                                                    | `bind_session(session_id,target_id)`、`route_request(context, payload)`                       | 不做适配器选择逻辑；不执行业务逻辑；不能直接调用外部系统                                  |
| Grounding 提供者注册表        | 维护适配器列表；**确定性优先级策略配置**：优先结构化适配器，失效自动切换兜底适配器                                                                                      | `select_providers(target_id)`、`list_available_adapters()`                                    | 不管理适配器子进程生命周期；不修改 Canonical 映像内容                              |
| Provider 运行时            | 管控适配器完整生命周期：deadline 超时、destroy 独立超时、stdout/stderr 隔离、资源强制回收、诊断采集                                                                | `spawn_adapter(name,config)`、`terminate(inst_id)`、`get_diag(inst_id)`                        | 禁止直接操作 MCP stdio 报文；stderr 诊断隔离，禁止泄漏进 MCP RPC 流               |
| 系统工作区 + Canonical 证据账本  | 持久化映像快照、checkpoint；仅被动提供快照读取，**不会主动向外推送完整数据**                                                                                    | `write_snapshot(snap,source_tag)`、`read_snapshot(snapshot_id)`、`list_checkpoints(target_id)` | 不主动向 MCP 客户端推送大批量快照；业务逻辑不能直接绕过账本写磁盘；MVP‑1 JSON 实现必须串行写队列      |
| 映像组装 & 可信度标记模块          | 把多适配器返回 RawEntity [] 合并为标准 Canonical 映像；实体匹配、计算 confidence、识别冲突、标记 possible_duplicate；domain_payload key 校验；非 GUI 跳过 occluded 计算 | `assemble_entities(raw_list:[]) → CanonicalSystemImage`                                      | 不发起 IO；不调用外部适配器；只做数据转换、冲突标记；坐标系不同跳过 IoU 比对                    |
| 可插拔适配器集合（IAdapter 抽象接口） | 读取外部系统；外部私有数据 ↔ RawEntity 双向转换                                                                                                   | 见 IAdapter 接口契约                                                                              | 禁止直接访问 MCP 通道；生命周期全部交给 Provider 运行时管理                         |
| 映像内存缓存                  | 内存高速缓存快照，减少重复拉取外部系统；磁盘持久化交给证据账本                                                                                                  | `cache_put(target_id,snapshot_id,snap)`、`cache_get(target_id,snapshot_id)`                   | 不负责磁盘持久化写入；缓存必须携带 snapshot_id 版本标识，禁止仅 target_id 做 key，规避并发覆盖 |

> 🔒骨架强制：**所有适配器必须实现统一 IAdapter 抽象接口**。新增一个外部系统，只实现该接口，上层全部模块完全不用改动。

## 三、适配器强制抽象接口 IAdapter【伪代码，开发直接参考】

```
interface IAdapter {
  // 1.探测：判断当前target是否本适配器可以处理；usable是启用唯一判定条件
  probe(target_id: string): Promise<{usable: boolean, confidence: "high"|"low"}>;

  // 2.采集：返回RawEntity[]原始实体，不组装完整Canonical映像
  collect_entities(target_id: string): Promise<RawEntity[]>;

  // 3.执行抽象动作，翻译成外部系统真实指令
  perform_action(target_id:string, action:AbstractAction): Promise<ActionResult>;

  // 4.获取适配器诊断、警告信息
  get_diagnostics(): Promise<AdapterDiagnostic>;

  // 5.销毁清理：🔒骨架强制，带独立超时；超时runtime强制杀死实例
  destroy(): Promise<void>;
}
```

### 行为约束🔒骨架强制

1. `probe()`作为 gatekeeper：**`usable`布尔值是是否启用适配器的唯一判定条件**；probe 返回的 confidence 仅表示探测结论可信度，**不参与适配器启用判定**。
2. `collect_entities()`输出`RawEntity[]`，**不要自己组装完整 CanonicalSystemImage**；组装统一交给【映像组装 & 可信度标记模块】，保证所有适配器输出归一化。
3. 适配器禁止自己做重试、无限循环；超时、终止全部交给上层 Provider 运行时的 deadline 管控。

> ✅【取舍表征｜交由 MC 承接团队选定实现】
>
> 1. spawn_adapter 全局默认 deadline 毫秒数值、destroy 独立超时毫秒数值，可配置；架构只规定存在这两套超时机制，不写死时间。
> 2. probe 执行策略：候选：并行 probe 带全局总超时 / 串行 probe；优先推荐并行；串行作为备选；总超时数值可配置。

## 四、完整端到端业务时序（文字时序图）

### 场景 A：正常通路（优先结构化适配器可用）

> 触发：上层 Agent 调用 MCP 工具`observe_target(target_id)`想要获取外部系统状态

1. MCP 接入模块收到 MCP‑JSON‑RPC 请求，校验报文，交给 Target 会话路由器。
2. 🔒骨架强制：路由器取出会话绑定的权威`target_id`；**入参 target_id 与会话绑定不一致直接返回结构化错误**。带上上下文下发 Grounding 提供者注册表。
3. 注册表调用各个适配器`probe(target_id)`，筛选可用适配器，按优先级排序（结构化适配器优先）。

> ✅【取舍表征｜交由 MC 承接团队选定实现】probe 并行 / 串行，总超时配置。

4. Provider 运行时 spawn 实例化选中适配器集合。
5. 调用适配器`collect_entities(target_id)`，拿到多份`RawEntity[]`。
6. 全部原始实体送入映像组装模块：
   - domain_payload key 合法性校验；
   - 同坐标系才执行 IoU 比对；
   - 补齐 entity_id、confidence、`possible_duplicate`、`conflict_with`；
   - GUI/DOM 计算 occluded；非 GUI 跳过 occluded；
   - 生成完整`CanonicalSystemImage`快照。
7. 快照一式两份：写入内存缓存；写入证据账本，标记`source_tag="external"`，生成`snapshot_id`。
8. MCP 接入**禁止把完整大映像塞进 RPC 返回报文！** MCP 仅返回精简摘要：`{"snapshot_id":"uuid","confidence_summary":"high","entity_count":12,"fallback_triggered":false}`。
9. 🔒骨架强制：Agent 调用`read_snapshot(snapshot_id)`读取完整快照时，校验快照内部 target_id 与会话绑定 target_id，不一致返回错误。

### 场景 B：全部结构化适配器 probe 失败，切换兜底适配器

1‑3 步骤同上；全部结构化适配器`probe()`返回`usable:false`。
4. 注册表启用兜底适配器。
5‑7：生成 Canonical 快照，设置`fallback_triggered=true`写入账本。
8‑9：返回快照摘要给 Agent。

### 场景 C：执行抽象动作

1. Agent 调用`perform_abstract_action(target_id,action)`。
2. 🔒骨架强制：路由校验 target_id 与会话绑定一致；同时校验`action.action_type`必须属于目标实体`abstract_actions`数组，不匹配返回结构化错误。
3. Provider 运行时拿到适配器实例，调用适配器`perform_action()`执行。
4. 返回`ActionResult`，其中`suggest_observe`置为 true 提示 Agent 重新采集快照。
5. ✅【取舍表征｜交由 MC 承接团队选定实现】按选定方案写入 ActionLog 动作日志；**不会自动调用 observe_target**。

## 五、对外暴露 MCP 工具集合（给大模型 Agent 调用）🔒骨架强制

> 中间层作为 MCP Server，向外只暴露 5 个核心工具，隐藏内部模块细节。

1. `bind_target(target_id)`

> MCP 会话绑定目标外部系统 target；会话绑定 target_id 为后续所有工具调用的权威真值。

2. `observe_target(target_id)`

> 触发采集外部系统，生成新 Canonical 快照。
> 校验：传入 target_id 必须与会话 bind_target 绑定保持一致，不一致返回结构化错误。
> 返回：`{snapshot_id,confidence_summary,entity_count,fallback_triggered:bool}`
> ⚠️**不返回完整实体列表，只返回快照 ID**。

3. `read_snapshot(snapshot_id)`

> 按需读取完整 CanonicalSystemImage 映像快照。
> 🔒骨架强制校验：快照 target_id 与会话绑定 target_id 必须匹配。

4. `perform_abstract_action(target_id,action)`

> 执行系统无关抽象动作原语。action 严格遵守 action json schema。返回`ActionResult`。

5. `list_checkpoints(target_id)`

> 🔒骨架强制：工具入口保留；✅【取舍表征｜MVP‑1 约定】MVP‑1 该工具直接返回空数组；checkpoint 完整恢复逻辑延后 MVP‑2 落地。

> 配套 MCP notification 异步推送🔒骨架强制：

- `provider‑diagnostic`：适配器警告、故障通知，载荷`AdapterDiagnostic`
- `load‑water‑level`：系统负载水位通知，载荷`LoadWaterLevelNotification`

> 内部 API（**不对外 MCP 暴露**）：`write_hypothesis_snapshot()`，用于写入`source_tag="agent_generated"`假设快照；同源回流闸门逻辑仅在内部 API 生效；公开 MCP 工具无法产生该类快照。
> ✅【取舍表征｜交由 MC 承接团队选定实现】内部 API 落地阶段 MVP‑2 / MVP‑3 二选一，记入 ADR。

## 六、分阶落地 MVP 规划

### MVP‑1｜最小可用原型，跑通完整数据流，零外部依赖

✅必须实现🔒骨架强制

1. CanonicalSystemImage、RawEntity、AbstractAction、ActionResult、ActionLog、AdapterDiagnostic、LoadWaterLevelNotification 全部 Schema；
2. MCP 接入模块，实现上面 5 个 MCP 工具；`list_checkpoints`返回空数组；
3. Target 会话路由器，全套 target_id 一致性校验；
4. Grounding 提供者注册表，usable 作为适配器启用唯一条件；
5. Provider 运行时：spawn/terminate、destroy 独立超时、deadline、stderr 隔离；
6. 映像组装模块：

> ✅【取舍表征｜MVP‑1 约束】实体匹配允许最简实现，只要可以产出`possible_duplicate`、`conflict`标记，完成三条测试用例即可；复杂编辑距离等逻辑可后续迭代，参数可配置。

- domain_payload key 合法性校验；
- 坐标系判断，不同坐标系跳过 IoU；
- possible_duplicate /conflict_with 标记；
- 非 GUI 跳过 occluded 计算；
7. 内存缓存；简易 JSON 证据账本，**强制实现串行单写队列（写锁），规避并发文件损坏；仅原型演示，禁止上生产**；
8. 适配器实现：`MockTestAdapter` + `MockFallbackAdapter`，模拟返回假数据，零外部依赖；
9. 完整端到端时序跑通：`observe_target → read_snapshot → perform_abstract_action`；
10. 降级通路：结构化适配器 probe 全部不可用时切换 fallback 适配器，设置`fallback_triggered=true`。
11. 三套必过测试用例：①正常结构化通路；②结构化全部失败走降级；③多 Mock 适配器输出矛盾实体，验证 conflict /possible_duplicate 标记。

❌MVP‑1 坚决不做

- 分布式多实例；
- 复杂 checkpoint 回滚高级逻辑；
- 真实 OCR / 截图视觉适配器（下移 MVP‑2）；
- 鉴权、沙盒（上层 Agent 运行环境承担，本层仅审计日志）。

### MVP‑2｜生产可用版本

✅新增交付

1. 证据账本替换为 sqlite 数据库；完善 checkpoint 完整恢复；`list_checkpoints`真正生效；

> ✅【取舍表征｜JSON 迁移 sqlite：迁移方案选定记入 ADR】

2. 实现第一批真实业务适配器（WindowsUIA、BrowserDOM 适配器）；
3. 注册表增强：适配器配置热加载、可配置优先级；
4. Provider 运行时增强：全局并发上限、任务排队（契约 5 能量带定义）；
5. MCP notification 完整实现故障、负载水位上报；
6. 缓存增加`snapshot_id`版本隔离，解决读写竞争；
7. ✅【取舍表征】可选择实现内部`write_hypothesis_snapshot` + 异源校验闸门；记入 ADR；
8. 完善全套访问审计日志（本层只留日志，鉴权交给上层）。

❌MVP‑2 不做：大规模集群分布式部署。

### MVP‑3｜完备商业增强版

1. 更多生态适配器：鸿蒙、安卓、IoT 设备适配器；
2. 分布式多实例支持，处理分裂态风险；
3. 高级冲突归并策略；
4. 完整可配置告警体系；
5. 性能指标监控、metrics。

## 七、同源回流漂移：异源校验闸门（内部 API，MCP 不对外暴露）

> 元理论提到的同源回流风险：Agent 把自己上一轮动作产生的 Canonical 映像写进证据账本，反复回流会造成偏移‑冲突‑漂移。

1. 账本区分两类快照来源：
   - `source=external`：来自外部适配器采集（真正外部系统，异源信号）
   - `source=agent_generated`：Agent 自己推理构造生成的映像（同源信号），**仅内部 API 可写入，MCP 客户端无入口**。
2. 闸门规则（可配置开关）：
   - 长任务迭代时，**不允许把 agent_generated 同源快照直接作为后续观测输入；必须重新调用`observe_target()`拉取来自外部适配器的 external 快照作为事实基准**。
   - agent_generated 快照可以存入账本做历史记录，但是禁止作为后续推理的事实依据。
3. notification 告警：当检测到业务逻辑试图拿`agent_generated`快照充当外部事实，推送诊断告警通知。

> 备注：这是运行时工程防护，不是 100% 根治，降低漂移概率。
> ✅【取舍表征｜交由 MC 承接团队选定实现】内部 API 落地 MVP‑2 / MVP‑3，记入 ADR；当前对外 MCP 工具不会产生该快照。

## 八、工程已知风险 & 坑清单

1. **适配器 probe 抖动坑**：部分系统接口瞬时报错，probe 时而 usable:true 时而 false。
   缓解：注册表层做 probe 结果滑动窗口防抖，不要单次 probe 失败立刻丢弃适配器。✅取舍表征，防抖参数记入 ADR。
2. **大快照 MCP 传输坑**：Canonical 映像实体很多，JSON 体积巨大。**严格遵守：observe 只返回 snapshot_id，不要返回完整实体；按需 read_snapshot**。禁止把几 MB 的快照塞进 MCP RPC 报文。🔒骨架强制。
3. **缓存并发读写竞争坑**：同一 target 并发多次 observe_target，内存缓存旧快照被新请求覆盖。
   缓解：缓存必须携带`snapshot_id`版本标识，不要只用 target_id 做 key。🔒骨架强制。
4. **适配器泄漏句柄坑**：适配器异常报错没有执行 destroy，句柄 / 连接僵尸残留。
   缓解：Provider 运行时无论成功失败，适配器实例离开作用域强制调用 destroy，加独立超时兜底。🔒骨架强制。
5. **domain_payload 逃逸风险**：代码审查必须拦截：上层业务逻辑直接读取 domain_payload。所有业务推理只允许读取顶层 Canonical 字段。🔒骨架强制。
6. **降级逻辑被绕过**：业务代码直接指定某一个适配器，绕过注册表优先级策略。
   缓解：MCP 工具层禁止传入 adapter 名字，只能传入 target_id，适配器选择全部由注册表内部完成。🔒骨架强制。
7. **账本无限膨胀**：大量快照持续写入磁盘。
   缓解：增加可配置快照 TTL，旧快照自动归档清理；保留 checkpoint 标记的快照不被清理。✅取舍表征，TTL 参数记入 ADR。
8. **跨坐标系实体匹配坑**：bounds IoU 只允许相同 coordinate_system 实体之间比对；不同坐标系仅使用文本标签匹配。🔒骨架强制。

## 九、明确本中间层**不负责什么**（划清边界，防止需求无限膨胀）

1. ❌**不是安全网关**：不实现文件 / 设备访问鉴权、沙盒隔离。鉴权、沙盒交给上层 Agent 运行环境；本中间层只做完整访问行为审计日志留存。（对应之前契约 3 数据配权限的分层分工）
2. ❌不会修复外部适配器、外部系统本身 Bug；只做隔离、报错、降级，不能修复下游系统问题。
3. ❌不做大模型推理。只负责映像转换、动作转发；模型推理完全在上层 Agent。
4. ❌MVP1‑MVP2 单实例架构，不处理分布式分裂态；分裂态风险只有 MVP3 多实例才需要处理。
5. ❌元理论里面的旺‑生‑休‑克‑死、六条动态契约，**不写进业务运行时代码做硬断言**，只在架构评审、故障复盘阶段使用。

> 冷峻收尾：架构图纸已经完备。骨架接口语义全部钉死；可变算法、阈值、二选一方案全部打上取舍表征标记。能不能落地，取决于第一个可运行适配器能不能跑通完整端到端链路。

---

# 附录 A｜通用映像转换中间层审计检查表（仅评审、故障复盘，不编译进业务代码）

> 引用说明：评审工具出自《狂潮・元理论 v2.4 + errata01 勘误页》独立文档。本篇工程文档不嵌入数学假说编号；涉及机理推演假说参阅原始封档文档。
> 边界声明：旺‑生‑休‑克‑死：借用五行意象描述组件耦合交互，不等同传统五行学说。

### 审计条目

1. **双流完整性审计**
- 权限‑状态流、边界‑执行流逻辑严格分离。大块状态映像禁止混杂进 MCP RPC 执行报文；采用 snapshot_id 按需读取模式。
- "停滞 / 无" 状态同样完成双流拆解。
2. **六条动态契约逐条核验**
- 信息配时序：所有快照携带`snapshot_id`版本标识；缓存并发版本隔离；domain_payload 来源 key 校验；
- 控制配轮询：具备任务并发上限、排队、超时熔断；
- 数据配权限：模块私有状态 / 公开状态边界清晰；所有外部访问行为审计留痕；鉴权沙盒交由上层；
- 一致就向前：适配器输出全部归一化为 Canonical 映像；冲突实体打上标记，禁止静默覆盖；不同坐标系实体匹配规则生效；
- 能量带定义：算力、适配器实例、IO 具备上限、过载降级策略；
- 状态要报备：故障、负载水位通过独立 notification 上报，不复用业务 RPC 作为唯一观测通道。
3. **五类运行工况边界梳理**
- 稳态、偏移态、冲突态正常触发条件；区分`possible_duplicate`与`conflict_with`语义；
- 分裂态：仅多实例分布式部署才触发，单实例无需处理；
- 崩溃态：适配器超时被终止，内存任务丢失，磁盘账本快照可恢复。
4. **旺‑生‑休‑克‑死组件耦合遍历**
- 逐个模块交互，完成五类耦合标记；识别存在「克、死」风险交互点，记录缓解手段。
5. **权限原子化协议逐条校验（每个模块、每个新增适配器）**
- 核心职责收敛单一；
- 私有状态：外部禁止读、禁止修改；
- 可公开状态：仅标准化快照、事件对外暴露；
- 允许接收输入白名单；
- 允许发出输出白名单；
- 越界黑名单明确禁止行为集合。
6. **同源‑异源风险专项审计**
- 识别同源回流点位；
- 异源校验闸门生效条件；区分有益结构同源，与带来漂移缺陷的结果同源。
7. **遗留风险登记（条件式描述，禁止无条件断言）**

> 格式：当【XX 条件触发】，则【发生 XX 现象】；缓解手段 XXX。

# 附录 B｜取舍表征 ADR 决策记录模板【所有✅取舍表征项落地必须归档】

> 目的：避免不同部署版本，对架构二选一方案、可配置阈值出现理解分歧；后续接手人员可以查阅历史决策。

```
ADR‑ID：________
模块/主题：________
候选方案列表：
  方案A：xxx，利弊简述
  方案B：xxx，利弊简述
最终选定方案：________
关键参数配置：________
为什么拒绝其他方案：________
生效版本：________
决策人员：________
日期：YYYY‑MM‑DD
备注：________
```

---

## 第三轮 11 项审查逐条处置汇总

表格

| 序号  | 审查点                                           | 处置方案                                                                   | 类型             |
| --- | --------------------------------------------- | ---------------------------------------------------------------------- | -------------- |
| ①   | 取舍表征缺少决策记录载体                                  | 新增附录 B ADR 模板；所有✅取舍表征落地必须归档 ADR                                        | 高优先级，补文档       |
| ②   | 骨架 / 取舍责任边界未说明                                | 文档头部标记说明补充责任归属                                                         | 高优先级，文本注释      |
| ③   | ActionLog 整体被标记取舍，但字段需要锁死                     | ActionLog 字段🔒骨架强制；存储载体二选一✅取舍表征                                        | 高优先级，Schema 修改 |
| ④   | possible_duplicate /conflict_with 语义、互斥关系未定义  | 补充🔒骨架强制字段语义：允许同时为 true，业务处理逻辑区分                                       | 高优先级，字段语义约束    |
| ⑤   | 跨坐标系 bounds IoU 比对逻辑缺失                        | 🔒骨架强制：仅 coordinate_system 完全一致才做 IoU 比对；不同坐标系仅使用 entity_kind + 文本标签匹配 | 高优先级，组装模块规则    |
| ⑥   | 非 GUI 领域 z_index=null，不应计算 occluded           | 🔒骨架强制：非 GUI/DOM 跳过 occluded 计算，visibility 只允许 visible/invisible       | 高优先级，字段语义约束    |
| ⑦   | domain_payload key 必须属于 source_providers      | 🔒骨架强制，assembler 增加校验，非法 key 返回错误                                      | 高优先级，组装模块校验    |
| ⑧   | load‑water‑level 通知缺少载荷 Schema                | 新增 LoadWaterLevelNotification🔒骨架强制 Schema                             | 高优先级，通知结构      |
| ⑨   | MVP‑1 实体匹配 “骨架” 实现程度模糊                        | 文字明确：MVP‑1 允许最简匹配，只要产出标记、跑通三套测试用例；复杂算法可延后迭代，参数可配置；✅取舍表征                | 中优先级，需求界定      |
| ⑩   | action_id 唯一性范围未定义                            | 🔒骨架强制：Agent 输出全局 UUID，中间层不做幂等去重，幂等交给上层 Agent                          | 中优先级，接口约束      |
| ⑪   | fallback_triggered=true 时 source_providers 校验 | 标记✅取舍表征；建议 assembler 实现，不作为 MVP‑1 阻塞条件                                 | 低优先级，增强校验      |

> 全部硬阻塞 MVP‑1 运行的缺口全部闭合；⑪属于质量增强，留给实现团队选择是否落地。
> 对外发布版本：v3.1 终稿。骨架语义全部钉死；可变部分全部打上取舍表征，配套 ADR 归档机制，防止后续版本语义漂移。
