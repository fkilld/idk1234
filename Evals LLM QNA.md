# LLM Evaluations & LLMOps — Interview Q&A

A reference covering how to build evals and eval frameworks, improve LLM quality, cut token cost, manage hallucination in production, and drive GenAI system-design rounds.

## Table of Contents

- [Building Evals & Eval Frameworks](#building-evals--eval-frameworks)
- [Improving LLM Quality](#improving-llm-quality)
- [Reducing Token Cost & Model Selection](#reducing-token-cost--model-selection)
- [Production Context & Hallucination](#production-context--hallucination)
- [GenAI / LLM System-Design Block](#genai--llm-system-design-block)

---

## Building Evals & Eval Frameworks

### Why evals are the core of LLM engineering.

You can't improve what you can't measure; prompts/models/retrieval change constantly, so evals are your regression suite and release gate. Without them you ship vibes. Treat evals as code: versioned dataset + metrics + thresholds in CI, run on every prompt/model/RAG change. The eval set is the most valuable asset — harder to build than the app.

### How to build an eval dataset.

Start from real traffic (logged production queries) + curated edge cases + known-failure examples. Label with the desired output or a rubric. Cover the distribution: common, rare, adversarial, ambiguous, multilingual, long-context. Keep a frozen golden set for regression + a growing set fed by production failures. Aim for quality and coverage over size; 100 well-chosen cases beat 10k random ones.

### Eval metric types — pick by task.

- **Deterministic (when ground truth exists):** exact match, F1, BLEU/ROUGE (weak for open text), JSON-schema validity, regex/assertion checks, tool-call correctness.
- **Semantic:** embedding similarity to reference.
- **LLM-as-judge (no single right answer):** score relevance/faithfulness/coherence on a rubric.
- **Task metrics:** retrieval recall@k/MRR/NDCG, end-to-end task success rate.

Prefer cheap deterministic checks first, judge only where needed.

### Reference-free evaluation (no ground truth).

LLM-as-judge on dimensions: for RAG → faithfulness (grounded in context), answer relevance, context relevance. General → helpfulness, coherence, instruction-following, safety. Plus pairwise comparison (A vs B, easier than absolute scores), self-consistency (sample N, measure agreement), and implicit user signals (thumbs, edits, regenerations, abandonment).

### LLM-as-judge: setup & biases.

- **Setup:** give judge the input, output, and a clear rubric with scoring criteria + examples; force CoT reasoning before the score; prefer pairwise or low-cardinality scales (1-3) over 1-10.
- **Biases:** position (favors first/second), verbosity (longer = better), self-preference (favors same-family outputs), leniency, formatting.
- **Mitigate:** swap/randomize order and average, calibrate against human labels, use a stronger/different-family judge than the system under test, ensemble judges, anchor the rubric with examples.

### Offline vs online eval.

- **Offline:** pre-deploy on a fixed dataset — accuracy, faithfulness, retrieval metrics, regression suite in CI; gates releases.
- **Online:** live traffic — latency p50/p95, cost/token, error rate, user feedback/acceptance, A/B outcomes, guardrail trigger rate, drift.

Offline catches regressions before ship; online catches real-world failures and distribution shift. You need both.

### Eval frameworks & tooling.

Ragas (RAG metrics: faithfulness, context precision/recall, answer relevance). DeepEval / promptfoo (pytest-style assertions + LLM metrics in CI). TruLens, OpenAI Evals. Observability+eval platforms: LangSmith, Arize Phoenix, Braintrust — run evals on production traces, track scores per release, dashboard drift. Pattern: assertions in CI for gating + trace-based online evals for monitoring.

### Designing an eval-driven dev loop.

Baseline current system on the eval set → change one thing (prompt, model, chunking, retriever) → re-run evals → keep if metrics improve, revert if not. Log every production failure back into the dataset (close the loop). Never tune on the test set — split eval/holdout. This turns prompt engineering from guesswork into experiment.

---

## Improving LLM Quality

### Levers to improve an LLM application (ordered by cost).

1. **Prompt engineering** (clearer instructions, few-shot, CoT, structured output) — cheapest.
2. **Retrieval/context quality** (better chunking, hybrid search, re-ranking) — biggest RAG lever.
3. **Model choice / routing** — upgrade hard cases.
4. **Fine-tuning** — for stubborn behavior/format/style.
5. **Agentic decomposition + tool use** for complex tasks.

Always re-measure on evals after each change; attribute gains to one lever at a time.

### Prompt-level improvements.

Be explicit and specific; put instructions first, context after; use delimiters to separate instructions from data (also a guardrail vs injection); few-shot examples to set format/edge handling; CoT for reasoning tasks; enforce structured output (JSON mode/function calling); add "say I don't know if unsupported" to cut hallucination. Iterate against evals, not intuition.

### When to fine-tune vs prompt vs RAG (for quality).

Prompt first. RAG for knowledge/freshness/citations. Fine-tune for behavior the model won't reliably follow via prompt: consistent format/tone, domain style, latency (shorter prompts), or narrow classification. Knowledge → RAG; behavior/format → fine-tune; often combine. Don't fine-tune to inject facts — it's lossy and stale.

---

## Reducing Token Cost & Model Selection

### Choosing the right model (Claude vs GPT vs open) for cost/quality.

Match model tier to task difficulty, not "best everywhere." Cheap/fast small models (Haiku, GPT-mini, open 8B) for classification, extraction, routing, simple Q&A. Mid models for most RAG/chat. Frontier (Opus/GPT-large) only for hard reasoning, agentic planning, or quality-critical paths. Benchmark candidates on YOUR eval set + measure $/request and p95 latency, not public leaderboards. Re-evaluate as new cheaper models ship.

### Model routing / cascade.

Classify query difficulty (heuristic or a cheap classifier model) → route easy to small model, hard to large. Cascade: try cheap model first, escalate to expensive only if confidence/quality check fails. Cuts cost dramatically since most traffic is easy. Tune the routing threshold on evals to protect quality on the hard tail.

### Token cost reduction techniques.

Trim context (only relevant retrieved chunks, not whole docs; re-rank then keep top 3-5). Shorter system prompts / fewer few-shots once fine-tuned. Prompt caching (reuse compute on the static prefix — system prompt, few-shot). Semantic caching (return cached answer for near-duplicate queries — skip the LLM entirely). Cap max output tokens. Summarize/compress chat history. Batch offline work. Distill a small model from a large one for high-volume paths.

### Prompt caching vs semantic caching.

- **Prompt caching:** provider reuses KV/compute for an identical leading prefix (system prompt + few-shot) across calls — big savings when many requests share static context; you still call the LLM.
- **Semantic caching:** embed the query, if cosine-similar to a past query above threshold return the stored answer — avoids the LLM call entirely for repeats/paraphrases. Risk: stale or wrong cache hits — tune threshold, scope by user/context, set TTL.

### Quantization & distillation for cost.

- **Quantization:** lower weight precision (8/4-bit) → less memory, faster, cheaper self-hosted inference, minor quality loss.
- **Distillation:** train a small student on a large teacher's outputs to keep most quality at a fraction of cost — ideal for a high-volume narrow task.

Both apply to self-hosted/open models; for APIs the lever is model tier + routing + caching.

---

## Production Context & Hallucination

### What causes hallucination in production.

Model asked beyond its knowledge / no grounding; retrieval miss or irrelevant context (RAG fetched the wrong chunks); context too long (lost-in-the-middle — model ignores mid-context); conflicting or outdated context; over-high temperature; ambiguous prompt; the model filling gaps to satisfy a confident-answer prior. Distinguish open-domain hallucination (no context) from RAG faithfulness failure (context present but answer ignores/contradicts it).

### Reducing hallucination — RAG path.

Fix retrieval first (hybrid search + re-ranking + better chunking) so the right context is present. Instruct "answer only from context; if not present, say you don't know." Add citations and verify each claim maps to a source (faithfulness check). Filter out low-similarity retrievals; if nothing clears the threshold, refuse rather than guess. Lower temperature. Keep relevant context near the top/bottom (mitigate lost-in-the-middle).

### Reducing hallucination — model/prompt path.

Ground every factual claim (retrieval or tools instead of parametric memory). Few-shot examples that model the "I don't know" behavior. Self-consistency (sample N, flag disagreement). Chain-of-verification (model drafts → generates checks → revises). Constrain scope and output format. Use a stronger model for high-stakes facts. Never reward confident guessing in the prompt.

### Detecting & measuring hallucination.

Faithfulness/groundedness: LLM-judge or NLI entailment — is each claim supported by retrieved context? Metrics: faithfulness score, % unsupported claims, contradiction rate, citation-accuracy. Self-consistency disagreement as a proxy. Run as a continuous production eval on sampled traces, alert on regression. Combine automated scoring with periodic human spot-checks for calibration.

### Context management at production scale.

Retrieve precisely (re-rank, top 3-5) rather than stuffing the window — more context isn't better (noise + lost-in-the-middle + cost). Order context by relevance; dedupe overlapping chunks. For long chat: summarize/trim old turns, keep system prompt + recent + retrieved. For long docs: map-reduce or hierarchical summarization. Monitor context-relevance as an eval dimension — bad context is the root of most RAG failures.

### Guardrails as a hallucination/safety layer.

- **Input:** PII redaction, prompt-injection/jailbreak classifiers, scope filters.
- **Output:** groundedness check, schema/format validation, toxicity/safety filter, refuse-on-no-context.
- **Defense in depth:** separate untrusted retrieved data from instructions, least-privilege tools, retry on violation. Log trigger rates and feed violations back into evals.

### Tying it together: an eval-first improvement workflow.

1. Instrument production with tracing + online evals (faithfulness, relevance, cost, latency).
2. Mine failures into the eval dataset.
3. Hypothesize a fix (prompt > retrieval > routing > fine-tune).
4. Validate offline on the golden set.
5. Canary/A-B online.
6. Promote if metrics + cost both improve, else revert. Repeat.

Cost and quality are co-optimized — every change measured on both axes.

---

## GenAI / LLM System-Design Block

### How to drive an LLM system-design round.

1. **Clarify:** task, users, scale (QPS, concurrent users), latency/cost budget, accuracy bar, data freshness, privacy.
2. **Pick the pattern:** prompt-only < RAG < agentic < fine-tuned — least complex that meets the bar.
3. **Architecture:** ingestion, retrieval, orchestration, model layer, guardrails, caching, observability.
4. **Evals + feedback loop** (how you'll measure and improve).
5. **Cost/latency optimization** (routing, caching, context trimming).
6. **Failure modes** (hallucination, injection, model/provider outage).

State tradeoffs; "it depends on the eval numbers" is the senior answer.

### Design a production RAG chatbot.

- **Ingestion (offline):** load → chunk (recursive, ~300-500 tok, overlap) → embed → vector DB + metadata; re-index on doc updates.
- **Query path:** input guardrails (PII/injection) → embed query → hybrid retrieve (vector + BM25) → re-rank (cross-encoder, keep top 3-5) → assemble prompt ("answer only from context, cite, else say I don't know") → LLM (stream) → output guardrails (groundedness, format) → cite sources.
- **Cross-cutting:** semantic + prompt caching, model routing, tracing + online faithfulness eval, feedback capture.
- **Scale:** stateless services, replicate, async ingestion, cache hot queries.
- **Tradeoffs:** chunk size (precision vs context), top-k (recall vs noise/cost), latency budget vs re-ranking.

### Design an agentic workflow system.

- **Components:** planner (decompose goal), tool layer (functions/APIs/retrieval with typed schemas), memory (short-term scratchpad + long-term vector store/state), controller/state machine (routing + termination), reflection (self-critique → retry/replan).
- **Pattern:** ReAct for dynamic/exploratory, plan-and-execute for well-defined pipelines, hybrid replanning on failure.
- **Reliability:** validate tool outputs, retry with backoff, cap iterations/cost/tokens, repeat-action detection (anti-loop), human handoff on low confidence.
- **Multi-agent:** supervisor delegates to specialists (LangGraph state graph).
- **Observability:** trace every step/tool call; eval task-success rate + cost per task.

Hardest parts: termination control and error recovery — emphasize them.

### Design a multi-tenant LLM gateway (the "traffic controller").

Single entry fronting multiple providers (OpenAI/Anthropic/open). Responsibilities: auth + per-tenant API keys, model routing (by cost/capability/health), rate limiting + token-budget metering per tenant (Redis), prompt + semantic caching, guardrails, request/response logging + tracing, usage metering → billing pipeline. Resilience: provider fallback (retry → alternate model/provider → cached/degraded response), circuit breaker per provider, timeouts, streaming passthrough. Cost control: route easy → cheap model, cascade with quality check, cap output tokens, dedupe via cache. Multi-tenancy: isolate quotas, keys, and logs per tenant; fair-share scheduling so one tenant can't starve others. This is where the API-gateway concepts meet LLMOps.

### Design a semantic search / retrieval service.

Ingestion: chunk + embed (pick embedding model on your eval set) → vector DB (HNSW/IVF index) + metadata for filtering. Query: embed → ANN search (top-k) → metadata filter → optional re-rank → return ranked results + scores. Hybrid: fuse BM25 + vector (reciprocal rank fusion) for exact-match + semantic recall. Scale: shard the index, cache embeddings + hot queries, batch embedding jobs. Freshness: incremental upsert on new docs, re-embed on model change (versioned index). Eval: recall@k/MRR/NDCG on a labeled query set. Tradeoffs: ANN recall vs latency (ef_search), index rebuild cost, embedding dim vs storage.

### Design an LLM-powered data extraction / structured-output pipeline.

Goal: docs → reliable JSON. Flow: pre-process (OCR/parse) → chunk if long → prompt with schema + few-shot → enforce structured output (function calling / JSON mode / constrained decoding) → validate (Pydantic/JSON-schema) → retry-with-error on failure → confidence/routing (escalate hard docs to a stronger model) → human-in-the-loop review queue for low-confidence. Eval: field-level accuracy vs labeled set, schema-validity rate, % needing human review. Cost: cheap model first, cascade up; cache repeats. Emphasize validation + retry loop and the human review fallback — extraction is judged on reliability, not cleverness.

### Design evaluation + monitoring for an LLM product (eval system itself).

Offline: versioned golden dataset (real + edge + adversarial) + metrics (deterministic checks, LLM-judge faithfulness/relevance, retrieval metrics) wired into CI as a release gate. Online: trace every request (retrieval, prompt, tokens, cost, latency), run sampled async evals on production traces (faithfulness, hallucination, safety), capture user feedback (thumbs/edits). Dashboards + alerts on metric/cost/latency drift; A/B new prompts/models. Feedback loop: production failures → labeled → added to golden set. Tools: Ragas/promptfoo/DeepEval (CI) + LangSmith/Phoenix/Braintrust (traces). The point: evals are infra, not a one-off.

### Latency & cost optimization for an LLM service (design-level).

- **Latency:** stream tokens (cut perceived wait), parallelize retrieval + guardrails, prompt caching on static prefix, semantic cache for repeats, smaller/faster model on the easy path, cap output length, speculative/batched where applicable.
- **Cost:** model routing + cascade (most traffic → cheap model), trim context (re-rank, top-k), shorter prompts (fine-tune in behavior), semantic caching to skip calls, batch offline jobs, self-host + quantize for high-volume narrow tasks.

Always: set a p95 latency + $/request budget and measure both on evals — never optimize one blind to the other.
