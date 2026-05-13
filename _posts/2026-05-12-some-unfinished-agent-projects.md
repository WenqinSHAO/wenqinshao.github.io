---
layout: post
title: 几个没写完的 Agent 小项目
date: 2026-05-12
---

最近断断续续折腾了些工作外的小项目，现在回头看出发点现在看都还挺有意思，但都实现得都不怎么样，用法语说是没有aboutir，没能落实到底，半途而废了。我这几个项目，大概都围绕一个问题：**今天到底什么样的 Agent 应用还值得自己做？**

![om]({{ site.baseurl }}/assets/images/xia_cao_xin.png)
<p align="center"><em>咸吃萝卜淡操心</em></p>


通用 Agent 已经越来越强了，Claude Code、Codex、Manus、Workbuddy 这类东西都在往“什么都能干一点”的方向走。那还有没有必要自己写一套 agentic flow？如果要写，写的到底是一个独立的应用，还是一个 skill，还是只是某种上下文和数据的沉淀？再往后看，token 越来越便宜又越来越贵，便宜是模型调用单价一直在卷，贵是 agent 真正跑起来以后 token 消耗很吓人。那 MaaS 到底靠什么赚钱？AI Cloud 最后是不是都要往企业现场交付和解决方案走？

这些问题都还没有想太明白，简单分享一下上手后的感谢吧。

## 师爷要有记性

第一个项目叫 [Shiye](https://github.com/WenqinSHAO/shiye/tree/lifelong-summary)，师爷。想做一个自己的 personal AI assistant，或者说一个外脑。最早的动机很简单：我觉得真正有价值的不是 agent 软件本身，而是**我自己的使用数据、笔记、历史对话、浏览痕迹、临时想法**。软件可以换，模型可以换，但这些东西如果能长期积累下来，应该会越来越值钱，至少对自己来说。

当时还有一个很朴素的不满：__为什么每次使用 AI 都要由用户自己决定要不要开新 session？__ context 快满了怎么办？之前说过什么要不要复制进来？这件事交给用户判断，总觉得很蠢。一个真正的个人助理，至少应该知道哪些上下文该延续，哪些该忘掉，哪些只是临时噪音。后来 Claude Code 的记忆机制有点接近这个方向。

Shiye 没有很好地做出来，尤其是主动提醒、个性化推送这些都太宽了。说“在恰当的时候主动给我建议”很容易，真要实现就发现什么叫恰当、什么叫打扰、什么叫建议，全都很难定义。最后比较有价值的反倒不是产品本身，而是它让我很早开始思考Agent memory和上手RAG。

memory 不只是“把文档塞进向量数据库，问的时候搜一下”。它还包括怎么捕获信息，怎么写入，怎么压缩，怎么更新，怎么遗忘，怎么处理冲突，怎么在模型需要的时候主动生成 query 去找相关信息。summary 也不只是摘要，它其实是在给 LLM 建一个更友好的 index。这个 index 可以按时间、人物、地点、topic、项目、任务状态来组织，也可以是更松散的、多 cue 的。**理想情况下，它甚至不应该是一个固定结构，而是随着使用不断长出来**。

这也是我后来关注 Zep、Graphiti 这类项目的原因。Zep/Graphiti 的核心点之一就是把 agent memory 做成 temporal knowledge graph，不只是静态地存文档，而是记录关系如何随时间变化。([Zep][2]) 这听起来很高级，但落到个人使用里，其实就是一个很日常的问题：我上个月觉得重要的东西，这个月还重要吗？我之前对某个人、某个项目、某个工具的判断变了没有？如果变了，旧信息是要删掉，还是作为历史保留？

## Shredder可以用通用电机

Shiye 之后，我的兴趣从“个人助理”转到了更具体的信息检索和研究流上。于是有了 [Shredder](https://github.com/WenqinSHAO/shredder/tree/agentic-search-clean-baseline)。

[Shredder](https://github.com/WenqinSHAO/shredder/tree/agentic-search-clean-baseline) 的起点是一个很具体的痛点。当时我要分析阿里近几年在 SIGCOMM、NSDI 上发表的论文，或者 Google 十几年基础设施研究的脉络。听起来像一个很适合交给 ChatGPT 或 Claude 的任务：你帮我找全文章，再总结一下。但实际用起来就发现，它们经常搜不全。不是总结能力不行，而是第一步搜索就会以漏，数据不全就会扭曲分析的结果。让它搜每年的 official accepted papers、program 页面，它能找一些，但很难保证全；你让它 filter 某家公司，也可能漏掉作者单位、项目名、并购前后的组织变化。于是我就想，能不能自己写一套 agentic search flow，至少**把搜索的完备性做得更可控一点**。

一开始野心很大。最好从搜文章，到下载 PDF，到结构化分析，到跨文章总结，到最后输出报告和 slides，全都一条龙。刚开始写 agentic workflow 的时候，总觉得只要把步骤拆清楚，让 agent 一步步干，就能自动化整个研究过程。真正写起来就发现，最费 token、最难搞的往往不是后面的总结，而是前面的搜索。query 怎么自动有化，搜索结果怎么去重，什么时候算搜够了，漏召怎么发现，错召怎么排除，provenance 怎么保存，每一步都可能把 flow 搞歪。

这时候我第一次强烈感觉到，agentic workflow 不只是 prompt engineering，而是系统工程。尤其是研究类任务，它最怕的不是答案写得不漂亮，而是遗漏不可见。一个模型总结十篇文章总结得再好，如果该有二十篇，那就是另一回事。

Shredder 写到后来也没有写完。倒不是这个方向完全错了，而是开发 agentic workflow 本身太像一个套娃：我让 coding agent 帮我写研究 agent，然后我要看 research agent 的 trajectory，再告诉 coding agent 哪些地方 overfit 了、哪些搜索策略不对、哪些地方过设计了。这个过程里我一直觉得，**trajectory 本身应该被结构化，应该能给出更细粒度的学习信号**，而不是最后只给一个“成了/没成”的 binary feedback。后来看到一些关于 agent trajectory、反思记忆、trajectory explorer 的工作，就觉得这个直觉并不奇怪。Agent 失败的时候，真正有价值的不是最后那句错答案，而是它到底在哪一步开始走歪。

再后来，Shredder 也就慢慢放下了。一个原因是 Workbuddy 这类工具出现以后，我可以薅试用期 credit，便宜到让我不太心疼 token。另一个原因是，我用 Workbuddy 写了 [net-search](https://github.com/WenqinSHAO/skill-net-search)，基本实现了 Shredder 最初最想要的那部分：有一个可积累的文献数据库，有 agentic search，先分析单篇文章，再跨文章总结架构、演进、关键组件、关键人物。

这时候我的想法又变了。原来我想做一个完整应用，后来发现真正值得沉淀的可能不是应用，而是 skill 和数据库。通用 Agent 已经够强了，它会读文件、改代码、跑命令、查资料。那我自己要补的东西，不一定是再写一个 Agent 外壳，而是给它一个更好的工作环境：稳定的数据结构，清楚的工作流说明，可复用的检索策略，能积累的中间产物。

## Agent + Skill的槽点

但 skill 这件事用起来也有不少槽点。

比如工具环境经常出问题。这个 skill 要 node，那个 skill 要 python，依赖版本反复配错。对于码农来说，大不了开个虚拟环境，写个 README，报错了自己修。但对非技术用户来说，这一步已经足够劝退。再比如我在 repo 里开发 skill，agent 有时候自作主张把它加载到全局 skill，结果 repo 里的版本、数据库和全局版本分叉了。再下一次调用，它甚至可能调错skill，是repo里的还是全局环境里的。还有一个更大的问题是，现在 skill 的门槛很低，阿猫阿狗都可以写，当然我也是阿猫阿狗之一。但企业级质量的 skill 还很少：那种装上去你知道它会稳定工作、不会乱动环境、不会乱读写文件、不会半路自由发挥的 skill，还不多，毕竟还有很大部分取决于Agent和底层大模型。

这也让我对“通用 Agent + skill”这个路线有点复杂的看法。它潜力很大，甚至我觉得对技术用户来说，Claude Code 加足够好的 skill，可能比很多专用 Agent 产品上限更高。Claude Code 本来就贴近代码库和本地开发环境，适合读项目、改文件、跑测试、用 CLI。 Manus 则更像一个面向普通用户的云端虚拟同事，它强调自己有 sandbox computer、联网、持久文件系统，可以安装软件、创建工具、在长任务里保留状态 （当然Claude也有cowork）。 两者不是简单谁强谁弱。Claude Code 更像一个很强的执行器，但前提是你愿意管理 repo、环境、权限和 artifact；Manus 更像把这些复杂性打包进产品体验里。

所以问题不是“通用 Agent 能不能做”，而是“通用 Agent 在什么边界里做”。workflow 越希望稳定遵从，就越要给它边界。专用 agentic flow 的价值，也许不在于能力比通用 Agent 强，而在于效率、可复查、可重放、可控成本和工作流遵从性。也就是说，你写的不是一个更聪明的 Agent，而是一个不那么容易乱跑的笼子。

## 搞抽象令人愉悦

这中间我还整理了一个比较好玩的 skill：[skill-abstract-art](https://github.com/WenqinSHAO/skill-abstract-art)。这个项目唯一算是做完了，原因也很简单：它没什么用，所以怎么样都可以算做完。

我当时想试一下，如果不让 Agent 调扩散模型，而是让它像人一样一笔一画地画画，它能画成什么样。直接写实肯定太难，那就降低难度：给它一些笔画、形状、纹理、透明度、层次，让它画抽象画。意境到了就行，不要求像。结果有些图还真挺有意思，但也暴露了一个问题：Agent 自己的画面想象力并没有那么好。你还是要给它一个大概的抽象方向。如果只让它自己想，它常常会忍不住往写实走，然后又写实不像。

这个项目对我没什么实际价值，但我还挺喜欢。Agent 应用不一定都要围绕生产效率，变现。我特别喜欢像[《史记知识库》](https://baojie.github.io/shiji-kb/index.html)这个项目，把《史记》整理成结构化知识图谱，做实体、事件、关系和索引，本质上也是一种 Agent/AI 辅助的信息组织项目，但它的吸引力不只是“省时间”，而是让人愿意进去逛。([AI驱动的历史知识图谱][2]) 有些东西就是因为好玩才值得做。[skill-abstract-art](https://github.com/WenqinSHAO/skill-abstract-art) 没探索出什么惊天结论，但我写的时候很开心，这也够了。

## 反刍

把这几个项目放在一起看，我觉得它们其实都被同一个趋势“吞掉”了一部分。

Shiye 不用继续做，是因为 Claude Code 这类通用 Agent 的记忆和 context compacting 已经部分接近 session-less 的体验。Shredder 不用继续做，是因为 net-search 加上 Workbuddy、Codex 或 Claude Code，基本已经能满足搜论文和分析论文的需求。net-search 也没太大必要继续折腾到端到端，因为最后写文章、做 PPT、画图，我还是更愿意回到 ChatGPT 的 web chat 里。图片生成、slides、文档排版这些事情，本地 skill 管起来很麻烦，加载哪个、不加载哪个、依赖在哪、输出放哪，都不如一个成熟产品顺手。

这听起来像是“自己做的东西被平台吃掉了”。平台吃掉的是外壳，不一定吃掉了过程里形成的判断。

比如 RAG。以前我会更自然地觉得，既然要做长期记忆，那就向量数据库、embedding、chunking、hybrid search 一套上去。现在会谨慎很多。chunking 真的是大坑，怎么切都可能破坏语义。切小了，上下文断；切大了，召回噪音多。有 chunk 就必然有语义割裂，那跨 chunk 的关系怎么补？靠 hierarchy summary 够不够？靠 graph 有多必要？这些问题不 hands-on 一下，很容易停留在“RAG 很有用”或者“RAG 都是垃圾”这种口号上。

我的体感是，小规模时 grep markdown 真挺好，尤其是写代码。多数 query 都很简单，我知道我要找的是函数名、配置项、错误信息、测试命令。这时候语义检索不一定比 grep 强，甚至可能添乱。但这不等于 semantic search 没用。上规模以后，尤其是处理笔记、邮件、浏览记录、论文、跨项目关系时，问题就变了。很多时候用户并不知道自己要搜什么，query 需要 Agent 主动生成，相关信息也不一定在同一个文件里。那时语义理解、query optimization、自动分类、动态调整类别、发现跨项目关系，就会变得很重要。

可以说，grep 解决的是“我知道我要找什么”。memory 想解决的是“我还不知道该从哪里想起”。
上次我们去那个，那个，哪儿，门口停车场老大了，你把车停在墙边，我们后来吃了一碗面才出来的，面汤头不错，那个到底是哪儿，你还记得嘛？

Agentic workflow 也是类似。自己写一套专用 flow，很多时候不是为了能力更强，而是为了让它更守规矩。通用 Agent 已经很能干了，但它的聪明有时候也是麻烦。你让它按照 skill 做，它做着做着就开始自由发挥；你让它保存中间结果，它可能换个格式；你让它先搜再分析，它可能分析着分析着又去搜；你让它不要过度设计，它可能特别热心地给你搭一个框架。这些都不是纯粹的 bug，甚至可以说是通用推理能力的副作用。它太像一个积极但不完全靠谱的实习生了。

## 咸吃萝卜淡操心

最后再说 token 经济学。因为写这些东西时最直观的感受就是：agent 真烧 token。

普通聊天还好，一旦进入 agentic workflow，token 消耗就不是一问一答了。它要搜索、读网页、读 PDF、整理中间结果、反思、重试、压缩上下文、调用工具、修自己的错。人看到的是一个任务，系统看到的是一大串轨迹。Workbuddy 试用期 credit 用起来很爽，也让我更清楚地意识到：很多 Agent 产品如果真按 token 原价卖，用户未必愿意买单；但如果按结果价值卖，供应商又要自己承担 token 成本。

这大概也是为什么现在 prompt caching、KV cache reuse 这么重要。OpenAI 的 prompt caching 文档就明确提到，系统会复用重复 prompt prefix 的缓存，从而降低重复上下文的计算开销。([OpenAI Developers][4]) 从商业上看，token 是成本单位，不是价值单位。用户不关心你烧了多少 token，他关心你有没有把论文搜全，有没有把报告写好，有没有帮他少开两小时会。MaaS 如果只是卖 token，很难长期卖出高溢价。模型能力再强，也会卷成本、卷延迟、卷稳定性。

**所以 AI Cloud 最后很可能不会满足于卖 LLM API、vector DB、memory substrate、agent framework**。这些中间层当然都能卖，但越往 agent 应用靠，越容易被上下游夹住。ISV 不一定想被绑定，客户也不一定愿意为中间件本身付高价。真正值钱的是端到端解决问题：从客户的垂直场景，到数据和权限，到 workflow，到模型和工具，到上线后的持续迭代。

这也是为什么 Forward Deployed Engineer 这个角色在 AI 时代又变得很有意思。Palantir 的 Forward Deployed AI Engineer 职位描述里，重点不是单纯写代码，而是直接和客户一起负责 GenAI strategy and implementation，把 end-to-end workflow 做到生产环境，再把现场经验反馈回产品。([Palantir FDE][5]) 这个模式很土，也很有效。AI 应用越往深处走，越不像 SaaS 自助开箱即用，越像带着工具和模型去现场打仗。因为只有现场才知道什么叫真的完成，什么叫可用，什么叫老板愿意付钱。

## 结个尾

回头看，我这几个小项目当然都不算成功。Shiye 没做成贴身师爷，Shredder 没 shred 完，net-search 写到够用就懒得继续，abstract-art 倒是做完了，但主要是因为它没什么用。Hands-on了就算还在场吧

[1]: https://code.claude.com/docs/en/memory?utm_source=chatgpt.com "How Claude remembers your project - Claude Code Docs"
[2]: https://github.com/getzep/zep?utm_source=chatgpt.com "GitHub - getzep/zep: Zep | Examples, Integrations, & More"
[3]: https://manus.im/docs/introduction/welcome?utm_source=chatgpt.com "Welcome - Manus Documentation"
[4]: https://developers.openai.com/api/docs/guides/prompt-caching?utm_source=chatgpt.com "Prompt caching | OpenAI API"
[5]: https://jobs.lever.co/palantir/636fc05c-d348-4a06-be51-597cb9e07488?utm_source=chatgpt.com "Palantir Technologies - Forward Deployed AI Engineer - Lever"
