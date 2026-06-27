LLM fundamentals

- Explain transformer architecture: attention, multi-head attention, positional encoding.
- Difference between encoder-only, decoder-only, encoder-decoder models.
- How does temperature, top-k, top-p sampling affect output?
- What is the context window and how do you handle inputs exceeding it?
- Explain tokenization (BPE, WordPiece) and its impact on cost/performance.

RAG

- Walk through a RAG pipeline end to end.
- Chunking strategies and trade-offs (fixed, semantic, recursive).
- Dense vs sparse retrieval; when to use hybrid search.
- How do you evaluate retrieval quality (recall@k, MRR, NDCG)?
- How do you reduce hallucination in RAG systems?
- Re-ranking: why and how (cross-encoders, Cohere rerank).

Fine-tuning & adaptation

- Full fine-tuning vs LoRA/QLoRA vs prompt tuning; when to pick each.
- When do you fine-tune instead of using RAG or prompting?
- Explain RLHF and DPO.
- How do you build a training dataset and avoid catastrophic forgetting?

Agentic systems

- Design an agentic workflow: planning, tool use, memory, reflection.
- ReAct vs plan-and-execute patterns.
- How do you handle multi-agent orchestration (LangGraph, supervisor pattern)?
- Error handling, retries, and loop/termination control in agents.
- MCP and tool-calling design.

Evaluation

- How do you evaluate an LLM application without ground truth?
- LLM-as-a-judge: setup, biases, mitigation.
- Offline vs online eval; what metrics for production.
- Detecting and measuring hallucination.

Production / system design

- Design a chatbot/RAG system for X scale; latency, cost, caching.
- Prompt caching, semantic caching, batching.
- Guardrails: PII, prompt injection, jailbreaks, output validation.
- Observability and tracing (Arize, LangSmith, Phoenix).
- Cost optimization: model routing, distillation, quantization.
- Streaming, rate limits, fallback strategies.

Prompt engineering

- Few-shot, CoT, self-consistency, structured output enforcement.
- How do you make outputs deterministic/JSON-reliable?

Behavioral / experience

- Walk me through a GenAI project you owned end to end.
- A time a model failed in production—how you debugged it.
- How you keep up with a fast-moving field.

Coding (live)

- Implement a basic RAG retriever / cosine similarity from scratch.
- Parse and validate LLM JSON output with retries.
- Implement a token-aware text chunker.

Want a deeper drill-down with model answers on any cluster?
