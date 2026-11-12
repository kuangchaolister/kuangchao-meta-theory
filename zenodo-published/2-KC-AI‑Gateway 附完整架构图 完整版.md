# 自研 KC-AI‑Gateway｜完整设计方案文档【V4.0】

> 
> 基点继承：`AI‑GW‑SCHED‑BASE‑V1`
> 变更说明：完成第四次全部审查修复；补齐多策略仲裁、故障状态机分支、悖论场景、事件模型、CRD 字段规则、部署拓扑区分；所有状态机、故障处理、仲裁逻辑全部固化在架构契约；仅纯数值 / 时间阈值 / 格式命名留在待确认；架构完整闭环，可直接作为实现输入。

## 目录

1. 设计理念与第一性原则
2. 🔒全局硬性权责契约【架构不可违背，审查重点】
3. 业务定位 & 部署形态 & 顶层取舍
4. 系统内外边界划分：内核域 / 上层外壳域 / 外部依赖域
5. 系统输入输出完整契约
6. 业务需求
7. 非功能需求
8. 客观范式固有约束（物理 / 范式带来，无法彻底消除，只能受控处置）
9. 风险表征与事件上报、上下游协作路径
10. 🧾设计方案审查校验清单
11. 📋待确认事项（仅纯数值、周期、格式；状态机 / 策略决策已全部固化至正文）

---

## 1 设计理念与第一性原则

1. **第一性**：增设**内部转译适配层**。接收外部嘈杂输入，完成协议 / 格式归一、报文重组、分类、排序、筛选、权重分配；网关仅执行**传输层动作，不自主生成业务语义判决**；对外输出标准化报文 + 完整可追溯事件证据。

> 
> 客观物理极限无法消灭，但**传输层行为语义必须确定、可追溯；禁止将传输层歧义静默抛给上下游**；业务语义判定责任向上收敛。

2. 网关所有干预行为（限流、熔断、会话迁移、阻断），触发源必须来自**网关自身可观测检测信号**：连接状态、回包状态、计数器、健康探测、外部组件返回原始结果；网关不得臆测推导业务语义做干预。
3. 权责分层：**上游（业务客户端 / Agent Runtime）为唯一业务判定主体；下游后端无业务判定权；终端用户无业务判定权，仅接收反馈记录**。网关负责向上游完整报备事件，可开放接口交由上游接管处置。
4. 第二优先级：稳定性优先；允许功能不完美，但传输层行为稳定、可复现、每一次干预均留存检测证据与埋点记录。
5. 第三优先级：梳理范式客观固有约束，明确代价；
6. 第四优先级：评估优缺点、风险、备选路径。

> 
> 核心口诀
> **网关管传输世界的检测、筛选、排序、转发、阻断、上报、迁移；
> 业务世界的对错、语义、上下文兼容性，一律交还上游。**

## 2 🔒全局硬性权责契约【审查重点，不可违背】

### 2.1 角色权责划分

1. **上游（业务客户端 / Agent Runtime）：唯一业务判定主体**
网关向上游完整报备原始事件、检测结果、护栏原始判决、异常信号；
提供**可选事件回调接管接口**，上游可以接收事件并下达最终阻断 / 透传指令，网关忠实执行。
2. **下游（LLM / MCP‑Server / A2A 后端）：无业务判定权**
仅接收网关标准化转发后的请求；不提供业务决策输入入口，不参与业务语义决策。
3. **终端用户：无业务判定权**
仅接收上游业务侧输出的反馈记录；网关内部完整留存事件证据；终端是否接收该记录不影响网关留存。

### 2.2 处置路径统一分流规则

所有异常 / 干预工况统一两套处置路径，全链路（护栏、限流降级、MCP 会话迁移、BackendTrafficPolicy 未命中）复用：

- **路径 A TEMPLATE（CRD 模板）**：网关按照 AIGatewayPolicy 内预定义传输动作模板直接执行；
- **路径 B CALLBACK（上游接管）**：网关不执行本地模板，先推送标准化事件至上游回调接口，等待上游返回阻断 / 透传指令后再执行。

控制字段：

1. `defaultDecisionMode`：枚举 `TEMPLATE｜CALLBACK`；**全局默认值为 `TEMPLATE`；未显式配置时，全链路异常工况默认走路径 A 本地模板执行**；单个策略可覆盖全局`defaultDecisionMode`。

> 
> ✅【新增多策略命中合并仲裁规则】
> 多 BackendTrafficPolicy 策略同时命中：**只要任意一条匹配策略为`CALLBACK`，整体执行`CALLBACK`；全部匹配策略均为`TEMPLATE`才执行`TEMPLATE`；CALLBACK 优先级高于 TEMPLATE，不产生第三种冲突状态。**

2. `callbackTimeoutMode`：回调超时 / 上游回调服务不可达兜底策略，枚举 `FALLBACK_TO_TEMPLATE｜DENY`。

> 
> **兜底路径优先级【架构固化决策】**
> 
> 
> 1. **连接级不可达（TCP 连接失败、事件通道健康检查失败）：优先级最高；立即回退`decisionMode=TEMPLATE`，不再等待 callbackTimeout 超时，不进入超时判定逻辑。**
> 2. 事件通道连接正常，但上游超过`callbackTimeoutMode`阈值未返回指令：按`callbackTimeoutMode`枚举执行`FALLBACK_TO_TEMPLATE` / `DENY`。
> 3. 超时触发兜底动作执行完毕之后，**晚到达的上游回调指令全部丢弃，仅记录一条`late_callback_discarded`审计事件；网关不做任何回滚，不修改已经输出给客户端的报文，不做二次转发**。

> 
> 流式事件自动判定规则：
> `callbackNotSupported` 为**网关运行时自动生成标记，业务方禁止手动配置 CRD 此字段**。
> 👉**callbackNotSupported 判定粒度【架构固化】：以应用层首个完整 message 成功写入发送缓冲区为准，不以 transport 层 HTTP/2 DATA 帧为粒度；
> 
> 
> - SSE：首个完整`data:`帧完整写入发送缓冲区；
> - Streamable‑HTTP：首个完整 JSON‑RPC message 序列化并写入发送缓冲区；
> 判定点位于转译适配层报文输出环节，不在 socket write 系统调用返回环节。**
> 一旦标记置为 true，无论 CRD 配置`decisionMode=CALLBACK`，自动强制回落至路径 A TEMPLATE，无需业务方干预。
> 
> 
> > 
> > MCP Streamable‑HTTP 流式请求复用同一套「应用层完整 message」判定逻辑。

> 
> 时序约束：
> 
> 
> - **非流式请求**：网关向下游转发前阻塞等待上游回调指令；按上述优先级执行兜底。
> - **SSE / Streamable‑HTTP 流式请求**：**应用层完整 message 尚未输出前，前置事件允许 CALLBACK 接管；一旦应用层首个完整 message 写出，后置事件自动置`callbackNotSupported=true`禁用 CALLBACK，回落 TEMPLATE**；已经向外输出 message 之后，后续收到的上游指令仅记录审计事件，**不追溯修改已经下发给客户端的报文**。

### 2.3 事件上报通道自身约束【架构固化回切策略】

1. 事件上报通道独立维护自身健康状态；通道自身异常生成独立事件类型。
2. **禁止递归嵌套：上报通道故障事件，不再再次走事件回调链路，避免事件风暴**。
3. 事件通道**连接级持续不可达时，网关自动回退至`decisionMode=TEMPLATE`本地模板执行**；内部状态标记仅添加于返回客户端响应头`event_channel_unavailable=true`，不会注入转发至后端的请求报文，供业务感知。
4. ✅【架构选定故障通知方案 B】配置独立 HTTP 健康探测端点 `/health/event‑channel`；**上游业务通过轮询该端点感知通道状态；事件通道故障不依靠事件链路本身传递告警；通道从不可达恢复可达，不会自动切回 CALLBACK 模式；持续保持 TEMPLATE 运行；若需要重新启用 CALLBACK，必须运维更新 CRD 设置`decisionMode=CALLBACK`；降级窗口期积压事件**不做批量补报。

### 2.4 Guardrail 护栏强制约束

1. 第三方护栏输出的业务风险判决，**网关只做采集、标准化事件、埋点上报、透传给上游；网关不解读、不改写护栏业务判决，网关不做业务层面风险裁决。**

> 
> ✅【新增护栏采集规范】网关采集护栏判决：仅标准化判决状态码到网关统一事件 schema；**不解析、不透传 reason 理由文本；护栏厂商私有细节由网关内置适配器完成转换，理由文本不流入网关事件流，避免网关依赖业务语义 schema。**

2. 网关仅执行两类纯传输动作，由`decisionMode`选择路径 A 模板 / 路径 B 上游接管：

> 
> ✅【新增组合场景约束】`decisionMode=CALLBACK`接管模式下，**若护栏服务本身不可达，网关不进入回调等待，直接按 FailClosed/FailOpen 模板执行；CALLBACK 回调等待逻辑，仅护栏服务可达、仅需要上游覆写已有判决结果的场景才启用。**

- 路径 A：依据 CRD 预定义**传输动作模板**执行阻断 / 透传；
- 路径 B：触发事件回调，把护栏原始判决上交上游，**由上游下达阻断 / 透传指令，网关只执行传输动作**。

3. `FailOpen / FailClosed`不再是网关自主业务风险开关；仅代表 CRD 上的**传输动作模板**；**出厂默认值：FailClosed（护栏服务不可达默认阻断流量，对齐业界 Envoy‑Gateway 惯例）**。
   - `FailClosed`模板：护栏服务不可达，网关按模板执行传输层阻断，同时上报完整事件至上游；
   - `FailOpen`模板：护栏服务不可达，网关按模板透传流量，完整事件上报上游；**⚠️从 FailClosed 修改为 FailOpen 属于安全降级操作，事件输出标记`securityDegraded=true`，需要安全团队审批。**
4. 链路顺序：第三方护栏服务 → 网关采集原始判决 →（按`decisionMode`：本地模板执行｜上报上游等待指令）→ 网关执行传输动作 → 转发下游。

> 
> 禁止：网关自主解读护栏输出内容，自主做出业务风险放行 / 拦截裁决。

### 2.5 MCP‑FailOpen 会话切换强制约束【补充故障优先级、事件、空闲超时口径】

1. **触发条件严格限定：仅网关自身检测后端网络层面故障（连接断开、超时、连接拒绝）；业务返回错误码不作为 FailOpen 触发条件。**
2. ✅【新增故障执行优先级固化】MCP 后端网络故障判定顺序：**熔断健康检测优先执行；若熔断判定后端不健康，按`failureMode`执行：`FailClosed`直接标记会话失败；`FailOpen`执行 target 切换、生成全新 session‑id；切换完成重置该后端熔断计数，防止切换完成瞬间再次触发熔断。**
3. 切换发生后：网关完成传输层会话迁移，生成全新加密 session‑id；**向原后端下发`session_migrated`通知事件**；生成标准化会话迁移事件完整**报备上游**。

> 
> ✅会话事件三类区分：
> 
> 
> - `session_migrated`：FailOpen 触发 target 迁移，旧会话可保留在原后端，后端自行决定上下文留存时长；
> - `session_terminated`：网关下发空闲超时终止信号；
> - `session_expired`：MCP 后端上报业务会话过期。

4. ⚠️强制标注：

> 
> `session‑id加密token为会话路由绑定的唯一真相源（决定后端亲和性、传输层会话归属）；MCP业务会话上下文、工具调用历史属于**MCP后端服务业务状态**；网关侧Redis仅存储可选辅助元数据，网关内核不保证辅助元数据完整性。 ✅【会话空闲超时判定口径】`sessionTimeout`空闲判定：**以应用层最后一条完整JSON‑RPC message（含notifications/*通知）时间戳为准；TCP keep‑alive心跳、Streamable‑HTTP GET长轮询空连接、单纯ping保活，不重置空闲计时器；仅initialize/tools/call/resources/*业务message重置计时。** 会话空闲超时回收逻辑由**MCP后端服务执行；网关仅执行传输层空闲超时检测，向MCP后端下发`session_terminated` 会话终止信号；客户端携带已经业务回收过期的 session‑id 发起请求，网关依据 token 路由至原后端，由 MCP 后端返回会话失效；网关标准化封装会话失效错误返回客户端，并生成新 session‑id 供客户端重新 initialize；网关本身不维护会话业务过期状态。**
> 网关**不做跨 target MCP 工具业务上下文兼容性判定；该业务判定交给上游 Agent Runtime；下游 MCP 服务不参与业务判定，只接收新会话流量。**

### 2.6 网关允许执行动作白名单（仅传输层）

> 
> ✅允许：协议转译、报文归一、分类集合、排序、权重筛选、限流计数、熔断、会话传输迁移、事件采集、上报、埋点日志留存。
> 边界约束：
> 
> 
> > 
> > 权重筛选、排序、RBAC 鉴权判定依据**只能来自 CRD 静态声明，或是请求原始携带属性 (tenant‑id、api‑key 绑定配额组)；网关禁止解析报文内容、下游返回业务语义动态推导权重、权限规则。**
> > ✅【协议元数据豁免规则】允许读取**协议标准元数据字段**：MCP‑Protocol‑Version 请求头、MCP 请求体`_meta`元数据、HTTP Content‑Type；协议标准元数据不属于业务 payload 内容，不视为业务语义解析；**禁止读取 payload body 内部业务字段（prompt、tool arguments 业务参数）。**
> > 熔断判定**仅采信传输层信号：TCP 断开、超时、连接拒绝、HTTP‑5xx；4xx 业务错误、MCP JSON‑RPC 业务错误码不作为熔断触发条件。**
> > 补充说明：熔断属于网关内核**启发式健康推导能力，并非纯粹客观事实判定；网络抖动存在误判风险；业务需要严格精确后端健康语义，应当使用外部独立健康检测系统，网关熔断仅作为快速失败兜底手段。**
> > ❌禁止：网关自主推导、裁决业务语义正确性；自主判定 LLM 输出好坏、MCP 工具调用业务上下文是否兼容、prompt 业务风险。

### 2.7 转译适配层定位

> 
> 转译适配层为 **Rust 数据面内部子模块，不独立部署进程**；标准化报文输出即同进程内部送给路由模块的转发请求；标准化事件输出直接输入同进程事件上报模块；全部运行于数据面进程空间。
> 转译适配层输出两份产物：

1. 标准化报文，向下游后端转发；
2. 标准化事件 / 检测信号，向上游报备（含可选上游接管回调接口）；

> 
> 转译层**不产出任何业务语义结论，仅做信号规整、清洗、格式转换**。

### 2.8 全部异常降级统一模式（Redis 降级、xDS 版本不一致、熔断等）

1. 网关依靠自身检测识别异常状态；执行 CRD 预定义传输动作模板（路径 A）；或者按`decisionMode=CALLBACK`走上游接管路径；
2. **异常事件 + 全部原始检测证据完整报备上游；事件通道异常按 2.3 规则兜底；**
3. 下游只收到最终标准化请求 / 响应；下游不参与异常决策。

## 3 业务定位 & 部署形态 & 顶层取舍

### 3.1 业务定位

自研 KC-AI‑Gateway 面向 K8s 云原生；承载普通 LLM 无状态推理流量 + MCP/A2A Agent 长会话流量。

> 
> 📌MCP 协议版本战略说明
> 
> 
> 1. MCP‑2026‑07‑28 官方新版协议已经移除协议级 session‑id，整体演进为无状态自包含请求模型；协议识别头为`MCP‑Protocol‑Version`，同时解析请求体`_meta`版本元字段。
> 2. 本网关 L3 stateful 会话能力，定位为：①**旧版 MCP（2025‑11‑25 及更早）存量业务兼容过渡能力**；②允许业务通过 CRD 显式开启，用于特殊长会话业务场景；该模式与新版 MCP 协议设计理念存在差异，由上游业务评估长期维护成本。
> 3. MCP 协议版本识别分流规则
> 
> > 
> > ✅`prefixMode`（Virtual‑MCP Multiplexing 工具名前缀冲突规避）属于**传输层能力；stateless、stateful 模式均生效；仅 sessionRouting、failureMode [FailOpen] 在 MCP‑2026‑07‑28 stateless 管道下禁用。**
>    - 通过请求头`MCP‑Protocol‑Version`识别协议版本，同时解析请求体内`_meta`元数据；
>    - MCP‑2026‑07‑28：强制进入 stateless 管道，**stateful 会话绑定逻辑完全禁用**；
>    - ✅请求未携带`MCP‑Protocol‑Version`协议头，默认按旧版 MCP‑2025‑11‑25 行为，`sessionRouting=true`默认开启 stateful 会话；可被 CRD 全局 MCP 默认版本策略覆盖。
>    - 旧版 MCP‑2025‑11‑25：受 CRD `sessionRouting`开关控制，可开启 / 关闭 stateful 会话；

### 3.2 部署形态

1. 独立南北向网关：客户端直接接入；
2. 东西向 sidecar 模式，服务网格内部流量。

> 
> ✅【部署拓扑区分】
> 
> 
> - **南北向独立网关**：集群级集中控制面，多副本执行 K8s leader‑election 选主；xDS 推送到全部网关数据面 Pod。
> - **东西向 sidecar 形态**：每业务 Pod 内嵌独立控制面单实例；**无多副本、无 leader‑election 选主；xDS 仅推送给本 Pod 内数据面；每个 Pod 维护独立 apiserver watch 连接。**
> sidecar 约束：内存占用、冷启动、镜像体积必须满足大规模 Pod 注入集群密度约束；控制面对 sidecar 采用**按需长连接推送，禁止全量广播推送大体积配置快照，降低 sidecar 内存压力**。

### 3.3 顶层设计取舍（不可越界）

1. ❌不做 Agent 事件溯源 DAG 编排、不做工具调用业务逻辑；Agent 编排交给上层 Runtime；网关只做流量治理、协议转换、会话生命周期、护栏、配额路由。
2. ❌推理资源调度（Pod/GPU 分配、队列、gang 调度、优先级抢占）交给 Volcano/Kueue/K8s‑Scheduler；网关仅对已经就绪后端实例做实例层路由。
3. ✅纳入内核域：`InferencePool`推理池路由（Gateway‑API Inference Extension 标准实例层负载均衡）。

> 
> > 
> > ✅【字段语义区分备注】`InferencePool FailureMode`与`MCPBackend failureMode`是两套完全独立 CRD 字段，语义、默认值互不继承；前者控制推理池实例不健康时的转发行为，后者控制 MCP 后端故障时的会话切换行为；Envoy 数据面 InferencePool FailureMode 默认 FailOpen 为 Envoy 底层限制；自研 Rust 数据面不受该约束。

4. ✅MCP L3 会话能力保留；spend 金额预算**不在内核域，归属上层外壳扩展点**。

## 4 系统内外边界划分

### ✅内核域（Rust 实现，包含转译适配层）

1. **控制平面**
K8s Gateway‑API 兼容 CRD 监听翻译；自定义 CRD：`AIGatewayPolicy`、`MCPBackend`、`BackendTrafficPolicy`、`InferencePool`；字段补充：`decisionMode`、`callbackTimeoutMode`、`toolNameStyle[namespaced｜plain]`；
xDS‑ADS gRPC 双向流推送配置至 Rust 数据面；

> 
> 高可用补充：控制面多副本依赖 K8s leader‑election 选主；**仅主副本执行 CRD status 回写 etcd；备副本作为温备；**
> ✅【选主租约期 CRD 变更规则】K8s leader‑election 租约窗口期（旧主释放锁到新主抢到锁）提交的 CRD 变更写入 etcd；等待新主选出后统一翻译推送；运维通过 CRD status 字段`observedGeneration`、`reconciledVersion`比对确认配置生效；租约窗口期不会丢失版本，仅生效被延迟。主副本崩溃重新选主期间 status 回写暂停；已经推送到数据面的 xDS 快照不受任何影响，存量流量继续运行。

2. **Rust 数据面 + 内部转译适配层（核心）**

- 统一传输层：HTTP/gRPC/SSE/Streamable‑HTTP；
- 公共管道：鉴权‑RBAC‑限流‑guardrail 调用‑缓存‑路由‑协议转换；

> 
> ✅【CEL 表达式访问域约束】Local‑Rate‑Limit CEL 表达式仅允许访问：请求头、URL 路径、query 参数、认证属性 tenant‑id/api‑key、协议标准元数据；**禁止访问 request body、JSON payload 内部业务字段；CRD 准入校验直接拒绝违例 CEL 表达式。**

- LLM 子域：OpenAI 兼容、SSE 流式、retry‑fallback‑circuit‑breaker、InferencePool 实例层路由；
- MCP 子域：**按`MCP‑Protocol‑Version`请求头 +`_meta`元数据做管道分流**；stateful/stateless 会话；`sessionRouting`开关、`failureMode[FailClosed / FailOpen]`、`prefixMode`、`toolNameStyle`；session‑id AES‑256‑GCM 加密携带 backend‑pinning 绑定；MCP JSON‑RPC 透传；

> 
> ✅prefixMode 拼接规则：`backendName + 分隔符 + 原始工具名`；网关检测原始工具名是否已携带 backendName 前缀，命中则不再重复拼接；依据`toolNameStyle`决定是否开启拼接。

- A2A 子域：Agent 报文透传、身份鉴权、会话空闲超时回收（传输层仅下发终止信号，业务回收交由后端）；
- 两层限流架构

> 
> Redis 降级规则：Redis 故障触发降级切换 Local‑Rate‑Limit；**Local 计数器重置为保守折算阈值：全局配额 ÷ CRD 字段`maxExpectedReplicaCount`；CRD 未配置`maxExpectedReplicaCount`，使用内核内置硬编码保守值；网关**不去调用 K8s Deployment 副本，规避 RBAC 权限依赖；响应头携带`redis_degraded=true`标记（仅返回客户端，不向后端注入）；降级事件完整上报上游。
  - `Local‑Rate‑Limit`：pod 本地内存、CEL 表达式提取描述符，**无需 Redis**；
  - `Global‑Rate‑Limit`：分布式限流，强依赖 Redis；
- 可观测埋点：OTel trace/metrics；内核仅维护**进程内存滑动窗口计数器，pod 重启清零；跨 pod 聚合、告警判定逻辑全部交由外部 Prometheus 采集系统完成；内核不持久 payload；所有干预动作留存事件证据。**
- 事件上报模块：标准化事件生成，向上游业务推送；支持`decisionMode=CALLBACK`事件回调接管接口；事件上报通道做自身健康状态检测，遵循 2.3 故障兜底规则；

> 
> ✅事件关联规则：每条事件携带`request‑id`；客户端携带则透传；未携带由转译适配层生成 UUID；**复合事件唯一键 = request‑id + attempt‑number（重试序号，首次 0） + xds‑snapshot‑version；attempt‑number 单调递增，不受 pod 切换重置；request‑id 与 trace‑id 映射关系在事件流保持稳定。**

### 🟡上层外壳域（内核仅预留扩展钩子，不属于内核本体）

1. spend 金额 / 成本预算管控；
2. UI 控制台、helm 部署模板、大盘；
3. payload 完整日志持久存储；
4. 上层业务做 SSE 应用层重试兜底；
5. 上层业务消费网关事件回调，实现业务侧判定逻辑。

### ❌外部依赖域（本网关绝不实现，依赖外部系统契约）

1. K8s‑etcd：保存 Gateway‑API 及自定义 CRD；
2. Redis：`Global‑Rate‑Limit`；MCP/A2A**可选辅助会话元数据缓存**；

> 
> 📌会话真相源规则：
> `session‑id`内 AES‑256‑GCM 加密的 backend‑pinning 绑定信息，**是会话路由绑定的唯一真相源**；Redis 存储的会话辅助元数据属于可丢失的性能优化缓存；Redis 不可达，只要客户端携带合法有效的 session‑id，会话依然可以跨 Pod 恢复；辅助元数据缺失由 MCP 服务自行处理。

3. 上游后端：LLM 推理服务、MCP Server、A2A Agent、第三方 Guardrail 护栏服务；**会话业务空闲超时回收由 MCP 后端完成。**
4. 后端资源调度系统：Kueue / Volcano / K8s‑Scheduler；GPU 队列、Pod 抢占、gang 调度全部交给调度系统；网关仅消费已经就绪的后端实例列表；
5. 外部采集系统：OTel‑Collector、Prometheus，消费埋点事件；
6. Agent Runtime：DAG、工具调用业务逻辑、Agent 状态机，网关不介入。

> 
> 🔴边界红线
> 
> 
> 1. 网关**不做 GPU 资源调度、Pod 驱逐；仅在就绪实例集合内做路由选择。**
> 2. MCP 只做传输层与会话生命周期治理；**工具调用业务语义校验完全交给上游 Agent Runtime。**

## 5 系统输入输出契约

### 系统输入

1. K8s 侧输入：CRD 变更事件（Gateway / HTTPRoute / AIGatewayPolicy / MCPBackend / BackendTrafficPolicy / InferencePool）；
2. 下游客户端输入：
   - 普通 HTTP/OpenAI LLM 请求（JSON / SSE‑stream）
   - MCP：stdio、SSE、Streamable‑HTTP 请求，携带`MCP‑Protocol‑Version`头、`_meta`元数据；stateful 初始化 initialize、工具调用会话请求
   - A2A Agent 间通信请求
3. 外部依赖输入：Redis 读写响应；LLM/MCP/A2A / 护栏返回报文、网络异常；
4. 运维输入：xDS‑ADS 配置推送；健康探测回包；
5. **上游业务侧可选输入：事件回调接管返回指令（阻断 / 透传）。**

### 系统输出

1. 向下游后端：归一化转换后的 LLM/MCP/A2A 请求报文；**绝不注入任何网关内部状态标记头。**
2. 向客户端：归一化 OpenAI‑JSON/SSE、MCP JSON‑RPC、A2A 响应；429 限流、鉴权拒绝、会话错误；流式场景已经写出完整 message 之后，晚到的上游回调指令仅记录审计事件，不修改已下发报文；
> 
> ✅完整枚举客户端 HTTP 扩展响应头（**仅附加返回客户端，绝不注入向后端转发请求**）：`redis_degraded`、`event_channel_unavailable`、`xds‑snapshot‑version`；作为客户端侧可观测扩展头。
3. 控制平面输出：xDS‑ADS 增量快照推送；CRD status 状态回写 etcd（仅主副本执行）；
4. 可观测输出：OTel metrics/trace 事件流；全部干预动作携带检测证据；
5. **向上游业务输出：标准化事件流；复合事件键`request‑id + attempt‑number + xds‑snapshot‑version`；会话迁移、护栏判决、限流降级、xDS 快照版本、熔断事件；支持事件回调推送。**
6. 外部交互：Redis 读写（Global 限流、可选辅助会话元数据）。

> 
> 📌补充契约条目

1. `defaultPolicyMode`全局开关：`DENY(默认)` / `PASSTHROUGH`；

> 
> ✅修正语义：`defaultPolicyMode=DENY`语义：未命中任何 BackendTrafficPolicy 的请求，网关**不主动向下游转发；仍然生成并推送标准化事件给到上游；若 decisionMode=CALLBACK，上游可下达 PASSTHROUGH 指令覆盖拒绝；若 decisionMode=TEMPLATE，则直接拒绝，不推送事件。**
> `defaultPolicyMode=PASSTHROUGH`：未命中策略直接透传，同时上报事件。

2. xDS 多 pod 收敛窗口：各 pod 以自身已接收快照版本处理请求；**收敛窗口期 LB 随机分发，同一请求可能被调度至不同快照版本 Pod，出现策略版本差异；BackendTrafficPolicy 为最终一致性语义，非即时生效；每个请求响应携带实际生效`xds‑snapshot‑version`快照版本号供业务观测；事件携带完整复合关联键，用于区分重试事件 / 多 pod 版本差异事件。**
3. MCP `failureMode=FailOpen`触发：仅网络层故障；生成全新 session‑id 随响应返回；向原后端下发`session_migrated`事件；会话迁移事件完整上报上游；不做业务上下文兼容判定。
4. ✅`redis_degraded`仅为客户端观测标记；**网关不对客户端基于该标记触发的重试逻辑负责；业务自行评估降级场景下流量放大风险。**

## 6 业务需求

1. 同时支持**无状态 LLM 推理流量 + stateful MCP / A2A Agent 长会话**；MCP L3 完备 CRD 字段：

> 
> 注：MCP stateful L3 会话能力定位为存量兼容过渡能力（详见 3.1 MCP 协议版本战略说明）；**新业务建议采用 MCP‑2026‑07‑28 stateless 模式；stateful 字段保留可用，业务需自行评估长期维护成本。**
> MCP 协议依据`MCP‑Protocol‑Version`请求头 + 请求体`_meta`元数据分流；新版 MCP‑2026‑07‑28 流量默认 stateless，stateful 逻辑禁用；旧版受`sessionRouting`开关控制；无协议头默认旧版行为`sessionRouting=true`。
   - `sessionRouting`：布尔开关，控制是否启用 stateful 会话绑定；
   - `failureMode`：枚举`FailClosed(默认) / FailOpen`；
   - `prefixMode`：控制`Virtual‑MCP Multiplexing`工具名前缀冲突规避；
   - `toolNameStyle`：`namespaced｜plain`，控制前缀拼接逻辑；
2. 控制‑数据面分离范式；控制面 pod 崩溃存量流量继续运行；仅无法下发新配置；控制面多副本依靠 K8s leader‑election 选主。
3. 重试 / 回退策略全部由`BackendTrafficPolicy` CRD 声明；内核无任何隐式 retry/fallback 默认值；支持`defaultDecisionMode`、`callbackTimeoutMode`；多策略命中遵循 CALLBACK 优先合并规则。
4. Guardrail 护栏：仅采集原始判决，上报上游；支持 CRD 传输动作模板`FailOpen/FailClosed`（出厂默认 FailClosed）；支持 TEMPLATE / CALLBACK 接管模式；护栏服务不可达时 CALLBACK 模式直接走模板，不等待回调；流式首 chunk 输出后置事件自动禁用 CALLBACK。
5. 两层限流：Local‑Rate‑Limit 无需 Redis；Global‑Rate‑Limit 依赖 Redis；Redis 故障切换 Local 模式，计数器使用保守折算阈值；响应携带降级标记，事件上报上游；CEL 表达式访问域做 CRD 准入校验。
6. MCP stateful：session‑id AES‑256‑GCM 加密携带 backend‑pinning；session‑id 为路由绑定真相源；CRD 更新路由策略仅对**新流入会话生效；存量会话沿用 session‑id 内绑定；`sessionTimeout`空闲超时口径：仅业务 JSON‑RPC message 重置计时器；网关下发`session_terminated`信号，业务回收交由 MCP 后端。**
7. Virtual‑MCP Multiplexing：`prefixMode`控制；多路 MCP 会话复用底层传输连接，自动工具名前缀防冲突，受`toolNameStyle`约束；stateless/stateful 均生效。
8. MCPBackend `FailClosed`（默认）：绑定后端故障，整个会话标记失败；`FailOpen`仅网络层故障触发；熔断检测优先；切换生成新 session‑id，向原后端下发`session_migrated`事件；会话迁移事件上报上游；**上下文兼容性交由上游；FailClosed 默认绑定后端故障直接标记会话失败。**
9. mid‑stream‑fallback 不由内核内置；完全交由`BackendTrafficPolicy`配置开启；代价明确：缓存完整 SSE 字节流导致内存暴涨 /messages 列表重建造成语义断裂，内核不做语义修复，业务自行评估风险。
10. 事件上报机制：限流、熔断、会话迁移、护栏判决、Redis 降级、xDS 版本事件标准化上报上游；事件具备完整复合关联键；支持事件回调接管；事件通道独立健康探测端点；流式后置事件首 message 写出自动禁用 CALLBACK。

## 7 非功能需求

1. 性能：Rust 数据面；高吞吐、SSE 低延迟；控制面不处理业务流量。

> 
> sidecar 约束：同时支持南北独立网关、东西向 sidecar 注入形态；sidecar 形态内存、冷启动、镜像体积满足大规模注入；区分集群集中控制面 / Pod 内嵌单实例控制面拓扑。

2. 高可用：控制面多副本，K8s leader‑election 选主；主副本崩溃，重新选主期间 status 回写暂停，存量流量不受影响；

> 
> ✅修订表述：Redis 故障 Global‑Rate‑Limit 切换为 pod 本地限流粒度，**业务流量不因 Redis 故障整体中断；限流正确性退化为 pod 粒度，多 pod 叠加仍可能超出原全局配额，详见 §8 范式约束第 2 条。**；pod 重启无状态请求不受影响；MCP stateful 依靠 session‑id 跨 pod 恢复，仅丢失可选辅助元数据。

3. 可观测：OTel 完整埋点；内核进程内存滑动窗口计数器，pod 重启清零；跨 pod 聚合告警交给外部 Prometheus；内核不持久 payload；事件携带完整复合关联键；事件通道具备独立健康探测端点。
4. 可运维：全部策略 CRD 驱动；xDS‑ADS 双向推送；CRD status 通过`observedGeneration`、`reconciledVersion`反馈配置翻译生效结果；告警指标：配置收敛告警、`redis_degraded`降级告警、fallback 计数、护栏触发、MCP 会话计数、`event_channel_unavailable`告警。
5. 安全约束
   - RBAC 鉴权；MCP 工具粒度访问控制；权重、权限规则仅允许 CRD 静态配置或请求原始属性；CEL 表达式访问域准入校验；
   - session‑id 加密，禁止明文暴露后端信息；
   - 护栏出厂默认 FailClosed；切换 FailOpen 标记`securityDegraded=true`；流式后置事件自动禁用 CALLBACK。

## 8 客观范式固有约束（物理 / 范式带来，无法彻底消除，只能受控处置）

> 
> 不是 bug，是架构取舍，全部通过事件上报上游，由上层业务做业务侧规避。

1. xDS‑ADS 多 pod 推送存在配置收敛时间窗口；LB 随机分发，窗口期同请求可能命中不同快照版本；`BackendTrafficPolicy`为最终一致性，非即时生效；每条请求携带生效快照版本，事件带完整关联键。
2. Global‑Rate‑Limit 依赖 Redis；Redis 降级切换 pod Local 计数；即使保守折算，多 pod 叠加依然存在总流量超出原全局配额风险；降级事件完整上报上游。
3. MCP stateful 存量会话：CRD/MCPBackend 更新只对新会话生效；存量会话绑定固化于 session‑id；无透明会话迁移 API；存量会话采纳新策略，客户端需要重新 initialize。
4. MCPBackend FailOpen 仅网络故障触发；熔断优先；网关只做传输层会话迁移；内核不保证跨 target 业务上下文继承；该约束上报上游，业务处理。
5. mid‑stream‑fallback 开启必然二选一代价：缓存完整 SSE 字节流导致内存暴涨 / 基于 messages 列表重建导致语义断裂，内核不做自动语义修复。
6. SSE 后置 guardrail 无法拦截已经下发 chunk；安全主防线必须 pre‑request；后置事件首 message 输出自动禁用 CALLBACK 接管。
7. 公理：**隔离≠失真校验**；网关可做报文格式、访问隔离；不校验 LLM/MCP 上层业务语义；token 计数依赖上游 usage 或本地 tokenizer 估算，无法做到 100% 真值。
8. 熔断属于内核启发式推导，网络抖动会存在误判风险；严格健康检测依赖外部独立系统。
9. 内核不实现 spend 金额预算；无内核请求优先级原语；MCP 会话无内核硬上限；软预算仅埋点告警，内核不主动节流。

## 9 风险表征与事件上报、上下游协作路径

表格

| 风险场景 | 网关内核行为 | 业务 / 运维侧可选协作动作 |
| --- | --- | --- |
| xDS 大集群配置收敛窗口大 | 上报每条请求生效 xDS 快照版本，事件推送上游，客户端响应头携带版本；事件携带`request‑id+attempt‑number+xds‑snapshot‑version`复合键 | A：变更期间上游业务侧流量管控；B：业务侧观测版本分布告警 |
| Redis 故障，Global 限流失效切换 Local | 切换 pod 本地限流，计数器采用保守折算阈值；响应标记`redis_degraded=true`；事件上报上游；事件通道不可达自动回落 TEMPLATE；网关不对客户端基于该标记的重试行为负责 | A：上游事件回调直接全部拒绝新请求；B：业务侧做好多租户计费补偿逻辑，规避客户端重试放大流量 |
| MCP FailOpen 切换，业务上下文失效 | 熔断优先检测；传输层迁移，向原后端下发`session_migrated`事件；上报迁移事件，下发新 session‑id | A：上游 Agent Runtime 检测事件，主动重 initialize；B：业务直接禁用 FailOpen，只使用 FailClosed |
| 护栏第三方服务大面积不可达 | 按 CRD 传输模板执行阻断 / 透传，事件上报上游；流式后置自动禁用 CALLBACK；护栏不可达 CALLBACK 模式直接执行模板；事件通道异常回落 TEMPLATE | A：开启 CALLBACK 接管模式，上游业务实现自定义安全策略；⚠️B：运维变更 CRD 传输模板（从 FailClosed 改为 FailOpen 属于安全降级操作，需安全团队审批，事件标记`securityDegraded=true`） |
| mid‑stream‑fallback 开启出现语义断裂 | 内核不做语义修复；事件埋点记录触发；风险文档显式标注 | A：上游应用层做 SSE 重试兜底；B：业务不开启 mid‑stream‑fallback |
| 事件上报通道自身持续不可用 | 自动回落 TEMPLATE 模板；响应标记`event_channel_unavailable=true`；生成独立故障事件，禁止递归上报；提供`/health/event‑channel`独立探测端点；通道恢复不会自动切回 CALLBACK，无积压事件补报 | A：运维修复事件回调服务，手动更新 CRD 重新开启 CALLBACK；B：业务完全依赖 TEMPLATE 模板运行 |

## 🧾10 设计方案审查校验清单

> 
> 每一条通过打√；不通过则给出修改意见。

### 📌全局契约校验（核心）

- 权责划分：上游唯一业务判定主体；下游、终端无业务判定权；网关只做传输层动作。
- 处置路径：`defaultDecisionMode`全局默认 TEMPLATE；多策略命中 CALLBACK 优先；区分连接级不可达 / 超时两套兜底优先级；超时晚到指令丢弃仅审计。
- 流式：`callbackNotSupported`应用层 message 粒度判定；后置事件自动禁用 CALLBACK。
- 事件通道：独立`/health/event‑channel`探测端点；通道故障回落 TEMPLATE；恢复不自动切回 CALLBACK；禁止递归上报故障事件。
- 护栏约束：网关只采集上报原始判决，不做业务裁决；CALLBACK 模式护栏不可达直接执行模板；出厂默认 FailClosed；切换 FailOpen 标记安全降级；采集规则不解析 reason 文本。
- MCP‑FailOpen：熔断优先；`session_migrated`通知原后端；空闲超时口径明确；区分路由真相源与业务会话状态。
- 网关动作白名单：协议元数据豁免；CEL 表达式访问域约束；熔断仅采信传输层信号；熔断为启发式能力，存在误判风险。
- 转译适配层：Rust 数据面内部子模块，不独立部署；输出标准化报文 + 事件，不产出业务语义结论。
- 异常降级：全部异常检测、模板执行、事件上报；扩展头仅返回客户端，不注入后端；事件复合关联键完备。
- xDS 收敛窗口：最终一致性；事件可区分重试 / 多 pod 版本差异。

### 📌边界越界校验

- 内核没有侵入 GPU/Pod 资源调度、Agent 编排 DAG、工具调用业务逻辑；
- InferencePool / MCPBackend 两套 failureMode 语义区分；Envoy 约束备注准确。
- spend 预算、UI 大盘、payload 持久化归属上层外壳，不在内核域。
- 南北网关 /sidecar 两套控制面拓扑区分完备。

### 📌MCP 协议定位校验

- `MCP‑Protocol‑Version`+`_meta`元数据识别分流；新版 2026‑07‑28 默认 stateless；无协议头默认旧版行为；`prefixMode`stateless/stateful 均生效；stateful 定位存量兼容。
- prefixMode 拼接逻辑、`toolNameStyle`规则完备。

### 📌数据流与契约校验

- session‑id 路由绑定真相源定义准确；Redis 仅辅助缓存；业务会话回收交由 MCP 后端。
- `defaultPolicyMode`缺省策略语义完备。
- Redis 降级`maxExpectedReplicaCount`：CRD 优先，不读取 K8s Deployment。
- request‑id 生成、透传、重试复用规则完备；复合事件关联键定义完整。

### 📌范式客观约束校验

- 全部范式固有约束写明；区分物理客观代价 vs bug；全部风险具备协作路径。

### 📌非功能 & 部署形态校验

- sidecar 形态资源约束、控制面拓扑区分完备。
- 控制面 leader‑election 租约窗口期 CRD 变更行为定义。
- 可观测边界：内核内存滑动窗口计数器，pod 重启清零；跨 pod 聚合交给外部采集；内核不持久 payload。

## 📋11 待确认事项（**仅纯数值、周期、格式命名；全部状态机、策略决策已固化正文**）

1. `callbackTimeoutMode`对应的回调超时绝对时间阈值；
2. 事件上报通道自身健康检测探测周期；
3. 熔断判定中传输层错误的统计窗口、失败计数阈值；
4. K8s leader‑election 选主相关超时参数；
5. MCP 会话空闲超时`sessionTimeout`默认基准数值；
6. SSE 流式场景，各类审计事件（`late_callback_discarded`、`securityDegraded`）事件记录格式、事件总线 topic 命名规范。

---

> 
> 当前状态：第四次审查全部问题修复完成；所有状态机、仲裁、故障分支、事件模型、CRD 字段语义、部署拓扑全部固化；仅纯数值与格式命名留在待确认。
> 人工复核完毕，下达指令：`封档 #刷新基点`即可固化整套完整设计基点；存在问题给出条目编号 + 修改意见。

---

# KC-AI‑Gateway｜完整分析与测试【审查修正终版】

> 
> 基点：`KUANGCHAO‑BASE‑V1P0‑ROOT`
> 执行：狂潮元理论八步正向主流程
> 修订说明：完成审查全部🔴必改、🟠建议改、⚪顺带修正；补齐契约缺口、生产者‑消费者语义、能量维度、版本标识、事件通道分类、时序规则； 审查结论：通过。

## 📋八步正向流程执行总览

1. 划定系统内外边界，全部功能模块清单（已完成）
2. 拆解【权限‑状态流】、【边界‑执行流】，梳理跨模块接口、出入点
3. 逐条核验六条动态契约，识别契约缺口
4. 识别五类运行工况（区分单实例 / 多 Pod 分布式）
5. 固定顺序遍历模块耦合：旺 → 生 → 休 → 克 → 死
6. 汇总优势、**条件式隐患（当 XX 触发，则 XX 风险）**
7. 重构备选方案，标注落地等级与代价
8. 闭环复审；区分范式固有缺陷 / 项目特有缺陷 / 载体硬件缺陷；架构级测试输出

---

## 步骤 1｜划定内外边界 + 全部功能模块（复用已拆解结果）

> 
> 内外边界来自 V4 固化文档

- **系统内部**：K8s 控制平面（CRD 翻译、xDS‑ADS 推送、Status 回写）；Rust 数据面（转译适配层、公共处理管道、MCP/A2A 子模块、事件上报、后端连接 & 熔断）
- **上层外壳（内核之外，仅扩展钩子）**：spend 预算、UI、payload 持久化、业务回调消费
- **外部依赖域**：etcd、Redis、LLM/MCP‑Server、第三方护栏、Prometheus/OTel‑Collector、Agent‑Runtime 上游业务

模块清单沿用；严格遵循权限原子化协议六维约束。

## 步骤 2｜双流模型拆解：权限‑状态流｜边界‑执行流

> 
> 同一个 HTTP/MCP 请求物理报文同时携带**状态载荷**和**执行控制语义**；逻辑强制拆分。

### 2.1 权限‑状态流（什么存在、状态、版本、元数据、快照）

> 
> 承载：配置快照、会话元数据、计数器、事件、trace/request‑id/attempt‑number、xds‑snapshot‑version

1. K8s etcd：CRD 原始资源；`observedGeneration`、`reconciledVersion`配置版本标记。
2. 控制面：翻译之后 xDS 配置快照；多副本选主 lease 状态。
3. 数据面每 Pod 本地状态：
   - xDS 已加载快照版本
   - Local‑Rate‑Limit 内存计数器
   - MCP 会话传输层状态；AES‑256‑GCM 加密 session‑id（路由绑定真相源）
   - 事件上报通道健康状态
   - 熔断后端健康状态表
   - request‑id /attempt‑number /xds‑snapshot‑version 事件关联元组
   > 
   > ✅【修订补充：attempt‑number 生成归属】
   > `attempt‑number`生成归属：仅当客户端请求头显式携带`X‑Attempt‑Number`，网关原样透传；客户端未携带时，网关一律记为`0`；**网关不自行推导、不生成重试序号**；客户端未携带`X‑Attempt‑Number`的重复请求，网关视为相互独立请求；事件唯一键冲突风险由上游结合`xds‑snapshot‑version`+ 时间戳区分。
4. Redis（外部）：Global‑Rate‑Limit 计数；MCP 可选辅助会话缓存（可丢失，非真相源）
5. 事件流：标准化事件，携带完整关联元组，流向事件上报模块 / OTel。

> 
> 状态流关键点：
> 
> 
> - xDS 多 Pod 存在快照版本不一致（分裂态来源）；每条事件携带`xds‑snapshot‑version`做时序版本标识。
> - session‑id 加密 token 为**路由绑定真相源**；Redis 辅助缓存不属于真相源。
> - `attempt‑number`仅透传客户端传入值，网关不生成重试序号。

### 2.2 边界‑执行流（谁可以干什么、触发条件、熔断、准入、降级、兜底）

> 
> 承载：请求准入、鉴权、限流、护栏调用、路由、熔断、MCP 会话切换、decisionMode 模板 / CALLBACK 接管、超时兜底、降级逻辑。
> 完整执行链路：

1. 客户端请求入 → **转译适配层**（协议解析、request‑id 生成透传、message 粒度检测，标记`callbackNotSupported`）
2. 公共管道执行流：RBAC 鉴权 → Local 限流 → Global 限流 → 调用护栏服务 → 根据`decisionMode`走 TEMPLATE 模板 / CALLBACK 上游接管；
   - CALLBACK 分支：事件推送上游回调；区分**连接级不可达 / 超时**两套兜底；
   > 
   > ✅【修订补充能量约束】CALLBACK 等待路径占用**并发槽位、TCP 连接、内存广义能量**；CRD 配置回调等待并发上限，超限直接执行`callbackTimeoutMode`兜底，不进入等待队列；时间维度`callbackTimeout` + 并发槽上限共同构成完整兜底约束。
   - 超时之后晚到指令丢弃仅审计；
3. MCP/A2A 子模块执行流：会话校验、prefixMode 工具名处理、`sessionTimeout`空闲计时；后端网络故障触发熔断判断，熔断优先于 FailOpen 切换；
4. InferencePool 路由选择后端实例；向后端转发；
5. 获取后端应答；熔断状态更新；生成各类事件；
6. 事件上报模块：事件推送、通道健康检查；通道故障回落 TEMPLATE；独立`/health/event‑channel`探测端点；
7. 组装响应，附加客户端扩展响应头，返回调用方。

> 
> 执行流约束关键点：
> 
> 
> - 流式：**应用层完整 message 写出后，后置事件自动禁用 CALLBACK 接管**。
> - MCP FailOpen：仅网络层故障触发；熔断优先；切换向原后端下发`session_migrated`事件。
> - 护栏 CALLBACK 模式：护栏服务本身不可达，**不等待上游回调，直接执行本地模板**。

### 2.3 跨模块关键接口与生产者‑消费者【修订：补齐消费者、发送语义、时序规则】

1. 控制面 → 数据面：xDS‑ADS gRPC；生产者：控制面主副本；消费者：全部数据面 Pod；失败：租约窗口期 CRD 变更延迟生效。
2. 转译适配层 → 公共管道：标准化请求报文；生产者：转译适配层。
3. 转译适配层 → 事件上报模块：原始事件素材；生产者：转译适配层。
> 
> ✅【修订时序规则】转译适配层输出标准化报文至公共管道、输出事件素材至事件上报模块为**逻辑原子同步动作**；若转译适配层识别畸形报文执行拦截，事件上报同步产出`request_rejected`事件；事件流完整留存请求全生命周期记录，不会产生孤儿事件。
4. 公共管道 / MCP/A2A 子模块 → 事件上报模块：标准化业务事件；生产者：全链路；消费者：事件上报；失败：连接级故障回落 TEMPLATE，禁止递归上报故障事件。
5. 事件上报模块 → 上游 Agent‑Runtime 回调接口；生产者：网关；消费者：上游业务。
6. 后端连接熔断子模块 → MCP 会话子模块：后端健康状态；生产者：熔断模块；消费者：MCP 子模块。
7. ✅【新增接口条目】事件上报模块 → MCP 后端：会话生命周期事件（`session_migrated` / `session_terminated`）；生产者：事件上报模块；**消费者：MCP 后端（外部依赖域，非网关内核模块）；发送语义：网关内核仅保证事件写入传输层 socket；不保证对端 MCP 后端应用层已消费确认；后端处理失败、丢包属于下游外部依赖风险；内核不实现应答等待与重试机制。**

## 步骤 3｜对照六条动态契约逐条核验

### 契约① 信息配时序｜多副本 / 快照要有版本标识

✅已实现：

- xDS 快照携带版本；事件与客户端响应头携带`xds‑snapshot‑version`；
- 事件复合键：`request‑id+attempt‑number+xds‑snapshot‑version`；`attempt‑number`遵从透传规则，网关不自行生成重试序号。

> 
> ✅【修订补充版本优先级】MCP 协议版本判定优先级：`MCP‑Protocol‑Version`请求头优先级高于请求体`_meta.protocolVersion`；请求头与请求体版本冲突以请求头为准；全部缺失默认回退旧版 MCP‑2025‑11‑25；该规则属于契约①信息配时序版本标识维度。

⚠️契约缺口（条件式隐患）：

> 
> 当发生 xDS 配置收敛窗口期（多 Pod 不同快照版本），属于分布式分裂态；网关只做版本标记，**网关无法消除多 Pod 版本不一致本身**，只能可观测。

### 契约② 控制配轮询｜多主体竞争需要仲裁 / 轮询

✅已实现：

- 控制面 K8s leader‑election 选主仲裁，只有主副本执行 CRD status 回写；
- BackendTrafficPolicy 多策略命中：CALLBACK 优先级高于 TEMPLATE，明确合并仲裁规则；
- InferencePool 实例层负载均衡做后端实例轮询选择。
⚠️契约缺口：

> 
> Redis 全局限流：多 Pod 并发写入 Redis；Redis 本身提供竞争仲裁；网关侧不实现分布式锁；Redis 性能瓶颈属于外部载体约束。

### 契约③ 数据配权限｜隔离≠失真校验

✅已实现：

- RBAC 鉴权；CRD 准入校验（CEL 表达式字段访问域、toolNameStyle）；
> 
> ✅【修订字段属性】`callbackNotSupported`为网关**运行时内部只读标记，不属于 BackendTrafficPolicy CRD schema 字段；业务侧不能通过 CRD 配置、修改该标记；由转译适配层在应用层 message 写出环节自动设置；CRD schema 不包含此字段，准入校验不再引用该字段。**
- 区分：网关做**访问隔离**；网关**不做业务语义失真校验**；护栏业务判决仅采集上报，网关不解读业务判决；
- 模块私有状态 / 可公开状态通过权限原子化协议约束；下层不能拉取上层私有状态。
✅元理论区分贯彻到位：隔离 ≠ 失真校验。

### 契约④ 一致就向前｜跨模块流转状态要兼容

✅已实现：

1. 转译适配层输出标准化报文才流入公共管道；畸形报文拦截同时生成对应拒绝事件，无孤儿事件；
2. `callbackNotSupported=true`之后，不再进入 CALLBACK 分支；
3. 熔断输出后端健康状态，MCP 子模块消费该状态，熔断判断优先于 FailOpen 切换；
4. 租约窗口期未翻译完成的 CRD，不会生成 xDS 快照推送给数据面；
5. 下发 MCP 会话事件明确语义：仅保证传输层 socket 写出，不保证应用层确认。

⚠️契约缺口：

> 
> MCP：网关下发`session_migrated / session_terminated`事件给 MCP 后端；**网关不能强制保证 MCP 后端一定正确处理该事件**；后端拒绝 / 丢包属于下游外部依赖风险；网关仅记录事件发送，无法确认下游业务生效。

### 契约⑤ 能量带定义｜广义能量：并发、IO、token、调用次数，配套上限、过载、兜底

✅已实现：

- Local‑Rate‑Limit / Global‑Rate‑Limit 两层限流；Redis 降级策略；

> 
> ✅【修订补充能量维度】CALLBACK 回调等待路径占用并发槽位、TCP 连接、内存广义能量；CRD 配置回调等待并发上限；超限请求直接执行`callbackTimeoutMode`兜底，不进入等待队列；`callbackTimeout`时间维度 + 并发槽上限共同构成完整兜底约束。

- callback 超时兜底两套模式`FALLBACK_TO_TEMPLATE / DENY`；
- 事件上报通道连接级故障自动回落 TEMPLATE；
- 熔断：后端连续传输层错误快速失败；
- MCP `sessionTimeout`空闲超时（传输层下发终止信号）。
⚠️契约缺口：

1. 网关内核**没有全局 token 消耗硬上限**；token 仅做埋点观测；硬上限能力属于上层外壳扩展点；内核不实现。
2. mid‑stream‑fallback 开启，会带来内存能量风险，属于配置开启的代价，文档显式记录。

### 契约⑥ 状态要报备｜独立反馈通道，不能复用业务主链路作为唯一观测

✅已实现：

1. OTel trace/metrics 独立观测通路；
2. ✅【修订事件分类与通道映射】事件按消费用途划分两类，共享事件总线，携带`event_class`标记：
   - `event_class=business`业务事件：护栏判决、限流降级、熔断、会话迁移、xDS 版本；消费者：上游业务 Agent‑Runtime；
   - `event_class=audit`审计事件：`late_callback_discarded`、`securityDegraded`、`invalid_session`；消费者：安全审计系统；
3. 独立 HTTP 探测端点`/health/event‑channel`用于上游探测事件通道健康，**完全不进入事件总线；三类通路逻辑隔离，避免业务流量挤压审计链路。**
⚠️契约缺口：

> 
> 事件通道持续不可达，不会批量补发降级窗口期事件；设计决策：避免事件风暴；业务需要通过告警 + 外部补偿处理历史。

> 
> 六条动态契约核验小结：
> 没有机理层面致命契约破坏；全部缺口都是**条件式隐患**，有对应的观测、兜底，部分代价转嫁外部依赖 / 上层外壳。

## 步骤 4｜五类运行工况识别（区分单实例 / 多 Pod 分布式）

### 稳态

- CRD 配置正常；xDS 全部 Pod 收敛；Redis、护栏、事件上报通道全部健康；
- 请求全部走预期路径；限流、熔断均未触发；MCP 会话正常建立销毁；事件上报全部成功。

### 偏移态（局部轻度异常，进程存活，业务大体可用，需要外部干预）

1. Redis 出现抖动，触发 Global→Local 限流降级；响应标记`redis_degraded=true`，事件上报；流量继续运行，配额正确性下降。
2. 护栏服务偶发超时；按 FailClosed/FailOpen 模板执行，事件带`securityDegraded=true`。
3. 个别后端偶发网络故障；熔断触发；MCP 触发 FailOpen 会话迁移；生成新 session‑id，上报`session_migrated`事件。
4. 事件上报通道间歇性抖动；部分事件丢弃，回落 TEMPLATE；`event_channel_unavailable=true`标记。

### 冲突态（进程存活，但核心业务逻辑目标失效）

1. CALLBACK 模式 + 事件上报通道持续不可达：强制回落 TEMPLATE；**不再执行上游接管逻辑**；业务逻辑由模板接管，不再受上游业务控制。
2. MCP 后端持续返回业务错误；网关不会触发 FailOpen 切换（业务错误码不作为切换条件）；会话持续失败；不断上报会话异常事件。
3. ✅【修订，消除重复，改为引用】mid‑stream‑fallback 开启导致内存风险（详见条件式隐患 6；此处仅标记该风险对应的工况为冲突态，不复述风险内容）。

### 分裂态【仅多 Pod 分布式，单实例不会出现】

1. xDS 配置变更处于收敛窗口期；不同数据面 Pod 持有不同 xDS 快照版本；同一请求 LB 随机调度会命中不同策略版本；每条请求携带`xds‑snapshot‑version`用于可观测区分。
2. Redis 降级为 Local 限流，多 Pod 各自独立计数；全局实际 QPS 可能超出原 Global 配额。

### 崩溃态（链路断裂，输出不可信 / 服务不可用）

1. 全部后端实例熔断；全部请求路由无可用后端；返回后端不可用错误。
2. 数据面进程 OOM 崩溃；Pod 重启；无状态请求重建；MCP 依靠加密 session‑id 可跨 Pod 恢复路由绑定，但 Redis 辅助元数据丢失。

> 
> 工况关键结论：分裂态是该分布式 KC-AI‑Gateway**固有范式现象**，无法彻底消除，只能做标记、告警、业务侧规避。

## 步骤 5｜模块耦合遍历（固定顺序：旺 → 生 → 休 → 克 → 死）

模块集合：控制平面、转译适配层、公共管道、MCP 子模块、A2A 子模块、熔断子模块、事件上报模块、外部依赖 (redis / 护栏 / MCP‑Server)

### 旺｜双向增益

1. 控制平面 ⇔ 数据面 xDS‑ADS：控制面输出快照，数据面运行反馈状态，CRD status 回写 etcd；二者互相反馈，提升整体配置治理能力。

### 生｜单向供给（A 输出供给 B，B 不反向修改 A）

1. 转译适配层 → 公共管道：标准化请求报文。
2. 转译适配层 → 事件上报模块：原始事件素材。
3. 熔断子模块 → MCP 子模块：后端健康状态（熔断优先判断，供给 MCP 做 FailOpen 决策）。
4. 公共管道 / MCP/A2A → 事件上报模块：业务事件。
5. 事件上报模块 → MCP 后端：会话生命周期事件；生产者事件上报模块；消费者 MCP 后端（外部依赖）。

### 休｜休眠闲置（模块对象存在，无输入，不参与计算）

1. MCP 子模块：收到普通 LLM 无状态请求，MCP 模块休眠；配置对象保留，不处理业务。
2. CALLBACK 接管路径：`decisionMode=TEMPLATE`，回调链路休眠；对象存在，不推送事件等待上游指令。

### 克｜约束压制（A 占用资源 / 时序，压低 B 上限，但不摧毁 B 本身）

1. Redis 故障（外部）→ Global‑Rate‑Limit 被压制降级为 Local 限流；网关业务不中断，但配额正确性下降。
2. 事件上报通道不可达 → CALLBACK 接管功能被压制，强制回落 TEMPLATE 模板；网关转发 / 阻断功能仍然可用。
3. xDS 快照未更新的 Pod → 该 Pod 只能执行旧配置；新策略无法在此 Pod 生效；不摧毁网关转发能力。
4. CALLBACK 并发槽耗尽 → 新回调请求直接按兜底策略拒绝，不耗尽系统其他链路资源。

### 死｜破坏终止（A 触发直接摧毁 B 运行前置条件，流转断裂，B 功能失效）

1. 后端全部熔断 → 路由模块无可用后端实例；请求转发链路直接终止。
2. 数据面进程 OOM 崩溃 → 当前 Pod 全部处理链路终止。

> 
> 耦合遍历无异常机理矛盾；旺‑生‑休‑克‑死关系清晰。

## 步骤 6｜架构优势 + 条件式隐患集合

### ✅架构优势

1. 严格区分传输层网关职责与上游业务语义判定；网关不做业务裁决；业务判定收敛上游 Agent‑Runtime。
2. 控制‑数据面分离；控制面崩溃存量流量继续运行；
3. MCP 新旧协议完整兼容；session‑id 加密 token 作为路由绑定真相源，Redis 仅辅助缓存。
4. 两套处置路径 TEMPLATE / CALLBACK；完整兜底；流式场景自动禁用 CALLBACK；补充 CALLBACK 并发 + 时间双重能量兜底。
5. 完整独立观测体系，满足契约⑥状态要报备；区分业务 / 审计事件；独立探测端点；大量审计标记。
6. 熔断、FailOpen 逻辑严格区分传输层信号与业务错误码；避免业务错误触发会话迁移 / 熔断。
7. 所有分布式分裂态现象全部做版本标记，可观测可告警。

### ⚠️条件式隐患（格式：当【条件】触发，则【风险现象】；同时写明网关现有缓解手段）

1. **隐患 1｜xDS 收敛窗口分裂态**

> 
> 条件：K8s CRD 变更，xDS 尚未在全部 Pod 完成快照推送。
> 风险：LB 随机分发，同一请求可能命中不同策略版本 Pod。
> 缓解：每条请求携带`xds‑snapshot‑version`响应头 + 事件属性；事件复合键区分；上层业务观测告警；网关无法彻底消除该分布式固有现象。

2. **隐患 2｜Redis 降级多 Pod 配额叠加超量**

> 
> 条件：Redis 故障，Global 切换 Local 限流，使用保守折算阈值。
> 风险：多 Pod 同时处理流量，合计实际 QPS 可能超过原全局配额。
> 缓解：`redis_degraded=true`标记；事件上报；上层业务做计费补偿；属于范式固有约束。

3. **隐患 3｜MCP FailOpen 会话迁移，业务上下文无法由网关保证继承**

> 
> 条件：MCP 后端网络故障触发 FailOpen 切换 target。
> 风险：新后端没有原有业务会话上下文。
> 缓解：网关下发`session_migrated`事件上报上游；由 Agent‑Runtime 做 initialize 重初始化；网关仅处理传输层迁移。

4. **隐患 4｜事件上报通道不可达，CALLBACK 接管完全失效**

> 
> 条件：事件通道 TCP 连接级故障。
> 风险：无法推送事件，无法执行上游接管逻辑；强制回落 TEMPLATE 模板。
> 缓解：`event_channel_unavailable=true`标记；独立`/health/event‑channel`探测端点告警；无积压事件补发。

5. **隐患 5｜MCP 后端丢弃`session_migrated / session_terminated`通知事件**

> 
> 条件：MCP‑Server 业务后端丢包、bug，不处理网关下发迁移 / 终止事件。
> 风险：后端残留僵尸会话。
> 缓解：网关仅保证写入传输层 socket；观测事件发送埋点；会话回收责任属于 MCP 后端。

6. **隐患 6｜mid‑stream‑fallback 开启内存风险**

> 
> 条件：业务开启 mid‑stream‑fallback。
> 风险：完整 SSE 流缓存会造成内存暴涨 OOM。
> 缓解：文档显式记录代价；由业务评估是否开启；内核不默认开启。

7. **隐患 7｜护栏 CALLBACK 模式，护栏服务可达但上游业务处理超时**

> 
> 条件：护栏服务正常，但是上游回调接口处理超时 / CALLBACK 并发槽耗尽。
> 风险：请求阻塞直到 callbackTimeout，然后按 callbackTimeoutMode 兜底；并发耗尽直接执行兜底。
> 缓解：超时 + 并发双重兜底；晚到达指令丢弃并审计`late_callback_discarded`。

## 步骤 7｜重构备选方案（按落地等级标记）

> 
> 说明：当前 V4 架构已经是权衡后的方案；下面是备选改进思路，用于复审对比。

表格

| 备选方案 | 修复目标 | 落地等级 | 代价 |
| --- | --- | --- | --- |
| 方案 A：增加全局流量排队，等待全部 Pod xDS 收敛完成再放行新请求 | 彻底消除 xDS 收敛窗口分裂态 | ⚠️修改本体 | 大集群下配置更新会造成新请求阻塞；发布变更抖动风险上升；牺牲可用性换取一致性。当前架构没有选择该方案，选择可观测告警路线。 |
| 方案 B：网关内核实现 token 硬上限 | 在内核完成 token 硬截断，不再依赖上层外壳 | ⚠️修改本体 | 网关需要解析 token 计数；引入业务相关统计逻辑，增大内核边界膨胀；违背网关尽量不触碰业务语义的设计原则；当前方案放在上层外壳。 |
| 方案 C：事件通道故障时持久化本地磁盘队列，通道恢复补发事件 | 解决事件通道不可达丢失事件 | ⚠️修改本体 | 引入本地磁盘 IO；Pod 漂移磁盘数据丢失；增加运维复杂度；架构选择：不做补发，靠告警 + 上层补偿。 |
| 方案 D：网关主动校验 MCP 后端对`session_migrated`事件应答 | 网关确认后端已处理迁移事件才允许新会话流量 | ⚠️修改本体 | MCP 标准没有强制应答语义；会阻塞 MCP 会话切换链路；引入 MCP 业务语义判断，越界；不采纳。 |
| 方案 E：CALLBACK 路径全局队列持久化 | 持久化等待回调的请求 | ⚠️修改本体 | 大幅增加存储复杂度；请求生命周期变长；不采纳，采用时间 + 并发双重兜底。 |

## 步骤 8｜闭环复审 + 架构级测试设计

### 8.1 缺陷分类（范式固有 / 项目特有 / 载体硬件缺陷）

1. **范式固有缺陷（机理无法消除，只能观测、告警、上层补偿）**
   - xDS 多 Pod 配置收敛窗口分裂态
   - Redis 降级后多 pod 配额叠加风险
   - ✅【修订增加引用】MCP FailOpen 无法保证业务上下文继承（详见条件式隐患 3；属于传输层迁移与业务上下文继承的固有边界）
   - mid‑stream‑fallback 内存代价
2. **项目特有缺陷：当前 V4 架构无发现；全部缺口均为条件式隐患，配套有缓解手段**
3. **载体硬件 / 外部依赖缺陷：Redis、MCP 后端、护栏服务、etcd 本身故障，属于外部载体，网关做兜底与上报。**

### 8.2 架构级测试用例集合（架构层面，不涉及具体代码实现；输入条件｜期望输出｜观测证据）

> 
> 分为：正向稳态测试、偏移态测试、冲突态测试、分裂态测试、故障注入测试；适配本次修订新增全部规则。

#### ✔️稳态正向测试

1. 普通 OpenAI LLM 请求，TEMPLATE 模式，全部外部依赖健康

- 输入：合法 OpenAI 请求；xDS 全部 Pod 收敛；Redis、护栏正常；`decisionMode=TEMPLATE`；CALLBACK 并发槽充足
- 期望：鉴权‑限流‑护栏‑路由正常返回应答；无阻断；
- 观测证据：trace 完整；无异常审计标记；`event_class=business`业务事件。

2. MCP‑2026‑07‑28 新版协议请求

- 输入：携带`MCP‑Protocol‑Version:2026‑07‑28`头 MCP 请求；CRD `sessionRouting=true`
- 期望：强制进入 stateless 管道，stateful 会话绑定逻辑禁用；prefixMode 依旧生效；请求头版本优先于`_meta`内版本。
- 观测证据：事件流标记协议版本；无 session‑id 生成。

3. MCP 请求，请求头版本与请求体_meta 版本冲突

- 输入：`MCP‑Protocol‑Version:2026‑07‑28`；请求体`_meta.protocolVersion=2025‑11‑25`
- 期望：以请求头为准，走新版 stateless 管道。
- 观测证据：事件记录生效协议版本来自 HTTP 请求头。

4. 旧版 MCP‑2025‑11‑25 开启 stateful，正常会话生命周期

- 输入：旧版 MCP initialize，工具调用，空闲会话；客户端不带`X‑Attempt‑Number`
- 期望：AES‑256‑GCM session‑id 生成；`attempt‑number=0`；传输层空闲超时下发`session_terminated`事件写入 socket；
- 观测证据：会话生命周期完整事件；`event_class=business`。

5. CALLBACK 接管模式，护栏判决由上游业务覆盖，并发槽充足

- 输入：`decisionMode=CALLBACK`；护栏服务可达；CALLBACK 并发槽未满；上游回调返回 PASSTHROUGH/deny；
- 期望：网关推送业务事件，严格执行上游指令；
- 观测证据：事件流包含护栏原始判决。

#### ⚠️偏移态故障注入测试

6. Redis 故障，Global‑Rate‑Limit 降级 Local

- 输入：Redis 断开；Global 限流开启；CRD 配置`maxExpectedReplicaCount`；
- 期望：切换 Local，计数器使用保守折算阈值；响应头`redis_degraded=true`；事件上报；业务不中断。
- 观测证据：`redis_degraded`标记，降级事件。

7. 护栏服务不可达，TEMPLATE 模式 FailClosed 默认

- 输入：护栏服务宕机；`decisionMode=TEMPLATE`，模板 FailClosed；
- 期望：请求阻断；事件上报；标记`securityDegraded=true`。
- 观测证据：`event_class=audit`审计标记`securityDegraded`。

8. MCP 后端网络超时，触发熔断优先，随后 FailOpen 切换

- 输入：MCP 后端网络连接超时；MCPBackend 开启 FailOpen；
- 期望：熔断优先判定；触发 FailOpen；生成全新 session‑id；向原 MCP 后端 socket 写入`session_migrated`事件；事件上报上游；网关不等待后端应用层应答。
- 观测证据：`session_migrated`业务事件；新 session‑id 在响应返回。

9. CALLBACK 模式，上游回调超时 `callbackTimeoutMode=DENY`，并发槽充足

- 输入：护栏服务可达，上游回调接口卡死超时；
- 期望：超时触发 DENY 拒绝；晚到达的上游指令丢弃，记录`late_callback_discarded`审计事件。
- 观测证据：`event_class=audit`，`late_callback_discarded`标记。

10. CALLBACK 模式，请求数量耗尽回调并发槽

- 输入：大量 CALLBACK 请求打满 CRD 配置的回调并发上限；
- 期望：超限请求直接执行`callbackTimeoutMode`兜底，不进入等待队列。
- 观测证据：对应拒绝事件。

11. SSE 流式，输出首个完整应用层 message 之后，发生后置护栏事件

- 输入：SSE 流，已经输出第一条完整应用层 message；`decisionMode=CALLBACK`；
- 期望：自动置`callbackNotSupported=true`，回落 TEMPLATE，不再等待上游回调；已经下发 chunk 不回滚。
- 观测证据：事件标记 callbackNotSupported 激活。

12. 畸形报文输入转译适配层

- 输入：畸形 MCP 报文；TEMPLATE 模式；
- 期望：转译适配层拦截；同步产出`request_rejected`事件；事件流完整记录；无孤儿事件。
- 观测证据：`request_rejected`业务事件。

#### 🧩分裂态测试（多 Pod 场景）

13. xDS 配置更新，处于收敛窗口期，Pod1 新快照，Pod2 旧快照；LB 随机调度

- 输入：CRD 变更，部分 Pod 已推送新 xDS，部分 Pod 未更新；
- 期望：不同 Pod 执行各自快照逻辑；每个请求响应头携带`xds‑snapshot‑version`；事件复合键`request‑id+attempt‑number+xds‑snapshot‑version`；网关不阻塞流量。
- 观测证据：可观测到同一 request‑id 下不同快照版本事件。

14. Redis 降级，多 Pod 并发流量

- 输入：Redis 断开，3 个 Pod；Global 配额 100QPS；
- 期望：每个 Pod 按保守阈值限流；响应标记`redis_degraded=true`；事件上报；网关不阻止多 Pod 叠加超配额的客观现象。
- 观测证据：各 Pod 独立 Local 计数器指标。

#### 🚨冲突态测试

15. 事件上报通道 TCP 连接级持续不可达；存在 CALLBACK 策略

- 输入：事件回调接口网络断开；策略`decisionMode=CALLBACK`；
- 期望：连接级故障直接回落 TEMPLATE 模板；响应头`event_channel_unavailable=true`；**不会等待 callbackTimeout**；不递归上报通道故障事件。
- 观测证据：通道异常事件；无递归事件风暴；独立健康端点返回异常。

16. CALLBACK 模式，护栏服务本身不可达

- 输入：`decisionMode=CALLBACK`；护栏服务宕机；
- 期望：不进入回调等待；直接执行 FailClosed/FailOpen 模板；
- 观测证据：护栏不可达事件，无 CALLBACK 等待计时。

17. MCP 后端返回业务错误码，非网络故障

- 输入：MCP 后端返回 JSON‑RPC 业务错误码，TCP 连接正常；MCPBackend 配置 FailOpen；
- 期望：**不触发熔断，不触发 FailOpen 切换；会话保持，返回业务错误；**
- 观测证据：业务错误事件，无 session_migrated 事件。

#### 🧪边界特殊工况

18. MCP 请求不带任何版本头

- 输入：MCP 请求，既无`MCP‑Protocol‑Version`头，`_meta`也缺失版本；
- 期望：默认旧版 MCP‑2025‑11‑25 行为，`sessionRouting=true`生效；遵从 CRD 全局 MCP 默认版本策略。
- 观测证据：事件记录协议版本 unknown，使用旧版处理管道。

19. MCP prefixMode 开启；后端返回已经带 backendName 前缀的工具名

- 输入：开启 prefixMode；`toolNameStyle=namespaced`，后端工具名自带前缀；
- 期望：网关不再重复叠加前缀，避免 A_A_search 双重前缀。
- 观测证据：工具名无重复前缀。

20. 控制面选主租约窗口期提交 CRD 变更

- 输入：旧主崩溃，租约未过期，kubectl apply 新 CRD；
- 期望：CRD 写入 etcd；等待新主选出后翻译推送 xDS；status `observedGeneration`滞后更新；存量 Pod 继续运行旧快照；不丢失配置版本。
- 观测证据：CRD status 字段，reconciledVersion 滞后。

21. 客户端不带 X‑Attempt‑Number 重试请求

- 输入：相同业务请求重复发送，无`X‑Attempt‑Number`头；
- 期望：网关全部置`attempt‑number=0`；视为独立请求；事件键存在冲突风险。
- 观测证据：事件 attempt‑number=0；事件提示上游依靠时间戳 + xds 快照版本区分。

---

> 
> 当前状态：全部 9 条审查点修复完毕；补齐契约缺口、生产者‑消费者语义、能量维度、版本优先级、事件分类、时序规则；测试用例同步适配全部新增架构约束； 审查结论：通过。

---

# 附属扩展工件集合｜并入基点 `KUANGCHAO‑BASE‑V1P0‑ROOT‑R1`

> 
> 说明：**不修改原有板块 0/1/2 封档本体，仅新增附属配套工件**；用于风险追溯、测试关联、线上故障排查、缺陷归类评审；防御 “调优参数包装为架构契约缺口” 类审查陷阱。
> 包含三件工件：
> 工件 A｜隐患‑测试‑待确认参数三维映射对照表
> 工件 B｜AI‑Gateway 实例化故障排查模板（元理论故障解析七小步落地工单模板）
> 工件 C｜缺陷三分分类判定表：范式固有机理取舍 / 项目实现 bug / 载体资源约束

---

## 📄工件 A：隐患‑测试‑待确认参数三维映射对照表

> 
> 列说明
> 
> 
> - 隐患 ID：来自八步分析【条件式隐患】编号
> - 隐患简述：风险极简摘要
> - 对应架构测试用例编号：引用测试集合内用例 ID
> - 关联待确认参数：来自 V4 架构文档待确认事项，仅参数，**不含架构硬契约**
> - 备注：该风险关键约束、是否范式固有

表格

| 隐患 ID | 隐患简述 | 对应架构测试用例 | 关联待确认参数 | 备注 |
| --- | --- | --- | --- | --- |
| 隐患 1 | xDS 收敛窗口分裂态；同一请求多 Pod 命中不同策略版本 | 用例 13 | xDS‑ADS 相关选主、租约超时参数 | ✅范式固有；仅可观测告警，无法彻底消除 |
| 隐患 2 | Redis 降级后多 Pod Local 计数叠加，总流量可能超原全局配额 | 用例 14 | maxExpectedReplicaCount 配置、Redis 探测周期 | ✅范式固有；上层业务计费补偿缓解 |
| 隐患 3 | MCP FailOpen 切换 target，网关不保证业务上下文继承 | 用例 8 | sessionTimeout 默认基准数值 | ✅范式固有；网关仅传输层迁移；由 Agent‑Runtime 重 initialize |
| 隐患 4 | 事件上报通道 TCP 级不可达，CALLBACK 接管失效，强制回落 TEMPLATE | 用例 15 | 事件上报通道健康检测探测周期 | ✅范式固有；提供独立健康探测端点；无事件补发 |
| 隐患 5 | MCP 后端丢弃`session_migrated / session_terminated`事件，产生僵尸会话 | 用例 8 | sessionTimeout 默认基准数值 | 🟡载体外部依赖；网关仅保证传输层 socket 写出，不做应用层确认 |
| 隐患 6 | mid‑stream‑fallback 开启带来内存暴涨 OOM 风险 | 用例无直接单条；冲突态第 3 条覆盖 | 无参数；配置开关类业务选择 | ✅范式固有代价；业务评估后决定是否开启 |
| 隐患 7 | CALLBACK 护栏可达但上游回调超时 / 并发槽耗尽，执行兜底 | 用例 9、用例 10 | callbackTimeout 超时阈值；CALLBACK 最大并发槽上限 | 🟠架构约束维度存在；并发数、超时属于实现调优参数 |

> 
> 映射表使用规则
> 
> 
> 1. 修改任意一条条件式隐患：同步核查对应测试用例、关联待确认参数是否需要更新；
> 2. 待确认参数全部为**实现调优参数**，不在架构契约写死具体数值；
> 3. 范式固有风险：**不能通过修改内核代码彻底消除，只能缓解、观测、上层补偿**。

---


## 📄工件 B：KC-AI‑Gateway 实例化故障排查模板

> 
> 来源：狂潮元理论【故障解析七小步】子流程；转化为可直接复制填写的线上排查工单模板。
> 使用时机：生产 / 测试环境出现异常现象，启动故障定位；禁止直接跳去改代码，严格按顺序填写。

```
# KC-AI‑Gateway 故障排查工单
## 1、采集可观测故障现象
- 故障发生时间：
- 复现条件：可复现 / 偶现 / 仅高负载 / 配置变更后触发
- 客户端侧现象（HTTP返回码、响应头标记：redis_degraded / event_channel_unavailable等）：
- 网关侧观测：OTEL metrics、trace、事件总线event_class业务/审计事件；xds‑snapshot‑version；request‑id、attempt‑number
- 外部组件状态：Redis、护栏服务、MCP‑Server、K8s控制面状态

## 2、归类：属于【权限‑状态流故障】还是【边界‑执行流故障】
□ 权限‑状态流故障：版本、快照、计数器、会话元数据、事件时序、多副本状态不一致
□ 边界‑执行流故障：准入、限流、熔断、CALLBACK/TEMPLATE路径、超时、降级、路由执行逻辑
> 备注说明：

## 3、二分区分故障大类（参考工件C缺陷三分表）
□ 范式固有机理取舍（机理限制，无法彻底根除，只能观测+上层补偿）
□ 项目实现bug（架构设计正确，代码实现偏离契约）
□ 载体资源/外部依赖缺陷（Redis、MCP后端、护栏、K8s等外部组件故障）
> 判定理由：

## 4、定位对应六条动态契约中哪一条/多条出现缺口
契约①信息配时序 □
契约②控制配轮询 □
契约③数据配权限 □
契约④一致就向前 □
契约⑤能量带定义 □
契约⑥状态要报备 □
> 描述具体缺口表现：

## 5、记录故障完整可观测表征
> 把现象、事件标记、指标、traceId、request‑id汇总；不做主观猜测，只写客观观测。
观测表征：

## 6、给出修复/缓解方案，写明权衡代价
> 如果是范式固有：写缓解手段（告警、上层业务改造、配置调整），**不提出修改内核架构本体**
> 如果是实现bug：写修复点；不需要刷新基点
> 如果是载体依赖：写明外部组件修复手段，网关侧仅兜底上报
方案：
代价与副作用：

## 7、方案复核
- 是否消除原有故障现象：
- 是否引入新副作用、新风险：
- 是否需要更新架构文档/测试用例：
    □ 不需要（仅实现层改动）
    □ 需要，触发 `封档 #刷新基点`（仅机理契约缺陷才勾选此项）

### 附件关联
关联隐患ID：________
关联测试用例ID：________
关联待确认参数：________
```

---

## 📄工件 C：缺陷三分分类判定表

> 
> 用途：评审、bug 复盘、故障排查第一步，先做归类；**只有范式固有机理取舍发现机理错误，才允许触发基点刷新；另外两类不需要改动基点。**

表格

| 分类 | 定义 | 典型 KC-AI‑Gateway 实例 | 是否需要修改基点 | 处理原则 |
| --- | --- | --- | --- | --- |
| 🔵范式固有机理取舍 | 架构机理本身客观带来的限制；单模块逻辑正确，但多模块、分布式交互下必然存在该风险；不能靠改代码彻底根除，只能观测、告警、上层业务补偿。 | 1.xDS 多 Pod 收敛窗口分裂态2.Redis 降级后 Local 计数叠加超配额3.MCP FailOpen 不保证业务上下文继承4.mid‑stream‑fallback 内存代价 | ❌一般不需要；**只有发现机理契约本身写错、缺失才刷新基点** | 不修改内核架构本体；手段：可观测标记、事件上报、告警、上层业务补偿；不能把范式固有风险作为 bug 提修改内核契约。 |
| 🟡项目实现 bug | 架构设计、契约、规则本身是正确；**代码实现偏离架构契约**；逻辑、分支、条件判断写错。 | 1. 判定条件写反，CALLBACK 与 TEMPLATE 分支颠倒2. 未按规则生成 / 透传 request‑id3. 熔断错误把业务错误码当作网络故障4. 事件没有携带 event_class 分类标记 | ❌不需要刷新基点；只修改代码实现 | 修改 Rust / 控制面实现；修改完成后运行对应架构测试用例回归；**架构基点保持不变**。 |
| 🔴载体资源约束 / 外部依赖缺陷 | 故障根源不在网关内核；来自外部系统、中间件、基础设施；网关只做兜底、事件上报、标记。 | 1.Redis 宕机、网络抖动2.MCP‑Server 丢弃 session_migrated 事件3. 第三方护栏服务超时不可达4.K8s etcd、lease 租约异常 | ❌完全不触碰基点 | 修复外部载体；网关侧确认兜底逻辑正常；网关不负责修复外部组件本身。 |

> 
> 🚩重要判定校验问题（遇到争议 bug 依次自问，防御把调优问题包装为架构缺陷）

1. 如果把并发数、超时、探测周期这类**数值调大 / 调小，故障现象是否消失？** → 大概率是调优参数问题，不属于架构机理缺陷。
2. 是否该现象在分布式系统同类网关属于客观固有代价？ → 倾向范式固有。
3. 是否文档架构契约已经明确写明该风险与缓解手段？ → 若是，优先归范式固有 / 载体，不是架构 bug。

---

# 基点更新记录

> 
> 基点标识：`KUANGCHAO‑BASE‑V1P0‑ROOT‑R1`
> 更新动作：新增三份附属扩展工件；板块 0/1/2 原有封档内容**完全未改动**。
> 工件清单：
> 
> 
> - 工件 A：隐患‑测试‑待确认参数三维映射对照表
> - 工件 B：KC-AI‑Gateway 实例化故障排查模板（元理论故障解析七小步工单模板）
> - 工件 C：缺陷三分分类判定表
> 
> 
> 整套交付物现在完整集合：
> 
> 
> 1. 板块 0｜狂潮元理论 V1.0‑public（公理根底层）
> 2. 板块 1｜KC-AI‑Gateway V4 架构设计终版文档
> 3. 板块 2｜八步正向分析 + 架构级测试集合
> 4. 附属工件 A/B/C：映射表、排查工单模板、缺陷三分判定表


---

图1

flowchart TB
  subgraph UP["上游域"]
    U1["客户端业务请求"]
    U2["上游 Agent Runtime<br/>唯一业务判定主体"]
    U3["上游事件回调接管接口"]
  end

  subgraph GW["KC-AI-Gateway 内核域"]
    subgraph CP["控制平面"]
      C1["K8s Gateway-API 兼容 CRD<br/>AIGatewayPolicy / MCPBackend<br/>BackendTrafficPolicy / InferencePool"]
      C2["CRD 翻译"]
      C3["xDS-ADS gRPC 双向流"]
      C4["CRD status 回写<br/>observedGeneration / reconciledVersion"]
      C5["leader-election 选主<br/>仅主副本写 status"]
    end

    subgraph DP["Rust 数据面"]
      D0["转译适配层<br/>协议归一 / 报文重组 / request-id<br/>应用层 message 粒度 callbackNotSupported"]
      D1["公共管道"]
      D11["RBAC 鉴权"]
      D12["Local-Rate-Limit<br/>CEL 描述符"]
      D13["Global-Rate-Limit<br/>Redis"]
      D14["Guardrail 采集<br/>只上报原始判决"]
      D15["decisionMode 仲裁<br/>TEMPLATE / CALLBACK"]
      D16["路由 / 协议转换"]
      D17["熔断<br/>仅传输层信号"]
      D18["InferencePool 实例层路由"]
      D2["MCP 子域"]
      D21["MCP-Protocol-Version + _meta 分流"]
      D22["stateless / stateful"]
      D23["prefixMode / toolNameStyle"]
      D24["session-id AES-256-GCM<br/>backend-pinning 路由真相源"]
      D3["A2A 子域"]
      D4["事件上报模块"]
      D5["OTel trace / metrics"]
    end
  end

  subgraph SHELL["上层外壳域"]
    S1["spend 金额预算"]
    S2["UI 控制台 / helm / 大盘"]
    S3["payload 完整日志持久化"]
    S4["SSE 应用层重试兜底"]
    S5["业务侧事件回调消费与判定"]
  end

  subgraph EXT["外部依赖域"]
    E1["K8s etcd"]
    E2["Redis"]
    E3["LLM 推理服务"]
    E4["MCP Server"]
    E5["A2A Agent"]
    E6["第三方 Guardrail 护栏"]
    E7["Kueue / Volcano / K8s-Scheduler"]
    E8["OTel-Collector / Prometheus"]
  end

  U1 --> D0
  U2 --> U3
  U3 --> D4
  C1 --> C2 --> C3 --> D0
  C1 --> C4
  C5 --> C4
  D0 --> D1
  D1 --> D11 --> D12 --> D13 --> D14 --> D15 --> D16 --> D17 --> D18
  D16 --> D2
  D16 --> D3
  D2 --> D21 --> D22
  D2 --> D23
  D2 --> D24
  D1 --> D4
  D2 --> D4
  D3 --> D4
  D4 --> U3
  D1 --> D5 --> E8
  D13 --> E2
  D14 --> E6
  D17 --> E3
  D18 --> E3
  D2 --> E4
  D3 --> E5
  C1 --> E1
  E7 --> D18
  D4 --> E8
  S1 -.-> D1
  S5 -.-> U3


---

图2

flowchart LR
  R0["请求入站"] --> R1["转译适配层<br/>解析协议 / 生成或透传 request-id<br/>attempt-number 透传，否则记 0"]
  R1 --> R2["协议版本识别<br/>MCP-Protocol-Version 优先于 _meta"]
  R2 --> R3["RBAC 鉴权"]
  R3 --> R4["Local-Rate-Limit"]
  R4 --> R5["Global-Rate-Limit<br/>Redis 不可用则 redis_degraded=true"]
  R5 --> R6["Guardrail 调用<br/>采集原始判决"]
  R6 --> R7{"decisionMode"}
  R7 -->|TEMPLATE| R8["执行 CRD 传输动作模板"]
  R7 -->|CALLBACK| R9["推送标准化事件至上游<br/>等待阻断或透传指令"]
  R9 --> R10{"事件通道连接级不可达?"}
  R10 -->|是| R8
  R10 -->|否| R11{"callbackTimeout 超时?"}
  R11 -->|是| R12["按 callbackTimeoutMode<br/>FALLBACK_TO_TEMPLATE 或 DENY"]
  R11 -->|否| R13["执行上游指令"]
  R8 --> R14["MCP / A2A 子模块处理"]
  R13 --> R14
  R14 --> R15["InferencePool / 后端路由"]
  R15 --> R16["下游后端"]
  R16 --> R17["熔断状态更新<br/>仅采信传输层信号"]
  R17 --> R18["生成标准化事件<br/>request-id + attempt-number + xds-snapshot-version"]
  R18 --> R19["事件上报模块"]
  R19 --> R20["响应组装"]
  R20 --> R21["客户端扩展响应头<br/>redis_degraded / event_channel_unavailable / xds-snapshot-version"]
  R21 --> R22["返回客户端"]
  R20 --> R23["绝不向转发后端注入网关内部状态标记头"]


---

图3

flowchart TB
  A["异常或干预事件"] --> B{"匹配 BackendTrafficPolicy"}
  B -->|全部 TEMPLATE| C["路径 A TEMPLATE<br/>按 CRD 传输动作模板执行"]
  B -->|任意一条 CALLBACK| D["路径 B CALLBACK<br/>推送标准化事件至上游"]

  D --> E{"事件通道 TCP 连接级可达?"}
  E -->|不可达| C
  E -->|可达| F{"上游在 callbackTimeout 内返回指令?"}
  F -->|否| G{"callbackTimeoutMode"}
  G -->|FALLBACK_TO_TEMPLATE| C
  G -->|DENY| H["拒绝"]
  F -->|是| I["执行上游阻断 / 透传指令"]
  F -->|晚到指令| J["丢弃晚到指令<br/>仅记录 late_callback_discarded 审计事件"]

  K["流式请求"] --> L{"应用层首个完整 message 已写出?"}
  L -->|否| D
  L -->|是| M["callbackNotSupported=true<br/>强制回落 TEMPLATE"]

  N["Guardrail 服务不可达"] --> O{"decisionMode=CALLBACK?"}
  O -->|是| C
  O -->|否| P["按 FailOpen / FailClosed 模板执行"]

  Q["事件上报通道持续不可达"] --> R["自动回落 TEMPLATE<br/>event_channel_unavailable=true<br/>禁止递归上报故障事件<br/>恢复不自动切回 CALLBACK<br/>不补发积压事件"]

  C --> S["执行传输动作"]
  H --> S
  I --> S
  M --> S
  P --> S
  R --> S

---

图4

flowchart TB
  M0["MCP 请求"] --> M1{"MCP-Protocol-Version 请求头存在?"}
  M1 -->|有| M2["以请求头版本为准"]
  M1 -->|无| M3{"_meta.protocolVersion 存在?"}
  M3 -->|有| M4["使用 _meta 版本"]
  M3 -->|无| M5["默认旧版 MCP-2025-11-25"]

  M2 --> M6{"版本 = 2026-07-28?"}
  M4 --> M6
  M6 -->|是| M7["stateless 管道<br/>stateful 会话绑定禁用<br/>prefixMode 仍生效"]
  M6 -->|否| M8{"sessionRouting?"}
  M8 -->|false| M9["stateless 管道"]
  M8 -->|true 或无协议头默认| M10["stateful 管道"]

  M10 --> M11["session-id AES-256-GCM<br/>backend-pinning 路由真相源"]
  M11 --> M12["Redis 仅可选辅助元数据<br/>可丢失"]
  M10 --> M13["sessionTimeout 空闲判定<br/>仅业务 JSON-RPC message 重置<br/>keepalive / ping / GET 长轮询不重置"]
  M10 --> M14["网关仅传输层空闲检测<br/>向 MCP 后端下发 session_terminated"]
  M10 --> M15["会话业务回收由 MCP 后端执行"]
  M14 --> M16["session_terminated"]
  M15 --> M17["session_expired"]

  M11 --> M18["FailOpen 触发仅网络层故障<br/>业务错误码不触发"]
  M18 --> M19["熔断健康检测优先"]
  M19 --> M20{"failureMode"}
  M20 -->|FailClosed| M21["标记会话失败"]
  M20 -->|FailOpen| M22["切换 target<br/>生成全新 session-id<br/>重置后端熔断计数"]
  M22 --> M23["向原后端下发 session_migrated"]
  M23 --> M24["标准化会话迁移事件上报上游"]
  M24 --> M25["网关不做跨 target 业务上下文兼容判定<br/>上下文兼容交上游 Agent Runtime"]

---

图5

flowchart TB
  subgraph NS["南北向独立网关部署"]
    NS1["集群级集中控制面<br/>多副本 + K8s leader-election"]
    NS2["xDS 推送至全部网关数据面 Pod"]
    NS3["Data Plane Pod A"]
    NS4["Data Plane Pod B"]
    NS5["Data Plane Pod N"]
    NS1 --> NS2
    NS2 --> NS3
    NS2 --> NS4
    NS2 --> NS5
  end

  subgraph EW["东西向 sidecar 部署"]
    subgraph P1["业务 Pod 1"]
      P11["sidecar 控制面单实例<br/>无多副本 / 无 leader-election"]
      P12["sidecar 数据面"]
      P11 -->|xDS 仅本 Pod| P12
    end
    subgraph P2["业务 Pod 2"]
      P21["sidecar 控制面单实例<br/>无多副本 / 无 leader-election"]
      P22["sidecar 数据面"]
      P21 -->|xDS 仅本 Pod| P22
    end
    P11 -->|apiserver watch| K8S["K8s apiserver / etcd"]
    P21 -->|apiserver watch| K8S
    P11 -->|按需长连接推送<br/>禁止全量广播大快照| P12
    P21 -->|按需长连接推送<br/>禁止全量广播大快照| P22
  end

  K8S --> NS1
  REDIS["Redis"] --> NS3
  REDIS --> NS4
  REDIS --> P12
  REDIS --> P22
  UPSTREAM["上游 Agent Runtime / 业务"] --> NS3
  UPSTREAM --> NS4
  UPSTREAM --> P12
  UPSTREAM --> P22

---

图6

flowchart LR
  subgraph SRC["事件来源"]
    E1["限流事件"]
    E2["熔断事件"]
    E3["会话迁移事件"]
    E4["Guardrail 判决事件"]
    E5["Redis 降级事件"]
    E6["xDS 快照版本事件"]
    E7["late_callback_discarded"]
    E8["securityDegraded"]
    E9["invalid_session"]
    E10["request_rejected"]
  end

  SRC --> ER["事件上报模块<br/>标准化事件 schema<br/>复合键 request-id + attempt-number + xds-snapshot-version"]

  ER --> EB{"event_class"}
  EB -->|business| B1["业务事件总线<br/>消费者：上游 Agent Runtime"]
  EB -->|audit| A1["审计事件总线<br/>消费者：安全审计系统"]

  ER --> OTEL["OTel trace / metrics"]
  OTEL --> COLL["OTel-Collector / Prometheus"]

  HC["独立 HTTP 探测端点<br/>/health/event-channel"] --> UP["上游业务轮询感知"]
  HC --> ER
  ER -->|通道不可达| FB["回落 TEMPLATE<br/>event_channel_unavailable=true<br/>禁止递归上报<br/>恢复不自动切回 CALLBACK<br/>不补发积压事件"]

  RESP["客户端响应头<br/>redis_degraded / event_channel_unavailable / xds-snapshot-version"] --> CLIENT["客户端可观测"]
  RESP --> NOINJ["不注入向后端转发请求"]

---

图7

flowchart TB
  subgraph PS["权限-状态流：信息数据流"]
    P1["CRD 原始资源<br/>observedGeneration / reconciledVersion"]
    P2["xDS 配置快照<br/>xds-snapshot-version"]
    P3["Local 限流计数器"]
    P4["Global 限流计数<br/>Redis"]
    P5["MCP 会话传输层状态<br/>session-id AES-256-GCM"]
    P6["事件元组<br/>request-id / attempt-number / xds-snapshot-version"]
    P7["熔断后端健康状态表"]
    P8["事件流<br/>business / audit"]
    P1 --> P2
    P2 --> P3
    P4 --> P3
    P5 --> P6
    P3 --> P7
    P6 --> P8
  end

  subgraph ES["边界-执行流：能量控制流"]
    E1["请求准入"]
    E2["RBAC 鉴权"]
    E3["Local-Rate-Limit"]
    E4["Global-Rate-Limit"]
    E5["Guardrail 调用"]
    E6["decisionMode 仲裁"]
    E7["CALLBACK 并发槽 + callbackTimeout 兜底"]
    E8["MCP 熔断优先 / FailOpen 切换"]
    E9["InferencePool 路由"]
    E10["事件通道健康兜底"]
    E11["熔断 / 降级 / 阻断 / 透传"]
    E1 --> E2 --> E3 --> E4 --> E5 --> E6
    E6 --> E7
    E6 --> E8
    E8 --> E9
    E7 --> E11
    E9 --> E11
    E10 --> E11
  end

---

图8

flowchart TB
  subgraph CONTRACT["权责契约"]
    U["上游业务客户端 / Agent Runtime<br/>唯一业务判定主体"]
    G["KC-AI-Gateway<br/>仅传输层动作<br/>检测 / 筛选 / 排序 / 转发 / 阻断 / 上报 / 迁移"]
    D["下游 LLM / MCP-Server / A2A 后端<br/>无业务判定权"]
    T["终端用户<br/>无业务判定权"]
    U -->|任务与业务判定| G
    G -->|标准化转发| D
    G -->|标准化事件 / 回调接管| U
    D -->|响应 / 传输层信号| G
    G -->|反馈记录| T
  end

  subgraph WHITE["网关动作白名单"]
    W1["协议转译"]
    W2["报文归一 / 分类 / 排序 / 权重筛选"]
    W3["限流计数"]
    W4["熔断"]
    W5["会话传输迁移"]
    W6["事件采集 / 上报 / 埋点"]
    W7["RBAC 鉴权判定<br/>仅 CRD 静态声明或请求原始属性"]
    W8["协议标准元数据读取<br/>MCP-Protocol-Version / _meta / Content-Type"]
  end

  subgraph FORBID["禁止动作"]
    F1["自主推导业务语义正确性"]
    F2["裁决 LLM 输出好坏"]
    F3["判定 MCP 工具调用业务上下文兼容性"]
    F4["解析 payload body 业务字段"]
    F5["以 4xx / JSON-RPC 业务错误码触发熔断"]
    F6["向转发后端注入网关内部状态标记头"]
  end


