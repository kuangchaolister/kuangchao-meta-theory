# KC‑TRSF‑SS 狂潮‑意图搜索变体 V2.3（完整待审查封档版）

> 
> 定位：基于 Transformer 物理组件收编重构的端到端意图搜索引擎
> 核心目标：彻底废除传统倒排索引主导模式，实现「多维意图矩阵 → 粗排召回 → 微数据引擎语义提纯 → Cross‑MHA 深度匹配 → 动态叠权收敛 →【可选双向微引擎反向商业分发】完整端到端闭环。
> 重要声明：本方案为范式级架构设计；算力优化、训练超参、底层部署细节属于后置工程实现范畴，不在本文档定义范围。
> 核心契约：**商业分发逻辑严禁污染 Intent_Matrix 意图矩阵，双流合一的对外契约必须取自商业分发之前的纯净链路节点。**

## 一、全局双流模型拆解

### 1. 权限‑状态流（系统里有什么，如何演变）

- 输入域：原始自然语言 Query、用户会话上下文（可选）
- 核心状态库
  - **高维意图矩阵 Intent_Matrix**：由 KC‑MHA 多头意图感知层生成，承载显性与隐性用户诉求，输出格式 `[5, hidden_dim]`
  - **软候选池（Soft Candidate Pool）**：多级混合索引召回得到的候选文档集合
  - **纯净索引版本树**：记录每一份文档抓取版本号、时间戳，用于拦截过期 / 废弃文档副本，防止旧数据污染
  - **语义骨架缓存池（可选后置工程）**：热点文档经微数据引擎提取后的语义骨架，用于降低在线推理开销
- 输出域：
1）叠权收敛后**纯净结构化结果 + intent_contract 契约（双流合一对外标准输出，取自商业分发前）**；
2）可选：经过双向微引擎反向分发 + LLM 融合之后面向用户展示的带商业标的结果。

### 2. 边界‑执行流（谁在什么条件下执行）

- 并发调度器：控制 KC‑MHA 五路意图头、微数据引擎多文档解析的并行计算
- 多级混合索引：HNSW 向量索引 + 辅助倒排文本索引，完成粗阶段召回
- 重排物理闸门：硬逻辑拦截商业竞价标签，排序仅由语义匹配信号驱动
- 双向微引擎：正向做文档语义骨架提纯；子模块 4‑A 反向分发（受开关控制），仅处理输出副本，不改动内部意图矩阵。

## 二、完整模块级架构设计（七大模块，含 4‑A 可选子模块）

### 模块 1：意图接入与预处理层（Input Gateway）

- 核心职责：接收用户原始输入，仅做输入清洗、特殊字符过滤，**不做关键词提取、不做查询改写**
- 输入：raw_query: string，用户上下文 context
- 输出：标准化查询文本，送入 KC‑MHA 多头意图感知层

### 模块 2：KC‑MHA 多头意图感知层（改造自 Transformer Encoder）

> 
> 本层负责生成意图矩阵与各头置信度；**训练阶段五路意图头投影矩阵 Wq/Wk/Wv 施加子空间正交损失**，抑制原生预训练模型带来的语义混杂；正交约束仅作用于本层，后续 Cross‑MHA 匹配层放开耦合限制。

五路路由意图头定义：

- Head_Entity：实体抽取（人物、物品、专有概念）
- Head_Causal：因果 / 逻辑意图，解析用户求解目的
- Head_Constraint：限制条件（时间、预算、规格、范围）
- Head_Implicit：隐性意图，结合会话上下文挖掘未直白表达诉求
- Head_Emotion：情绪 / 场景识别（求知、冲动、应急、娱乐等）

关键改造点：

1. 屏蔽 Decoder 文本生成通路，仅使用 Encoder 通路；
2. 废弃简单均值池化，采用**意图感知注意力池化**，以各头 [CLS] 向量作为 Query 对序列加权求和；
3. 输出标准契约（可直接对外输出，用于对接 KC‑TRSF‑TJ 推荐变体）

```
{
  "intent_matrix": [[...], [...], [...], [...], [...]],
  "confidence_scores": [float, float, float, float, float],
  "raw_query": "用户原始查询文本"
}
```

> 
> 说明：`confidence_scores`由各意图头注意力分布熵经过归一映射得到，值域 0~1，作为后续叠权动态权重来源。

> 
> 冷启动逻辑：新用户 / 无可用会话上下文时，Head_Implicit 置信度强制置为≤0.05，叠权层通过 mask 将权重平滑转移至其余四路意图头。

### 模块 3：粗排召回层（多级混合索引）

向量库（Milvus/Qdrant）为主，构建**多级混合索引：HNSW 向量索引 + 辅助倒排文本索引**

1. 以意图矩阵池化表征执行向量检索，召回 Top‑500 候选文档；
2. 倒排索引仅作为补充增强手段，**向量语义信号为主，倒排字面命中不参与主排序，仅用于挽回向量漏召的冷门文档**；
3. 候选送入下一阶段前，先查询纯净索引版本树，直接过滤标记为废弃的文档副本，避免无效计算。

### 模块 4：微数据引擎（抽筋拔髓・前置语义降维层）

> 
> 定位：在线实时语义提纯模块，解决 Cross‑MHA 算力爆炸，同时过滤 SEO 注水、冗余废话噪声。
> ⚠️架构固有取舍：语义骨架提炼会存在小概率丢失细微边界条件、脚注、次要约束信息的风险；**Cross‑MHA 匹配仅使用语义骨架，返回给用户的最终结果仍然输出完整原始文档**。
> ⚠️取舍：在线实时解析会引入额外推理时延；热点文档骨架缓存属于后置工程优化，不在本范式内定义。

执行流程：

1. 接收粗排输出的 500 份候选文档；
2. 根据文档元类型（新闻、学术、手册、普通网页）差异化策略，剥离冗余文本、SEO 注水内容，提取高密度语义节点（语义骨架）；
3. 输出文档新鲜度、文档类型元信息，供叠权层时间衰减计算使用；
4. 输出提纯后的语义骨架 Token 序列，交给 KC‑TRSF‑CrossMHA；
5. 时间衰减相关打分逻辑不在本模块执行，元信息仅做输出，打分计算统一收敛到动态叠权层。

#### 模块 4‑A：双向微引擎‑反向分发子层【commercial_mode 开关控制，可选子模块】

> 
> ⚠️核心铁律：**本模块只接收动态叠权收敛完成后的结果副本；绝不修改、污染原始 Intent_Matrix；双流合一 intent_contract 契约取自本子模块之前。**
> 输入：叠权收敛纯净结果集、原始 intent_matrix、经过风控网关过滤之后的商业标的物库。
> 执行逻辑：

1. 意图锚定：Top‑3 必须 100% 纯意图匹配内容，商业标的严禁侵入前 3 展示位；底层硬编码，业务不可修改；
2. 利用原始 intent_matrix 对商业标的物做关联语义召回；
3. **双重约束：位置约束（禁止进入 Top3） + 分数约束（商业标的得分不允许超过第 3 位纯文档匹配得分）**，执行柔性降权；
4. 全部商业输出强制打上`<sponsored>`标签；
5. 交付 LLM 做语言润色融合；契约约束：**LLM 只做话术改写润色，禁止重排结果相对优先级；输出网关二次校验拦截 LLM 乱序输出**。
边界工况：若无匹配商业候选，直接透传原始纯净结果，不生成虚构商业推荐。

### 模块 5：KC‑TRSF‑CrossMHA 交叉注意力匹配层

> 
> 范式要点：KC‑MHA 意图头保留正交约束；**本层放开正交强制约束，允许文档同时匹配多路意图，支持合理意图耦合**。

1. Query 侧：使用来自 KC‑MHA 的 Intent_Matrix；
2. K/V 侧：全部使用微数据引擎输出的文档语义骨架表征；
3. 执行交叉注意力计算，输出每篇文档对应五路意图的独立匹配分数`CrossMHA_Score_i`。

### 模块 6：动态叠权收敛层（Deterministic Stacking）

1. 权重规则：严禁静态写死权重；由`confidence_scores`做 Softmax 归一化生成五路动态权重

\(W_i=\frac{e^{\text{confidence}_i}}{\sum_{j=0}^{4}e^{\text{confidence}_j}}\)
2. 单文档得分公式：

\(Score=\sum_{i=0}^{4}W_i \times CrossMHA\_Score_i \times TimeDecay(doc\_type)\)

> 
> TimeDecay：差异化时间衰减；学术、标准手册类文档衰减系数弱于新闻、时效性网页。

3. 物理熔断：文档携带`<sponsored>`商业竞价标签或`<SEO_Spam>`垃圾标签，直接权重清零过滤；`<SEO_Spam>`标签由上游抓取网关指纹规则生成。
4. 读取纯净索引版本树，保证只保留文档有效版本。

> 
> 【**双流合一标准输出点**】本层执行完毕，输出纯净`intent_contract`，供给 KC‑TRSF‑TJ；4‑A 反向分发子层只拿副本做展示。

### 模块 7：输出网关与执行边界（Result Gateway）

契约约束：

- 能量带控制：单次搜索总超时 500ms 熔断；最大输入序列长度、并发 QPS 上限做硬限制；
- 高负载降级：QPS 逼近阈值时自动下调粗排 Top‑K，由 500 逐级降低至 200/100；向量库故障时，**禁止回退传统倒排索引**，仅可使用骨架缓存做极简意图匹配返回；
- 状态报备：输出附带 Debug 解释字段 Explain_Debug，记录哪一路意图头贡献主要得分，用于审计排查；
- 4‑A 子层开启时，网关校验 LLM 输出，防止商业标的优先级被大模型篡改。

## 三、工况判定与系统行为

1. **稳态**：查询意图清晰，五路置信度分布合理，链路在超时阈值内返回排序结果。
2. **偏移态‑意图模糊**：五路`confidence_scores`全部低于 0.3；
系统行为：优先向用户发起追问，引导补充诉求；备选方案返回高泛化度候选文档集合。**微数据引擎不承担常识生成任务。**
3. **冲突态‑意图撕裂**：五路 confidence_scores 方差显著偏大，各路意图置信度严重分化；
系统行为：强制反问用户澄清矛盾诉求，不强行猜测输出结果。
4. **崩溃态**：关键模块超时 / 异常；返回提示，绝不降级回退倒排索引链路。

## 四、架构固有取舍说明（预印本必须附带，用于防御评审质疑）

1. **微数据引擎信息丢失风险**：语义骨架在线提纯会丢弃冗余内容，存在小概率丢失次要约束、脚注细节；匹配使用骨架，用户侧返回完整原文。
2. **算力取舍**：微数据引擎降低 Cross‑MHA 算力，但引入在线解析开销；热点骨架缓存属于后置工程手段。
3. **粗排固有短板**：即便使用多级混合索引，粗排阶段依旧存在漏召可能性；深度匹配上限被粗排候选池约束。
4. **分层正交约束**：KC‑MHA 意图感知训练施加投影子空间正交约束，保障五个意图头分工；Cross‑MHA 匹配层放开约束，允许意图耦合，二者不可混淆。
5. **时间衰减差异化代价**：经典文档弱化时间衰减，会保留部分老旧过时观点，依赖上游抓取网关质量管控。
6. **反向分发子层取舍**：反向商业召回存在弱关联误匹配风险；双重分数 + 位置约束缓解，但无法完全根除；LLM 融合存在篡改顺序风险，依赖网关后校验；商业库需要前置独立风控网关过滤虚假标的。

## 五、完整可参考伪代码

```
def kc_trsf_ss_search(user_query: str, context, tenant_id: str, commercial_mode=False):
    # 1. KC‑MHA意图感知层
    intent_matrix, confidence_scores = KCMHA_Encoder(user_query, context)

    # 2.多级混合索引粗排召回Top‑500
    coarse_candidates = MultiLevelVectorDB.search(
        query_vector = intent_aware_pooling(intent_matrix),
        top_k = 500,
        namespace = tenant_id
    )
    # 过滤纯净索引版本树中的废弃副本
    coarse_candidates = filter_outdated_doc(coarse_candidates)

    # 3.微数据引擎：抽筋拔髓提取语义骨架，输出元信息
    skeleton_list, doc_meta_list = MicroDataEngine.forward_extract(coarse_candidates)

    # 4.Cross‑MHA深度交叉匹配（Query=意图矩阵；K/V=语义骨架）
    cross_scores = KC_TRSF_CrossMHA(intent_matrix, skeleton_list)

    # 5.动态叠权收敛【双流合一原始契约输出点，商业分发使用副本】
    import numpy as np
    W = np.exp(confidence_scores) / np.sum(np.exp(confidence_scores))
    final_results = []
    for idx, doc in enumerate(coarse_candidates):
        doc_meta = doc_meta_list[idx]
        if "sponsored" in doc.tags or "SEO_Spam" in doc.tags:
            continue
        score = sum(W[j] * cross_scores[idx][j] for j in range(5)) * TimeDecay(doc_meta["doc_type"])
        final_results.append((doc, score))
    final_results.sort(key=lambda x:x[1], reverse=True)
    pure_intent_contract = {
        "intent_matrix":intent_matrix.tolist(),
        "confidence_scores":confidence_scores.tolist(),
        "raw_query":user_query
    }
    pure_result_payload = {"intent_contract":pure_intent_contract,"result_list":[i[0] for i in final_results[:10]]}

    # 6.可选4‑A双向微引擎反向商业分发子层
    if commercial_mode:
        display_payload = 
CommercialDistributor(pure_result_payload)
        return display_payload
    return pure_result_payload
```

---

### KC‑TRSF‑SS V2.3 图示说明

SS 为【人主动向外找信息】意图搜索变体，严格遵循元架构`intent_contract`跨组件契约。

1. 所有外部原始搜索信息不会自动沉淀进长期记忆，仅可被 JY 接收放入会话临时工作区；
2. 内置可信度过滤、多源熔断降级，故障不允许幻觉编造内容；
3. 输出标准化契约，可输入 JY 记忆、TJ 推荐，实现整套体系流转。




> 
> 【封档标记】KC‑TRSF‑SS V2.3 完整待审查版
> 版本变更摘要：
> 
> 
> 1. 将双向微引擎反向分发作为模块 4‑A 可选子模块，不破坏原有 V2.2 模块编号；
> 2. 严格固化契约：双流合一`intent_contract`取自叠权收敛完成、商业分发之前；商业分发仅操作副本；
> 3. 商业标的新增**分数 + 位置双重约束**；补充商业候选为空工况；增加 LLM 输出网关校验；
> 4. 商业标的物前置依赖独立风控网关；全部取舍声明补齐；伪代码同步更新。