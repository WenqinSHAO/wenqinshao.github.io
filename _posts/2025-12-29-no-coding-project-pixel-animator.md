---
layout: post
title: 不写一行代码：我用 Coding Agent 写了一个像素动画小玩具
date: 2025-12-29
---


中年牛马的快乐就是这么简单：花不了几块钱，就可以角色扮演项目经理，PUA AI coding agent 连轴转两天。我动动小手指提需求，它埋头写代码，最后我收果子、验收、完成 KPI。

<div style="display: flex; justify-content: center; align-items: flex-start; gap: 20px; flex-wrap: wrap;">
  <div style="flex: 1; min-width: 300px; text-align: center;">
    <img src="{{ site.baseurl }}/assets/images/e-ink-draw.jpeg" alt="e-ink-draw" style="max-width: 100%; height: auto;">
    <p><em>支持墨水屏，手写笔</em></p>
  </div>
  <div style="flex: 1; min-width: 300px; text-align: center;">
    <img src="{{ site.baseurl }}/assets/images/montage-2025-12-26T15-49-22-009Z.gif" alt="montage" style="max-width: 100%; height: auto;">
    <p><em>高清动画样片</em></p>
  </div>
</div>

当然这只是玩笑。真实体验经常在这几种情绪之间来回横跳：
- “这也太厉害了吧，提 PR 比我码字还快。”
- “这个功能一下就实现对了，可以啊。”
- “诶？怎么又给我引入了个新 bug？”
- “这个 bug 怎么又回来了？”
- “这样改不行，我得给更具体的提示。”

提需求提到恨不得给 agent 打电话把它喊过来当面讲清楚。

言归正传。这两天 AI coding agent 的热度又上来了。面对 AI 辅助开发，连 Andrej Karpathy 都说自己在软件开发上有一种“要掉队”的感觉（原文：[https://x.com/karpathy/status/2004607146781278521](https://x.com/karpathy/status/2004607146781278521) Claude Code 的负责人 Boris Cherny 也公开提到（原文：[https://x.com/bcherny/status/2004887829252317325](https://x.com/bcherny/status/2004887829252317325)最近 30 天里，他合入了 259 个 PR、接近 40k 行新增/38k 行删除，而这些代码“都不是自己写的”，而是 Claude Code + Opus 完成的。同时，不少公众号也在写：大厂一边加码 AI 和基建，一边又在“优化程序员队伍”。

我其实算比较早接触 AI 辅助开发，也用它写过点小东西（比如这个博客网站）。但显然这几个月的 AI coding agent 已经不是“代码补完”那么简单了。于是我打算换个姿势：**不写一行代码**，做个小项目，看看自己能触及的AI辅助开发已经到了什么样子。

---

## 我做了个什么：pixel-animator

项目叫 `pixel-animator`：一个手绘像素 GIF 的小玩具，也能把多个 GIF/片段剪辑成视频。链接：[https://github.com/WenqinSHAO/pixel-animator](https://github.com/WenqinSHAO/pixel-animator)

我让 ChatGPT 查了一下，类似项目也有，比如：

- [https://github.com/HatAndBread/pixel-animator](https://github.com/HatAndBread/pixel-animator)
- [https://github.com/piskelapp/piskel](https://github.com/piskelapp/piskel)
  
都能画像素 GIF。我的这个项目更偏“像素小电影”：除了逐帧画（chunk editor），还做了一个 montage/剪辑模式，把多个片段拼起来再导出视频（至少我没看到第二家这么做的，欢迎打脸）。

另外我不太熟 web app 开发。以前写过 TypeScript + React，npm 这套把我折腾得挺烦。所以这个项目主打“民科开发模式”，刻意选一条不合常态、但更适合我动手的路：

- **单个 HTML 文件本地运行**：浏览器直接打开，不需要 npm，不依赖外部服务，离线可用。
- **触摸屏 / e-ink 手写笔适配**：主要在 PC + 鼠标上用，但我自己有个墨水屏设备，希望“有浏览器就能画”。适配肯定不可能一步到位（#8/#10 其实就一直在打这个补丁）。

同时，这类项目对 AI 辅助开发也挺“刁钻”的：我理想的交互很多很难靠测试用例讲清楚；而且 AI 也没有墨水屏设备，它很难知道在那个环境下到底 work 不 work。很多时候我做的事其实不是写代码，而是不停测试、写测试报告、再把问题描述回灌给它。

---

## AI 辅助开发的三种交互方式

体验里很关键的一件事，是 AI 和人到底怎么协作。以 GitHub Copilot 体系为例，我把交互分成三个层次（从微观到宏观）：

1. **代码补全（Inline completion）**
   Copilot 从这里起家，和 VS Code 的整合也不错。但我这次基本没用——原因很简单：几乎没有一行代码是我亲手敲出来的。

2. **IDE Chat**
   在 IDE 里对话，它能读代码、改代码、准备 commit，甚至能调用一些工具（比如终端/bash）。用起来像“交互式小助手”。其他一些 terminal 形态的工具虽然 UI 不一样，但交互感觉类似，我就先都归到这一类。

3. **Coding Agent**
   在 GitHub 网站上提交任务，它会自动提一个 PR。你在 PR 下面做 code review、提意见，它再继续改。整体更像“一个独立开发者”。

表面上都是“AI 帮我写代码”，但工作单位完全不同：

- 补全：单位是“一行/一小段代码”。
- IDE chat：单位是“一个明确的小修改”。
- coding agent：单位是“一个 PR 的交付物”。

这次让我觉得“变了”的，就是第 3 种。

---

## 为什么我觉得 coding agent 更强：不是更聪明，而是更像交付

同样花钱（premium request / session 那套），coding agent 给我的“成功率体感”更高。我现在倾向于认为，原因主要是两个。

### 1) 成功率的度量单位变了

- 用 IDE chat 时，我会看：**这次 chat 能不能把问题解决掉，最好顺便把文档和 commit 也准备好**。它更适合交互式的小问题，或者我已经知道大概在哪，怎么改的问题，到那里去找答案。对我来说就是节省“我自己动手改”的时间。

- 用 coding agent 时，我关注的是：**这个 PR 最终交付得怎么样**。
  每次 code review 都像重新触发一轮 agent 操作：读代码、查资料、动手改、跑验证、甚至给截图，还会把原因和修改方法写进 PR 说明。我不需要把每一步都塞进对话里，它会自己做很多“为交付服务”的动作。

IDE chat 更像我和实习生 peer coding——我方向大致对，只是懒得改，让它按我的思路把活干了；coding agent 更像我把一个 feature 交给团队——我用项目经理/产品经理的视角讲清楚我要什么，中间怎么实现我不管，最后我验收merge。至少理想状态是这样。

### 2) 可调用工具更丰富，验证半径更大

coding agent能调用的工具更多：linux 沙箱、bash、浏览器、截图……
这类 tool use 对前端尤其有用：很多 bug 不是语法问题，是视觉/交互问题。能截图，沟通成本直接下降一大截。
当然它也更贵：调用这些工具往往在 request 之外按时间额外计费。从这个角度看，coding agent 比 IDE chat 更“能花钱”所以也更能耐些。

---

## Context 依然是瓶颈

coding agent 好用的副作用是：我会忍不住往里塞更多需求。起初我的心态很直接：一次 request/一个 session 有成本，它只改两行我觉得亏。所以我会在一个 request 里写很多 todo，甚至想在一个 PR 里 follow 多个 request，看看它到底能做到什么程度。

比如在PR #8里，我要求实现 e-ink light theme、触控手势、自动 zoom-lock，还顺手把“已完成/已知问题”列成 checklist等。

东西一多，它的注意力问题就暴露得非常典型：

- **拆东墙补西墙**：改 A 引入 B，小问题到处漏。
- **在同一类问题上打转**：反复尝试相似解法，像在原地绕圈。
- **有疲惫感**：后面明显“思考变少”，中间过程记录也变少。
- **明知有错也说好了**：嘴上说 all fixed，实际上还有明显问题。

我对coding agent的“context”的体感是这样的：同一个 PR 里，跨request它确实会复用一部分上下文；但只要你在一个 PR 下继续叠 request，它脑子里那坨“项目地图”就会被你不断扰动。反过来，重新开一个 PR，它就几乎是 fresh start，要重新读项目、重新建立地图。

只要 context 还是有限的，“记忆 scope”又是交互模式里内置的一套 heuristic，那 AI 辅助开发的核心技巧就有：**边界管理**。这件事和人打交道很像——执行者的注意力是有限的，需要专注。管理者手上盘着十几件事、对接三十多人，往往就会“盯不深”；AI 也是一样。难点是试探：怎么判断它已经被你 PUA 到“动不了了”。

扯远了。回到经验层面，我现在为了提高 PR 成功率、尽量避免自己下场找 bug 源头，会更愿意把力气用在：怎么测试、怎么发现新问题、怎么设计新 feature、怎么描述新 feature、怎么验收、怎么排优先级上。具体做法是：宁可把 request 用在更小、更可验收的范围，也不要在一个 request 里塞太多跨模块的大目标。因为“改 AI 写的代码”有时候比自己写更痛苦，尤其当它为了改bug堆出一堆 case-by-case patch 的时候。

---

## 一个典型教训：case-by-case 修补会掩盖真正该 refactor 的结构问题

这个项目里，我很早就遇到一个结构性矛盾：

- 我有两个 editor mode（不同布局、不同快捷键、不同缩放逻辑）
- 它们需要“工作内存隔离”
- 但我最开始是 case-by-case 修补：哪里坏了补哪里

这个锅其实赖我这个“项目经理”。我先让它把 chunk editor（逐帧画 GIF）打磨到稍微像样，脑子一热又让它加了 montage editor（剪辑拼接动画、导出视频）。代码结构一开始就没为“双 mode”做系统设计：mode 切换时什么复用、什么隔离、交互边界怎么划。coding agent 也不会主动替我系统化思考这些问题。随着两边功能越加越多，技术债开始反过来影响 feature 开发。

我遇到一个很典型的 bug：**chunk editor 的 canvas 泄漏到 montage editor**。表现是：在 montage 模式下做某些操作（比如缩放、鼠标划过某些区域），会突然看到 chunk editor 的 canvas 更新，像“幽灵画布”一样冒出来。#6/#10 其实都在围绕类似问题打补丁：

- #6 里它自己总结的 root cause 是“全局事件监听没有检查 `currentMode`”，导致在 montage 模式下按 `A` 会触发 chunk 的 add frame，滚轮也会影响 chunk 的 brush/zoom；修复方式就是在 wheel/keyboard 入口加 `currentMode` 判断。
- 但到了 #10，又出现另一种“漏法”：它把 leak 的修复落到一个更局部的状态清理上（切换到 montage 时 remove 掉 `.zoomed` 之类的缩放状态）。这种 patch 当下能止血，但也容易给后续埋“另一个入口又漏”的雷。

case-by-case 的确可以短期快速把眼前 bug 改掉，但长期会变成：

* 复用逻辑和隔离逻辑搅在一起
* hotkey、缩放、事件监听互相污染
* 每加一个特例，就更难回头做统一抽象

如果要反思，这和自己写项目也一样：有些 feature 是要靠底层重构支撑的，而重构的大方向需要架构师/项目负责人根据“未来可能的 feature”做判断。比如新增 editor mode 后，mode 切换逻辑到底怎么设计：是彻底两套数据隔离、数据驱动 UI 与 canvas；还是简单点在 CSS/事件入口做 mode 判断就行？这完全取决于后续要怎么演进。

如果项目管理者/架构师不在这些大方向上提前下功夫，要么团队疲于奔命，要么几代迭代后无力支撑新功能，产品整体落后，最后影响 GTM。

---

## The last catch：几个很具体的小技巧

### 1) 把“系统性 review”当成固定动作

每个 PR 结束，或者每隔几个 PR，我会让 agent 做一次系统 review：

* 有没有明显的逻辑漏洞
* 有没有没清干净的 TODO
* 有没有 corner case 没考虑
* 有没有必要做代码复用/接口抽象

看似会多花一些 request，但能早一点暴露问题总是好的。当然对这种 review 也别期待太高——很多架构性缺失它未必看得出来。一个做法是把这些 review 要点写进项目 prompt，让它每次 PR 结束前自动自检一遍。

### 2) 描述问题要“可搜索”

尤其是前端视觉/布局问题，纯自然语言很容易鸡同鸭讲。我尽量这样描述：

* 直接引用代码里的变量名/函数名
* 精确到 HTML 的 class/id
* 指向具体的 UI 元素、状态、操作路径
* 必要时点名某个函数/类/文件

因为 agent 会大量用 grep 搜代码仓。当然自然语言也不是不能用，比如“左边窗口上方那个文本框”，大部分时候它也能定位。但我的目标是尽量减少它犯低级错误的次数——每次犯错都是沟通成本、计算成本和时间成本。

但这里也有个现实：代码不是我写的，我还是得花一点时间理解基本数据结构和关键路径在哪，不然我连“可搜索的描述”都写不出来。

### 3) README / TODO 对 agent 很重要

每次开新 PR，它都像第一天上班。README 和 TODO 就是它的“入职培训材料”。有准确的中间状态记录，一方面能简化我写需求的成本，一方面也能节省它的 context，不用到处乱翻。

副作用也很明显：文档如果不更新，误导非常严重。一个小窍门是：**每个 PR 结束前舍得花一个 request，让 agent 把文档更新好**。不只是给人看，也是给下一次的 agent 看。（#10 里它就会在 commit 说明里顺手写“更新 TODO/README 反映已修复的问题”。）

### 4) 测试从未如此重要

简单的单元测试，现在 agent 已经能效劳。但这不意味着你就不用投入测试了。现在能自动化生成的测试，多数更像防呆：文件格式检查、静态 layout 检查之类，必要但有限。

大部分问题在这些以外，是产品使用层面的，需要代入用户路径去看。我这次“0 代码开发”的过程中，本质上就是在做测试、写测试报告。比如 #11 里那种“unsaved changes tracking / 模式切换时只检查当前编辑器 / load/new 时检查两边防止丢数据”; 还有触摸屏模式下，如何防止下滑触发pull refresh丢失工作内容。其实都是测试路径逼出来的需求，而不是“凭空拍脑袋”的功能点。

---

## 最后：AI 辅助开发到底意味着什么

对我个人来说——一个代码写得不多的手残——AI 辅助开发给我的最大信心是：只要我肯花点钱，很多 idea 就能以很低的时间成本落地，动手能力和动手意愿都被放大了。这个项目如果完全由我自己写，还得避开工作、挤时间，可能要 4 个星期起步（瞎估）。那我大概率就不写了，或者写一半兴趣转移。但如果只要一个周末，再加一顿饭的花销，我就很愿意经常整点活，给自己找点 cosplay 包工头的乐子。

至于 one man company 能不能成立，短期看 agent 确实把“小项目外包”的成本打下来了：以前可能要花几千块、两星期（同样瞎说没考证），现在也许一天、百来块就能搞定。但商业逻辑不会因为省这点钱就发生质变——只是对特别小的 business 来说，门槛更低、供给更泛滥了。这个背景下，能帮大家变现的平台，才可能是大生意。

至于软件开发行业会怎么变，我回答不了，就不瞎扯了。吃瓜听戏，风雨要来，一叶小舟又能怎地？

---

## 附：相关链接（开发过程与代码）

```text
项目仓库：
https://github.com/WenqinSHAO/pixel-animator

PR 列表（开发过程）：
https://github.com/WenqinSHAO/pixel-animator/pulls?q=is%3Apr+is%3Aclosed
```
