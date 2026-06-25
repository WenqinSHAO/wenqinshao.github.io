---
layout: post
title: 化句成章，我的第一个Anthropic Skill
date: 2026-03-01
---

这是一篇关于「从零做第一个 Anthropic Skill」的实战复盘。  
我尝试把零散句子自动整理成可发布文章，并用同一份原始输入对比两条路径：`Prompt 直写` vs `Skill 流水线`。结论是：Skill 的流程价值很高，但工程和 token 成本也比预期更重。

## 第一部分：从想法到实现

### 1) 我为什么会做这个 Skill

我现在对 Anthropic Skills 的理解是：它更像“可复用的领域流程单元”，特别适用于相对复杂、同时又带有 ad-hoc 属性的自动化工作流。  
和 MCP 相比，Skill 可以同时封装 Prompt、脚本、MCP 工具，或者三者组合；和早期写死的 workflow 相比，它的组合成本更低、可演化性更高；和通用 Agent 只给 TODO 的模式相比，它能承载更明确的领域 SOP，让执行路径更可控。

### 2) 真实应用场景：洞察报告生成

我自己遇到的一个场景是写洞察报告。比如分析某 OTT 在一个研究方向上的近年论文脉络、合作网络和技术路线时，流程通常可以拆成：

1. 准备原始数据。
2. 对论文和作者做结构化抽取（可 Skill 化）。
3. 做主题归类、关系梳理和阶段总结（可 Skill 化）。
4. 基于中间结果生成可交付内容（如报告或胶片，可 Skill 化）。

这类任务的共性是：步骤多、循环多、上下文长，且中间结果要反复复用。  
如果只靠一次性 chat prompt，尤其在样本数较大（比如 50+ 篇文章）时，稳定性会明显下降。更合适的方式是让 Skill 负责高频子任务，让 Agent 做全局调度、进度跟踪和最终整合。

### 3) 这个最小化 Skill 是怎么做的

基于这个判断，我试着做了一个最小化 Skill，想体验一把自定义工作流的 baby step：把零散思考、短句甚至半句，整理成语言顺畅、逻辑连贯的文章。

成品：
<https://github.com/WenqinSHAO/skill-sentences-to-article/blob/main/SKILL.md>

实现方式非常克制：

1. 只用 vanilla Claude Code，不依赖其他 Skill。
2. 后端模型使用 DeepSeek API。

这个 Skill 不追求“文风炫技”，只做三件事：

1. 把碎片观点整理成逻辑链。
2. 在不改变原始立场的前提下补全表达。
3. 产出可直接发布的技术文章初稿。

我没有走“强类型参数 + 复杂 schema + 多级 validator”的路线，原因很直接：

1. 任务本质是写作与重写，不是结构化抽取。
2. 约束过强会让输出变僵，损失表达自然度。
3. 工程维护成本会快速上升，迭代速度反而下降。

所以主要精力放在 Prompt 工程：任务定义、边界约束、少量示例和风格锚点。

目标工作流程是：

1. Claude Code 读原始输入。
2. Claude Code 把内容喂给 LLM backend，决定哪些东西要搜索。
3. 使用 `SearXNG` 搜索内容。
4. Claude Code 改写原始输入，加入搜索信息。
5. 调用 Skill 整合。

### 4) 做完之后：价值与代价

实际做完后的结论是：这个 Skill 有价值，但比我预期更“重”，最终成品的质量也不如 ChatGPT。

体感主要有三点：

1. **工程体积偏大**：即便坚持最小化，落地复杂度仍高于“在聊天中直接改写”。
2. **Token 成本偏高**：累计消耗超过 12,000,000 tokens。
3. **现金成本可接受但不低**：仅开发简单测试这个 Skill 的DS token成本约 3 RMB。

## 第二部分：同一输入，两条路径对比

### 对比设置

我选了同一段输入，分别走了两条路径生成两篇文章：

1. **路径 A（Prompt 直写）**：ChatGPT 5.2 Extended Thinking，prompt：
```
输入是针对一个主题提炼出来的目标问题，核心思想，技术亮点，洞察建议等离散的点，请查阅相关文献，组织输出几段完成的文章。
采用总-分-总的一般结构。开篇总，主要介绍问题，设立阅读预期，结尾的总主要用于总结核心的洞见和对未来的建议（而不用顾及具体怎么实施的问题）。
```
2. **路径 B（Skill 流水线）**：Claude Code + DeepSeek API + SearXNG 检索，再进入 `sentences-to-article`。
```
read notes from ./notes.md, search with mcp `searxng` for additional info, fact check and reference citing, enrich the orignal notes, before passing it to skill `sentences-to-article` for a technical blog posts envisioned for Agentic application buiders and prodcut managers. the artical should be in chinese, technical serious writing, output to `./article_output/skill_with_search_chs.md`. 
```



<style>
.article-compare-wrap {
  border: 1px solid #ddd;
  border-radius: 10px;
  padding: 14px;
  margin: 20px 0;
  background: #fafafa;
}
.article-compare-top {
  font-size: 0.95rem;
  line-height: 1.6;
  margin-bottom: 12px;
}
.article-compare-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
}
.article-card {
  background: #fff;
  border: 1px solid #e6e6e6;
  border-radius: 8px;
  padding: 12px;
}
.article-card h3 {
  margin-top: 0;
}
.article-card blockquote {
  margin: 0.8em 0;
  padding-left: 0.8em;
  border-left: 3px solid #ccc;
  color: #444;
}
.raw-material-pre {
  margin: 0;
  padding: 12px;
  border: 1px solid #e6e6e6;
  border-radius: 8px;
  background: #fff;
  font-size: 0.92rem;
  line-height: 1.6;
  white-space: pre-wrap;
  max-height: 300px;
  overflow-y: auto;
}
.article-full-grid {
  display: grid;
  grid-template-columns: minmax(0, 1fr) minmax(0, 1fr);
  gap: 12px;
}
.article-full-pane {
  background: #fff;
  border: 1px solid #e6e6e6;
  border-radius: 8px;
  padding: 12px;
  min-width: 0;
}
.article-full-pane h3 {
  margin-top: 0;
}
.article-full-scroll {
  border: 1px solid #eee;
  border-radius: 6px;
  background: #fcfcfc;
  padding: 10px;
  max-height: 640px;
  overflow-y: auto;
}
.article-full-content {
  font-size: 0.9rem;
  line-height: 1.6;
}
.article-full-content > :first-child {
  margin-top: 0;
}
.article-full-content table {
  display: block;
  overflow-x: auto;
  max-width: 100%;
}
@media (max-width: 860px) {
  .article-compare-grid {
    grid-template-columns: 1fr;
  }
  .article-full-grid {
    grid-template-columns: 1fr;
  }
}
</style>

### 原始输入与双栏全文对照（限高可滚动）

<div class="article-compare-wrap">
  <div class="article-compare-top">
    <strong>原始输入（raw material）：</strong>下面这段就是两条 pipeline 共同处理的输入素材。  
    为了便于对照，我保留了原始“散句”风格。
  </div>
  <pre class="raw-material-pre">现在有不少文章说RAG已死
真正需要的不是RAG而是Agent记忆
根据OpenClaw等通用Agent项目，背后的context管理机制也没用用到RAG而是直接写到markdown，要搜索的时候直接grep查文件
Agent Coding平台也没有用RAG，而是用IDE通过Language Server Protocol结合大量grep构造编码context
但我觉得RAG背后的技术并没有死，而是technical marketing要求在技术名词上推陈出新
比如阿里的Sirchmunk项目号称是给RAG判了死刑，但是最基础的知识族构建还是要基于语义相似性，基于embedding计算余弦相似性
另外就是依托LLM做了大量语义上的处理，clustering，比传统RAG里的indexing或是graph构建更加消耗LLM token的消耗
本质是更复杂的中间结构设计，更LLM-based各种搜索整合，FTS+Dense+query optimization
RAG中的向量检索技术单独拿出来的确已经不胜任现代基于LLM的Agent应用构造
但是也没有像数据库一样的通用方案，因为Agent应用的context构造其实是有特定领域的业务逻辑的，不是通用的
比如coding agent的context构造方法和金融分析agent的contxt构造方法就有所不同，不同workspace的组织也会带来差异，比如一个关键文件法在哪里，有哪些关键文件
这些都会造成搜索的过程是不一样的，怎么优化query，去哪里搜，用什么工具搜，搜完之后怎么组合起来，这些都不是单纯向量检索可以回答的
怎么实现上述这些，有些要靠LLM自身年的同样的可泛化的能力，有些要靠领域知识的注入，没有唯一的答案
但这些里面多少都会有向量检索的参与，只是是很小一部分</pre>
</div>

{% capture article_a_md %}
当下“RAG 已死”的流行说法，更多是在给一种**过于简化的实现范式**写讣告：把知识库切块、做向量索引、Top-k 召回、整段塞进 prompt，然后期望模型稳定地产生可核验的结论。但 Agent 化之后，问题的重心从“如何召回相似文本”迁移到了“如何在任务过程中持续构造、更新、压缩与验证上下文”。RAG 作为一类“外部非参数记忆 + 生成”的系统思想并没有过时（它最初就被定义为把显式检索记忆与生成模型结合）。([arXiv][1]) 只是今天的 Agent 需要的，是更像**运行时（runtime）的 context 管理**：既要跨回合持久化，又要可搜索、可重放、可审计，还要对成本与时延负责。

于是你看到不少通用 Agent 项目把“记忆”落在最朴素的形态：**Markdown 文件 + 目录约定 + grep**。这并非“返祖”，而是工程理性——文件是可读、可 diff、可版本控制、可追责的源事实（source of truth），而搜索只是对源事实的访问路径之一。OpenClaw 的做法很典型：记忆以工作区里的 Markdown 为准，系统提供 `memory_get` 做定点读取，`memory_search` 做语义召回；并且明确把**混合检索（BM25 + 向量）**、MMR 重排、时间衰减等“检索工程”纳入默认能力，甚至还能切换到一个把 BM25、向量与 reranking 组合起来的本地侧车（QMD）。([OpenClaw][2]) 这其实已经不是“有没有 RAG”，而是“RAG 的粒度与边界在哪里”：检索不一定叫 RAG，但检索型记忆仍在系统里扮演关键角色。

同样的逻辑也出现在你提到的 Sirchmunk 这类“宣称给 RAG 判死刑”的项目上：它在叙事上强调“embedding-free / indexless”，但在关键的“知识族（cluster）复用”机制里，仍然把 **KnowledgeCluster 的 embedding** 作为字段，并用“把查询向量化、余弦相似度阈值命中则复用”的流程做加速与复用（文档里直接给出 cosine similarity ≥ 0.85 的复用门槛）。([GitHub][3]) 更有意思的是，它把检索的“底座”部分交给了 ripgrep / ripgrep-all 这类确定性的全文搜索工具，同时把“证据采样—归纳—结构化沉淀”交给 LLM 去完成，最终产物是可复用的结构化对象（KnowledgeCluster）。([GitHub][3]) 这恰好印证了你的判断：很多“RAG 已死”更像是技术营销的换皮——死的是“只靠向量 Top-k 的幼稚链路”，不是“语义相似性与检索增强”的基本技术路线。

在 Coding Agent 场景里，这个分化更明显：代码上下文构造的主力往往不是向量库，而是 **IDE 级语义索引与符号级检索**——Language Server Protocol（LSP）提供“跳转定义、查引用、补全”等结构化能力，让 Agent 直接以“符号”和“语法/类型约束”来定位上下文；在此基础上再辅以 grep/全文检索做补充，是更符合代码这种强结构语料的方式。([Microsoft GitHub][4]) Serena 这类工具明确把“基于 LSP 的符号理解”作为核心，并强调其价值在于**用更少 token 找到更对的上下文**（省成本、提质量），这与“向量召回塞大段文本”的范式几乎是两套思路。([GitHub][5]) 但这并不意味着“检索消失”，而是检索被重新拆分为：符号检索（结构化）、全文检索（确定性）、语义检索（召回补洞）、以及最后的重排/证据组合（面向生成的可核验输入）。

从研究视角看，“Agent 记忆”也并不是对 RAG 的替代，而是把 RAG 提升为一个**分层、可调度的虚拟上下文系统**：MemGPT 明确借鉴操作系统的虚拟内存分层，把“有限上下文窗口”变成一个可管理的资源，通过在不同记忆层之间搬运信息来获得“近似无限上下文”的效果。([arXiv][6]) Generative Agents 则把“经历—反思—计划”做成闭环：完整记录经验，抽象出更高层的反思，再在行动前动态检索相关记忆来驱动行为。([arXiv][7]) 这些系统共同指向同一个事实：检索只是记忆系统的一个算子（operator），真正的难点在于**什么时候写、写成什么结构、什么时候读、读多少、如何验证**——也就是你说的“更复杂的中间结构设计 + LLM-based 的语义处理 + FTS/Dense/Query Optimization 的整合”。

因此，今天更准确的表述应当是：**“向量检索单打独斗已不胜任，但检索增强并未死亡，而是被纳入更大的 context engineering 体系。”**一方面，工程实践正在把“混合检索 + 重排”当作常规配置，并且在评测里能看到混合检索与 reranking 对指标的显著拉升，同时也暴露了计算成本的陡增（这与你提到的 token/推理开销一致）。([arXiv][8]) 另一方面，学术界也在把 RAG 重新表述为更复杂的决策与协作过程（例如把 RAG 视作多智能体协作决策问题的 CoRAG），说明“检索—生成”仍是活跃研究母题，只是形态更系统化。([arXiv][9])

最后回到你最关键的洞察：**不会出现“像数据库一样通用”的 Agent context 方案**。因为 context 的构造不是纯信息检索问题，而是“任务—工具—风险—成本”共同约束下的业务逻辑问题：Coding Agent 的最优上下文往往围绕符号依赖与变更面收敛；金融分析 Agent 则围绕口径一致性、时点、出处与证据链组织；不同 workspace 的目录约定、关键文件位置、以及团队写作规范都会改变“去哪里搜、怎么改写 query、如何拼装证据”的策略。未来更稳健的方向，不是继续争论“RAG 死没死”，而是把**中间对象（证据片段、约束、查询计划、可复算 IR/图结构）**做成一等公民：让检索、融合、归因、回放成为默认能力；同时承认并拥抱“领域差异”，把可泛化能力交给模型，把不可泛化的部分沉淀为可审计的结构与策略库。这样，向量检索会自然回到它应处的位置——不是主角，但始终是工具箱里不可或缺的一把刀。

[1]: https://arxiv.org/abs/2005.11401?utm_source=chatgpt.com "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks"
[2]: https://docs.openclaw.ai/concepts/memory "Memory - OpenClaw"
[3]: https://github.com/modelscope/sirchmunk/ "GitHub - modelscope/sirchmunk: ️ Sirchmunk:  Raw data to self-evolving intelligence, real-time."
[4]: https://microsoft.github.io/language-server-protocol/ "Official page for Language Server Protocol"
[5]: https://github.com/oraios/serena "GitHub - oraios/serena: A powerful coding agent toolkit providing semantic retrieval and editing capabilities (MCP server & other integrations)"
[6]: https://arxiv.org/abs/2310.08560?utm_source=chatgpt.com "MemGPT: Towards LLMs as Operating Systems"
[7]: https://arxiv.org/abs/2304.03442?utm_source=chatgpt.com "Generative Agents: Interactive Simulacra of Human Behavior"
[8]: https://arxiv.org/html/2506.22644?utm_source=chatgpt.com "Evaluating Hybrid Retrieval Augmented Generation using Dynamic Test ..."
[9]: https://arxiv.org/pdf/2602.18734?utm_source=chatgpt.com "Rethinking Retrieval-Augmented Generation as a Cooperative Decision ..."

{% endcapture %}

{% capture article_b_md %}
### RAG技术的演进与Agent记忆系统的崛起：从向量检索到上下文引擎

> author: Wenqin
> date: 2026-03-01

随着大模型技术的快速发展，关于"RAG已死"的讨论在技术社区中频繁出现。然而，深入分析2025-2026年的技术演进趋势，我们发现RAG技术并未消亡，而是正在经历深刻的转型——从独立的检索增强生成框架演变为智能体（Agent）生态体系中的关键子模块。本文将基于最新的技术实践案例，分析RAG技术的演进路径、Agent记忆系统的兴起、向量检索在新架构中的定位，以及领域特定context构造的挑战，为Agentic应用构建者和产品经理提供技术决策参考。

#### 从独立框架到生态子模块：RAG技术的演进轨迹

##### RAG技术的角色转变

检索增强生成（RAG）技术在2023-2024年间曾被视为解决大模型知识时效性和领域适应性的标准方案。然而，随着Agentic应用的兴起，传统的RAG架构面临新的挑战。根据InfoQ在2025年底的技术总结，RAG正从独立框架演变为智能体生态的关键子模块，其核心价值从"外部知识检索"转向"Agent上下文构造"。

这一转变的驱动力来自两方面：一方面，大模型上下文窗口的持续扩大（如支持数十万甚至数百万tokens的模型出现）使得部分检索需求可以通过长上下文直接满足；另一方面，Agent应用对个性化和记忆能力的强烈需求催生了更复杂的上下文管理系统。

##### Agentic RAG成为复杂场景标配

在医疗诊断、法律咨询、金融分析等复杂应用场景中，2025年的趋势显示Agentic RAG正在成为标配。这种新型RAG架构不仅支持静态文档的检索，更重要的是能够处理Agent在交互过程中产生的动态记忆数据。根据CSDN在2025年6月的分析，Agentic RAG具备动态记忆管理能力，支持长期记忆库的创建、更新、检索和遗忘机制，这是传统RAG所不具备的。

#### Agent记忆系统的兴起：以OpenClaw为代表的实践案例

##### 记忆与检索的技术分离

OpenClaw项目在2026年初的爆火（GitHub星标突破20万+）标志着Agent记忆系统进入了成熟应用阶段。腾讯云开发者社区的分析文章明确指出："AI Agent = 大模型(思考) + Memory(记忆) + RAG(知识) + MCP(手脚) + Skills(经验)"。这种划分方式清晰地表明了记忆系统在Agent架构中的独立地位。

OpenClaw采用的"工具驱动、按需检索"策略具有重要启发意义。与传统的自动注入记忆模式不同，OpenClaw通过`memory_search`（语义搜索）和`memory_get`（精确取回）两个工具实现记忆检索，将"何时需要记忆、需要什么记忆"的决策权交给Agent本身。Heyuan110.com的技术分析指出，这种设计带来两个直接好处：节省tokens消耗，以及更符合Agent的自主决策逻辑。

##### 混合检索策略与透明存储

OpenClaw的记忆系统采用混合检索策略：向量检索占70%权重，关键词检索占30%权重。这种设计既利用了语义相似性检索的灵活性，又保留了精确匹配的准确性。同时，所有记忆都以明文Markdown文件的形式存储，用户可以随时查看、修改和删除，确保了系统的透明度和可控性。

更重要的是，OpenClaw的上下文压缩机制体现了工程化思维：在压缩对话历史前，系统会先让Agent将重要内容写入记忆文件，然后执行压缩。这避免了重要信息的丢失，相当于"清理房间前先将贵重物品放入柜子"。

#### 向量检索在新架构中的定位：辅助手段而非核心机制

##### 向量检索的降级趋势

原始笔记中提到一个关键观点："RAG中的向量检索技术单独拿出来已经不胜任现代基于LLM的Agent应用构造。"这一判断得到了多个技术实践的支持。在OpenClaw的混合检索系统中，向量检索只是70%的权重组成部分；在Coding Agent平台中，IDE工具如Language Server Protocol（LSP）和grep命令构成了更主要的代码理解机制。

Medium的文章"The IDE Solved This Twenty Years Ago"指出，现代开发环境早就解决了代码检索的问题。当开发者悬停在函数上时，VS Code不是通过grep文件，而是询问语言服务器。这反映了专业领域应用需要领域特定工具而非通用向量检索。

##### 语义相似性计算的基础性作用

然而，向量检索技术并未完全失效。即使在号称"给RAG判死刑"的项目中，最基础的知识族构建仍然需要基于语义相似性，通过embedding计算余弦相似性。向量检索的核心价值已经从"主要检索机制"转变为"辅助语义理解工具"。

在实际应用中，向量检索通常与全文检索（FTS）、关键词检索、领域特定搜索工具结合使用，形成多层次的检索体系。这种转变反映了技术实用化的趋势：没有单一的完美解决方案，只有最适合具体场景的组合策略。

#### 领域特定的Context构造：没有银弹解决方案

##### Agent应用的多样性需求

技术分析揭示了一个重要现实：不存在像数据库一样通用的Agent context构造方案。不同领域的Agent应用具有完全不同的context构造需求：

1. **Coding Agent**：依赖Language Server Protocol（LSP）、grep、代码结构分析工具。Serena项目（GitHub: oraios/serena）为编码代理提供了类似IDE的工具套件，包括实现LSP的语言服务器。

2. **金融分析Agent**：依赖金融数据源、时间序列分析、领域特定知识库、实时市场数据接口。

3. **医疗诊断Agent**：需要医疗文献数据库、病历分析工具、医学影像处理能力。

这种差异的根本原因在于不同领域的知识表示、问题空间和决策逻辑存在本质区别。

##### Workspace组织的影响

即使是同一类型的Agent，不同workspace的组织方式也会导致context构造方法的差异。原始笔记中明确指出："不同workspace的组织会带来差异：关键文件在哪里、有哪些关键文件。这些都会影响搜索过程：怎么优化query、去哪里搜、用什么工具搜、搜完之后怎么组合。"

这种领域特定性要求Agent系统设计者深入了解目标应用场景，开发领域适应的工具链和检索策略。通用向量数据库无法满足这种精细化的需求。

#### 未来趋势：Context Engine的整合框架

##### 从碎片化到统一化

2025-2026年的技术演进显示，RAG、Agent Memory、MCP工具和服务接口等概念正在向统一的"Context Engine"框架收敛。知乎专栏文章"2025年RAG已死？2026年做Agentic和上下文工程？"提出了明确的观点："不要纠结用RAG还是Memory，统一到'Context Engine'的框架下思考。"

这种统一框架的核心思想是：所有Agent对数据的访问需求——无论是静态文档（通过RAG）、动态交互日志（通过记忆），还是结构化服务接口（通过MCP封装）——都应该由一个统一的上下文管理引擎来处理。

##### Context Engine的关键特征

基于现有技术实践，未来的Context Engine可能具备以下特征：

1. **统一的数据访问层**：为Agent提供一致的API来访问各种类型的数据源。

2. **智能的检索策略选择**：根据查询类型、数据特性和应用场景，自动选择最合适的检索方法（向量检索、全文检索、关键词匹配、API调用等）。

3. **可组合的结果整合**：能够将来自不同源的检索结果进行智能整合，消除冗余，补充缺失信息。

4. **动态的上下文管理**：支持实时更新和增量索引，能够根据交互进展调整上下文内容和结构。

5. **透明的可解释性**：所有检索过程和结果来源都可以追溯和解释，支持人工审计和干预。

#### 结论：构建适应性强、可扩展的Agent上下文系统

基于对RAG技术演进和Agent记忆系统实践的分析，我们提出以下可操作性建议：

1. **采用分层检索架构**：避免对单一检索技术的过度依赖，构建包含向量检索、全文检索、关键词匹配、领域特定工具的多层次检索体系。

2. **实施工具驱动的记忆管理**：参考OpenClaw的实践经验，将记忆检索设计为可调用的工具而非自动注入的过程，赋予Agent更多的自主决策权。

3. **强化领域特定能力建设**：深入了解目标应用场景的业务逻辑和数据特征，开发领域适应的工具链和context构造策略。

4. **向统一的Context Engine演进**：在系统架构层面规划上下文管理引擎的整合，为不同类型的Agent提供一致、高效的数据访问能力。

5. **保持系统的透明性和可控性**：确保记忆存储和检索过程对用户透明，支持人工干预和审计，避免形成不可控的"黑箱"系统。

RAG技术并未死亡，而是正在经历必要的演进和分化。向量检索从"核心机制"转变为"辅助工具"，Agent记忆系统从"附加功能"升级为"核心组件"，这一过程反映了AI Agent技术从理论探索走向实际应用的自然演进。对于Agentic应用构建者和产品经理而言，理解这一演进趋势，把握技术转型的关键节点，将有助于构建更加强大、灵活和实用的智能体系统。

**引用来源：**
1. 腾讯云开发者社区，"OpenClaw 的Agent记忆与 RAG 有何区别？"，2026-02-28
2. Heyuan110.com，"OpenClaw 记忆实施策略解析：工具驱动的 RAG 与'按需回忆'"，2026-01-31
3. InfoQ，"从RAG 到Context：2025 年RAG 技术年终总结"，2025-12-25
4. Medium，"The IDE Solved This Twenty Years Ago"
5. GitHub，oraios/serena项目文档
6. CSDN，"RAG已死？一文带你讲清2025年RAG的重点新趋势！"，2025-06-25
7. 知乎专栏，"2025年RAG已死？2026年做Agentic 和上下文工程？"
8. Elasticsearch官方博客，"使用Elasticsearch 管理智能体记忆"，2026-01-16
{% endcapture %}

<div class="article-compare-wrap">
  <div class="article-compare-top">
    <strong>双栏全文对照：</strong>左侧是 ChatGPT 5.2 Extended Thinking（Prompt 直写）原文，右侧是 Claude Code + DeepSeek API + SearXNG + Skill（流程化生成）原文。  
    两侧都设置了固定高度，直接滚动即可逐段比较。
  </div>
  <div class="article-full-grid">
    <section class="article-full-pane">
      <h3>文章 A 原文（ChatGPT 5.2 Prompt 直写）</h3>
      <div class="article-full-scroll">
        <div class="article-full-content">
          {{ article_a_md | markdownify }}
        </div>
      </div>
    </section>
    <section class="article-full-pane">
      <h3>文章 B 原文（Claude Code + DeepSeek API + SearXNG + Skill）</h3>
      <div class="article-full-scroll">
        <div class="article-full-content">
          {{ article_b_md | markdownify }}
        </div>
      </div>
    </section>
  </div>
</div>

## 第三部分：第三方评审（Gemini 3 / ChatGPT 5.2）

### Gemini 3 的评审（免费版）

>从（AI爱好者，AI从业者，Agentic App开发/产品经理）这类人的视角，同时作为专业编辑评审比较下面两篇关于统一主题文章的优劣，点评一下如果你是读者你会更喜欢那一篇为什么？

Thinking模式下输出（我自己也更喜欢A）：

你好！作为一名在 AI 圈子里摸爬滚打的专业编辑，同时也是一名关注 Agent 落地实践的开发者，我仔细研读了这两篇关于“RAG 演进与 Agent 记忆”的文章。

这两篇文章虽然共享了一套“底层逻辑”，但**气质和目标受众**完全不同。如果把文章比作产品，文章 A 是一份**深度技术内参**，而文章 B 则是一份**标准行业白皮书**。

以下是我的评审点评：

---

### ⚖️ 文章优劣对比分析

| 维度 | 文章 A（深度内参型） | 文章 B（行业综述型） |
| --- | --- | --- |
| **核心视角** | **工程理性与系统架构**。侧重于“为什么这么做”以及技术底层的拆解。 | **市场趋势与方法论**。侧重于“发生了什么”以及行业标准的总结。 |
| **语言风格** | **犀利、硬核、有极客感**。使用了大量圈内术语（LSP、sidecar、non-parametric memory）。 | **工整、规范、有结构感**。标准的公文/媒体报道体，易于快速扫描。 |
| **洞察深度** | **极高**。提出了“RAG 是运行时的 context 管理”这种极具启发性的抽象。 | **中等**。更多是整合已知信息，将其归纳为“Context Engine”框架。 |
| **可读性** | 需要读者有较强的背景知识，否则会觉得信息密度过载。 | 极佳。分级标题清晰，要点明确，非常适合非技术背景的 PM 阅读。 |
| **真实感** | 像是一位资深架构师在深夜复盘时的**手记**，充满实战思考。 | 像是一名资深分析师撰写的**结项报告**，引用考究（虽然是模拟的未来日期）。 |

---

### 📝 深度点评

#### 文章 A：给“懂行的人”看的解药

文章 A 最出彩的地方在于它**戳穿了技术营销的泡沫**。它敏锐地捕捉到了 Coding Agent 中 LSP（符号索引）与向量检索的区别，这说明作者真的写过代码或深度研究过 Agent 的工程痛点。

* **优点**：它不是在复述趋势，而是在重新定义问题（例如：RAG 是一种思想而非某种工具）。
* **缺点**：排版比较密集，缺乏视觉缓冲，对“小白”爱好者不太友好。

#### 文章 B：给“管事的人”看的指南

文章 B 的组织非常严密，非常适合作为团队内部的技术对齐文档。它把碎片化的信息（OpenClaw、Serena、MCP）整合进一个名为“Context Engine”的宏大叙事中，给出了清晰的 1-2-3 建议。

* **优点**：结构化思维极强，结论部分（5 条建议）具有极高的**实操参考价值**。
* **缺点**：稍显“AI 味”较重，观点表达比较四平八稳，缺乏文章 A 那种令人拍案叫绝的独到观察。

---

### 🏆 读者偏好：我会选哪一篇？

作为一名 **Agent 开发者/产品经理**，我会更倾向于 **文章 A**。

**为什么？**

1. **它提供了“认知差”**：文章 B 告诉我的大部分是“行业共识”，而文章 A 告诉我的是“工程细节”。例如，它提到 Sirchmunk 依然保留了 embedding 门槛值（0.85）来做复用，这种**细节拆解**对开发者做技术选型和优化非常有价值。
2. **它解释了“为什么”**：文章 A 解释了为什么 Markdown + grep 不是返祖，而是“工程理性”。这种从底层哲学出发的辩护，能帮助我在面对老板或客户质疑“为什么不用向量数据库”时，给出更有力的回答。
3. **它更有“生命力”**：文章 A 的文字带有明显的个人思考印记。在 AI 生成内容泛滥的时代，这种带有**专业偏见和实战汗水**的文章，才是资深读者愿意反复研读的。

### ChatGPT 5.2 的评审

5.2 Thinking模式，同样prompt：

下面我会用三类读者视角（AI 爱好者 / AI 从业者 / Agentic App 开发或产品经理）去读，同时以“专业编辑评审”的标准拆解两篇的优劣；最后回答“如果我是读者更喜欢哪一篇、为什么”。

---

下面我会用三类读者视角（AI 爱好者 / AI 从业者 / Agentic App 开发或产品经理）去读，同时以“专业编辑评审”的标准拆解两篇的优劣；最后回答“如果我是读者更喜欢哪一篇、为什么”。

---

## 先给结论

* **我更喜欢文章 A**：它更像“有主张的技术评论 + 证据链”，论证密、例子硬、反直觉点多，读完能形成更稳的认知框架（尤其适合你列的三类目标读者）。
* **文章 B 更像“结构化综述/公众号式技术解读”**：读起来更顺、更像一篇可发布的长文，但存在更明显的“信息来源层级不稳 + 细节可核验性不足 + 内容重复”的编辑问题，导致信任感和“含金量密度”弱于 A。

---

## 三类读者视角：分别会偏好什么？

### 1) AI 爱好者视角（追求“新观点/反转/洞见”）

**更偏好 A。**

* A 的核心叙事是“RAG 没死，死的是过于简化的范式；Agent 时代真正难的是 runtime context 管理”，这个切入更锋利，也更贴近社区争论的本质。
* A 用 Sirchmunk 这种“宣称 embedding-free”的案例做反证很有“拆穿营销叙事”的爽感，而且是可核验的：它明确写了 reuse 阶段用 **cosine similarity ≥ 0.85** 做语义复用，并维护 embedding 字段。([GitHub][1])
* B 的表达更“稳妥”，但亮点更少：大段是把常见共识重新分段描述（演进、崛起、定位、无银弹、未来趋势），对爱好者来说容易有“我知道你要说什么”的疲劳感。

### 2) AI 从业者视角（关心“可落地的工程取舍/成本-时延-可靠性”）

**仍更偏好 A，但会希望它更好读；B 的“行动建议”更像交付物。**

* A 的优势在于它把工程现实讲透：文件作为 source of truth、grep/FTS 的确定性、混合检索+重排+时间衰减、LSP 符号级检索等，都是“真正做过系统”的语气与颗粒度；并且它引用到 OpenClaw 记忆系统的混合检索、MMR、temporal decay 这些“可配置细节”，更贴近工程决策。([OpenClaw][2])
* B 的优势是“更像一份 briefing”：标题—摘要—分节—趋势—建议，读者可以快速抓重点；但它在若干关键数据/判断上，**要么引用层级偏二手，要么缺少可定位链接**。比如：

  * OpenClaw stars 用“20万+”是对的方向，但如果写成“截至某日 X 万”会更专业；目前 GitHub 页面显示 **Star 243k**（抓取时）。([GitHub][3])
  * “向量 70% / BM25 30%”这点其实在 OpenClaw 文档配置里能直接对齐（vectorWeight/textWeight），但 B 没把“来自官方配置”这层证据钉牢，读者只能“相信作者”。([OpenClaw][2])

### 3) Agentic App 开发 / 产品经理视角（关心“架构范式、边界、路线图、风险/合规/可审计”）

**A 更像“架构判断力”，B 更像“产品文档风格”。**

* A 在“为什么不会有通用 context 数据库”这段，对 PM/架构师很有价值：它把 context 构造解释为任务逻辑约束下的决策系统（工具、风险、成本），并提出“中间对象一等公民”（证据片段、查询计划、可复算 IR/图结构）这种更像产品路线的抽象。
* B 的“Context Engine 统一框架”对 PM 也友好，但它更像口号，需要更硬的边界定义：Context Engine 的最小闭环是什么？接口长什么样？谁负责评估/重排/归因？失败如何回退？这些在 A 里以“算子/调度/可审计”方式更接近可落地系统。

---

## 作为专业编辑：两篇各自的“优点 / 问题 / 怎么改”

### 文章 A

**优点**

1. **论证强**：不是“讲趋势”，而是在“拆范式 + 给替代框架”，并用多个项目/论文互相佐证。
2. **证据链更偏一手**：arXiv、项目 docs、GitHub readme 这种可核验引用占比高；Sirchmunk 的反例非常加分。([GitHub][1])
3. **概念密度高且不空**：runtime context 管理、分层记忆、符号检索 vs 语义检索的边界，都能落到具体机制。

**问题（编辑层面）**

1. **读者负担偏重**：长段落+高术语密度，缺少“路标”（小标题、图示、结论框）。
2. **论点层级可再清晰**：哪些是“事实陈述”，哪些是“作者判断”，可以用显式标记分开，提升可信度。
3. **信息过载**：例子很多，但读者可能记不住“到底想让我带走哪 3 条”。

**怎么改（最有效的 3 刀）**

* 开头加一个 **TL;DR（3 句）**：RAG 死的是 Top-k stuffing；Agent 时代是 context engineering；未来是中间对象+可审计 pipeline。
* 中段用 4 个小标题拆开：“文件即真相 / 混合检索+重排 / 代码场景的符号检索 / 研究视角的分层记忆”。
* 结尾把“不会有通用方案”的论证，改成 **“决策清单”**（领域约束、证据链要求、成本上限、回放审计、工具可用性）。

---

### 文章 B

**优点**

1. **结构友好**：标题、分节、要点、结论建议，很适合公众号/团队内部分享。
2. **对 PM 友好**：最后的“可操作建议 1-5”是天然的行动项。
3. **叙事更平滑**：读者不需要太强背景就能跟上。

**问题（编辑+可信度层面）**

1. **来源层级不稳**：大量引用来自资讯/社区二手文章（CSDN、知乎等），但关键判断（趋势、比例、爆火数据）没有在正文里做“可定位引用”。相较之下，A 的一手引用更让人放心。
2. **“新信息”比例偏低**：很多段落是把同一观点换句话说（RAG 转型、记忆崛起、向量降级、无银弹、统一框架），容易产生重复感。
3. **部分表述偏“营销式确定”**：例如“成为标配”“成熟应用阶段”等，建议改成带条件的工程语言（在什么场景/约束下、什么证据支持）。
4. **缺少反方压力测试**：比如“长上下文替代部分检索”的边界、Agent memory 的失败模式（污染、遗忘策略、对齐/安全）没有展开。

**怎么改（最有效的 3 刀）**

* 把“引用来源”改为 **正文脚注/行内引用**，并优先用一手材料：OpenClaw 官方 docs（可直接写出 vectorWeight/textWeight、MMR/decay 公式）、GitHub readme、Elastic 官方博客等。([OpenClaw][2])
* 删掉 15–25% 的重复段，换成 2 个“硬例子”：一个 coding agent（LSP+符号索引），一个知识密集领域（合规证据链+审计回放）。
* 给 “Context Engine” 一个更工程化的定义：输入/输出契约、检索策略选择器、证据融合与归因、缓存与回放、评测指标。

---

## 如果我是读者，我会更喜欢哪一篇？为什么？

**我会选文章 A。**理由很直接：

1. **更“有判断力”**：不是把材料铺开，而是把争论焦点从“RAG 死没死”迁移到“context engineering 怎么做”，这对真正要做 Agent 的人更值钱。
2. **证据更硬、反例更漂亮**：Sirchmunk 的“embedding-free 叙事 vs 实际 cosine reuse”这种拆解，会显著提升读者对作者的信任感。([GitHub][1])
3. **信息密度更高**：同样篇幅下，A 提供的“可复用 mental model”更多；B 的优势主要是“好读”，但在专业读者这里，通常会输给“更准、更硬、更可核验”。


[1]: https://github.com/modelscope/sirchmunk/ "GitHub - modelscope/sirchmunk: ️ Sirchmunk:  Raw data to self-evolving intelligence, real-time."
[2]: https://docs.openclaw.ai/concepts/memory "Memory - OpenClaw"
[3]: https://github.com/openclaw/openclaw "GitHub - openclaw/openclaw: Your own personal AI assistant. Any OS. Any Platform. The lobster way. "
