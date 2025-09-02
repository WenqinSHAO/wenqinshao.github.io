---
layout: post
title: LLM外部记忆系统 2： A Plain, Bold System Desgin
date: 2025-09-02
---


LLM 要成为可长期进化的 Agent，必须配一套“外部记忆系统”。仅靠更长上下文会带来成本上升与 **Context Rot**（信息被多轮拼接稀释/扭曲）；需要在“表示—检索—压缩/保留—注意力接口—治理/度量”上形成闭环。

和AI助手一起开开脑洞，生成出下面这么个“大白话”方案： **MU（Memory Unit）** 方案、**混合索引**（lexical/vector/temporal/graph/KV）与**学习式控制器**（Learning‑to‑Retrieve & Learning‑to‑Write/Retain）。

![Memory System seen by Dali]({{ site.baseurl }}/assets/images/ChatGPT-img-memory.png)
<p align="center"><em>The Memory Orchard — melting clocks drip over index trees while an agent nets glowing MUs from a token stream.</em></p>
---

## TL;DR

* Use a **unified Memory Unit (MU)** schema and a **hybrid index fabric** (lexical + dense + temporal + graph + KV) with a learning router.
* Treat improvement with usage as **two loops**: **Learning‑to‑Retrieve** and **Learning‑to‑Write/Retain** (compression & tiering under token/\$/latency budgets).
* Expose **three interfaces** to the LLM: **Pre‑attn** (context assembly), **Mid‑attn** (cross‑attention adapters), **Post‑attn** (write‑back).
* Make governance first‑class: multi‑tenant isolation, privacy labels, provenance/audit, and right‑to‑be‑forgotten.
* Compare with **MemOS/MemCube**: we focus on the external layer for fast ROI and auditability; MemCube aims to unify parametric & non‑parametric.

---

## 1) Why Agents Need External Memory (recap)

* **Cost & dilution**: longer contexts raise inference cost and often bury the signal.
* **Context Rot**: Chroma shows reliability degrades as inputs get longer and repeatedly re‑stitched; hence **Context Engineering**: organize/place/trim instead of blindly adding tokens.
* **Cross‑session consistency**: real apps need durable, auditable knowledge reuse—not ad‑hoc prompts.

**Design principles**

1. **Computational scalability** (sub‑linear growth; explicit latency/recall/cost trade‑offs).
2. **Less design, more learning** (let policies learn routing, compression, retention, write‑back).
3. **Detachable** (decouple memory from the model; portable across LLMs/agents).
4. **Tenant isolation** (namespaces, quotas, rate limits).
5. **Privacy & auditing** (provenance, labels, right‑to‑be‑forgotten).

---

## 2) Architecture at a Glance

```
[Agent/LLM]
   ↑  Post‑attn write‑backs (F3)
   │  Pre‑attn bundles (F1)
   ↓  Mid‑attn cross‑attention (F2)
[Memory Controller]
   ├── Retrieval policy (D)
   ├── Retention/Compression policy (E)
   └── Privacy/Provenance guard (G)
[Hybrid Index Fabric] (B)
   ├── Lexical (BM25)   ─┐
   ├── Vector (HNSW/IVF) ├─► Fusion & Rerank
   ├── Temporal index    ┤
   ├── Graph index       ┘
   └── KV
[Stores]
   ├── Hot (KV/cache)
   ├── Warm (vector/graph)
   └── Cold (summaries/archives)
[Ingestion & Distillation] (C)
[Schema & Ontologies] (A)
[Eval/Ops/Safety] (H)
```

Design purpose for each component:

* **Schema (A)** names/structures memory and carries privacy/provenance.
* **Ingestion (C)** cleans/merges/abstracts raw interactions.
* **Hybrid index (B)** picks the right tool(s) to find evidence; the system decides where/how deep to search.
* **Controller (D/E)** acts like a scheduler: how to search and what to keep/where to tier.
* **Interfaces (F)** decide how evidence is fed to the model (Pre/Mid/Post‑attn).
* **Governance (G)** controls who can see what and handles redaction/invalidation.
* **Eval/Ops (H)** measures quality/cost/stability and feeds learning signals back.

---

## 3) Components (with borrowable lines of work)

### A. Unified Memory Unit (MU)

A typed carrier for **events/entities/spans/time/relations** with `provenance / privacy / ttl / links`.

**Sketch (v0)**

```json
{
  "id": "mu_xxx",
  "who": ["entity:user_123"],
  "what": "Fixed webhook HMAC verification bug",
  "when": {"start": "2025-08-12T14:03:00Z", "end": "2025-08-12T14:15:00Z"},
  "where": ["repo://payments/webhook", "issue://#4821"],
  "why": "Elevated settlement failure rate",
  "how": "Rollback + patch; add HMAC check",
  "source": {"type": "tool", "uri": "ci://job/98765"},
  "provenance": {"hash": "…", "signature": "…"},
  "confidence": 0.82,
  "privacy_label": "team",
  "ttl": {"policy": "days", "value": 180},
  "links": [{"type": "causes", "target": "mu_incident_4711"}],
  "embeddings": {"text": "…", "entity": "…"}
}
```

**Ops**: `write(mu)`, `amend(id, patch)`, `forget(criteria)` (with index invalidation).

---

### B. Hybrid Index Fabric (lexical/vector/temporal/graph/KV)

One **logical** index over many **physical** indices; a learned router decides where/how deep to search; results are fused and re‑ranked.

**References to learn from**

* **Late Interaction (ColBERT)** for cost‑effective ranking.
* **GraphRAG** for graph‑guided retrieval/summarization over entity/topic communities.
* **Temporal indexing** for interval slicing and backtracking.
* **Approx + cache** with HNSW/IVF/PQ.
* **Zep**: time‑aware graph memory for agents.

  * Two **timelines (T, T')**; hierarchical **episode/entity/community** organization.
  * **Edge invalidation** to expire outdated facts (version‑like control).
  * Trade‑offs: complexity and **prompt‑orchestration dependency**; still far from a fully trainable controller.

**Fusion essentials (our approach)**

* Score calibration across **lexical/dense/temporal/graph**; learned weighting.
* Query‑type routing (factoid/temporal/relational) + dynamic **fan‑out**.
* Cost‑aware **cut‑offs** under token/latency budgets.
* **Carrier alignment**: align raw chunks/summaries/graphs to canonical **MU** before fusion.

---

### C. Ingestion & Distillation (from raw → MU)

**Pipeline**

1. **Extraction** of events/entities/spans from dialogue/tool logs/docs.
2. **Dedup & Merge** (hash + soft match) across sources.
3. **Entity Linking** to a shared ontology.
4. **Abstraction** via hierarchical summarization (session→topic→event) and **sketches** for quick recall.
5. **Labeling** (`privacy_label/provenance/confidence`, data residency).
6. **Validation** (PII scrub, poisoning/contradiction checks).
7. **Write** into EventMU/EntityMU/SummaryMU/SketchMU.

**Write policy (pseudo‑code)**

```python
if utility_pred(x) - storage_cost(x) > tau:
    write(MU(x), tier = 'hot' if hit_prob(x) > p else 'warm')
else:
    skip_or_summarize(x)
```

**Signals**: source quality, future hit‑rate, answer consistency, conflict strength, privacy sensitivity.

**Summarization‑first retrieval**: hit **SummaryMU/SketchMU** first; **lazy‑load** original chunks on low coverage/low confidence.
**Evidence packs**: every write/read carries snippets + sources + confidence for auditability.

**Figure 1**: *Retrieval & Distillation — Write Policy State Machine + Feedback Loop* (see attachment in chat).

---

### D. Learning‑to‑Retrieve (gets better with use)

**Action space**: index routing, fan‑out, cut‑offs, reranker, interface choice (Pre‑attn / Mid‑attn), token budget.
**Reward**: `a·TaskSuccess + b·EvidenceCoverage − c·Latency − d·TokenCost`.
**Learning**: contextual bandits → off‑policy RL with counterfactual replay.
**Warm‑starts**: **ATLAS** (as reranker/teacher), **Titans** (test‑time memorization for routing expansion), **Late‑Interaction** for stable re‑ranking.

---

### E. Learning‑to‑Write/Retain (gets cheaper with use)

Decide **what to write**, **how to compress**, **where to tier (hot/warm/cold)**, and **when to forget**; predict future utility and maintain retrieval‑worthiness after compression (summary/sketch).

---

### F. Interfaces: Pre‑attn / Mid‑attn / Post‑attn

* **Pre‑attn (Context Assembly)**: structured evidence bundles with citations—most controllable, higher tokens.
* **Mid‑attn (Cross‑Attention Adapters)**: read external memory encoders (e.g., **KBLaM** KV pairs)—token‑efficient, requires training/adapters.
* **Post‑attn (Write‑back)**: persist new facts/decisions as MUs.

---

### G. Portable API & Governance (with Measurement + Feedback)

**Core API**

```
POST /memory/write
GET  /memory/read?q=…&top_k=…&privacy=…
PATCH /memory/amend/:id
POST /memory/forget {criteria}
GET  /memory/provenance/:id
```

**Example**

```http
GET /memory/read?q=failed+webhook&top_k=8&privacy=team&time=2025-07..2025-08
```

**Governance**: namespaces/quotas, region pinning, audit logs, redaction + index invalidation, poisoning alerts.
**Measurement & Feedback**: hit\@k, coverage, latency, token/\$; explicit thumbs‑up/down; task‑level success; feed logs into bandit/RL loops; SLOs (p95 latency, audit traceability, TTI for forgets).

---

### H. Metrics & Ops

**Storage/system**: write/read QPS, p50/p95 latency, compression ratio (raw→summary/sketch), index rebuild/backfill, rebalance time, distributed robustness under failures.
**Retrieval/quality**: hit\@k, MRR, precision\@k, calibration AUC, evidence coverage, summary‑first fallback rate.
**Agent tasks**: EM/F1, success rate, tool‑success, end‑to‑end latency, plan churn.
**Cost**: tokens per solved task, \$ per solved task, delta from policies.
**Safety/compliance**: PII leakage, TTI for forget, audit traceability, cross‑source corroboration.

---

## 4) Comparative View with MemOS (MemCube vs MU)

**Why these axes?** They determine **cost, controllability, portability, and time‑to‑production**: carrier design, cross‑carrier migration, retrieval interface (unit‑answer cost), attention coupling (tokens/latency), governance (enterprise readiness), and cost.

| Axis                    | MemCube                                              | MU (this work)                                                        | Trade‑off & impact                                          |
| ----------------------- | ---------------------------------------------------- | --------------------------------------------------------------------- | ----------------------------------------------------------- |
| **Carrier**             | General cube for text/activation/weights w/ versions | Event/entity‑centric structured item (time/relations/privacy)         | MemCube is more universal; MU aligns with retrieval & audit |
| **Migration**           | Cross types (text↔activation↔weights)                | External layer only; do not touch weights; adapter‑driven             | MemCube enables unified evolution; MU lowers risk/cost      |
| **Retrieval interface** | System‑level memory scheduling                       | Hybrid‑index + learned routing for **cheaper answers**                | Complementary: MemCube supplies, MU finds                   |
| **Attention coupling**  | Lifecycle focus; fewer interface details             | Explicit **Pre/Mid/Post‑attn** options (KBLaM/NTM/DNC/Memory‑Decoder) | MU path is more engineering‑ready                           |
| **Governance**          | Emphasized                                           | First‑class API for quotas/rights/forget                              | Faster enterprise fit                                       |
| **Cost/latency**        | Impl‑dependent                                       | Optimized explicitly for token/\$/latency                             | MU focuses on unit‑answer cost                              |

**Does MemCube “solve” fusion?** Not entirely. It harmonizes **carriers** (parametric↔non‑parametric), but **score calibration/fusion** across lexical/dense/graph/temporal still lives in the retrieval layer.
**When to use which?** If you pursue end‑to‑end unification and have research bandwidth, MemCube is attractive; for short‑term product ROI with auditability, MU + hybrid index + learned controllers is pragmatic.

---

## 5) Open Questions

* **Schema/Ontology**: causal/time encoding; confidence calibration; merge vs provenance fidelity.
* **Hybrid Index**: carrier‑level vs score‑level fusion; dynamic fan‑out under budgets; summary‑first sweet spot.
* **Ingestion/Write policy**: minimal write‑set for long‑horizon utility; privacy‑preserving abstraction.
* **L2R**: reward design and counterfactual evaluation.
* **L2W/Retain**: predicting future utility; retrieval‑worthiness after compression.
* **Interfaces**: token‑efficiency vs stability (Pre vs Mid); how many memory tokens/slots?
* **Governance**: verifiable tenant isolation and forget semantics across LLMs.
* **Benchmarks**: combine storage‑style metrics (throughput/compression/robustness) with agent‑task metrics.

---

## Last catch

Treat memory as a **system resource** and close the loop across **schema → index → learning → interfaces → governance → ops**. Here the focus is more on the external layer delivers quick wins on cost and auditability; MemOS/MemCube offers a broader unification. They are complementary—together moving us toward **steadier, cheaper, more auditable** agent memory system design.
