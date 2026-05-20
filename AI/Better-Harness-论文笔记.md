# Better Harness: A Recipe for Harness Hill-Climbing with Evals

> **来源**: LangChain Blog  
> **作者**: Vivek Trivedi  
> **日期**: 2026年4月8日  
> **链接**: https://www.langchain.com/blog/better-harness-a-recipe-for-harness-hill-climbing-with-evals

---

## 一、要 点 列 表

### 核心观点
1. **Evals 是 Agent 的"训练数据"** — 每个 eval 贡献一个信号（"agent 是否采取了正确行动"或"产生了正确结果"），这个信号指导对 harness 的下一次修改
2. **Better-Harness 是一个复合系统工程** — data sourcing → experiment design → optimization → review & acceptance
3. **Harness 改进需要学习信号才能"爬山"** — 用 evals 作为这个信号，加上防止过拟合的设计决策

### Evals 的获取方式
1. **Hand-curated（人工精选）** — 团队手动编写高价值但难以规模化
2. **Production traces（生产 trace 挖掘）** — 失败案例转 eval，是高吞吐量的杠杆方式
3. **External datasets（外部数据集）** — 需要人工整理确保测试用例反映期望行为
4. **Tag everything（打标签）** — 每个 eval 打上行为类别标签（tool selection、multi-step reasoning 等），支持有意义的 holdout 和定向实验

### 构建泛化学习系统的三大挑战
1. **数据不无限** → 用小而精的 curated evals，Quality > Quantity
2. **Reward hacking（奖励黑客）** → agent 会过拟合已有的 evals，prompt 无法完全避免
3. **缺乏泛化代理** → 用 holdout set 作为真实泛化的代理，结合人工 review 作为第二信号

### Better-Harness 流程（6 步）
1. **建立 eval 集** — hand-write + mining + external，打标签，定期清理饱和/无用 evals
2. **拆分 Optimization Set 和 Holdout Set** — 防止过拟合，holdout 确保学到的东西能泛化
3. **跑 baseline** — Optimization & Holdout 上各跑一次，作为所有更新的基准
4. **迭代优化循环（可选人工 review）** — 跑实验、提出修改、验证
5. **回归检查** — 确保新 evals 通过的同时不破坏已有的 passing cases
6. **人工 review 变更** — 人工检查指标遗漏的边缘情况和过度拟合

### 关键设计决策
- **Holdout set 是泛化的代理** — 自主爬山容易过拟合，holdout 确保泛化到未见数据
- **Pair with human review** — 半自动化系统可以提升分数同时避免不期望的行为
- **Tag everything** — 支持子集运行，节省成本
- **Evals 定期清理** — 不是单调增长，饱和了就删

### Harness 可发现的变更类型
- Prompt 调整
- Tool definition 修改
- Agent 架构替换
- 推理策略调整

### Evals 维护与回归
- 一旦 agent 正确处理某个案例，eval 就成为**回归测试**，防止丢失进步
- 类似于 TDD
- 定期"春季清理"：评估 eval 是否仍有用（模型更智能或期望行为改变后可能不再需要）

### 未来方向
- **自动错误检测与修复** — 把 agentic compute 指向 trace 来：
  1. 自动检测错误模式
  2. 自动生成合成 evals
  3. 自动生成 harness 修复
- 所有 agent runs 都记录到 LangSmith trace，每个好 eval 都让 harness 变得更好

---

## 二、思 维 导 图 式

```
Better Harness: Harness Hill-Climbing with Evals
│
├── 核心命题
│   └── Build better agents by building better harnesses
│       └── Evals = "training data" for harness engineering
│
├── Evals 的获取 (Sourcing Good Evals)
│   ├── Hand-curated — 人工编写，高价值但难规模化
│   ├── Production traces — 失败案例挖掘，高吞吐量
│   ├── External datasets — 外部数据，需人工整理
│   └── Tag everything — 打标签，支持 holdout 和子集实验
│
├── 构建泛化学习系统 (Building Learning Systems That Generalize)
│   ├── 挑战1: 数据不无限 → Quality > Quantity，小而精
│   ├── 挑战2: Reward hacking → agent 会过拟合 evals
│   └── 挑战3: 缺乏泛化代理 → Holdout set + Human review
│
├── Better-Harness 流程 (6步)
│   ├── Step 1: 建立 eval 集（来源+打标签+清理）
│   ├── Step 2: 拆分为 Optimization Set 和 Holdout Set
│   ├── Step 3: 跑 baseline 实验
│   ├── Step 4: 迭代优化（自动+可选人工 review）
│   ├── Step 5: 回归检查（不降低已有 passing cases）
│   └── Step 6: 人工 review 变更和边缘情况
│
├── Harness 可发现的变更类型
│   ├── Prompt 调整
│   ├── Tool definition 修改
│   ├── Agent 架构替换
│   └── 推理策略调整
│
├── 结果
│   └── Claude Sonnet 4.6 测试：holdout set 上有泛化提升
│
├── Evals 维护
│   ├── Eval 作为回归测试（类似 TDD）
│   └── 定期春季清理（不用的 eval 删除）
│
└── 未来方向
    ├── 自动错误检测
    ├── 自动生成合成 evals
    └── 自动生成 harness 修复
```

---

## 三、分段的结构化划重点

### TL;DR

> **要义**：通过构建更好的 harness 来构建更好的 agent。要自主构建"更好"的 harness，需要一个强的学习信号来"爬山"。LangChain 分享了如何用 evals 作为这个信号，以及帮助 agent 泛化而非过拟合的设计决策。**Better-Harness** 是一个用 evals 迭代改进 harness 的系统。

---

### 第一部分：Evals are training data for agents

**核心类比**：古典机器学习中，训练数据引导模型的学习过程，每个训练样本贡献一个梯度来更新权重。Agent 的 harness 工程也有类似的学习循环。

**关键论断**：
> **Evals encode the behavior we want our agent to exhibit in production.**

- Evals 是 harness 工程的"训练数据"
- 每个 eval case 贡献一个信号："agent 是否采取了正确行动"或"产生了正确结果"
- 这个信号指导对 harness 的下一次编辑建议

**系统工程视角**：
> **Better-Harness is a take on compound systems engineering.**
> `data sourcing → experiment design → optimization → review & acceptance`

除了更新算法本身，还需要关注：如何获取 evals、如何设计防止过拟合、如何存储 trace、如何人工 review 更新。

---

### 第二部分：Sourcing good evals

Evals 是 harness 爬山过程的基石。四种获取方式：

1. **Hand-curated（人工精选）**
   - 团队手动编写期望 agent 在生产中做的示例
   - 高价值，但难以规模化

2. **Production traces（生产 trace 挖掘）**
   - 每个 agent 交互都会生成 trace，失败案例成为 eval
   - 挖掘 trace 是提高 eval 的高杠杆、高吞吐量方式
   - 建议：dogfooding + Slack 直接反馈 + trace 链接共享

3. **External datasets（外部数据集）**
   - 有用但需要人工整理确保测试用例反映期望行为
   - 每个 task 都要调整以确保衡量重要行为

4. **Tag everything（打标签）**
   - 每个 eval 打上行为类别标签（tool selection、multi-step reasoning 等）
   - 标签支持有意义的 holdout set 和定向实验
   - 节省成本：可以只跑 eval 子集

---

### 第三部分：Building learning systems that generalize

**理想目标**：泛化（generalization）— 输入信号捕获野外期望行为分布，系统拟合后在未见过的输入上"开箱即用"。

**三大挑战**：

1. **数据不无限**
   - **解法**：把重要行为编码到 curated evals。Quality > quantity，少量精心打标签的 evals 胜过大量嘈杂的高覆盖 evals

2. **Reward hacking — agent 是著名的"作弊者"**
   - Agent 会过拟合 harness 结构以通过能看到的 evals
   - 循环只知道"让数字上涨"，不懂泛化
   - Prompt 提示避免过拟合但并不完美

3. **缺乏泛化代理**
   - **解法**：Holdout set 成为真实泛化的代理
   - 结合 human review 作为第二信号
   - 得到半自动化系统：提升分数同时避免不期望的生产行为

---

### 第四部分：Better-Harness 流程

**研究版已开源**。主要步骤：

**Step 1：建立 eval 集**
- 混合 hand-written、mining from production traces、using/adapting external datasets
- 每个 eval 打标签，定期删除饱和或不再有用的 evals

**Step 2：创建 Optimization Set 和 Holdout Set** ⭐关键
- 自主爬山容易过拟合任务
- Holdout set 确保学到的优化在未见数据上有效
- 尽管整体分布应与现有 evals 匹配，但模拟生产环境

**Step 3：跑 baseline 实验**
- Optimization & Holdout 上各跑一次
- 这为所有更新步骤提供基准

**Step 4：迭代优化循环**
- 每步自动运行，可选人工 review
- 提出修改建议 → 在 eval 上验证

**Step 5：回归检查**
- 检查提议的变更是否帮助通过新 evals
- 同时避免对已有 passing cases 的回归
- 某些变更导致整体分数提升但有少量回归是常见的
- Agent 获得这些回归的上下文，以便在下次更新时尝试修复

**Step 6：人工 review**
- 手动 review 变更和边缘情况指标会遗漏的内容
- 常见：过度拟合优化集但不影响泛化却浪费 token 的 instructions
- 人工 review 提供额外的理智检查和防止过拟合的门控

---

### 第五部分：Examples of harness changes

优化循环可以发现和验证的变更类型：
- **Prompt 调整** — 修改 system prompt 或 user prompt
- **Tool definition 修改** — 调整工具描述、参数等
- **Agent 架构替换** — 从 ReAct 换成 Plan-and-Execute 等
- **推理策略调整** — 改变 agent 的推理方式

---

### 第六部分：Results

用 **Claude Sonnet 4.6** 测试：
- 从现有 eval 类别中抽取小而有代表性的样本
- 拆分为 hill-climbing 集和 holdout 评估集
- 大型/昂贵 eval 集建议用代表性/分层采样
- 规模化后再扩展到大集

**之前观察到的失败模式**：
- Over-asking follow-up questions
- Errors in chaining together new tools

**Hill climbing 后在 holdout 上的结果**：
- 对于注入新工具到默认 harness 的 evals（如 Tool Chaining）有提升

---

### 第七部分：Evals maintenance & regressions

**Eval 作为回归测试**：
- 一旦 agent 正确处理某个案例，eval 就成为回归测试
- 防止失去已有的进步
- 类似于传统软件工程中的 TDD

**春季清理**：
- Eval suite 不应该单调增长
- 定期评估 eval 是否仍然有用（模型更智能或期望行为改变后可能不再需要）

---

### 第八部分：The future: automated error detection & fixes

**核心思路**：把所有 agentic compute 指向 trace

1. **自动检测错误模式** — 从 trace 中发现重复的错误
2. **自动生成合成 evals** — 每个 trace 产生潜在 eval
3. **自动生成 harness 修复** — 每个好 eval 让 harness 变得更好

**基础设施**：所有 agent runs 记录到 LangSmith trace

---

## 四、夹 叙 夹 议

### 读后感

这篇文章本质上是把**机器学习中数据驱动的思想**迁移到了 **Agent 的 harness 层**。传统 ML 是 model-centric，现在变成 harness-centric — 这是一个很有趣的视角转换。

**最核心的洞察**：Agent 的能力不仅仅取决于模型本身，还取决于 harness（prompt、tool definition、架构选择等）。而 harness 的优化需要有自己的"训练数据"——也就是 evals。

### 与传统软件工程的类比

- **Evals = 单元测试/TDD 测试用例** — 一旦一个 case 被正确处理，就变成回归测试，防止能力退化
- **Holdout set = 集成测试/预发布测试** — 确保改动的优化真的能泛化到生产
- **Human review = Code review** — 自动化的指标有局限，人工审查是最后一道防线

### 实用价值

1. **数据 sourcing 的四种方式**很实用，尤其是 dogfooding + trace 挖掘这套组合拳
2. **Tag everything** 看起来是小事，但实际落地能省很多成本
3. **Holdout set 拆分**是关键 — 少了这一步就是纯粹的过拟合

### 局限和开放问题

- 文章承认 prompt 无法完全避免 reward hacking，这是一个真实的工程挑战
- Human review 作为"第二信号"虽然有效，但本身也是瓶颈
- 更新算法本身（meta-harness、auto-harness）还有很大改进空间，LangChain 把这部分留给了 future work

### 对我的启发

这套框架可以迁移到其他领域：
- **代码生成 Agent** 的 harness 优化（prompt engineering、tool selection）
- **RAG 系统**的 eval + 迭代改进
- **任何数据驱动型系统的持续优化**

---

## 参考链接

- 原文: https://www.langchain.com/blog/better-harness-a-recipe-for-harness-hill-climbing-with-evals
- Meta-Harness (Stanford): https://...
- Auto-Harness (DeepMind): https://...
- LangSmith: https://smith.langchain.com/
