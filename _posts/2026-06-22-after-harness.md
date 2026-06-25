---
layout: post
title: Harness之后
date: 2026-06-06
---

pre-training 获得的通用学习能力，能不能以比 pre-training / post-training 更高的效率，从 agentic trajectory 里提炼出具有一定泛化性的规律。

![kupka]({{ site.baseurl }}/assets/images/ChatGPT-kupka.png )
<p align="center"><em>After harness, the missing half</em></p>

## 1. Loop 本来就是基本编程范式，那 agentic loop 新在哪里？

Loop 本身不新。观察、计划、行动、反馈、重试，这些都是基本控制结构。把它叫成 [loop engineering](https://addyosmani.com/blog/loop-engineering/) 有提醒作用，但如果只是说“agent 应该有 loop”，那基本等于说“程序应该会 while”。

关键在于：**loop 跑完之后，系统有没有留下东西？**

如果一次 agentic 执行只是完成任务，然后状态清空，下一次从头再来，那它本质上还是一个复杂 workflow。看起来很 agentic，实际是“一次性聪明”。

[ReAct](https://arxiv.org/abs/2210.03629) 把 reasoning trace 和 action 交织起来，让模型边想边做，解决了“只推理不行动”或“只行动不解释”的问题。

[Reflexion](https://arxiv.org/abs/2303.11366) 往前走了一步：它不更新模型参数，而是把任务反馈转成 verbal reflection，放进 episodic memory，影响下一次尝试。

这两个工作放在一起看，已经把问题从“agent 如何行动”推进到了“agent 如何从行动结果中调整后续行为”。

**未来的 agentic 应用，不会止步于 loop，而会围绕执行后的能力沉淀展开。** Loop 是执行结构，不是学习机制。没有沉淀的 loop，只是更贵的 retry。

## 2. Memory 只是记录过去，为什么还不等于 learning？

通用 Agent 的 memory 系统可以记录过去的成功案例、失败案例、用户偏好、项目背景，然后在下一次任务中检索出来。看上去像是实现了“从过去经验中学习”。但记得多，不意味着能力强。

哪怕在 pre-training 阶段，模型也要跨过从 memorization 到 generalization 的门槛。把样本记住，和从样本中学到可迁移规律，是两件事。Agentic memory 也是一样。

如果 memory 只是“过去发生了什么”的记录，它只是材料库。真正缺的是：系统能不能基于模型已有的智能，用比 pre-training 和 post-training 更高的效率，从 agent trajectory 里提炼出有一定泛化能力的 heuristic、policy，甚至局部 world model。

[Generative Agents](https://arxiv.org/abs/2304.03442) 把 observation、memory、reflection、planning 组织成一个可运行的 agent architecture。它说明 memory 不只是日志，而可以被 reflection 抽象成更高层的行为依据。

[MemGPT](https://arxiv.org/abs/2310.08560) 则把问题放在 memory hierarchy 和 context management 上：模型需要在有限 context window 内管理长期记忆。它解决的是“怎么让模型用得上更长的上下文”，但这仍然不自动等于 learning。

[Engram](https://engram.com/) 的方向更激进：不只是每次让模型重新读取 workspace，而是让模型从公司或个人上下文中学习，把上下文内化进模型本身。它把问题从 retrieval 推到了 training / internalization。

**Agentic learning 不是存更多 memory，而是把 memory 转化成可执行、可复用、可更新的过程性知识。**

更准确的链条是：

```text
episode memory → pattern extraction → heuristic / policy → reusable procedure
```

Memory 是原料，不是成品。没有抽象，memory 很快就会变成上下文垃圾场。

## 3. Skill 远不止是 prompt

如果 skill 只是一个更长、更复杂的 prompt，那它和 prompt optimization 没有本质区别，只是 prompt 穿了件马甲。

Skill 真正有意义的地方在于：它可以成为一种稳定的 procedural state。它不只是写“你应该怎么回答”，还可以包含任务流程、工具使用方式、领域假设、失败案例、边界条件、schema、测试样例，甚至小脚本。

未来需要进化的不只是 prompt，而是所有影响 agent 行为的 operational artifact：

```text
prompt
skill document
tool policy
workflow
routing rule
schema
script
test case
permission rule
retrieval policy
adapter
```

[Voyager](https://arxiv.org/abs/2305.16291) 是一个很早但很关键的例子。它在 Minecraft 里持续探索，把成功行为保存成 executable skill library。这里的 skill 不是一句提示词，而是可调用、可组合、可迁移的代码能力。

[GEPA](https://arxiv.org/abs/2507.19457) 优化的是 prompt / text artifact：它从 trajectory 中反思失败原因，修改 prompt，并用 Pareto frontier 合并不同尝试中的互补经验。[SkillOpt](https://arxiv.org/abs/2605.23904) 的本质区别其实不大；就通用性而言，甚至未必强于 GEPA。

[AlphaEvolve](https://arxiv.org/abs/2506.13131) 则把 code 当成 evolution 对象。它不是优化自然语言，而是让模型直接修改算法代码，再用 evaluator 筛选更好的版本。

未来真正重要的问题是：**系统从 trajectory 里提炼出的 know-how，应该落在哪种 asset 里？**

有些经验适合写成自然语言 skill。
有些适合变成 tool routing rule。
有些适合变成 workflow。
有些适合变成 schema 或 test。
有些稳定到一定程度后，才适合压进 [LoRA](https://arxiv.org/abs/2106.09685) / adapter / 参数。

所以 agentic evolution 的关键，不是 prompt evolution、skill optimization、code evolution 谁更高级，而是 asset selection 以及对应的更新方法：会不会灾难性遗忘，冲突怎么办，是不是要版本管理等。

```text
know-how → choose carrier → update asset → validate → reuse
```

## 4. LoRA / adapter 能固化能力，为什么还需要显式 skill？

把能力写进参数当然有吸引力。它紧凑、便宜，推理时简单，也更像传统意义上的“模型学会了”。

从这个角度看，[LoRA](https://arxiv.org/abs/2106.09685) / adapter 也应该被看作 skill asset 的一种可能形态。

显式 skill 和 latent skill 适合不同阶段。显式 skill 像操作手册：可读、可审查、可局部修改、可回滚。Latent skill 像压缩后的习惯：便宜、高频、推理时负担小，但不透明。这不是谁替代谁的问题，而是生命周期问题。

[LoRA](https://arxiv.org/abs/2106.09685) 的基本思路是冻结 base model，只训练低秩适配参数，用更少的训练参数完成下游适配。

[LatentSkill](https://arxiv.org/abs/2606.06087) 把 textual skill 转成 plug-and-play LoRA adapter，让 skill 知识从 context space 进入 weight space，减少每次注入 skill text 的 token 成本。[Skill-to-LoRA](https://arxiv.org/abs/2606.16769) 的思路也类似：不是压缩 SKILL.md 文档本身，而是学习这个 skill text 会诱导出的行为变化，用 skill-specific LoRA 在运行时激活对应行为。

**短期内，agentic 系统的主要学习形态更可能是显式的 skill / policy / workflow 更新；长期稳定下来的部分，才适合被压缩进 adapter 或参数。**

更合理的路径是：

```text
trajectory（目前最费人工的应该在高质量 trajectory 的选择上）
→ explicit skill / policy / workflow
→ validation over time
→ distillation into adapter / SFT / RL update
→ keep explicit spec + tests for governance
```

## 5. 从个人 copilot 到 agent 加持的超级团队

个人 copilot 的价值已经比较清楚：帮一个人写、查、总结、执行、规划。

但下一阶段的问题，未必只是“个人 assistant 更聪明”。更大的变化可能是：人和 agent 共同组成新的工作系统，甚至新的团队形态。

新的瓶颈会从“产出不够”变成“负责不过来”。人 + agent 能输出的量，可能逐渐超过个人能理解、能验证、能负责的量。

软件工程里已经有苗头：代码生成更快了，但很多程序员开始不再完整阅读自己生成或合并的代码。代码多了，责任边界、验证成本、系统理解能力没有同步扩张。

所以关键问题不是“怎么生成更多”，而是：**怎么以最低成本实现可靠负责？**

对外部交付，不能只靠信任，必须靠验证。对内部协作，也不能每一步都重新验证，否则 scale 不起来。系统需要一种介于“盲目信任”和“全量复查”之间的机制。

组织学里有一个相关概念叫 [transactive memory](https://en.wikipedia.org/wiki/Transactive_memory)：团队不需要每个人知道所有事，但需要知道“谁知道什么”“什么知识在哪里”“什么时候该调用谁”。放到 agentic team 里，这个问题会被放大。

[LLM multi-agent](https://arxiv.org/abs/2501.06322) 研究也已经开始讨论 role、coordination protocol、centralized / decentralized structure、shared memory 等问题。[DecentMem](https://arxiv.org/abs/2605.22721) 这类工作进一步把 self-evolving multi-agent system 和 decentralized memory 放在一起，说明 memory topology 本身会影响团队效率、通信成本和 agent 多样性。

**未来 agentic 应用的高阶形态，不是每个人拥有一个更聪明的 assistant，而是人和 agent 共同组成可以持续积累经验的工作系统。**

多 agent 不是多几个 bot 拉群聊。低效的大规模，只会加速破产。真正难的是组织问题：

```text
task decomposition
role assignment
context sharing
verification
permission boundary
responsibility boundary
collective memory
experience consolidation
```

从个体经验到团队经验，从 agent memory 到 organization memory，这里会出现新的生产关系。

大概率不是套用现成大公司组织模板，而是根据任务图、权限边界、风险等级、上下文依赖和历史表现，动态生成组织结构。

也就是：

**生成式的组织关系，动态演进的生产关系。**

## 6. 真正的分水岭是什么？

**pre-training 获得的通用学习能力，能不能以比 pre-training / post-training 更高的效率，从 agentic trajectory 里提炼出具有一定泛化性的规律。**


![kupka]({{ site.baseurl }}/assets/images/ChatGPT-the-missing-half.png )
<p align="center"><em>After harness, the missing half</em></p>

模型不只是临时读取上下文，而是能从自己参与过的工作世界里抽象出 procedure、policy、skill 和 workflow，并写回系统。

### 先例

[ReAct](https://arxiv.org/abs/2210.03629)、[Reflexion](https://arxiv.org/abs/2303.11366)、[Generative Agents](https://arxiv.org/abs/2304.03442)、[Voyager](https://arxiv.org/abs/2305.16291)、[GEPA](https://arxiv.org/abs/2507.19457)、[SkillOpt](https://arxiv.org/abs/2605.23904)、[AlphaEvolve](https://arxiv.org/abs/2506.13131)、[LatentSkill](https://arxiv.org/abs/2606.06087)、[Skill-to-LoRA](https://arxiv.org/abs/2606.16769)、[Engram](https://engram.com/)，其实都在碰同一个大问题的不同切面，差别只在于，最后沉淀到哪里。

[ReAct](https://arxiv.org/abs/2210.03629) 沉淀在执行轨迹结构里。
[Reflexion](https://arxiv.org/abs/2303.11366) 沉淀在 verbal memory 里。
[Generative Agents](https://arxiv.org/abs/2304.03442) 沉淀在 memory + reflection + planning architecture 里。
[Voyager](https://arxiv.org/abs/2305.16291) 沉淀在 executable skill library 里。
[GEPA](https://arxiv.org/abs/2507.19457) 沉淀在 prompt artifact 里。
[SkillOpt](https://arxiv.org/abs/2605.23904) 沉淀在 skill document 里。
[AlphaEvolve](https://arxiv.org/abs/2506.13131) 沉淀在 code artifact 里。
[LatentSkill](https://arxiv.org/abs/2606.06087) / [Skill-to-LoRA](https://arxiv.org/abs/2606.16769) 尝试沉淀在 adapter 里。
[Engram](https://engram.com/) 试图直接把 workspace context 内化进模型。

未来 agentic 应用的主线，不是 agent 替人完成更多任务，而是 agentic system 在完成任务的过程中，持续形成自己的可复用能力。

更完整的链条是：

```text
trace
→ reflection
→ pattern extraction
→ asset selection
→ skill / policy / workflow / adapter update
→ evaluation
→ reuse
```
