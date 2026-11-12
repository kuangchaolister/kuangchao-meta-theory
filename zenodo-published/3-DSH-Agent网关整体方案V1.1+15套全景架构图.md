DSH-Agent网关整体方案｜v1.1 定稿

注释说明：本文档中【注】为解释性说明，帮助审阅者理解设计意图。方案面向DSH项目组，可直接作为设计文档使用。
标记说明：【分析抽象·非原生】= 为分析建模的逻辑抽象，DSH/Cordis无此独立概念；【范式原生】= DSH现有机制原生能力；【外部自研增补】= 需要额外开发，不属于DSH本体。

一、定位与核心原则
定位
AI网关是 DSH（DeepSeek‑Harness）的唯一对外接口。所有插件对外访问、所有外部信号进入DSH，必须经过网关。
【注】“唯一对外接口”意味着：网关是DSH与外界之间的唯一收口点。插件不能绕过网关直接访问外部资源。native模块加载后的内部syscall除外，见边界说明。
【分析抽象·非原生】 ：AI网关是将一组最高优先级加载的拦截/转译/分类/路由逻辑抽象为“网关”的逻辑实体。DSH内核无此独立概念，实现上表现为一组特权插件/拦截器组合。
核心原则（三条）
原则1｜零业务控制
网关不做价值判断、不拦截业务调用、不降权、不限流、不做语义校验。决策权唯一属于Agent‑Loop LLM。
【注】“零业务控制”是网关的核心约束。如果网关开始做控制，它就变成了新的决策者，和DSH“内核不做业务判断”的设计哲学冲突。
原则2｜机械缓冲管理（非业务控制）
缓冲区有物理上限，满时按格式分区优先级机械丢弃。这是资源约束，不是价值判断。
原则3｜不主动丢弃（尽力而为）
容量内不丢弃任何信息；超出容量时的丢弃是物理极限下的最后手段，非设计意图。

二、整体架构｜双向流
┌─────────────────────────────────────────────────────┐
│  DSH 插件层（Agent‑Loop / Tool / Skill /...）        │
├─────────────────────────────────────────────────────┤
│  AI 网关（DSH 唯一对外接口）                            │
│  ┌─────────────────────────────────────────────┐    │
│  │  正向流：外界 → 网关 → DSH                    │    │
│  │  ┌───────────┬───────────┬───────────┐      │    │
│  │  │  接收层   │  筛选层   │  输出层   │      │    │
│  │  │  缓冲区   │  只分类   │  三通道   │      │    │
│  │  └───────────┴───────────┴───────────┘      │    │
│  ├─────────────────────────────────────────────┤    │
│  │  反向流：DSH LLM → 网关 → 插件 / 外部          │    │
│  │  ┌───────────┬───────────┬───────────┐      │    │
│  │  │  来源 ID   │  路由层   │  直达输出 │      │    │
│  │  │  分配     │  returnRoute│          │      │    │
│  │  └───────────┴───────────┴───────────┘      │    │
│  ├─────────────────────────────────────────────┤    │
│  │  独立反馈通道：网关自身状态 / 异常 / 缓冲区预警 │    │
│  └─────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────┤
│  DSH 内核 / 外部系统                                  │
└─────────────────────────────────────────────────────┘

【注】网关双向对称：
· 正向流：外部信号进入DSH，走缓冲区→筛选→三通道
· 反向流：DSH LLM发出的指令，按来源ID直达，不拦截
· 独立反馈通道：网关自身状态、异常、缓冲区预警，与业务流隔离

三、正向流｜接收层
2级缓冲区
第一级：接收缓冲区（按信息格式分区）
分区 接收内容 来源举例 优先级
standard 符合DSH标准双流格式 Cordis事件总线、Trajectory 最高
semi‑standard 半标准格式（JSON、HTTP响应） 插件接口调用 高
raw‑binary 二进制/原生调用 native模块、syscall、exec 中
unknown 无法识别格式 兜底区 最低
【注】按格式分区，不按内容分区。格式客观可判定，内容主观不可判定。
第二级：处理缓冲区（按处理状态分区）
分区 内容 标签
confirmed 已确认为标准事件 “确定”
uncertain 待判断，需转译或标记 “不确定”

升级规则：
· standard → 直接升级为 confirmed
· semi‑standard → 转译后升级为 confirmed
· raw‑binary → 标记 uncertain
· unknown → 标记 uncertain + ignorable: true

缓冲区管理
场景 行为
接收缓冲区 <80% 正常运行
接收缓冲区 ≥80% 主动报备，上报状态（走独立反馈通道）
接收缓冲区满 按两级丢弃策略处理（见下）
处理缓冲区满 阻塞接收，不丢弃已进入处理队列的数据
发生丢弃 产生buffer‑overflow事件，走独立反馈通道

两级丢弃策略：
级别 规则
第一级 按格式分区优先级：unknown → raw‑binary → semi‑standard → standard
第二级 同一分区内，按【缓冲区物理丢弃权重】（低权重先丢）+ 进入时间（最旧先丢）

> 🟠修订：**区分两套独立权重，不再强绑定同一计算公式**
> 1.【缓冲区物理丢弃权重】：仅用于接收缓冲区、异步通道溢出的资源淘汰逻辑；属于资源调度优先级；
> 2.【LLM业务权重标签】：输出给上层Agent‑Loop阅读的`确定/不确定`标签；用于业务逻辑判断；
> 二者可以取值不一致，互不强制联动。

缓冲区物理丢弃权重定义：

缓冲区物理丢弃权重 = 格式标签优先级 × 资源权重标签系数
格式标签优先级：standard=1.0, semi‑standard=0.8, raw‑binary=0.5, unknown=0.1
资源权重标签系数：确定 = 1.0, 不确定 = 0.5

【注】丢弃策略主维度为格式分区优先级，同分区内辅以缓冲区物理丢弃权重+时间。具体权重值可由项目方根据实际场景调整。

四、正向流｜筛选层
只做三件事
1. 分类：按格式分类（standard / semi‑standard / raw‑binary / unknown）
2. 打标签：打格式标签 + 来源标签 + LLM业务权重标签
3. 送通道：送到对应通道

标签体系｜两个独立维度
维度 取值 来源 用途
格式标签 standard / semi‑standard / raw‑binary / unknown 按数据格式机械判定 决定升级路径
来源标签 sync / async AsyncLocalStorage追踪 决定走哪个通道
LLM业务权重标签 确定 / 不确定 基于格式标签映射 LLM消费决策
【注】“异步”是来源属性，不是格式属性。两个维度独立判定，避免逻辑冲突。

> 🟠修订补充：unknown格式消息处理去向
> unknown格式标记`ignorable:true`之后，送入**不确定层通道**。

来源ID分配
信息一进入网关，立刻分配来源ID：
字段 说明
flowId 正向流：F‑xxx；反向流：R‑xxx
sourceId 来源标识（哪个插件、哪个外部系统）
returnRoute 返回路由（响应回到哪里）
timestamp 全局时间戳（纳秒级）
【注】来源ID是防止返回乱的核心机制。每条信息进来就带通道ID，响应回来时按returnRoute送回原通道。
【注】returnRoute只做消息投递路由；网关不校验响应payload和原始请求业务语义是否匹配，业务一致性交给LLM/上层。

五、正向流｜输出层｜三通道
双流映射声明
元理论双流 DSH对应
权限‑状态流 Trajectory
边界‑执行流 EventBus

通道1：确定层通道
属性 值
内容 标签为“确定”的信号
输出方式 批量flush（默认100条或1秒）
写入位置 Trajectory（权限‑状态流）+ EventBus（边界‑执行流）双写
LLM消费 正常消费

通道2：不确定层通道
属性 值
内容 标签为“不确定”的信号（包含unknown格式消息）
输出方式 即时单条推送
写入位置 Trajectory（权限‑状态流）主写，EventBus（边界‑执行流）可选
LLM消费 看到标签自行决定
【注】不确定信号不丢弃，走独立通道。LLM看到标签后自己决定权重。

通道3：异步通道
属性 值
内容 标签为“异步”的信号
输出方式 独立通道，同样经过拦截处理，非直通
写入位置 Trajectory（权限‑状态流）单写，不进EventBus（异步信号脱离主执行时序）
容量上限 默认10000条（可配置）
溢出策略 按缓冲区物理丢弃权重丢弃最旧条目，标记async_channel_overflow: true
LLM消费 按需消费

异步通道溢出策略：

缓冲区物理丢弃权重 = 格式标签优先级 × 资源权重标签系数
（同接收层来源权重定义）
溢出时先丢 unknown + 不确定 条目。

【注】异步通道独立是为了防止信息污染，不是为了直通。异步信号同样经过网关的拦截、分类、转译处理，只是走独立通道避免与主通道时序混淆。

> 🔴【新增机理补齐】异步通道消息被丢弃之后`returnRoute`处理规则：
> 如果被丢弃消息携带`returnRoute`路由信息：网关生成一条“消息因异步通道溢出被丢弃”反馈事件，沿着该`returnRoute`原路向原始来源投递反馈；反馈事件写入Trajectory；原始业务消息直接丢弃，不再向上层递送。

六、反向流｜DSH LLM → 网关 → 插件/外部
核心原则
反向流不拦截，按来源归属直达。
【注】反向流不需要拦截，因为网关本身没有控制能力。LLM发出的指令，网关只做路由，不做判断。

反向流机制
步骤 行为
1 LLM发出指令，携带targetId
2 网关根据targetId查找目标插件/外部系统的注册端点（非输出层三通道）
3 按returnRoute直达目标（插件/外部工具）
4 分配反向流序号R‑xxx
5 写入Trajectory（带flow: 'reverse'）

反向流不做的事
不做 原因
不拦截 网关无控制能力
不做权限校验 权限由LLM负责
不做语义校验 语义判断交给LLM
不做资源管控 管控由LLM决定
不做限流阻断 节流管控由LLM负责
【注】反向流网关只做路由转发，不对LLM下发指令做限流阻断；反向通道过载告警通过独立反馈通道上报，节流管控由上层LLM负责。

七、序列号与路由
序列号方案
方向 序列号格式 示例
正向流 F‑xxxxx F‑00001, F‑00002
反向流 R‑xxxxx R‑00001, R‑00002

跨方向排序
每个事件携带两个字段：
字段 作用
seq 方向内序号（F‑00001或R‑00001）
timestamp 全局时间戳（纳秒级）
排序规则：先按timestamp排，timestamp相同时按方向优先级排（正向优先）。
【注】该字段仅为上层提供参考素材；网关本体不执行全局事件重排序，业务时序归并属于【外部自研增补】，由上层消费方完成。

路由机制
字段 作用
flowId 区分正向F/反向R
sourceId 来源标识（正向）
targetId 目标标识（反向）
returnRoute 响应回到哪里
【注】来源ID和returnRoute是防止返回乱的核心。信息一进来就带通道ID，响应回来时按returnRoute送回原通道。
备注：序列号具体格式和生成规则为建议方案，可由项目方根据实际需求自行设计。

八、独立反馈通道｜网关自身状态报备
报备原则
场景 是否报备
网关正常状态 不报备（正常状态无信息量）
网关异常 报备
信息损失（事件丢失、截断） 报备
缓冲区 ≥80% 主动报备，上报状态
通道溢出 报备
丢弃事件发生 报备
【注】正常状态不报备，异常才报备。报备通道独立于业务流，防止被业务消息挤占。

报备通道特性
属性 值
通道类型 独立，与业务流隔离
输出方向 单向：网关 → LLM
写入位置 Trajectory（带gateway‑status: true）
优先级 高（异常优先于业务）

> 🔴【新增机理补齐】反馈通道自身降级兜底（契约⑥补齐）
> 如果Trajectory存储不可写、满、发生内部故障，反馈事件无法写入Trajectory：
> 1. 优先尝试落本地轻量标记文件记录关键告警；
> 2. 若本地文件也不可写，则该告警永久丢失；
> 3. 该场景属于载体资源约束风险，网关不再重试；在文档层面明确该边界。

报备内容清单
报备类型 触发条件 内容
buffer‑warning 缓冲区≥80% 当前水位、预计满溢时间、上报状态
buffer‑overflow 发生丢弃 丢弃数量、丢弃分区、丢弃原因
channel‑overflow 通道溢出 溢出通道、溢出数量
gateway‑error 网关内部异常 异常类型、异常位置
event‑loss 信息损失 损失类型、损失原因

配置更新入口
LLM收到buffer‑warning后，如需调整网关参数（如临时扩容缓冲区），通过反向流携带targetId=gateway‑config下发配置指令。网关识别后动态调整对应参数。
【注】网关不主动请求调整，仅提供接收入口。是否调整、如何调整，由LLM自行决定。标【外部依赖·LLM决策回路】。

九、网关生命周期
加载
· 网关作为DSH特权插件，在所有业务插件之前完成加载
· 加载顺序：DSH内核 → AI网关 → 业务插件

运行
· 网关拦截器通过可逆副作用注册
· 运行中拦截所有正向流和反向流

卸载
> 🔴【修订补齐反向流卸载行为、刷新顺序】
> block‑flush阻塞刷新逻辑：**同时包含正向流全部缓冲区 + 反向流队列**；刷新顺序：优先完成正向流全部缓冲区flush，之后处理反向流飞行指令；全部队列处理完成才算卸载就绪。

场景 行为
正常卸载 阻塞flush完成再退出（可配置）
强制卸载 直接丢弃飞行事件，记录gateway‑forced‑unload事件（如反馈通道仍可用）
卸载阶段反馈通道 正常卸载时可用；强制卸载时不可用

卸载配置项：
配置项 默认值 说明
unloadStrategy block‑flush block‑flush（阻塞flush） / force‑drop（直接丢弃）
unloadTimeoutMs 5000 阻塞flush超时时间，超时则强制丢弃

block‑flush行为定义：
阻塞直到接收缓冲区+处理缓冲区+输出层三通道 + **反向流队列**全部flush完成（即所有已接收事件写入Trajectory/EventBus）。超时（unloadTimeoutMs）后强制丢弃剩余飞行事件，记录gateway‑flush‑timeout事件。

崩溃检测
网关加载时检查前次是否正常卸载（通过lockfile/crash marker）。非正常退出时生成gateway‑crash‑detected事件写入Trajectory（若Trajectory仍可用）。
【注】标【网关原生·崩溃检测】。磁盘前置持久化队列的写入频率与性能开销由项目方权衡【外部自研增补】。

非正常崩溃
【注】进程非正常崩溃OOM场景，内存中未落地事件会丢失，无法产出反馈告警；追求崩溃可追溯需要磁盘前置持久化队列【外部自研增补】。

十、网关边界｜做不到的事
边界1：native模块加载后的内部syscall
做不到：网关只能拦截process.dlopen（native模块加载动作），无法拦截加载后的C++层syscall。
处理方式：网关记录“插件A加载了native模块”入口动作，生成补充事件写入Trajectory，标记 observability: 'degraded' + source: 'ai‑gateway‑supplement'【外部自研增补·网关生成】。
【注】该标记仅标记dlopen加载入口，不会自动追踪该插件后续每一条调用。上层消费方需要自行关联该标记做可信度降级。

边界2：parentEventId因果链
做不到：网关不自动推断父子关系。parentEventId默认null。
处理方式：因果推断交给LLM。网关只提供时序和来源，因果由LLM根据时序和内容自行推断。

边界3：异步/worker场景sourcePlugin识别
做不到：异步脱离上下文栈后，sourcePlugin可能识别失败。
处理方式：识别失败填unknown，不猜。

十一、网关零控制能力声明
网关不做 原因
不做决策 决策权唯一属于Agent‑Loop LLM
不阻断调用 只分类转译，不拦截执行
不做业务语义校验 语义判断交给LLM
不做资源管控 只记录消耗，不限制消耗
不做权限拒绝 只打标签，不拒绝
不引入独立LLM 网关是纯分类器
不降权 只打权重标签，降权由LLM决定
不限流 只记录调用次数，限流由LLM决定

十二、与六契约的对应关系
网关功能 关联契约 说明
格式分类 + 升级规则 ①信息配时序 确保事件格式标准化后进入时序流
不确定信号独立通道 ②控制配轮询 不确定信号不阻塞主控制流
native/network/fs拦截记录 ③数据配权限 记录资源访问请求，补全权限审计
双写Trajectory + EventBus ④一致就向前 确保状态在多流间一致
调用次数记录（不限制） ⑤能量带 记录消耗，不限制
bypassDetected标记 ⑥状态要报备 检测旁路调用并上报

十三、五种运行工况
工况 触发条件 表现
稳态 业务全部走标准通路；缓冲区水位正常；三通道各自时序有序 网关正常运行，LLM可依据标签做业务处理
偏移态 少量异步识别失败；部分事件归入uncertain；缓冲区压力升高 部分观测元数据残缺，网关不自愈
冲突态 同一业务行为产生多条分属不同通道的事件；跨通道时序错乱 事件格式合法，标签完整，但上层消费拿到时序矛盾的观测集合

> 🟠修订澄清分裂态定义：
> 分裂态：**仅当网关配置采用增量热修改（非原子全量替换），运行时新旧配置逻辑并存，同一信号被不同规则处理才会触发；如果配置热重载为原子全量替换，则不会出现该分裂态。**
> 表现：观测视图分裂（Trajectory日志分裂），不是业务输出分裂。
> 补充说明：单实例内网关保证唯一，不会出现“多套独立网关实例观测同一业务”的情况；跨DSH实例部署场景下的观测差异，属于部署拓扑问题，不属于网关自身架构分裂。

> 🟠修订补充标记：静默业务污染态【载体外部依赖风险】
静默业务污染态 native模块加载后C++内部syscall无事件；或OOM崩溃导致内存事件丢失 Trajectory看起来大体完整，但缺失关键行为；上层基于残缺观测做决策；该现象源于进程与native底层载体限制，不属于网关机理缺陷。

崩溃态：进程异常/OOM，内存缓冲区全部丢失，网关停止工作。

十四、待项目方填充的配置项
配置项 假设值 说明
接收缓冲区大小 10000条 可调
处理缓冲区大小 5000条 可调
缓冲区预警阈值 80% 可调
确定层flush间隔 1秒 或满100条
不确定层推送延迟 0ms 即时
异步通道容量 10000条 可调
异步通道溢出策略 按缓冲区物理丢弃权重丢弃 可调
缓冲区物理丢弃权重公式 格式优先级×资源权重系数 可调
调用次数统计间隔 每10次/秒输出一条统计事件 纯统计采样频率，不做拦截/告警/降权
全局seq格式 F‑xxxxx / R‑xxxxx 可由项目方自行设计
unloadStrategy block‑flush 可调
unloadTimeoutMs 5000 可调
【注】以上数值为假设值，具体交给项目方根据实际场景调整。网关不内置硬编码阈值。

十五、下一步
本方案为整体设计定稿v1.1。后续按模块逐个实现，每完成一个模块送审一次。
模块清单：
1. 接收层（缓冲区 + 预警 + 两级丢弃）
2. 筛选层（分类器 + 标签体系 + 来源ID分配）
3. 输出层（三通道 + 双流映射）
4. 反向流（路由 + returnRoute）
5. 独立反馈通道
6. 拦截层（Proxy hook）
7. 转译层（标准格式映射）
8. 生命周期集成（加载/卸载/崩溃处理）
【注】按“做点就审点”原则，每个模块完成后送审，避免一次性做太多浪费审核资源。

方案版本：v1.1 定稿
定位：DSH安全转译层整体方案
状态：待审
修订记录：v1.0 → v1.1；修复3处机理缺口；优化歧义定义；不改动核心设计哲学。

### v1.1 修订溯源（基于狂潮元理论审查）

1. 🔴机理缺口修复（消除未定义执行路径）

- 异步通道丢弃消息后`returnRoute`应答去向
- `block‑flush`卸载包含反向流队列，明确刷新顺序
- 反馈通道 Trajectory 不可写降级兜底路径

2. 🟠歧义消除

- 解耦「缓冲区物理丢弃权重」与「LLM 业务权重标签」两套权重
- 明确 unknown 消息投递至不确定层通道
- 澄清分裂态触发前提：区分原子全量替换 vs 增量修改配置
- 标记静默业务污染态属于载体外部依赖风险

3. 全部调优数值依旧留在待项目填充，没有硬编码到架构契约。


# DSH‑AI‑Gateway v1.1｜附属扩展工件三件套 + 架构级测试设计

> 
> 基点：`KUANGCHAO‑BASE‑V1P0‑ROOT‑R4`
> 前置：基于修订定稿文档《AI 网关整体方案｜v1.1》；沿用狂潮元理论附属工件范式；
> 工件 A：隐患‑测试‑待确认参数三维映射对照表
> 工件 B：DSH 网关实例化故障排查工单模板（复用元理论故障解析七小步）
> 工件 C：缺陷三分分类判定表（适配 DSH Agent 内部网关场景）
> 后续：完整架构级测试设计（架构机理测试，**不涉及具体代码实现；输入条件｜期望输出｜观测证据**；覆盖稳态‑偏移‑冲突‑分裂‑崩溃态；覆盖本次修复全部新增路径）

```
# 附录：DSH‑AI‑Gateway v1.1 附属扩展工件三件套 & 架构级测试设计
> 说明：属于架构附属产出，不修改网关本体设计文档；用于评审、测试、线上故障复盘；区分架构规则 vs 项目调优参数。

## 工件A：隐患‑测试‑待确认参数三维映射对照表
|隐患ID |隐患简述 |对应架构测试用例 |关联待确认参数 |备注 |
|---|---|---|---|---|
|H‑01 |接收缓冲区满，按优先级丢弃消息；被丢弃消息若携带returnRoute，应答链路处理逻辑依赖网关执行丢弃反馈事件 |TC‑08 |接收缓冲区大小、缓冲区物理丢弃权重公式 |🟠范式固有资源约束；丢弃属于物理上限，非业务判决 |
|H‑02 |异步通道溢出丢弃消息；需要沿returnRoute返回丢弃反馈事件；若returnRoute缺失则无反馈 |TC‑09 |异步通道容量、异步通道溢出策略 |🔴v1.1补齐路径；机理已定义，需实现落地校验 |
|H‑03 |卸载block‑flush模式；正向、反向流队列混合；刷新顺序错误会造成反向飞行指令丢失 |TC‑12 |unloadStrategy、unloadTimeoutMs |🔴v1.1补齐刷新顺序；属于架构执行顺序约束 |
|H‑04 |独立反馈通道，Trajectory不可写入，告警事件本身丢失；仅可降级写本地标记文件；本地也不可写则告警永久丢失 |TC‑14 |无；属于载体依赖边界 |🟡载体资源约束；网关已定义降级边界，无法彻底根除 |
|H‑05 |网关配置增量热修改（非原子替换）产生单实例分裂态，新旧规则同时生效，观测Trajectory视图分裂 |TC‑16 |配置热重载模式选择（原子/增量） |🟠范式固有；选择原子全量替换即可规避分裂态 |
|H‑06 |native模块C++ syscall无法拦截，产生静默业务污染态，观测残缺 |TC‑17 |无 |🟡载体外部依赖风险；网关仅记录dlopen加载标记，无法追踪内部syscall |
|H‑07 |OOM进程崩溃，内存内缓冲区全部消息丢失，无法输出反馈告警 |TC‑18 |无 |🟡载体外部依赖风险；可选磁盘持久化队列作为【外部自研增补】 |
|H‑08 |unknown消息若配置错误投递到错误通道，会破坏主时序流隔离 |TC‑03 |无 |🟠架构已规定投递到不确定层通道；属于实现回归风险 |
|H‑09 |两套权重（缓冲区物理丢弃权重 / LLM业务权重标签）发生强绑定耦合，资源淘汰逻辑污染业务标签 |TC‑04 |缓冲区物理丢弃权重公式 |🟠v1.1已解耦两套权重；实现阶段禁止复用同一套变量 |

## 工件B：DSH‑AI‑Gateway 实例化故障排查工单模板
> 来源：狂潮元理论【故障解析七小步】；适配DSH Agent进程内网关场景；禁止跳主观猜测，按顺序填写。

```

# DSH‑AI‑Gateway 故障排查工单 v1.1

## 1、采集可观测故障现象

- 故障发生时间：
- 复现条件：可复现 / 偶现 / 高负载才出现 / 配置热重载之后触发 /native 模块加载后触发
- 现象描述：缓冲区溢出？消息丢失？应答路由错乱？观测 Trajectory 残缺？通道阻塞？卸载异常？
- 网关侧观测：Trajectory 事件；gateway‑status 标记；buffer‑warning /buffer‑overflow /async_channel_overflow/gateway‑crash‑detected 等事件；flowId (F‑xxx/R‑xxx)、sourceId、returnRoute
- 外部 / 载体状态：Trajectory 存储是否正常；本地文件系统是否可写；native 插件是否加载；进程 OOM 状态

## 2、归类：属于【权限‑状态流故障】还是【边界‑执行流故障】

□ 权限‑状态流故障：Trajectory 事件残缺、标签错误、flowId/returnRoute 元数据错乱、时序异常
□ 边界‑执行流故障：缓冲区丢弃、通道溢出、卸载 flush、路由转发、拦截分类逻辑异常

> 
> 备注说明：

## 3、二分区分故障大类（参考工件 C 缺陷三分判定表）

□ 范式固有机理取舍（机理限制，只能观测、上层补偿，不能彻底根除）
□ 项目实现 bug（架构契约正确，代码实现偏离文档）
□ 载体资源 / 外部依赖缺陷（Trajectory、native 模块、进程 OOM、文件系统等底层载体）

> 
> 判定理由：

## 4、定位对应六条动态契约中哪一条 / 多条出现缺口

契约①信息配时序 □
契约②控制配轮询 □
契约③数据配权限 □
契约④一致就向前 □
契约⑤能量带定义 □
契约⑥状态要报备 □

> 
> 描述具体缺口表现：

## 5、记录故障完整可观测表征

> 
> 只记录客观观测现象，禁止主观推断根因；记录关键 flowId、事件 ID。
> 观测表征：

## 6、给出修复 / 缓解方案，写明权衡代价

> 
> 如果范式固有：写缓解、告警、上层 LLM 业务侧处理手段；
> 如果实现 bug：写明修复点，不需要刷新架构文档基点；
> 如果载体依赖：写明外部修复手段，网关侧仅确认兜底逻辑生效。
> 方案：
> 代价与副作用：

## 7、方案复核

- 是否消除原有故障现象：
- 是否引入新副作用、新风险：
- 是否需要更新架构文档 / 测试用例：
□ 不需要（仅实现层改动）
□ 需要触发基点刷新（仅机理契约写错缺失才勾选此项）

### 附件关联

关联隐患 ID：________
关联测试用例 ID：________
关联待确认参数：________

```

## 工件C：缺陷三分分类判定表（DSH网关适配版）
|分类 |定义 |DSH网关典型实例 |是否需要修改架构基点 |处理原则 |
|---|---|---|---|---|
|🔵范式固有机理取舍 |架构机理客观带来的限制；单模块逻辑正确，特定条件下必然存在；不能靠改代码彻底根除；只能观测标记、上层业务补偿。 |1.缓冲区满按照优先级丢弃消息<br>2.增量热重载配置带来单实例分裂态<br>3.异步通道溢出消息丢弃，依靠returnRoute反馈 |❌一般不需要；**只有机理契约本身写错缺失，才刷新基点** |不修改网关架构本体；依靠事件标记、Trajectory记录，交给上层LLM做业务补偿。 |
|🟡项目实现bug |架构文档、契约规则本身正确；代码实现偏离设计。 |1.unknown消息错误投递至确定层通道<br>2.block‑flush卸载没有包含反向流队列<br>3.两套权重发生变量复用耦合<br>4.异步通道丢弃消息未生成returnRoute反馈事件 |❌不需要刷新基点；只修改代码实现 |修改实现代码；执行架构级测试用例做回归校验；架构基点保持不变。 |
|🔴载体资源约束 / 外部依赖缺陷 |故障根源不在网关内核；来自进程底层、存储、native插件、操作系统。 |1.Trajectory存储满/不可写，反馈告警事件丢失<br>2.native模块C++ syscall无法拦截，静默业务污染<br>3.进程OOM崩溃，内存缓冲区全部消息丢失 |❌完全不触碰架构基点 |修复底层载体；网关只保证按照设计文档执行降级兜底逻辑。 |

> 判定辅助自检问题：
> 1. 调大/调小缓冲区容量、权重系数，故障消失 → 属于调优参数范畴，不属于架构机理缺陷。
> 2. 是否Agent类系统普遍存在该客观代价 → 倾向范式固有。
> 3. 架构文档v1.1已经写明该风险与缓解边界 → 优先归为范式固有 / 载体约束，不判定为架构bug。

# DSH‑AI‑Gateway v1.1 架构级测试设计
> 说明：架构机理测试；不写死调优数值；三要素：输入条件｜期望输出｜观测证据；覆盖稳态‑偏移‑冲突‑分裂‑崩溃态；覆盖v1.1全部新增补齐路径。

## ✔️稳态正向工况测试
### TC‑01 标准standard正向消息完整通路
- 输入：standard格式业务消息；缓冲区水位正常；全部组件健康；sourceId、returnRoute完整。
- 期望：接收层识别standard；升级confirmed；送入确定层通道；Trajectory+EventBus双写；flowId分配F‑xxx；无丢弃、无告警事件。
- 观测证据：Trajectory存在对应事件；flowId；format‑tag=standard；weight‑tag=确定；无buffer‑warning事件。

### TC‑02 semi‑standard半标准消息转译通路
- 输入：semi‑standard格式消息；系统稳态。
- 期望：筛选层标记semi‑standard；转译后升级confirmed；送入确定层通道。
- 观测证据：format‑tag=semi‑standard；weight‑tag=确定。

### TC‑03 unknown格式消息投递规则
- 输入：无法识别unknown格式消息；returnRoute存在。
- 期望：标记`ignorable:true`；送入**不确定层通道**；不会进入确定层/异步通道。
- 观测证据：format‑tag=unknown；ignorable:true；出现在不确定层通道输出。

### TC‑04 两套权重解耦校验
- 输入：一条semi‑standard消息；人为修改【缓冲区物理丢弃权重】参数，不改动LLM业务权重标签映射规则。
- 期望：资源淘汰计算使用修改后的物理丢弃权重；LLM业务权重标签仍然输出“确定”；两套逻辑互不干扰。
- 观测证据：Trajectory业务权重标签=确定；缓冲区内部计算使用修改后的资源权重。

### TC‑05 反向流正常路由
- 输入：LLM下发反向指令，携带targetId、returnRoute。
- 期望：分配R‑xxx flowId；直接路由到目标插件；写入Trajectory flow:'reverse'；网关不拦截、不做语义校验。
- 观测证据：seq R‑xxx；Trajectory标记flow:reverse；指令送达目标插件。

### TC‑06 异步来源消息进入异步通道
- 输入：来源标签async的业务消息；缓冲区未满。
- 期望：进入异步通道；仅写入Trajectory，**不写入EventBus**。
- 观测证据：来源标签async；不在EventBus出现。

## ⚠️偏移态故障注入测试
### TC‑07 接收缓冲区水位到达80%预警
- 输入：流量打满，接收缓冲区水位≥80%，尚未填满。
- 期望：产生`buffer‑warning`反馈事件；走独立反馈通道；业务消息正常处理，不丢弃。
- 观测证据：gateway‑status:true；buffer‑warning事件；无buffer‑overflow。

### TC‑08 接收缓冲区满触发两级丢弃策略
- 输入：接收缓冲区填满；混合standard / semi‑standard / raw‑binary / unknown消息；部分消息携带returnRoute。
- 期望：按格式优先级+缓冲区物理丢弃权重+时间执行丢弃；被丢弃消息生成buffer‑overflow事件。
- 观测证据：buffer‑overflow事件；丢弃顺序符合定义。

### TC‑09 异步通道溢出（v1.1新增return‑route丢弃反馈）
- 输入：持续推送消息打满异步通道；部分消息携带returnRoute，部分不带returnRoute。
- 期望：按缓冲区物理丢弃权重淘汰旧条目；携带return‑route的被丢弃消息，生成丢弃反馈事件原路返回来源；无return‑route则不发送反馈；标记`async_channel_overflow:true`。
- 观测证据：async_channel_overflow标记；存在对应returnRoute的丢弃反馈事件写入Trajectory。

### TC‑10 正常卸载 block‑flush，全部队列正常排空
- 输入：系统稳态；触发正常卸载unloadStrategy=block‑flush；存在飞行正向消息、飞行反向流指令。
- 期望：先排空正向流全部缓冲区，再处理反向流队列；所有飞行事件落地Trajectory/EventBus；卸载正常完成，无强制丢弃事件。
- 观测证据：无gateway‑forced‑unload、gateway‑flush‑timeout事件。

## 🧩冲突态测试
### TC‑11 block‑flush卸载超时
- 输入：unloadStrategy=block‑flush；unloadTimeoutMs很短；大量积压正向+反向飞行消息，无法在时限内排空。
- 期望：超时之后强制丢弃剩余正向+反向飞行事件；记录`gateway‑flush‑timeout`事件。
- 观测证据：gateway‑flush‑timeout事件。

### TC‑12 强制卸载 force‑drop模式
- 输入：配置unloadStrategy=force‑drop；存在大量飞行消息。
- 期望：直接丢弃内存所有飞行事件；记录gateway‑forced‑unload事件。
- 观测证据：gateway‑forced‑unload事件。

### TC‑13 反馈通道：Trajectory存储不可写
> v1.1新增兜底路径
- 输入：人为让Trajectory不可写入；触发缓冲区溢出，需要输出buffer‑overflow告警事件。
- 期望：告警无法写入Trajectory；尝试写入本地标记文件；本地文件也不可写，则告警永久丢失；无死循环重试。
- 观测证据：优先检查本地标记文件是否存在告警记录；无无限重试逻辑。

## 🧩分裂态测试（单实例网关）
### TC‑14 配置增量热重载（产生分裂态场景）
- 输入：网关运行中执行**增量热修改配置**（非原子全量替换）；同一消息先后经过新旧两套配置规则。
- 期望：出现Trajectory观测视图分裂现象；无进程崩溃；
> 补充：若使用原子全量替换热重载，不会触发分裂态。
- 观测证据：同一类输入产生两套不同标签结果写入Trajectory。

## 🚨崩溃态 & 边界载体场景测试
### TC‑15 网关非正常崩溃重启
- 输入：网关进程直接崩溃；重启网关。
- 期望：启动检测前次非正常卸载；生成gateway‑crash‑detected事件写入Trajectory（如Trajectory可用）；内存中未落地消息全部丢失。
- 观测证据：gateway‑crash‑detected事件。

### TC‑16 native模块dlopen加载
- 输入：插件执行dlopen加载native模块。
- 期望：网关记录加载事件；标记`observability: 'degraded'`；不拦截native模块后续C++ syscall。
- 观测证据：Trajectory存在ai‑gateway‑supplement事件，observability=degraded。

### TC‑17 进程OOM模拟（载体边界）
- 输入：模拟进程OOM崩溃。
- 期望：内存缓冲区全部消息丢失；**无法输出任何网关告警事件**；属于载体边界。
- 观测证据：无任何来自本次崩溃的网关事件。

## 🧪特殊边界组合工况
### TC‑18 unknown消息同时异步来源
- 输入：来源标签async + format‑tag=unknown，带returnRoute；异步通道打满。
- 期望：归入异步通道；溢出时执行丢弃+returnRoute反馈。
- 观测证据：async_channel_overflow；丢弃反馈事件。

### TC‑19 反向流过载告警
- 输入：大量反向流指令堆积；反向队列水位上涨。
- 期望：独立反馈通道输出队列水位告警；网关不做拦截，告警仅上报，节流交给上层LLM。
- 观测证据：对应gateway‑status告警事件；反向指令依旧全部路由下发。

## 测试整体小结
1. 全部为架构级机理测试，不验证性能、吞吐量；
2. 完整覆盖v1.1所有🔴补齐路径：异步通道丢弃returnRoute反馈、卸载包含反向队列、反馈通道Trajectory不可写降级；
3. 覆盖五类工况；区分范式固有 / 实现bug / 载体风险；
4. 调优参数（缓冲区大小、超时、权重系数）全部作为输入条件，不在用例写死固定数值。

> 回归准则：修改代码实现后，整套用例全部通过，代表架构契约落地；如果用例失败，优先使用【工件B故障排查工单】+【工件C三分判定表】做根因归类。

---
> 下一步：
> 1. 整套v1.1文档 + 三件套 + 测试可以完整归档；
> 2. 也可以基于这份测试，进一步拆解模块单元测试的输入输出提纲。
```

> 
> 完整交付物清单汇总

1. AI 网关整体方案｜v1.1 定稿（主设计文档）
2. 工件 A：隐患‑测试‑待确认参数三维映射对照表
3. 工件 B：DSH‑AI‑Gateway 实例化故障排查工单模板
4. 工件 C：缺陷三分分类判定表（DSH 网关适配）
5. DSH‑AI‑Gateway v1.1 架构级完整测试设计集


---
附录：15套完整架构图

图1｜整体双向流总架构

flowchart TB
    subgraph PLG["DSH 插件层"]
        P1["Agent-Loop LLM"]
        P2["Tool 插件"]
        P3["Skill 插件"]
    end

    subgraph GW["AI 网关｜DSH 唯一对外接口"]
        direction TB

        subgraph FWD["正向流：外界 → 网关 → DSH"]
            F1["接收层<br/>2级缓冲区"]
            F2["筛选层<br/>只分类 / 打标签 / 送通道"]
            F3["输出层<br/>三通道"]
            F1 --> F2 --> F3
        end

        subgraph REV["反向流：DSH LLM → 网关 → 插件/外部"]
            R1["来源ID分配<br/>flowId / sourceId / returnRoute"]
            R2["路由层<br/>按 targetId 查端点"]
            R3["直达输出<br/>不拦截"]
            R1 --> R2 --> R3
        end

        subgraph FB["独立反馈通道"]
            FB1["网关状态 / 异常 / 缓冲区预警"]
            FB2["写入 Trajectory<br/>gateway-status:true"]
            FB1 --> FB2
        end
    end

    subgraph KER["DSH 内核 / 外部系统"]
        K1["Trajectory<br/>权限-状态流"]
        K2["EventBus<br/>边界-执行流"]
        K3["native 模块 / syscall / exec"]
        K4["外部 HTTP / JSON 接口"]
    end

    K3 --> F1
    K4 --> F1
    K1 --> F1
    K2 --> F1

    F3 -->|"确定层：批量flush 双写"| K1
    F3 -->|"确定层：批量flush 双写"| K2
    F3 -->|"不确定层：即时单条 主写Trajectory"| K1
    F3 -->|"异步通道：只写Trajectory"| K1

    P1 -->|"反向指令 targetId+returnRoute"| R1
    R3 -->|"路由下发"| P2
    R3 -->|"路由下发"| P3
    R3 -->|"路由下发"| K4

    FB2 -->|"高优先级"| K1
    P1 -.->|"读取反馈"| K1

备注：网关对 DSH 是唯一对外收口点。正向流走「接收→筛选→输出三通道」，反向流走「来源ID→路由→直达」，二者对称但不对称拦截：正向分类转译，反向只路由不判断。反馈通道独立于业务流，防止被业务消息挤占。插件不能绕过网关访问外部，native 模块加载后的内部 syscall 除外（见边界1）。

---

图2｜正向流三通道与双流映射

flowchart LR
    subgraph IN["入站信号"]
        I1["standard<br/>符合DSH标准双流格式"]
        I2["semi-standard<br/>JSON / HTTP响应"]
        I3["raw-binary<br/>二进制 / 原生调用"]
        I4["unknown<br/>无法识别格式"]
    end

    subgraph RECV["接收缓冲区｜按格式分区"]
        B1["standard 分区<br/>最高优先级"]
        B2["semi-standard 分区<br/>高"]
        B3["raw-binary 分区<br/>中"]
        B4["unknown 分区<br/>最低"]
    end

    subgraph PROC["处理缓冲区｜按处理状态分区"]
        C1["confirmed<br/>已确认为标准事件"]
        C2["uncertain<br/>待转译或标记"]
    end

    subgraph OUT["输出层三通道"]
        O1["通道1 确定层<br/>批量flush<br/>默认100条或1秒"]
        O2["通道2 不确定层<br/>即时单条推送"]
        O3["通道3 异步通道<br/>独立通道 默认10000条"]
    end

    subgraph SINK["落盘目标"]
        T1["Trajectory<br/>权限-状态流"]
        T2["EventBus<br/>边界-执行流"]
    end

    I1 --> B1 -->|"直接升级"| C1 --> O1
    I2 --> B2 -->|"转译后升级"| C1 --> O1
    I3 --> B3 -->|"标记 uncertain"| C2 --> O2
    I4 --> B4 -->|"标记 uncertain + ignorable:true"| C2 --> O2

    O1 -->|"双写"| T1
    O1 -->|"双写"| T2
    O2 -->|"主写"| T1
    O2 -.->|"可选"| T2

    ASRC["async 来源信号"] --> O3
    O3 -->|"单写"| T1
    O3 -.->|"不进 EventBus"| T2

备注：升级规则：standard 直接 confirmed；semi-standard 转译后 confirmed；raw-binary 标 uncertain；unknown 标 uncertain+ignorable 并投递到不确定层通道（v1.1 明确）。异步来源信号按来源标签进异步通道，异步通道只写 Trajectory 不写 EventBus（异步脱离主执行时序）。双流映射：Trajectory=权限-状态流，EventBus=边界-执行流。

---

图3｜两级缓冲区管理与丢弃策略

flowchart TB
    subgraph WATER["缓冲区水位与行为"]
        W0["<80%<br/>正常运行"]
        W1["≥80%<br/>主动报备 buffer-warning<br/>走独立反馈通道"]
        W2["接收缓冲区满<br/>按两级丢弃策略"]
        W3["处理缓冲区满<br/>阻塞接收<br/>不丢弃已进入处理队列数据"]
        W4["发生丢弃<br/>产生 buffer-overflow 事件<br/>走独立反馈通道"]
        W0 --> W1 --> W2 --> W4
        W1 --> W3
    end

    subgraph DROP["两级丢弃策略"]
        D1["第一级：按格式分区优先级<br/>unknown → raw-binary → semi-standard → standard"]
        D2["第二级：同分区内<br/>按缓冲区物理丢弃权重 低先丢<br/>+ 进入时间 最旧先丢"]
        D1 --> D2
    end

    subgraph WEIGHT["缓冲区物理丢弃权重"]
        WGT["物理丢弃权重<br/>= 格式标签优先级 × 资源权重标签系数"]
        F1["格式标签优先级<br/>standard=1.0<br/>semi-standard=0.8<br/>raw-binary=0.5<br/>unknown=0.1"]
        F2["资源权重标签系数<br/>确定=1.0<br/>不确定=0.5"]
        F1 --> WGT
        F2 --> WGT
    end

    subgraph DECOUPLE["两套权重严格解耦 v1.1"]
        X1["缓冲区物理丢弃权重<br/>仅用于资源淘汰"]
        X2["LLM业务权重标签<br/>确定 / 不确定<br/>仅用于上层业务判断"]
        X1 -.->|"互不强制联动"| X2
    end

    W2 --> D1 --> D2
    WGT --> D2
    DECOUPLE --> WGT

备注：按格式分区不按内容分区，因为格式客观可判定、内容主观不可判定。丢弃是物理上限下的最后手段，非设计意图。v1.1 核心修订：物理丢弃权重与 LLM 业务权重标签解耦，取值可不同、互不强制联动，防止资源淘汰逻辑污染业务标签。

---

图4｜异步通道溢出与 returnRoute 反馈（v1.1 补齐）

flowchart TB
    A["异步通道持续入流"] --> B{"通道容量达上限?"}
    B -->|否| C["正常入队<br/>只写 Trajectory"]
    B -->|是| D["按物理丢弃权重淘汰最旧条目<br/>unknown+不确定优先"]
    D --> E{"被丢弃消息带 returnRoute?"}
    E -->|是| F["生成『消息因异步通道溢出被丢弃』反馈事件<br/>沿 returnRoute 原路投递原始来源<br/>反馈事件写入 Trajectory"]
    E -->|否| G["不发送反馈"]
    F --> H["原始业务消息直接丢弃<br/>不再向上层递送"]
    G --> H
    D --> I["标记 async_channel_overflow:true"]


备注：这是 v1.1 三处🔴机理缺口之一。异步通道溢出时若丢弃消息带 returnRoute，必须生成丢弃反馈事件沿原路回投并写 Trajectory；无 returnRoute 则无反馈。避免出现"消息被丢但来源永远等不到应答"的未定义执行路径。

---

图5｜反向流机制

flowchart LR
    L["LLM 发出指令<br/>携带 targetId"] --> G1["网关分配反向流序号 R-xxx"]
    G1 --> G2["按 targetId 查找目标插件/外部系统<br/>注册端点"]
    G2 --> G3["按 returnRoute 直达目标<br/>插件 / 外部工具"]
    G3 --> G4["写入 Trajectory<br/>flow:'reverse'"]
    G4 --> T["Trajectory"]

    subgraph NO["反向流不做的事"]
        N1["不拦截"]
        N2["不做权限校验"]
        N3["不做语义校验"]
        N4["不做资源管控"]
        N5["不做限流阻断"]
    end

    OV["反向队列过载"] -.->|"仅告警"| FB["独立反馈通道上报<br/>节流交给上层 LLM"]

备注：反向流不拦截、按来源归属直达，因为网关本身无控制能力，LLM 是唯一决策者。网关只做路由转发；反向通道过载通过独立反馈通道告警，节流管控由上层 LLM 负责，网关不做限流阻断。

---

图6｜序列号、来源ID 与路由

flowchart TB
    subgraph ID["信息入网关即分配来源ID"]
        F1["flowId<br/>正向 F-xxx / 反向 R-xxx"]
        F2["sourceId<br/>来源标识"]
        F3["returnRoute<br/>响应回到哪里"]
        F4["timestamp<br/>全局时间戳 纳秒级"]
        F5["seq<br/>方向内序号"]
    end

    subgraph SORT["跨方向排序规则"]
        S1["先按 timestamp 排"]
        S2["timestamp 相同时<br/>方向优先级：正向优先"]
        S1 --> S2
    end

    subgraph ROUTE["路由机制"]
        R1["flowId 区分正/反向"]
        R2["sourceId 正向来源"]
        R3["targetId 反向目标"]
        R4["returnRoute 响应投递路由"]
    end

    ID --> SORT
    ID --> ROUTE
    SORT --> NOTE1["仅提供排序参考素材<br/>网关本体不做全局重排序<br/>业务时序归并属外部自研增补"]
    ROUTE --> NOTE2["returnRoute 只做消息投递路由<br/>不校验响应payload与原始请求业务语义是否匹配<br/>业务一致性交给 LLM/上层"]

备注：来源 ID 与 returnRoute 是防返回乱核心机制。跨方向排序字段仅作参考，网关本体不执行全局事件重排序，业务时序归并属【外部自研增补】。序列号格式为建议方案，可由项目方自定。

---

图7｜独立反馈通道 + 自身降级兜底

flowchart TB
    subgraph TRIG["触发报备的场景"]
        T1["网关异常"]
        T2["信息损失：事件丢失 / 截断"]
        T3["缓冲区 ≥80% buffer-warning"]
        T4["通道溢出 channel-overflow"]
        T5["丢弃事件 buffer-overflow"]
        T6["网关内部异常 gateway-error"]
        T7["信息损失 event-loss"]
    end

    subgraph CH["反馈通道特性"]
        C1["独立于业务流"]
        C2["单向：网关 → LLM"]
        C3["写入 Trajectory<br/>gateway-status:true"]
        C4["优先级高，异常优先于业务"]
    end

    TRIG --> CH

    subgraph DEGRADE["v1.1 反馈通道自身降级兜底"]
        D1{"Trajectory 可写?"}
        D2{"本地轻量标记文件可写?"}
        D3["告警永久丢失<br/>属于载体资源约束风险<br/>网关不再重试"]
        D1 -->|是| OK["正常写入"]
        D1 -->|否| D2
        D2 -->|是| LOCAL["落本地标记文件"]
        D2 -->|否| D3
    end

    CH --> DEGRADE

    subgraph CFG["配置更新入口"]
        CF1["LLM 收到 buffer-warning 后<br/>通过反向流 targetId=gateway-config 下发"]
        CF2["网关识别后动态调整参数"]
        CF3["网关不主动请求调整<br/>是否调整由 LLM 决定<br/>属外部依赖·LLM决策回路"]
        CF1 --> CF2 --> CF3
    end

    C4 --> CFG
    NOTE["正常状态不报备<br/>异常才报备"]
    NOTE -.-> TRIG


备注：正常状态不报备（无信息量），异常才报备。反馈通道独立防止被业务消息挤占。v1.1 补齐：Trajectory 不可写时降级到本地标记文件；本地也不可写则告警永久丢失，网关不重试，属载体资源约束。配置调整入口只提供接收入口，是否调整由 LLM 决策。

---

图8｜网关生命周期状态机

stateDiagram-v2
    [*] --> LOADING: DSH内核 → AI网关 → 业务插件
    LOADING --> CHECK_CRASH: 检查 lockfile / crash marker
    CHECK_CRASH --> RUNNING: 前次正常卸载
    CHECK_CRASH --> RUNNING: 前次非正常退出，生成 gateway-crash-detected 事件写入Trajectory

    RUNNING --> RUNNING: 拦截器通过可逆副作用注册
    RUNNING --> RUNNING: 拦截正向流+反向流

    RUNNING --> UNLOADING: 触发卸载

    state UNLOADING {
        [*] --> BLOCK_FLUSH: unloadStrategy=block-flush
        BLOCK_FLUSH --> FLUSH_FWD: 先排空正向流全部缓冲区
        FLUSH_FWD --> FLUSH_REV: 再处理反向流飞行指令
        FLUSH_REV --> FLUSH_DONE: 全部队列落地Trajectory/EventBus
        BLOCK_FLUSH --> FLUSH_TIMEOUT: 超时 unloadTimeoutMs 默认5000
        FLUSH_TIMEOUT --> FORCE_DROP: 强制丢弃剩余正向+反向飞行事件
        [*] --> FORCE_DROP2: unloadStrategy=force-drop
    }

    UNLOADING --> [*]: 正常卸载完成
    UNLOADING --> CRASHED: 超时强制丢弃
    CRASHED --> [*]: 记录 gateway-flush-timeout / gateway-forced-unload

备注：加载顺序 DSH 内核 → AI 网关 → 业务插件，网关作为特权插件最先加载。卸载 block-flush 同时包含正向全部缓冲区+反向队列，刷新顺序：先正向后反向。强制卸载时反馈通道不可用。崩溃检测通过 lockfile/crash marker；OOM 非正常崩溃内存事件丢失，磁盘前置持久化队列属【外部自研增补】。

---

图9｜五种运行工况

flowchart TB
    subgraph STABLE["稳态"]
        S1["业务全走标准通路"]
        S2["缓冲区水位正常"]
        S3["三通道各自时序有序"]
    end

    subgraph OFFSET["偏移态"]
        O1["少量异步识别失败"]
        O2["部分事件归入 uncertain"]
        O3["缓冲区压力升高"]
        O4["观测元数据残缺<br/>网关不自愈"]
    end

    subgraph CONFLICT["冲突态"]
        C1["同一业务行为产生多条分属不同通道事件"]
        C2["跨通道时序错乱"]
        C3["事件合法标签完整<br/>但上层拿到时序矛盾的观测集合"]
    end

    subgraph SPLIT["分裂态｜仅单实例网关"]
        P1["触发前提：配置增量热修改<br/>非原子全量替换"]
        P2["运行时新旧配置逻辑并存"]
        P3["同一信号被不同规则处理"]
        P4["观测视图分裂 Trajectory 日志分裂<br/>不是业务输出分裂"]
        P0["若原子全量替换热重载<br/>不出现分裂态"]
        P1 --> P2 --> P3 --> P4
        P0 -.-> P4
    end

    subgraph SILENT["静默业务污染态｜载体外部依赖风险"]
        L1["native 模块加载后 C++ syscall 无事件"]
        L2["或 OOM 崩溃导致内存事件丢失"]
        L3["Trajectory 大体完整但缺失关键行为"]
        L4["上层基于残缺观测做决策<br/>不属于网关机理缺陷"]
        L1 --> L3
        L2 --> L3
        L3 --> L4
    end

    subgraph CRASH["崩溃态"]
        X1["进程异常 / OOM"]
        X2["内存缓冲区全部丢失"]
        X3["网关停止工作"]
        X1 --> X2 --> X3
    end

    STABLE --> OFFSET --> CONFLICT
    CONFLICT -.-> SPLIT
    SPLIT -.-> SILENT
    SILENT -.-> CRASH

备注：v1.1 修订明确：分裂态仅在增量热修改配置（非原子全量替换）时触发，是单实例内观测视图分裂，不是业务输出分裂；跨 DSH 实例观测差异属部署拓扑问题不属网关架构分裂。静默业务污染态归为载体外部依赖风险（native syscall 不可拦截 / OOM 内存丢失），不是网关机理缺陷。

---

图10｜与六契约的对应关系

flowchart LR
    subgraph FUNC["网关功能"]
        F1["格式分类 + 升级规则"]
        F2["不确定信号独立通道"]
        F3["native / network / fs 拦截记录"]
        F4["双写 Trajectory + EventBus"]
        F5["调用次数记录 不限制"]
        F6["bypassDetected 标记"]
    end

    subgraph CON["六条动态契约"]
        C1["① 信息配时序<br/>事件格式标准化后进入时序流"]
        C2["② 控制配轮询<br/>不确定信号不阻塞主控制流"]
        C3["③ 数据配权限<br/>记录资源访问请求，补全权限审计"]
        C4["④ 一致就向前<br/>状态在多流间一致"]
        C5["⑤ 能量带定义<br/>记录消耗不限制"]
        C6["⑥ 状态要报备<br/>检测旁路调用并上报"]
    end

    F1 --> C1
    F2 --> C2
    F3 --> C3
    F4 --> C4
    F5 --> C5
    F6 --> C6


备注：六契约与网关功能一一对应。网关不主动满足契约的"控制"语义，只做观测与报备，控制决策留给 LLM。

---

图11｜网关边界｜做不到的事

flowchart TB
    subgraph B1["边界1：native 模块加载后的内部 syscall"]
        B11["能做到：拦截 process.dlopen 加载动作"]
        B12["做不到：拦截加载后的 C++ 层 syscall"]
        B13["处理：记录『插件A加载了native模块』<br/>生成补充事件写 Trajectory<br/>observability:'degraded' + source:'ai-gateway-supplement'"]
        B11 --> B13
        B12 --> B13
    end

    subgraph B2["边界2：parentEventId 因果链"]
        B21["做不到：不自动推断父子关系"]
        B22["parentEventId 默认 null"]
        B23["处理：因果推断交给 LLM<br/>网关只提供时序和来源"]
        B21 --> B23
        B22 --> B23
    end

    subgraph B3["边界3：异步/worker 场景 sourcePlugin 识别"]
        B31["做不到：异步脱离上下文栈后识别可能失败"]
        B32["处理：识别失败填 unknown，不猜"]
        B31 --> B32
    end

    B1 --> MARK1["标记仅标记 dlopen 加载入口<br/>不自动追踪后续每条调用<br/>上层需自行关联做可信度降级"]
    B2 --> MARK2["因果由 LLM 根据时序和内容推断"]
    B3 --> MARK3["宁可 unknown 不猜"]

备注：三条边界是网关的能力硬边界。native syscall 无法拦截 → 静默业务污染态载体风险；parentEventId 不推断 → 因果交 LLM；异步 sourcePlugin 识别失败填 unknown 不猜。

---

图12｜零控制能力声明

flowchart LR
    subgraph GW["网关做什么"]
        Y1["分类"]
        Y2["转译"]
        Y3["打标签"]
        Y4["记录消耗"]
        Y5["路由转发"]
        Y6["观测报备"]
    end

    subgraph NO["网关不做什么"]
        N1["不做决策<br/>决策权唯一属于 Agent-Loop LLM"]
        N2["不阻断调用<br/>只分类转译不拦截执行"]
        N3["不做业务语义校验"]
        N4["不做资源管控<br/>只记录消耗不限制消耗"]
        N5["不做权限拒绝<br/>只打标签不拒绝"]
        N6["不引入独立 LLM<br/>网关是纯分类器"]
        N7["不降权<br/>只打权重标签，降权由 LLM 决定"]
        N8["不限流<br/>只记录调用次数，限流由 LLM 决定"]
    end

    LLM["Agent-Loop LLM<br/>唯一决策主体"] -->|"接收标签和事件"| DEC["业务判断"]
    GW -->|"只提供素材"| LLM
    NO -.->|"边界禁止"| GW

备注：核心原则「零业务控制」：网关一旦开始做控制就变成新决策者，与 DSH 内核不做业务判断的哲学冲突。网关是纯分类器+转译层+观测层，决策权唯一属于 Agent-Loop LLM。

---

图13｜三件套工件与测试设计关系

flowchart TB
    subgraph MAIN["主设计文档"]
        M1["AI 网关整体方案 v1.1 定稿"]
    end

    subgraph ART["附属工件三件套"]
        A1["工件A：隐患-测试-待确认参数三维映射对照表<br/>H-01 ~ H-09"]
        A2["工件B：故障排查工单模板<br/>故障解析七小步"]
        A3["工件C：缺陷三分分类判定表<br/>范式固有 / 实现bug / 载体约束"]
    end

    subgraph TEST["架构级测试设计"]
        T1["稳态 TC-01 ~ TC-06"]
        T2["偏移态 TC-07 ~ TC-10"]
        T3["冲突态 TC-11 ~ TC-13"]
        T4["分裂态 TC-14"]
        T5["崩溃态 & 边界载体 TC-15 ~ TC-17"]
        T6["特殊边界组合 TC-18 ~ TC-19"]
    end

    M1 --> ART
    M1 --> TEST
    A1 -.->|"隐患ID关联用例"| TEST
    A2 -.->|"故障复盘入口"| A3
    A3 -.->|"归类后决定是否刷新基点"| M1

    subgraph RULE["判定自检问题"]
        R1["调大/调小容量、权重，故障消失? → 调优参数非架构缺陷"]
        R2["Agent 类系统普遍存在的客观代价? → 倾向范式固有"]
        R3["架构文档已写明风险与缓解? → 优先归范式固有/载体"]
        R1 --> R2 --> R3
    end

    A3 --> RULE

备注：三件套用于评审、测试关联、线上故障复盘。工件A把隐患映射到测试用例和待确认参数；工件B是排查工单模板，严格按七小步填；工件C三分判定表决定是否刷新架构基点——只有范式固有机理本身写错缺失才刷新，实现 bug 和载体约束都不刷新。


---

图14｜v1.1 三处🔴机理缺口修复路径

flowchart TB
    subgraph FIX1["修复1：异步通道丢弃 returnRoute 应答去向"]
        X1["异步通道溢出丢消息"]
        X2["检查 returnRoute"]
        X3["有：生成丢弃反馈事件原路回投 + 写Trajectory"]
        X4["无：不反馈"]
        X1 --> X2 --> X3
        X2 --> X4
    end

    subgraph FIX2["修复2：block-flush 卸载包含反向流队列"]
        Y1["卸载触发 block-flush"]
        Y2["刷新顺序：先正向全部缓冲区"]
        Y3["再处理反向流飞行指令"]
        Y4["全部落地 Trajectory/EventBus 才算卸载就绪"]
        Y1 --> Y2 --> Y3 --> Y4
    end

    subgraph FIX3["修复3：反馈通道 Trajectory 不可写降级兜底"]
        Z1["告警事件需写入 Trajectory"]
        Z2{"Trajectory 可写?"}
        Z3["本地轻量标记文件"]
        Z4["告警永久丢失<br/>不再重试<br/>载体资源约束"]
        Z1 --> Z2
        Z2 -->|是| Z5["正常写入"]
        Z2 -->|否| Z3
        Z3 -->|"文件也不可写"| Z4
    end

    subgraph FIX0["两处🟠歧义消除"]
        W1["物理丢弃权重 vs LLM业务权重标签解耦"]
        W2["unknown 消息投递到不确定层通道"]
        W3["分裂态前提：区分增量热改 vs 原子全量替换"]
        W4["静默业务污染态标记为载体外部依赖风险"]
    end

备注：v1.1 相对 v1.0 的全部修订。三处🔴是消除未定义执行路径的机理缺口；两处🟠是歧义定义澄清。全部调优数值仍留待项目方填充，未硬编码进架构契约。核心设计哲学未改动。


图15｜完整请求生命周期全景（正向+反向+反馈合流）


sequenceDiagram
    participant EXT as 外部/native
    participant GW as AI网关
    participant TR as Trajectory
    participant EB as EventBus
    participant FB as 反馈通道
    participant LLM as Agent-Loop LLM
    participant PLG as 目标插件

    EXT->>GW: 信号入站
    GW->>GW: 分配 flowId/sourceId/returnRoute/timestamp
    GW->>GW: 接收缓冲区按格式分区
    GW->>GW: 升级规则 standard/semi→confirmed, raw/unknown→uncertain
    alt 确定层
        GW->>TR: 批量flush 双写
        GW->>EB: 批量flush 双写
    else 不确定层
        GW->>TR: 即时单条 主写
    else 异步通道
        GW->>TR: 单写 不进EventBus
        Note over GW: 溢出时按物理丢弃权重淘汰<br/>带returnRoute生成丢弃反馈
    end

    LLM->>GW: 反向指令 targetId+returnRoute
    GW->>GW: 分配 R-xxx
    GW->>PLG: 按 returnRoute 直达 不拦截
    GW->>TR: 写 flow:'reverse'

    Note over GW,FB: 缓冲区≥80% / 溢出 / 丢弃 / 异常
    GW->>FB: buffer-warning / buffer-overflow / gateway-error
    FB->>TR: gateway-status:true 高优先级
    TR-->>LLM: 读取反馈
    LLM->>GW: 如需调整 targetId=gateway-config

备注：全景时序合流图。正向流三通道分流落盘，反向流直达不拦截，反馈通道独立高优先级写入 Trajectory。异步通道溢出是唯一会主动向来源回投反馈的路径。配置调整闭环由 LLM 决策，网关只提供接收入口。



































































