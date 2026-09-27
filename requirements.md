# Tokentide: Adaptive Model Router + Cost/Latency Observability Platform

## High-Level Summary

A middleware gateway that sits in front of LLM calls, classifies each incoming request by complexity, routes it to the cheapest model capable of handling it correctly, and logs token/cost/latency/quality metrics for every request. Over time, the observability data feeds back into routing decisions (e.g., adjusting thresholds when a model tier underperforms on a given task type).

**Problem it solves:** Most LLM applications default to one model for everything, wildly overpaying for simple tasks or underpowering complex ones. There's rarely a systematic, measured way to make the cost/quality tradeoff — this project builds that system, with the observability platform as the feedback mechanism that makes routing decisions data-driven rather than guessed.

**Non-goal (for now):** This is not a general APM/observability tool for arbitrary services — scope is LLM-call-specific (tokens, cost, model choice, output quality).

---

## Key Requirements

### Routing
- Classify incoming requests by task complexity (e.g., simple factual/extraction vs. multi-step reasoning/ambiguous)
- Classification method should itself be cheap: heuristics (prompt length, keyword patterns) and/or a small/cheap model — not a full-size LLM call just to decide which LLM to call
- Route to the appropriate model tier based on classification (e.g., small/fast model for simple tasks, large/capable model for complex ones)
- Support a configurable fallback/escalation path (if a cheap model's output looks low-confidence or fails a quality check, escalate to a bigger model)
- Routing decisions should be overridable/configurable per use case (not a black box)

### Observability
- Log per-request: input tokens, output tokens, total cost, latency, model used, routing decision, and a quality signal (even a simple proxy metric)
- Aggregate reporting: cost and latency broken down by model tier, task category, and time window
- Ability to compare "what if we always used the expensive model" vs. actual routed cost, to quantify savings
- Track routing accuracy over time (did the classifier send the request to a model that produced acceptable output?)

### System Requirements
- Should work as a drop-in layer in front of any LLM call, model-provider agnostic
- Feedback loop: observability data should be usable to retune routing thresholds (manually first, semi-automatically later)

---

## Success Metrics

- **Cost savings:** % reduction in total token/dollar cost vs. an "always use the top-tier model" baseline, at equivalent or acceptable quality
- **Routing accuracy:** % of requests routed to a model tier that produced acceptable output (no escalation/failure needed)
- **Misroute rate:** % of requests sent to too-weak a model that then needed escalation or produced poor output (the failure mode to minimize)
- **Latency impact:** added ms from the classification/routing step itself, as a % of total request latency
- **Observability coverage:** % of requests with complete logged telemetry (tokens, cost, latency, model, quality signal)
- **Quality parity:** output quality on a benchmark eval set, routed vs. always-large-model, should stay within an acceptable margin

---

## MVP Definition

A single-process library that:
1. Classifies incoming prompts into **2 tiers** (simple vs. complex) using a heuristic and/or one lightweight classifier
2. Routes to **2 model options** (one cheap/fast, one capable/expensive)
3. Logs per-request tokens, cost, latency, and model used to a local file/DB
4. Produces a basic aggregate report: total cost, cost breakdown by tier, average latency by tier
5. Has a small hand-curated eval set (20–30 prompts of known "should be simple" / "should be complex") to measure routing accuracy from day one

**Explicitly out of scope for MVP:** more than 2 model tiers, automatic threshold tuning, escalation/fallback logic, dashboard UI (a generated report/CSV is enough).

---

## Milestones / Phases

### Phase 1 — MVP (see above)
Core classify-and-route loop with baseline logging and reporting. Goal: prove the routing pattern works and establish a baseline cost-savings number vs. always-large-model.

### Phase 2 — Escalation & Confidence-Based Fallback
- Add a quality/confidence check on cheap-model output; escalate to the expensive model when confidence is low
- Track and report the escalation rate (a proxy for how well the initial classifier is performing)
- Expand to 3 model tiers (small/medium/large) if useful

### Phase 3 — Expanded Classification & Task-Type Awareness
- Move beyond binary simple/complex into task-category-aware routing (e.g., extraction vs. reasoning vs. creative writing may route differently even at similar "complexity")
- Improve the classifier (e.g., train a small model on your growing eval set instead of pure heuristics)
- Grow the eval set substantially (100+ labeled prompts across task types)

### Phase 4 — Full Observability Dashboard
- Build a real dashboard (cost/latency/quality trends over time, breakdowns by tier and task type)
- "What-if" comparison view: actual routed cost vs. always-large-model vs. always-small-model
- Alerting on anomalies (e.g., sudden spike in escalation rate or cost)

### Phase 5 — Feedback-Driven Threshold Tuning
- Use accumulated observability data to semi-automatically retune routing thresholds (e.g., periodic recalibration job based on recent escalation/misroute rates)
- Optional: expand to support more than 2–3 model tiers with a more general cost-optimization policy
- (Optional tie-in point with Filtrium's guardrails layer, if you later choose to integrate them)
