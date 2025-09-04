---
layout: post
title: LLM外部记忆系统 3： 小脑和脚手架
date: 2025-09-04
---

书接上回 [LLM外部记忆系统 2： A Plain, Bold System Desgin]({% post_url 2025-09-02-a-plain-bold-external-memory-system-for-agents %})，这次主要倒下面这些饺子：
- 哪些部分靠训练（小脑），那些部分靠高质量高性能的脚手架（受到[Google LangExtract][7]的启发）
- 这个外部记忆系统怎么和Agent的大脑LLM对接，怎么保持接口稳定，是不是可以用DSPy来描述一下
- raw data -> Memeory Unite -> 多维 indexing 的工作流，怎么为各类cue-based recall服务

文章基本就是GPT-5写的。看看我调教得怎么样。

![Small Brain for LLM memory by Hieronymus Bosch]({{ site.baseurl }}/assets/images/ChatGPT-small-brains.png)
<p align="center"><em>A surreal Bosch-style landscape imagining an AI’s external memory: strange towers as index fabrics, glowing vessels as hot and cold storage, and a bizarre brain-creature orchestrating the flows — the small brain at the system’s core.</em></p>

## TL;DR

**Purpose.** Deepen the architecture of the external memory system with two things:

* **Small, learnable brains** (tiny policies we can train on logs/rewards) that co‑optimize **what to read** and **what to write**.
* **Compute‑efficient scaffolds** (cheap, deterministic recipes) inspired by **Anthropic’s Contextual Retrieval** and **LangExtract** so we get strong retrieval and grounded MUs **without** heavy training. 
* **Scope boundary.** All **context/attention tricks** (Titans‑style neural long‑term memory, **KBLaM** rectangular attention, **KV‑cache compression**) belong **inside** your base LLM (F2) and are **out of scope** for external small brains. We just adapt our **budgets** to whatever F2 can handle. ([arXiv][2], [ICLR Proceedings][3])

---

## The Essence： one learned policy + thin contracts

```
                ┌──────────────────────────────────────────────┐
                │                   LLM                        │
                │       (internal context/attention)           │
                └───────────────▲──────────────────────────────┘
                                │  receives top‑K context
                                │  emits candiate writes
┌───────────────────────────────▼──────────────────────────────┐
│                 Memory Wrapper  (thin boundary)              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │              RW‑Policy  (learned small brain)          │  │
│  │  READ: pools/K/rerank/bypass   WRITE: CRUD+payload     │  │
│  └───────────────────────┬────────────────────────────────┘  │
│                          │  ReadPlan / WritePlan             │
│           (Telemetry + Reward Router spans entire box)       │
└───────────────┬──────────┴──────────┬───────────────┬────────┘
                │                     │               │
        ┌───────────────────┐  ┌─────────────────┐    │
        │ Index Fabric Shim │  │  MU Extractor   │    │
        │  BM25+vector+RRF  │  │  (LangExtract   │    │
        │  over‑retrieve150 │  │   + context_prefix)  │
        └────────┬──────────┘  └────────┬─────────┘   │
                 │                      │             │
                 ▼                      ▼             ▼
              [ Candidates ]      [ Unified MU Store ]  (hot recent-N / warm / cold)
```

**One unified small brain?** What we **save** determines what we can **cheaply retrieve**. Jointly learning READ+WRITE avoids the suboptimal split. Memory‑R1 already hints that learned **write ops** and **answer‑time distillation** together beat heuristics, even with **\~152 QA pairs** and retrieval of **\~60 items** before distill. We keep that signal but train a **single head** to co‑optimize quality **and** cost. ([arXiv][4])

---

## Unified Memory Unit (MU)

A single **event‑sourced MU** with a `kind` tag. No separate tables—just **different views**.
This laye of MU bridge the raw data blob and the index fabric.

* **Base (always)**:
  `mu_id, kind∈{claim, episode, profile}, text, context_prefix, source_uri, char_span, entities[], temporal{asserted_at, valid_from, valid_through}, embeddings{passage,entity,claim}, provenance, privacy, quality{conf, extractor_version}, lineage{parents[]}`

* **Optional payloads (Knowledge Graph, trajectory, etc. enhancements)**:
  `claim → relations[S–P–O]` • `episode → state_text, plan_or_tooltrace, outcome, reward, links{prev,next}` • `profile → attributes{k:v}, consent`

* **Two practical choices baked‑in**

  1. **`context_prefix` at ingest** Splitting docs into small chunks speeds search but loses context. Prepending a chunk‑specific, 50–100 token summary **before** embedding and **before** BM25 indexing **recovers that context** and **cuts top‑20 retrieval failures** by \~**49%**, and by **\~67%** when you **rerank top‑150 → top‑20**. ([Anthropic][1])
  2. **Temporal validity + lineage** A single MU + `kind` keeps indexing simple, supports **append‑only episodes** and **updatable facts**, and preserves audit/provenance. Zep’s temporal KG experience shows how to manage evolving truths via **time validity** and **invalidation**. ([arXiv][6])


---

## What’s learned vs. what’s scaffold (and why)

| Surface                              | **Learnt small brain**                                                                       | **Scaffold (deterministic, swappable)**                                                                                            | Rationale                                                                                                                                                                                 |
| ------------------------------------ | -------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **RW‑Policy** (READ+WRITE)           | Chooses K, pools (`claim/episode/profile`), whether to rerank/bypass; emits CRUD + summaries | —                                                                                                                                  | Jointly optimizes **answer quality & cost & store health** under one reward. Memory‑R1 shows RL works; Memento shows episodic reuse can be learned without touching the LLM. ([arXiv][4]) |
| **MU Extractor**                     | —                                                                                            | **LangExtract** to produce MUs with **grounded spans** + **context\_prefix** at ingest                                             | Grounded extraction + contextualized chunks give robust retrieval **without training** (prefix from Anthropic; grounding from LangExtract). ([Anthropic][1], [LangExtract][7])            |
| **Index Fabric Orchestrator (shim)** | —                                                                                            | **Hybrid BM25+vector** with **rank‑fusion**, **over‑retrieve (\~150)**, **simple rerank primitive**; **bypass** for <\~200k tokens | These are well‑understood IR recipes; keep them fast & inspectable. Anthropic provides defaults (150→rerank→20 + “no‑RAG” regime). ([Anthropic][1])                                       |
| **Telemetry & Reward Router**        | Give a single scalar reward to RW‑Policy                                                     | Central service computes recall\@K/NDCG, token & latency cost, store deltas, contradiction/grounding rate (BEIR norms)             | Lets you start with SFT, then flip to RL **without changing interfaces**. ([arXiv][8])                                                                                                    |

> Anthropic shows that adding a **50–100 token context prefix per chunk** and fusing **BM25 + embeddings** **cuts retrieval failures by \~49%**, and by **\~67%** when reranking **top‑150 → top‑20**; if your active KB slice is **<\~200k tokens**, you can even **skip RAG** and stuff the whole KB using prompt caching. ([Anthropic][1])

---

## The learnt part: let's describe with DSPy

### A) **Memory Wrapper** — the boundary around the main LLM

Gives structure for **query/read/write/reward** without tying you to an implementation.

```python
import dspy

class MU(dspy.Prediction):
    mu_id:str; kind:str; text:str; context_prefix:str
    entities:list[str]; temporal:dict; relations:list[tuple]
    provenance:dict; embeddings:dict; lineage:dict; privacy:str

class ReadPlan(dspy.Prediction):
    pools:dict            # e.g., {"claim": 12, "episode": 8, "profile": 0}
    rerank:bool; bypass:bool; filters:dict  # entities/time/privacy
    budget:dict           # token, latency

class WritePlan(dspy.Prediction):
    ops:list[dict]        # [{"op":"ADD|UPDATE|DELETE|NOOP", "mu": MU}, ...]

class RewardTaps(dspy.Prediction):
    task_score:float; tokens:int; latency_ms:int
    store_delta:int; contradictions:int; grounding_rate:float

class MemoryWrapper(dspy.Module):
    def __init__(self, rw_policy):
        self.rw = rw_policy           # injected (see controller below)
        self.retrieve = ...           # scaffold (index orchestrator call)
        self.commit = ...             # scaffold (CRUD executor)
    def forward(self, query:str, write_candidates:list[MU]=None):
        plan = self.rw.decide(query=query, store_sketch=self._sketch())
        cands, stats = self.retrieve(plan)         # hybrid fusion + over-retrieve
        answer = self._reason(query, cands)        # call base LLM
        wplan = self.rw.judge_and_write(answer=answer,
                                        write_candidates=write_candidates or [])
        self.commit(wplan)
        self._route_rewards(self.rw, self._measure(answer, cands, stats))
        return answer
```

### B) **Memory Controller (RW‑Policy)** — small LLM you can train

One head decides **READ** & **WRITE**; same signature from heuristics → SFT → RL.

```python
class RWAct(dspy.Signature):
    # Inputs are cheap summaries, not raw store
    query:str; store_sketch:dict; write_candidates:list[MU]; budget:dict \
      -> read_plan:ReadPlan, write_plan:WritePlan, stop:bool, signals:dict

class RWPolicy(dspy.Module):
    def __init__(self): self.act = dspy.Predict(RWAct)
    def decide(self, query, store_sketch):     # READ
        rp, _, _, sig = self.act(query=query, store_sketch=store_sketch,
                                 write_candidates=[], budget=self._budget())
        return rp
    def judge_and_write(self, answer, write_candidates):    # WRITE
        _, wp, _, sig = self.act(query=answer, store_sketch=self._sketch(),
                                 write_candidates=write_candidates,
                                 budget=self._budget())
        return wp
```

**Training path (small LLM).**
**Stage A — SFT:** warm‑start with (i) reranker preferences for READ (BEIR‑style pairwise/listwise), (ii) heuristic labels for WRITE (merge→UPDATE, unsafe delete blocked). ([arXiv][8])
**Stage B — RL:** cost‑aware reward

$$
R = r_{\text{task}} - \lambda_{\text{tok}}\text{(tokens)} - \lambda_{\text{lat}}\text{(latency)} - \lambda_{\text{store}}\Delta\text{(storage)} - \lambda_{\text{churn}}\#\text{writes}
$$

Optimize with PPO/GRPO (as in Memory‑R1). If episodic reuse is helpful, the policy learns to request from the `episode` pool (soft‑Q flavor of Memento), without changing interfaces. ([arXiv][4])

---

## The scaffolding part: Index Fabric as an **efficient shim**

**Role.** A thin, fast layer between RW‑Policy and physical indices. It **compiles a high‑level ReadPlan** into backend‑specific queries, **executes hybrid search** (BM25 + vector), **over‑retrieves (≈150)**, applies a **simple rerank primitive**, and returns candidates + stats.

**Do we have to build from scratch?** No. Mature engines already offer **hybrid search + fusion**:

* **Vespa** (hybrid retrieval + flexible ranking). ([docs.vespa.ai][9])
* **OpenSearch** (hybrid pipelines combining BM25 + vector). ([OpenSearch Documentation][10])
* **Weaviate** (hybrid / RRF fusion). ([Weaviate Docs][11], [Weaviate][12])
* **Milvus** (hybrid and multi‑vector search). ([Milvus][13])

These make an excellent **shim**: keep it deterministic, observable, and cheap; let the RW‑Policy decide “**which pools/how big K**,” not *how* to score BM25.

**Who formulates index‑specific queries?**

* The **RW‑Policy** outputs a **generic ReadPlan** (pools, K per kind, filters, rerank/bypass hints).
* The **shim compiles** that into index‑specific DSL (e.g., Vespa YQL, OpenSearch DSL, Weaviate GraphQL), performs **rank‑fusion**, and hands back candidates. This split keeps the policy **portable** and the shim **simple**. ([docs.vespa.ai][9], [OpenSearch Documentation][10], [Weaviate Docs][11])

**Defaults you can adopt now (Anthropic recipe).**

1. Ingest‑time: compute a **context\_prefix (50–100 tok)** and index **raw + contextualized** chunks.
2. Query‑time: **BM25+embedding fusion** → **over‑retrieve \~150** → **rerank → top‑20**.
3. If KB slice **<\~200k tokens**, **bypass RAG** and stuff the context with **prompt caching**. ([Anthropic][1])

---

## The spine allowing for learning and evolution: Telemetry & reward (vertical and minimal)

* **What we measure:** retrieval recall\@K / NDCG (BEIR norms), grounding rate (span coverage), contradictions, tokens, latency, store deltas, and final task/judge score. ([arXiv][8])
* **One Reward Router:** transforms metrics into a single scalar **R** and **only updates the RW‑Policy** (keeps scaffolds stable).
* **Optional guards:** a tiny **retrieval evaluator** (CRAG) to accept/broaden/skip; or **retrieve/critique** tokens (Self‑RAG) if you want the LLM to hint when to call memory. ([arXiv][14])

---

## Possible next-steps

1. **Scaffolds first.**
   * Plug **LangExtract** at ingest to produce MUs with **span grounding** + **context\_prefix**. ([LangExtract][7])
   * Stand up the **index shim** on Vespa / OpenSearch / Weaviate / Milvus with **hybrid fusion** and a **simple rerank**. ([docs.vespa.ai][9], [OpenSearch Documentation][10], [Weaviate Docs][11], [Milvus][13])
   * Adopt Anthropic defaults: **over‑retrieve \~150 → rerank → top‑20**; enable **no‑RAG** for **<\~200k tokens**. ([Anthropic][1])
2. **One small brain.** Train **RW‑Policy (SFT)** on logs/heuristics to pick **pools/K** and **CRUD**.
3. **Flip to RL** with the **cost‑aware reward** (Memory‑R1‑style PPO/GRPO). Keep the same DSPy signatures. ([arXiv][4])
4. **(Optional)** If the tasks are tool‑heavy, let RW‑Policy request the `episode` pool; add a soft‑Q head later if needed (Memento). ([arXiv][5])

---

## Why mid-attn excluded ?

```text
↑ Post-attn write-backs (F3) write-backs 
│ Mid-attn cross-attention (F2) ? 
↓ Pre-attn bundles (F1) retrieval
```

Either this mid-attn is decomposed into multiple-rounds of the post-attn and pre-attn ops following some sort of test-time scaling techniques, either this is sth built-in of the LLM attention mechnism, such as:

- Titans adds a neural long‑term memory to attention, scaling beyond long windows; 
- KBLaM integrates an external KB as continuous KV pairs via rectangular attention; 
- KV‑cache compression (MiniKV/KV‑Compress/surveys) cuts inference memory; 
- KG-attetion soft in-context distillation/retrieval

These materially change how much context the LLM can internally carry, while our RW‑Policy simply adapts its K and bypass decisions to those capacities.

---

## The last catch

* Keep the **RW‑Policy** small and learnable; keep the **shim + extractor** boring, fast, and inspectable.
* Use **Anthropic’s retrieval recipe** to lock in immediate wins; **unify** MUs and **route** a single, cost‑aware reward.
* Let **F2** evolve with the model; your external system stays stable and **cost‑controlled**.

---

## References

* **Anthropic — Contextual Retrieval** (context prefixes; BM25+embeddings; **top‑150 → rerank → top‑20**; **49%/67% failure reduction**; **<\~200k tokens → no‑RAG**). ([Anthropic][1])
* **LangExtract** (grounded, schema‑driven extraction; provider‑pluggable). ([LangExtract][7], [GitHub][16])
* **Memory‑R1** (RL‑learned CRUD + answer‑time distillation; **\~152 QA pairs**; **\~60 retrieval** before distill; “recent‑50” temporal snapshot). ([arXiv][4])
* **Memento** (case‑bank with **soft‑Q** retrieval; continual adaptation **without** LLM FT). ([arXiv][5])
* **Zep** (temporal KG; validity intervals; **invalidation** and history‑preserving updates). ([arXiv][6])
* **Hybrid search engines** (shim choices): **Vespa** tutorials/docs; **OpenSearch** hybrid pipelines; **Weaviate** hybrid/RRF; **Milvus** hybrid/multi‑vector. ([docs.vespa.ai][9], [OpenSearch Documentation][10], [Weaviate Docs][11], [Weaviate][12], [Milvus][13])
* **Internal F2 (for completeness):** **Titans** (neural long‑term memory); **KBLaM** (rectangular attention over KB); **KV‑cache compression** (MiniKV, KV‑Compress). ([arXiv][2], [ICLR Proceedings][3], [ACL Anthology][17])
* **LLM外部记忆系统 2： A Plain, Bold System Desgin** (baseline architecture & goals). ([Snow River Notes 钓雪记][15])

[1]: https://www.anthropic.com/news/contextual-retrieval "Introducing Contextual Retrieval \ Anthropic"
[2]: https://arxiv.org/pdf/2501.00663?utm_source=chatgpt.com "arXiv:2501.00663v1 [cs.LG] 31 Dec 2024"
[3]: https://proceedings.iclr.cc/paper_files/paper/2025/hash/803485352e61e3ebf41221e4776c9fd4-Abstract-Conference.html?utm_source=chatgpt.com "KBLaM: Knowledge Base augmented Language Model"
[4]: https://arxiv.org/pdf/2508.19828 "Memory-R1: Enhancing Large Language Model Agents to Manage and Utilize Memories via Reinforcement Learning"
[5]: https://arxiv.org/pdf/2508.16153 "Memento: Fine-tuning LLM Agents without Fine-tuning LLMs"
[6]: https://arxiv.org/abs/2501.13956v1?utm_source=chatgpt.com "Zep: A Temporal Knowledge Graph Architecture for Agent Memory"
[7]: https://langextract.io/?utm_source=chatgpt.com "LangExtract - Extract Structured Data From Any Text"
[8]: https://arxiv.org/abs/2104.08663v1?utm_source=chatgpt.com "[2104.08663v1] BEIR: A Heterogenous Benchmark for Zero-shot Evaluation ..."
[9]: https://docs.vespa.ai/en/tutorials/hybrid-search.html?utm_source=chatgpt.com "Hybrid Text Search Tutorial - Vespa"
[10]: https://docs.opensearch.org/latest/vector-search/ai-search/hybrid-search/index/?utm_source=chatgpt.com "Hybrid search - OpenSearch Documentation"
[11]: https://docs.weaviate.io/weaviate/search/hybrid?utm_source=chatgpt.com "Hybrid search | Weaviate Documentation"
[12]: https://weaviate.io/blog/hybrid-search-explained?utm_source=chatgpt.com "Hybrid Search Explained - Weaviate"
[13]: https://milvus.io/docs/milvus_hybrid_search_retriever.md?utm_source=chatgpt.com "Milvus Hybrid Search Retriever"
[14]: https://arxiv.org/pdf/2401.15884?utm_source=chatgpt.com "Corrective Retrieval Augmented Generation - arXiv.org"
[15]: https://wenqinshao.github.io/2025/09/02/a-plain-bold-external-memory-system-for-agents.html "LLM外部记忆系统 2： A Plain, Bold System Desgin | Snow River Notes 钓雪记"
[16]: https://github.com/google/langextract/blob/main/langextract/providers/README.md?utm_source=chatgpt.com "langextract/langextract/providers/README.md at main · google ... - GitHub"
[17]: https://aclanthology.org/2025.findings-acl.952/?utm_source=chatgpt.com "MiniKV: Pushing the Limits of 2-Bit KV Cache via Compression and System ..."
