---
layout: post
title: Harness之后
date: 2026-06-06
---

对各种新造名词，多少有点疲劳。比如最近说的 loop engineering，但 loop 难道不是最基本的程序设计范式之一吗？看作一个已经在被广泛实践的 harness / control / 的一个子集可能更合适。

agent的大的问题一直都是base model 之外，什么机制是真正有必要的？这是一个不停变化动态的问题，CoT/ReAct/workflow被base model吸收，或是变得更加ad hoc/generative。我之前的几个项目感受也是过多的harness，比如workflow，有损任务表现。

今年最重要的方向/看点，可能是即插即用的自进化机制。无论是 GEPA、Skill1、SkillOpt，还是后面会出现的一堆名字，它们本质上都在指向同一件事：agent 不只是执行任务，而是持续改造自己的资产（assets）。Skill 只是一个方便的载体；真正需要自进化的是 prompt、data、policy、script、tool、tool chain、memory、安全规则、评估样本，以及所有能沉淀为组织能力的 agentic asset。

人 + agent 的“超级个体”已经开始出现。还没被解决的问题是：超级团队怎么建？当多个 agents、多人、多种记忆、多套工具和多条轨迹共同演化时，什么是最根本的不应该被base model吸收的技能点？

![om]({{ site.baseurl }}/assets/images/ChatGPT-after-harness.png )
<p align="center"><em>After harness</em></p>
