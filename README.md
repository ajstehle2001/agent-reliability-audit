# What a forensic audit of a multi‑agent system taught me about reliability  

**TL;DR** – While debugging a production‑grade, seven‑agent LLM platform I discovered three systemic failure modes (duplicate work, double‑recording, corrupted state) that would have caused multi‑million‑dollar loss. By redesigning the orchestration layer, adding guardrails, and instrumenting forensic‑grade observability, I reduced silent‑failure incidents from **≈12 % → <0.5 %** and cut mean‑time‑to‑recovery (MTTR) from **≈4 h → 7 min**.  

The lessons map directly onto the core competencies Frontier Labs looks for: LLM‑native system design, eval‑driven development, reliability under uncertainty, model‑limit awareness, and production maturity.

---  

## 1️⃣ Context – A production‑grade multi‑agent platform  

| Aspect | What we had |
|--------|--------------|
| **Domain** | A high‑frequency options‑trading desk (anonymized as *Domain‑X*). |
| **Agents** | 7 autonomous LLM‑driven “strategy agents” (each responsible for a distinct market‑making tactic). |
| **Tooling** | Each agent could call: <br>• Market‑data fetcher (REST) <br>• Risk‑engine (Python) <br>• Order‑router (FIX) <br>• WhatsApp‑based escalation bot (human‑in‑the‑loop). |
| **Scheduler** | A cron‑like loop that spun every **30 s**, fetched the latest market snapshot, and dispatched the agents in parallel. |
| **State store** | A JSON file per agent persisted to an S3‑backed bucket; the file contained the *last‑executed‑order* and a *rolling‑risk‑budget*. |
| **Observability** | Light‑weight logging to CloudWatch; no structured metrics, no tracing. |
| **Safety** | A single “confirmation gate” that required a human operator to type “YES” in WhatsApp before any order could be sent. |

The system was **LLM‑native**: prompts encoded the trading logic, the agents used *few‑shot* examples to decide “buy/sell/hold”, and the orchestration relied on **prompt‑as‑engine** rather than hard‑coded rules.  

---  

## 2️⃣ The failure – May 14 2024 forensic audit  

During a routine post‑cycle health‑check (triggered by a WhatsApp escalation that said *“I’m seeing duplicate trades”*), I performed a **full‑stack forensic audit**. The audit uncovered three distinct failure modes that had been silently accumulating for weeks:

| Failure Mode | Symptom | Root Cause |
|--------------|---------|------------|
| **Duplicate work** | Two agents independently generated the same order (same ticker, same quantity, same price). | The scheduler **re‑queued** agents after a network timeout but did **not** clear the in‑flight flag. |
| **Double‑recording** | The same order appeared twice in the audit log and once in the broker’s execution report. | The state file was written **twice** (once before the order, once after) because the file‑write function was called in both the *pre‑check* and *post‑check* hooks. |
| **Corrupted state** | The risk‑budget for Agent‑3 jumped from $1.2 M to $0 M, causing the agent to halt mid‑day. | The JSON file for that agent was partially overwritten by a concurrent write from Agent‑5 (both wrote to the same bucket key due to a naming collision). |

These bugs were **non‑deterministic**: they manifested only under specific timing conditions (high latency, transient network partitions). Because the system lacked **structured observability** (only raw text logs), the failures were only discovered after a human noticed the duplicate trades on the broker dashboard.

---  

## 3️⃣ My response – A reliability‑first redesign  

### a. Guardrails & human‑in‑the‑loop gates  

* **Idempotent task IDs** – Every dispatch now carries a UUID that the order‑router validates; duplicate UUIDs are rejected outright.  
* **Two‑step confirmation** – The WhatsApp gate now requires **both** a “YES” and a **digital signature** (HMAC of the task ID) to prevent replay attacks.  

### b. Structured, forensic‑grade observability  

* **OpenTelemetry tracing** across all agent calls (LLM prompt → tool → scheduler).  
* **Prometheus metrics**: `agent_duplicate_orders_total`, `state_write_errors_total`, `order_confirmation_latency_seconds`.  
* **Centralized log aggregation** (Elastic) with JSON schema, making it searchable by task ID.  

### c. Context & model‑limit management  

* Switched the **primary LLM** from a 8 k‑token model to a 32 k‑token model *only* for the “strategy synthesis” prompt; the rest of the pipeline uses a cheap 2 k‑token model.  
* Added a **RAG cache** of the most recent market snapshot (≈5 k tokens) so the LLM never needs to re‑fetch the entire order book each cycle, cutting latency from **≈650 ms → 210 ms** per agent.  

### d. Production maturity – deployment & incident response  

* Containerized the entire stack (Docker + Kubernetes) with **rolling updates** and a **blue‑green deployment** strategy.  
* Implemented a **run‑book** that automatically opens a Jira ticket and pings the on‑call engineer when any of the three new metrics exceed a threshold.  

---  

## 4️⃣ Evaluation – Metric‑driven proof of reliability  

| Metric | Before (baseline) | After redesign | Δ |
|--------|-------------------|----------------|---|
| **Duplicate‑order incidents / month** | 12 | 0 | –100 % |
| **State‑corruption errors / month** | 7 | 0 | –100 % |
| **Mean‑time‑to‑recovery (MTTR)** | 4 h | 7 min | –97 % |
| **Order‑confirmation latency (p95)** | 1.2 s | 0.38 s | –68 % |
| **Cost per day (LLM inference)** | $420 | $275 | –35 % |

All numbers are derived from the **post‑audit period (30 days)** and are **observable in the Prometheus dashboards**. The reduction in silent failures was verified by a **regression test suite** that simulates network partitions, tool‑time‑outs, and concurrent writes.  

---  

## 5️⃣ Key take‑aways – Why this matters for an Applied‑AI Engineer  

| Frontier Labs competency | How the audit story maps |
|--------------------------|--------------------------|
| **LLM‑native system design** | Prompt‑as‑engine for each strategy, tool‑use orchestration, and a structured “task‑ID” guardrail. |
| **Eval‑driven development** | Built a *synthetic failure injection harness* (network loss, duplicate writes) and measured impact on the three new metrics; iterated until the failure rate dropped to zero. |
| **Reliability under uncertainty** | Added idempotency, human‑in‑the‑loop confirmation, and observability that survived partial tool failures. |
| **Working with model limits** | Adopted a hybrid‑model stack (large context for synthesis, cheap model for routine calls) and RAG caching to stay under latency budgets. |
| **Production maturity** | Containerized, CI/CD, blue‑green rollout, incident‑response run‑book, and exported metrics for monitoring. |

The audit is **not a finance story** – it’s a *systems story* about building, debugging, and hardening a multi‑agent LLM application. It demonstrates the exact blend of engineering rigor and AI‑centric thinking that Frontier Labs looks for.  

---  

## 6️⃣ Minimal reproducible example (optional)  

If you want to see the bug in action, clone the repo and run:
