---
layout: post
title: "KV Cache Reuse as a Transport and Control Problem"
date: 2026-06-05
excerpt: "An AI-generated HTML report on why KV cache reuse is not just a cache-hit problem, but a transport, placement, routing, and control problem."
---

I generated an HTML report on KV cache reuse in LLM serving:

[Read the English report]({{ site.baseurl }}/kv_cache_reuse_transport_report_final.html)

The report looks at KV cache reuse as a transport and control problem. The main question is not just whether a request can hit reusable state, but whether compatible KV state can be found, governed, routed, transported, and bound into the inference engine before recomputation becomes the better choice.

The focus is therefore on the serving-system layer around reuse: distributed KV stores, prefill/decode disaggregation, external memory tiers, direct transfer paths, CXL-like memory pooling, KV-aware routing, object compatibility, and the economics of moving large reusable state through a real cluster.

This is another follow-up to the `net-search` skill I mentioned in [a previous post]({% post_url 2026-05-12-some-unfinished-agent-projects %}). As with the NVIDIA research-to-product report, the final HTML is only the visible artifact. The more useful part is the trail behind it: search plans, source ledgers, intermediate notes, claim checks, prompts, and the scripts used to turn a broad technical question into something reviewable.

I put the project artifacts here: [skill-net-search/projects/kv-cache-serving-v2](https://github.com/WenqinSHAO/skill-net-search/tree/main/projects/kv-cache-serving-v2). The basic idea is still the same: instead of treating a research agent as a black box, give a general coding agent a structured workspace where the intermediate state is inspectable, reusable, and easy to audit.
