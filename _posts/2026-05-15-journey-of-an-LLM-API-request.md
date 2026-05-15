---
layout: post
title: Journey of an LLM API request
date: 2026-05-15
---

我先使用自研的 [net-search](https://github.com/WenqinSHAO/skill-net-search) skill 进行文献检索、筛选与结构化分析，再借助自动化分析 pipeline 归纳相关系统的架构分层、技术路线和演进逻辑。把生成的markdown作为上下文输入后，生成的图示不再只是概念性框架，而能更准确地呈现已有系统、研究工作与整体 serving stack 之间的关系。

![om]({{ site.baseurl }}/assets/images/llm_serving_stack.png)
<p align="center"><em>KV-Aware LLM Serving Stack</em></p>


![om]({{ site.baseurl }}/assets/images/llm_serving_journey.png)
<p align="center"><em>Journey of an LLM API Request with KV Reuse</em></p>

多年前在 G 家，NEO 还会讲 **A Day in the Life of a Packet**，因为很多码农并不知道 TCP 下面发生了什么。后来云基础设施把这些细节成功藏了起来，除非搞云网络的，大家确实也不太需要知道了。到了 LLM serving，网络还有没有一席之地呢？
