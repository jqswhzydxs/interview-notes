# 大模型工程面试题：重点梳理与 Agent 项目实践

> 阅读对象：`05 大模型-大模型工程面试题.md`。
>
> 本文不逐题复述原文，而是重组知识主线、给出面试必要回答，并把可迁移到 Agent 项目的思想落到架构、策略和实现顺序。
>
> 当前工作区只有面试笔记，没有具体 Agent 项目源码，因此项目建议采用技术栈无关的表达；拿到源码后还需映射到实际模块、模型供应商和评测数据。

## 一页结论

这份文档真正值得掌握的不是 12 道题的独立答案，而是下面八个工程判断：

1. **LLM 的关键变化是把任务统一成“条件生成”。** Prompt、Few-shot、RAG、Tool Calling 和 Agent 都建立在这个统一接口之上。
2. **Transformer 原理要讲到数据流，而不是只背架构图。** 输入经独立投影得到 Q/K/V，注意力完成跨位置的信息聚合，FFN 做逐位置非线性变换；Decoder 通过因果掩码执行自回归生成。
3. **长上下文要分清三个瓶颈。** Prefill 的注意力计算、Decode 的 KV Cache/内存带宽、应用侧的有效上下文利用率不是同一个问题；GQA、FlashAttention、RAG 和上下文压缩分别处理不同层次。
4. **训练方法必须按两个维度组织。** “改哪些参数”是全量微调、LoRA、QLoRA；“优化什么目标”是 SFT、偏好优化或强化学习。两组名词不是并列关系。
5. **微调是行为学习手段，不是动态知识库。** 稳定格式、风格、策略和任务模式可考虑微调；价格、库存、制度、文档事实应进入 RAG、数据库或工具。
6. **Post-training 是组合流水线。** SFT、拒绝采样、DPO、PPO/GRPO、AI Feedback 可以前后组合；不存在一种方法替代全部环节。
7. **解码策略应由任务的可验证性决定。** 精确任务优先结构化输出、约束解码和低随机性；开放任务再提高采样多样性；推理任务可用多候选生成加验证器，而不是迷信单次“思维链”。
8. **Agent 项目最值得借鉴的是“反馈闭环”。** 先记录 Trace 和 Badcase，再构造黄金集、chosen/rejected 和可验证奖励；优先迭代 Prompt、上下文、工具与工作流，数据积累足够后再决定是否 LoRA/SFT/DPO。

如果面试时间有限，优先准备 **2、3、6、8、10、11、12**，再补 **5、9**。第 **4、7** 适合作为原理和趋势追问；第 **1** 用来建立总框架，不宜停留在“参数大、数据多”的定义上。

## 文档脉络

原文 12 道题可以重组为五层：

| 层次 | 对应题目 | 核心问题 |
| --- | --- | --- |
| 统一生成范式 | 1～2 | LLM 为什么能用一个接口处理多任务；Transformer 如何实现理解和生成 |
| 长上下文与推理效率 | 3～5 | KV Cache、访存、注意力复杂度、位置、Tokenizer 和上下文预算 |
| 模型能力怎样获得和适配 | 6～9 | 预训练、SFT、Scaling、微调决策、LoRA/QLoRA |
| 行为怎样继续优化 | 10～11 | RLHF、PPO、DPO、GRPO、拒绝采样和 AI Feedback 的关系 |
| 输出怎样被选择 | 12 | 贪心、Beam、Temperature、Top-K/Top-P、Self-Consistency 和推测解码 |

### 第一层：从“一任务一模型”到统一条件生成（1～2）

原文先用传统 NLP、BERT 和 LLM 三个阶段解释范式变化：

```text
传统 NLP：任务拆分 + 专用模型/任务头
  ↓
BERT：通用预训练表示 + 每个下游任务单独适配
  ↓
Decoder LLM：任务写进上下文 + 自回归生成统一输出
```

这里最重要的不是“传统模型小、LLM 大”，而是三个统一：

- **训练接口统一**：大规模文本可转成 next-token prediction。
- **任务接口统一**：分类、抽取、翻译、总结、代码都可以写成条件生成。
- **适配接口统一**：Zero-shot、Few-shot、RAG、工具结果都通过上下文进入模型。

Transformer 是这套范式的底座。面试时应能从输入一路讲到输出：

```text
Token IDs
  → Embedding + Position
  → Q/K/V 线性投影
  → Scaled Dot-Product Attention
  → Multi-Head 信息聚合
  → FFN 非线性变换
  → Residual + Normalization
  → 多层堆叠
  → LM Head 生成下一个 token 的概率分布
```

Encoder、Decoder 的本质区别是注意力可见范围：Encoder 通常双向建模输入；Decoder 使用因果掩码，只读取当前位置之前的 token。Decoder-only 适合通用生成，但 Encoder 在 Embedding、Rerank、分类等 Agent 子模块中仍然很有价值。

### 第二层：长上下文不是单一的 `O(N²)` 问题（3～5）

原文把 MHA 的问题归为显存、访存和复杂度，方向是对的，但面试时最好进一步区分阶段：

| 阶段 | 主要瓶颈 | 常见优化 |
| --- | --- | --- |
| Prefill | 长输入的注意力计算和中间激活 | FlashAttention、分块/稀疏注意力、缩短输入 |
| Decode | 每步读取权重和 KV Cache，常为 memory-bound | GQA/MQA/MLA、量化、Paged KV Cache、Continuous Batching |
| 应用上下文 | 无关内容、Lost in the Middle、Token 成本 | RAG、Rerank、摘要、状态外置、引用回查 |

MQA/GQA 与 FlashAttention 不在同一层：

- **MQA/GQA 改模型结构**，减少 K/V 头数，从而压缩 KV Cache。
- **FlashAttention 改计算实现**，用 tiling 和在线 softmax 减少 HBM 与片上 SRAM 之间的读写；它是精确注意力，标准稠密版本的算术复杂度仍是二次量级。
- 两者可以叠加；应用侧仍然要控制无效上下文，不能把底层加速当成“随便塞满窗口”的理由。

位置编码回答的是“Attention 如何知道先后和相对距离”。RoPE 的工程价值在于相对位置信息、零额外可学习参数和与现代推理栈的兼容，但长上下文扩展仍需要 Scaling、继续训练和专门评测，不能说成无损无限外推。

Tokenizer 则把字符问题转成了三个工程预算：

- **费用预算**：API 价格按输入/输出 token，而不是字数。
- **窗口预算**：系统提示、工具 Schema、历史、检索片段和预留输出共享同一窗口。
- **延迟预算**：输入越长，Prefill 越慢；工具列表和历史消息不是免费的。

### 第三层：能力形成与领域适配（6～9）

原文用“预训练定能力、SFT 教交互、对齐教偏好”建立了直觉。更严谨的表达是：

```text
预训练/继续预训练：学习语言、知识和通用模式
  ↓
SFT：学习指令、格式、任务示范和目标行为
  ↓
偏好优化 / RL：在多个可行行为中提高更优行为的概率
```

Scaling Law 的面试价值不在背 `1:20`，而在说明三种预算要一起看：参数量决定部署成本，数据量和质量决定训练充分度，训练算力决定可达到的组合。Chinchilla 的比例是特定假设下的 compute-optimal 经验结论，不是自然常数或数据上限。

微调选型要先问需求，再问方案：

```text
事实缺失或频繁变化？ → RAG / DB / Tool
格式不稳定？          → Schema + Few-shot + Validator
流程决策不稳定？      → 显式 Workflow + Tool Policy + Eval
稳定行为仍无法学会？  → SFT
多个合格答案需排序？  → DPO / Preference Optimization
可验证任务需探索？    → RL / GRPO 类路线
```

“改哪些参数”和“学什么目标”是正交维度：

| 参数更新方式 | 可搭配的目标示例 |
| --- | --- |
| Full Fine-tuning | SFT、DPO、RL |
| LoRA | SFT、DPO，部分 RL 场景 |
| QLoRA | 资源受限下的 SFT、DPO |

LoRA 的关键价值是保存和训练增量权重。它便于版本化、回滚和多任务适配；若合并进基础权重，可以没有额外的 LoRA 分支计算。若要在线动态挂载多 LoRA，则仍需考虑加载、显存、Batch、路由和未合并计算开销。

### 第四层：Post-training 是组合管线（10～11）

把几类方法按“数据从哪里来、如何优化”整理最容易记：

| 方法 | 训练信号 | 核心操作 | 主要代价 |
| --- | --- | --- | --- |
| SFT | 单个目标回答 | 最大化示范回答似然 | 只会模仿数据覆盖的行为 |
| 拒绝采样 + SFT | 多候选中的高分样本 | 生成、筛选、再监督学习 | 采样和筛选成本；受当前策略限制 |
| DPO | chosen/rejected 偏好对 | 直接优化相对偏好目标 | 依赖离线偏好数据及其分布 |
| PPO/RLHF | 奖励模型或环境奖励 | 在线采样并做策略优化 | 系统复杂、训练不稳定、Reward Hacking |
| GRPO 类方法 | 同题多候选的相对奖励 | 用组内相对基线做策略优化 | 多样本生成开销；奖励和组分布敏感 |
| RLAIF/AI Feedback | 强模型或规则给反馈 | 生成偏好、评分或修订数据 | 教师偏差、成本和自我强化风险 |

两条代表性思路：

- **DPO**：在 KL 正则化偏好学习设定下，将显式 Reward Model + 在线 RL 的一部分流程改写为离线分类式目标。它不是“任意 PPO 过程的完全等价替代”。
- **GRPO**：用同一问题多个回答的组内相对奖励代替独立 Value Model；对数学、代码、工具调用等有确定验证器的任务尤其有吸引力，但一次生成多个候选会增加推理成本。

Agent 工程通常不需要自己训练 PPO/GRPO，但完全可以复用其设计思想：**同题生成多个计划或动作，用确定性验证器打分，选择可执行且证据完整的候选，并把结果回流为评测或训练数据。**

### 第五层：解码是任务策略，不是固定参数（12）

解码策略回答“如何从每步概率分布选择 token”。面试时不要只背参数定义，而要把它和任务约束绑定：

| 任务 | 首选机制 | 原因 |
| --- | --- | --- |
| Tool Call / JSON / SQL AST | 严格 Schema、约束解码、语法校验 | 结构正确比语言多样性重要 |
| 分类、抽取、路由 | 低随机性 + 固定标签/枚举 | 便于复现和评测 |
| 数学、代码推理 | 单次低温或多候选 + 执行验证 | 正确性可验证，可用候选搜索换准确率 |
| 普通对话 | 适度 Temperature/Top-P | 平衡自然度和稳定性 |
| 创意生成 | 较高采样多样性 + 质量过滤 | 任务本身允许多个合理答案 |

Beam Search 在开放式对话中通常不是默认方案，因为最高似然序列不等于最有帮助的回答，且 Beam 会增加状态和缓存成本。但它并非与 KV Cache 或 FlashAttention “不兼容”，在翻译、受约束生成和候选搜索中仍有用途。

Self-Consistency 的可迁移本质是“多样化搜索 + 外部聚合”，推测解码的本质是“小模型提案 + 大模型验证”。前者提升任务质量但增加调用成本，后者在保持目标分布的正确实现中用于加速；两者不能混为一谈。

## 面试重点：按优先级准备

### P0：必须能结合项目讲清楚

#### 1. Transformer 与 Decoder-only（第 2 题）

**必要回答：**

> Transformer 用 Self-Attention 让每个 token 直接聚合其他位置的信息。输入通过独立的线性层得到 Q/K/V，用 `softmax(QKᵀ/√d_k)V` 计算相关性和聚合结果；除以 `√d_k` 是为了控制点积方差，避免 softmax 过度饱和。Encoder 通常做双向建模，Decoder 用因果掩码做自回归生成。Decoder-only 通过 next-token prediction 统一了大规模预训练和多任务生成接口，所以成为通用生成模型的主流。

**追问要点：**

- Multi-Head 让不同子空间学习不同关系；不要断言每个头都有稳定、可解释的人类语义。
- Attention 负责跨位置通信，FFN 负责逐位置非线性变换；事实知识并非只存于 FFN。
- Encoder 没有消失；Embedding、Rerank、分类和检索仍广泛使用。

#### 2. MHA、GQA、KV Cache 与 FlashAttention（第 3 题）

**必要回答：**

> 自回归 Decode 会缓存历史 token 的 K/V，避免每步重算前缀，但 KV Cache 随 Batch、长度、层数、KV 头数和头维度增长。MQA 让所有 Query Head 共享一组 K/V，GQA 让一组 Query Head 共享 K/V，在表达能力和缓存之间折中。FlashAttention 不改变模型语义，而是用分块和在线 softmax 减少 GPU HBM 读写。GQA 是结构优化，FlashAttention 是 kernel/实现优化，可以同时使用。

**追问要点：**

- 能写出 KV Cache 估算式，但先确认 dtype、KV heads、layers、batch 和是否量化。
- FlashAttention 降低额外显存和 IO，不把标准稠密注意力的计算复杂度变成线性。
- Prefill 与 Decode 的瓶颈不同；不要只用一句 `O(N²)` 概括推理。

#### 3. 训练和 Post-training 全景（第 6、10、11 题）

**必要回答：**

> 预训练学习通用分布和能力，SFT 用高质量示范教指令与目标行为，偏好优化或 RL 再区分“可接受”和“更好”。PPO/RLHF 显式学习奖励并在线优化策略；DPO 用离线偏好对直接优化相对偏好；GRPO 用同题多候选的组内相对奖励替代 Value Model；拒绝采样是生成多候选、筛高分样本、再做 SFT。真实模型通常组合使用这些阶段。

**追问要点：**

- Post-training 在业界通常是包含 SFT 在内的预训练后阶段，不宜限定成“SFT 之后”。
- DPO 是特定 KL 正则化偏好建模下的直接目标，不是所有 PPO/RLHF 的通用数学等价物。
- GRPO 省 Value Model，不代表训练无成本；Group Sampling、Reward 设计和长回答偏置仍要治理。

#### 4. 微调决策树（第 8～9 题）

**必要回答：**

> 我不会把微调作为默认答案。动态事实先用 RAG/工具，严格格式先用 Schema、Few-shot 和验证器，复杂流程先显式编排；只有稳定行为模式仍无法可靠获得，或需要小模型替代大模型降低长期推理成本时，才评估 SFT。参数更新上选 Full/LoRA/QLoRA，训练目标上选 SFT/DPO/RL，两者是正交的。微调后必须同时做目标任务、通用能力和安全回归。

**追问要点：**

- 数据质量、覆盖和评测先于 rank、learning rate 等超参。
- LoRA 可单独存储、版本化和回滚；合并部署与动态 Adapter Serving 的取舍不同。
- LoRA 降低灾难性遗忘风险，但不保证不会遗忘。

#### 5. 解码与结构化输出（第 12 题）

**必要回答：**

> 解码策略本质是从 next-token 分布选择下一个 token。贪心和低温更稳定，Top-K/Top-P 截断长尾并保留多样性，Beam 同时搜索多条高概率序列。Agent 的工具参数和 JSON 输出不应只靠 Temperature=0，而应使用严格 Schema/约束解码、业务校验和失败重试；推理类任务可以多次采样，再用执行器或验证器选答案。

#### 6. Token 与上下文预算（第 5 题）

**必要回答：**

> Tokenizer 把文本映射为模型处理的 ID 序列，现代模型多使用 BPE、Unigram/SentencePiece 等子词或字节级方案。工程上不能用字数粗略替代真实 token 计数；系统提示、历史、工具 Schema、RAG 片段和输出预留要统一预算，并按目标模型 tokenizer 实测成本和截断边界。

### P1：体现原理和架构深度

#### 7. 位置编码与长上下文（第 4 题）

讲清无位置编码的 Self-Attention 对排列是等变的；sin/cos、RoPE、ALiBi 的区别在注入位置和归纳偏置。重点说明 RoPE 外推不是免费能力，必须用长文检索、多跳推理、位置敏感和不同证据位置的数据评测。

#### 8. Scaling Law 与“涌现”争议（第 7 题）

讲清 Chinchilla 是固定训练算力下的数据/参数分配结论；部署时还要考虑训练一次、推理多次的总成本。涌现应表述为“部分任务和指标观察到能力跃迁”，同时指出离散指标可能把连续提升显示成突变，不背固定的参数阈值。

#### 9. LoRA 数学与部署（第 9 题）

若 `W ∈ R^(d_out×d_in)`，常见写法是：

```text
ΔW = B A
A ∈ R^(r×d_in)
B ∈ R^(d_out×r)
r ≪ min(d_in, d_out)
```

标准初始化通常让一个矩阵随机、另一个为零，使初始 `ΔW=0`。训练后可合并 `W' = W + scale·BA`；如果保留多个未合并 Adapter 做在线路由，则换取灵活性但承担运行时复杂度。

### P2：理解即可，不宜喧宾夺主

- 固定模型名、参数量、上下文长度、速度提升倍数和“某年必问”会快速过时。
- RoPE Scaling、FlashAttention kernel、GRPO loss 的公式细节，只有岗位偏训练/推理基础设施时才需要深挖。
- “某模型一定采用某方法”只能依据公开技术报告，不要从社区实现反推闭源训练配方。
- Token/字数换算、LoRA 显存和 KV Cache 数字必须注明模型、dtype、Batch、长度和实现条件。

## 可以直接用于自己 Agent 项目的设计

### 1. 推荐架构

```text
用户请求
  ↓
任务分类与风险策略
  ├─ 精确/结构化任务 → 低随机性 + Schema/约束解码
  └─ 开放/探索任务   → 适度采样 + 候选质量过滤
  ↓
上下文构建器
  ├─ Token Budget
  ├─ 历史摘要与状态外置
  ├─ RAG + Rerank + 引用
  └─ Tool/Skill 候选裁剪
  ↓
模型路由
  ├─ 小模型：分类、抽取、路由、简单工具参数
  └─ 强模型：复杂规划、多跳推理、疑难恢复
  ↓
Agent / Tool 状态机
  ├─ 计划、调用、观察、验证
  ├─ 最大步数、Deadline、取消和 Checkpoint
  └─ 结构校验、权限、审批、幂等
  ↓
验证与答案合成
  ├─ 规则/Schema/单元测试/数据库约束
  ├─ 引用一致性和事实核对
  └─ 必要时多候选投票或 Judge
  ↓
Trace + Feedback + Eval Store
  ├─ Golden Set / Badcase
  ├─ chosen / rejected
  └─ Prompt → RAG/Workflow → SFT/DPO 的升级依据
```

### 2. 建立统一 Token Budget

每次模型调用都显式预算，不在超限时临时从尾部粗暴截断：

```yaml
context_window: 128000
reserved_output: 8000
safety_margin: 2000
budgets:
  system_and_policy: 6000
  tool_schemas: 8000
  conversation_recent: 16000
  task_state: 4000
  retrieved_evidence: 60000
  scratch_or_plan: 6000
overflow_policy:
  - remove_duplicate_evidence
  - rerank_and_reduce_chunks
  - summarize_old_history
  - externalize_tool_results
  - fail_with_actionable_reason
```

建议记录每个组成部分的真实 token、Prefill 延迟和最终利用情况。窗口能容纳不等于信息能被模型稳定使用。

### 3. 按任务切换解码和验证策略

```yaml
task_profiles:
  tool_call:
    randomness: low
    strict_schema: true
    validator: schema_and_business_rules
  knowledge_answer:
    randomness: low
    citations_required: true
    validator: evidence_entailment
  code_generation:
    candidates: 3
    validator: compile_and_tests
  brainstorming:
    randomness: high
    candidates: 5
    validator: deduplicate_and_rank
```

Agent 的可靠性不能只靠统一 `temperature=0.7`。可验证任务应该让验证器承担主要质量控制，模型负责提出候选。

### 4. 把“Post-training 思想”变成应用反馈闭环

即使不训练模型，也可以复用同样的数据闭环：

```text
线上 Trace / 用户反馈 / 人工复核
  → 聚类 Badcase
  → 写成可重复运行的 Eval
  → 每题生成多个候选轨迹
  → 规则、工具结果或人工选 chosen/rejected
  → 先优化 Prompt、检索、工具和 Workflow
  → 仍存在稳定行为缺口时，沉淀 SFT/DPO 数据
```

建议保存的数据结构：

```yaml
case_id: agent_042
input: 用户原始目标
context_version: ctx_v7
model: provider/model/version
trajectory: []
candidate_outputs: []
verifier_results: []
chosen_id: candidate_2
rejected_ids: [candidate_1, candidate_3]
failure_category: wrong_tool_argument
human_reason: 订单号来自旧对话，未重新确认
prompt_version: agent_v12
tool_versions: {}
```

这套数据既能做回归评测，也能在达到规模和质量门槛后转成 SFT 或偏好数据。

### 5. 用“小模型提案 + 强模型/工具验证”控制成本

可借鉴推测解码和级联模型的思想，但在应用层实现：

1. 小模型负责意图分类、工具候选、参数初稿和简单摘要。
2. 规则和 Schema 先做零成本验证。
3. 高风险、低置信度或复杂多步任务升级给强模型。
4. 可执行任务用真实工具结果或测试作为最终验证，不用强模型自信度代替证据。

路由指标应同时观察任务成功率、升级率、误降级率、P95 延迟和单任务成本。

### 6. 什么时候才值得给 Agent 微调

同时满足以下条件再进入微调立项：

- 已有稳定且高频的目标行为，不是仍在变化的产品需求。
- Prompt、Few-shot、RAG、Schema、工具和 Workflow 已经优化过。
- 至少有成体系的训练集、验证集和独立回归集。
- 错误集中在可学习的行为模式，而不是上游数据、权限或工具故障。
- 调用规模足以覆盖训练、部署、维护和基础模型升级成本。
- 有明确回滚方案，并保留基础模型上的对照评测。

对多数 Agent 项目，优先级通常是：

```text
Eval/Trace
  → Prompt 与 Schema
  → RAG 与 Tool/Workflow
  → 模型路由与候选验证
  → 小模型蒸馏或 LoRA/SFT
  → 有偏好数据后再评估 DPO
  → 只有可验证高价值任务且资源充足才考虑 RL
```

### 7. 推荐实现顺序

#### P0：先建立可靠基线

1. 为每种任务定义成功条件和验证器。
2. 统一 token 计数、上下文预算和截断策略。
3. 结构化任务启用严格 Schema，并增加业务校验。
4. 固化模型参数、Prompt、工具和检索版本到 Trace。
5. 建立 30～50 条种子评测，覆盖正常、边界和失败路径。

#### P1：增加反馈和成本控制

1. 将线上 Badcase 自动回流为候选评测样本。
2. 按任务选择模型和解码策略，不使用全局固定参数。
3. 对代码、SQL、工具调用等任务增加执行验证。
4. 对复杂任务试验多候选生成，只在收益超过成本时保留。
5. 建立小模型/大模型级联和升级门槛。

#### P2：数据成熟后再训练

1. 清洗 Trace，去除泄露、错误标签和不可复现样本。
2. 稳定目标行为整理为 SFT 数据；相对偏好整理为 chosen/rejected。
3. 先在小规模 LoRA/QLoRA 上验证收益和回归风险。
4. 训练后与 Prompt/RAG 基线做同一评测集的成本—质量对比。
5. 只有明确胜出才上线，并保留 Adapter/模型版本回滚。

## 原文中需要谨慎表述的地方

1. **“传统 NLP 都是判别式”过度概括。** 传统 NLP 也有语言模型和 Seq2Seq 生成模型；更准确的对比是过去常用专用任务建模，而现代 LLM 以统一生成接口覆盖更多任务。
2. **“LLM 必须百亿/千亿参数”不是定义。** 是否属于 LLM 不应由固定参数阈值决定；现代小模型也可能具备强指令和工具能力。
3. **“没有位置编码时两句话结果完全一样”需更严谨。** Self-Attention 对输入排列是等变的：输入位置交换，输出也相应交换；它缺少绝对/相对顺序语义，不是所有张量逐项不变。
4. **“Decoder-only 完胜”只适合通用生成语境。** Encoder 和 Encoder-Decoder 在 Embedding、Rerank、分类、翻译等任务仍可能更高效。
5. **FlashAttention 没有把标准稠密注意力的算术复杂度降为 `O(N)`。** 它通过 IO-aware tiling 避免物化完整注意力矩阵，降低额外显存和 HBM 访问。
6. **GQA/MQA 的固定精度损失和速度倍数不能当通用结论。** 结果取决于模型规模、KV 组数、训练方式、硬件、序列长度和 kernel。
7. **RoPE 不是“训练 2K 可直接稳定推到任意长度”。** Scaling 只能改善外推，稳定上线通常还需要继续训练和按位置/长度分桶评测。
8. **Tokenizer 的字数换算只能粗估。** 中文、英文、代码、数字、空格和模型词表差异很大，应使用目标模型 tokenizer 实测。
9. **Llama 2 的高质量 SFT 标注不是约 100 万条。** 论文明确写的是 27,540 条自有高质量 SFT annotations；原文“约 100 万”应修正。
10. **“Alpaca、Vicuna 基本用 QLoRA”时间线不对。** QLoRA 论文的代表模型是 Guanaco；不能把更早发布的 Alpaca、Vicuna 直接归为 QLoRA 训练。
11. **原文 LoRA 维度和初始化示例有问题。** 对 `ΔW=BA`，常见维度是 `A: r×d_in`、`B: d_out×r`；标准初始化通常一个随机、一个为零，不是 A/B 都随机。
12. **“LoRA 推理零开销”有条件。** 合并权重后可以没有额外分支计算；动态多 Adapter 服务通常不合并，需要承担路由、批处理和 kernel 开销。
13. **LoRA Merging 代数上可加，不代表能力一定无冲突。** 不同 Adapter 的数据、目标和缩放可能互相干扰，合并后必须重新评测。
14. **Chinchilla `1:20` 是 compute-optimal 经验点，不是训练定律或数据上限。** 面试时要说明固定算力与部署成本目标。
15. **不存在通用的“50B～100B 涌现阈值”或“推理至少 30B”。** 能力取决于任务、数据、架构、训练、提示和评测指标；固定阈值会快速过时。
16. **Post-training 通常包含 SFT。** 把它定义为“SFT 之后的所有阶段”过窄；更稳的定义是预训练之后用于指令、偏好、安全和推理能力塑形的一组训练阶段。
17. **DPO 不是任意 PPO 的等价替换。** 它来自特定 KL 正则化 RLHF/偏好模型假设下的推导，工程上是离线偏好优化方法。
18. **PPO 的“四个模型”是逻辑角色，不等于显存固定四倍。** Reward/Value 规模、冻结、共享骨干、量化、分片和卸载都会改变实际资源。
19. **GRPO 省掉 Value Model，不等于显存必然接近 DPO。** 它需要为同一问题采样多个回答；是否还需要 Reward Model 取决于奖励能否由规则或环境直接给出。
20. **Beam Search 与 KV Cache、FlashAttention 不是“不兼容”。** Beam 会增加缓存管理和计算成本，但主流框架可以支持；是否使用取决于任务目标。
21. **代码、JSON、SQL 不应只依赖贪心解码。** 更可靠的方案是受约束生成、Schema/语法校验、执行验证和有限修复。
22. **固定参数和固定模型名单是快照。** 面试回答应附模型版本、日期、硬件、dtype、Batch、上下文长度和评测集。

## 面试表达建议

不要把“知道论文名”伪装成“做过训练”。可以把回答分成两层：

> 在我的 Agent 项目里，我实际落地的是模型路由、上下文预算、结构化输出、工具校验、Trace 和评测闭环；训练方法方面，我理解 SFT/DPO/GRPO 的目标和数据要求，但在没有足够高质量数据前不会为了技术名词贸然微调。我们先用线上 Badcase 构造 Eval 和 chosen/rejected，确认错误确实来自稳定行为模式，再评估 LoRA/SFT 是否比 Prompt、RAG 或 Workflow 更划算。

一个完整的项目回答可以这样组织：

> 我先按任务的确定性选择模型和解码策略：工具参数、抽取和路由使用低随机性、严格 Schema 和业务验证；开放生成才使用更高采样。上下文由统一 Token Budget 管理，历史状态外置，知识通过 RAG 和工具按需进入。复杂或可验证任务可以生成多个候选，用执行器、测试或规则选择，而不是让模型自评。全链路记录模型、Prompt、上下文、工具版本和结果，把 Badcase 变成回归集。只有当问题稳定、数据足够且调用规模能覆盖维护成本时，才用 LoRA/SFT 学行为，用 DPO 学相对偏好。

这段回答可以把 Transformer、Tokenizer、上下文、解码、微调、对齐和 Agent 生产实践串成一条线，比逐个背名词更有说服力。

## 面试前 10 分钟速记

1. LLM 的关键是统一条件生成接口，不只是参数大。
2. Q/K/V 来自同一输入的独立投影；除 `√d_k` 为控制方差。
3. Encoder 双向理解，Decoder 因果生成；Encoder 仍适合检索和分类。
4. KV Cache 省重算但吃显存；GQA 减 KV heads。
5. FlashAttention 降 IO 和额外显存，标准稠密计算仍是二次量级。
6. 长上下文分 Prefill、Decode、有效利用率三层。
7. Token 预算必须包含 System、历史、工具、RAG 和输出预留。
8. 预训练学能力，SFT 学示范，偏好/RL 学“哪个更好”。
9. 参数方式与目标正交：LoRA/QLoRA × SFT/DPO/RL。
10. 动态知识用 RAG/工具，稳定行为才考虑微调。
11. LoRA 合并可无分支开销；动态 Adapter 服务仍有成本。
12. Chinchilla 是固定算力下的配比，不是 `1:20` 铁律。
13. 涌现没有统一参数阈值，指标会影响是否看起来“突变”。
14. DPO 是离线偏好优化；不是所有 PPO 的完全等价物。
15. GRPO 用组内相对奖励代替 Value Model，适合可验证任务。
16. 拒绝采样是“多生成、筛高分、再 SFT”。
17. JSON/Tool Call 先用约束和验证，不只靠 Temperature=0。
18. Agent 的最小闭环是 Trace → Badcase → Eval → 改进 → 回归。

## 进一步核对的权威资料

- [Transformer：Attention Is All You Need](https://arxiv.org/abs/1706.03762)
- [FlashAttention：IO-aware 的精确注意力](https://arxiv.org/abs/2205.14135)
- [GQA：从 Multi-Head 模型到 Grouped-Query Attention](https://arxiv.org/abs/2305.13245)
- [Chinchilla：Training Compute-Optimal Large Language Models](https://arxiv.org/abs/2203.15556)
- [涌现指标争议：Are Emergent Abilities of Large Language Models a Mirage?](https://arxiv.org/abs/2304.15004)
- [LoRA：Low-Rank Adaptation of Large Language Models](https://arxiv.org/abs/2106.09685)
- [QLoRA：4-bit Quantized Model + LoRA](https://arxiv.org/abs/2305.14314)
- [Llama 2：27,540 条高质量 SFT 标注与 RLHF 流程](https://arxiv.org/abs/2307.09288)
- [DPO：Direct Preference Optimization](https://arxiv.org/abs/2305.18290)
- [DeepSeekMath：GRPO 的提出与数学推理训练](https://arxiv.org/abs/2402.03300)
- [DeepSeek-R1：Cold Start、RL、拒绝采样和多阶段训练](https://arxiv.org/abs/2501.12948)
