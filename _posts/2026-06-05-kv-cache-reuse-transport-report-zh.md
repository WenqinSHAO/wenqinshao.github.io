---
layout: post
title: "KV Cache 复用：传输与控制视角"
date: 2026-06-05
excerpt: "一份 AI 生成的 HTML 报告，分析为什么 KV cache 复用不只是缓存命中问题，而是放置、传输、路由与控制问题。"
---

我生成了一份关于 LLM serving 中 KV cache 复用的 HTML 报告：

[阅读中文报告]({{ site.baseurl }}/kv_cache_reuse_transport_report_final_zh.html)

这份报告把 KV cache 复用当成一个传输与控制问题来写。关键不只是一个请求能不能命中可复用状态，而是兼容的 KV 状态能不能被找到、管住、路由、搬到位，并且在重算更划算之前被推理引擎直接使用。

所以报告的重点放在复用周围的 serving 系统层：分布式 KV store、prefill/decode 解耦、外部内存层级、direct transfer path、CXL 类内存池、KV-aware routing、对象兼容性，以及在真实集群里搬运大块可复用状态时的经济账。

这也是之前那篇[几个没写完的 Agent 小项目]({% post_url 2026-05-12-some-unfinished-agent-projects %})里提到的 `net-search` skill 的又一个后续。和 NVIDIA 研究到产品转化报告一样，最后的 HTML 只是最容易展示的产物。更有价值的是它背后的过程：搜索计划、来源账本、中间笔记、claim 检查、prompts，以及把一个很宽的技术问题收敛成可复查报告的脚本。

这些中间产物我也放到了这里：[skill-net-search/projects/kv-cache-serving-v2](https://github.com/WenqinSHAO/skill-net-search/tree/main/projects/kv-cache-serving-v2)。想法还是那套：不要把 research agent 当成黑盒，而是给通用 coding agent 一个结构化工作区，让中间状态可以检查、可以复用，也更容易审计。
