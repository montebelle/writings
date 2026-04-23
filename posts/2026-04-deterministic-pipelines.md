# Deterministic pipelines beat routing agents

*April 2026*

Most multi-agent-system papers published in 2023–2024 assume the right architecture is a *router*: a general-purpose agent reads the task, decides which specialist to hand off to, and coordinates the work. LangGraph-style state machines where the edges are LLM decisions. AutoGen's conversational routing. It's intuitive — more capability, more flexibility, more emergent behavior. It's also, empirically, where MAS systems fail most often.

The alternative: a deterministic pipeline with LLM reasoning *inside* each stage, not *between* them. Write the stage graph by hand. Let each stage be dumb about what comes next. The LLM is a powerful reasoner within its stage's scope and has no autonomy about the graph's structure.

I've run both on real production tasks. The deterministic pipeline wins on every axis that matters.

---

## What the empirical literature says

The case against routing-agent architectures has been building:

**Agentless (Xia et al., arXiv 2407.01489, 2024).** On SWE-bench Lite — a standard software-engineering benchmark — a deterministic 3-stage pipeline (localize → repair → validate) outperformed every agentic framework tested. No agent routing. No tool-use decisions at runtime. Just a fixed graph with LLMs doing the heavy thinking inside each stage. The paper's thesis: agentic overhead adds failure modes without adding capability on tasks whose structure is known.

**MAST 2025 (Cemri et al., ICLR, arXiv 2503.13657).** A taxonomy of 14 failure modes in multi-agent systems based on empirical analysis of real deployment failures. The mode that dominates: *coordination failures* — misrouted hand-offs, context loss between agents, wrong agent picked for the job. These are not failures of individual reasoning. They are failures of the coordination layer that routing architectures *add*. Removing the coordination layer removes the most common failure mode.

**MultiAgentBench (Zhu et al., ACL 2025).** Benchmarks router-based MAS vs deterministic pipelines vs single-agent baselines on a suite of realistic tasks. Deterministic pipelines match or exceed router-based MAS on most tasks at a fraction of the cost. Single-agent baselines beat router-based MAS on some.

**RouteLLM (Ong et al., arXiv 2406.18665, 2024).** The specific case of *learned* routing between models (not agents). Learned routers help when model capabilities are reliably separable — a harder precondition than it sounds. For the more common case of similar-capability models on heterogeneous tasks, the learned router's overhead often exceeds its value.

**Blackboard-pattern MAS (Li et al., arXiv 2510.01285, 2025).** Comparison of master-slave MAS (router hands off to specialists, specialists report back) vs blackboard MAS (typed shared state, specialists read/write regions, no explicit routing). Blackboard beats master-slave by 13–57% on information-discovery tasks in their experiments. The insight: shared structured state beats message-passing with routing decisions embedded in the messages.

The weight of evidence is one-sided. Deterministic structure with LLMs inside stages outperforms routing agents with LLMs choosing the stages.

---

## Why routing is hard

Routing is a capability the LLM is not particularly good at, and for which it cannot be easily trained:

- **Meta-decisions are rare in training data.** Your LLM saw plenty of examples of "here's a task, here's a solution." It saw far fewer examples of "here's a task, and here is the correct decomposition into sub-tasks, and here is the right specialist for each." Routing is a meta-task, and the training distribution underweights it.
- **Routing errors compound.** If your router has a 5% error rate on any individual decision and a task requires 6 routing decisions, your end-to-end success rate is 0.95⁶ = 74%. Fixed pipelines don't compound.
- **Debugging routing is miserable.** When a router picks the wrong specialist on step 3 and the run fails at step 7, you can't easily tell whether the problem was a bad route decision, a bad specialist, a bad handoff payload, or a context loss in between. The blame is distributed across four things that all look like "the agent was wrong."
- **Emergent behavior sounds good and is usually bad.** Router-based MAS produces behavior nobody specified. Sometimes that's creative; usually it's a subtle failure nobody noticed until a user complained.

Determinism is not a lack of sophistication. It's an acknowledgment that the structure of the task is usually known in advance and hiding that structure inside an LLM's routing choices adds noise without adding signal.

---

## What the pattern looks like

I ran a 5-stage intelligence pipeline over enterprise market data: orchestrator → forecasting stage → competitive-intel stage → creative-performance stage → influencer stage. The orchestrator *does not route*. It fires each stage in a fixed order. Each stage reads what it needs from a typed shared state file, writes its outputs to its namespaced region, and passes enriched state forward. No stage decides what the next stage is.

Inside each stage, the LLM does its hardest thinking: generate competitive analysis across 7 analytical lenses; score 45 creative-attribute combinations against engagement metrics; rank 8 influencer-fit dimensions. Those are genuinely LLM-hard tasks. The LLM reasoning is at the unit-of-work level, where the training distribution lines up.

The orchestrator is ~300 lines of boring code: read state, call stage, validate output against gates, write state, advance. That code does not fail. The stages fail in legible ways — a specific gate violates, a specific API is down — and the pipeline either skips the stage with a `degradedFeatures[]` tag or halts with a clean error, not a spiral of recursive debugging.

## Validation gates at each stage

Determinism alone isn't enough. You want per-stage output validation so bad LLM outputs don't propagate silently:

1. **Schema + taxonomy gate.** Does the output parse? Are the taxonomic fields inside the expected set?
2. **Evidence-citation audit.** Every headline claim in the stage output must trace to ≥ 2 sources. No unsourced assertions allowed downstream.
3. **Calibration-logging gate.** The output goes into an append-only ledger tagged with stage, timestamp, input hash, model used, cost. Makes audit and regression-testing possible weeks later.

Gates fire on every stage. Failures are either retried (transient) or routed to a `degradedFeatures[]` tag with a banner on the final output ("this brief is missing the influencer stage because the scoring API was down"). Graceful degradation beats pretending the stage worked.

## The typed runner envelope

Every pipeline run returns a typed envelope:

```json
{
  "ok": true,
  "reason": "success",
  "report": { ... },
  "gateResults": [
    { "gate": "schema",     "result": "pass" },
    { "gate": "evidence",   "result": "pass" },
    { "gate": "calibration","result": "pass", "meta": { ... } }
  ],
  "degradedFeatures": [],
  "cost_usd": 1.47,
  "meta": {
    "models_used": ["claude-sonnet-4-6", "claude-haiku-4-5"],
    "wall_clock_seconds": 147,
    "mast_modes": []
  }
}
```

`mast_modes` tags the run against MAST 2025's 14-mode taxonomy if anything went wrong — useful for drift analysis across runs.

---

## When a routing agent is actually the right move

Determinism fails when the stage graph is genuinely dynamic. Two cases:

**Open-ended research tasks.** "Go find out everything relevant about X." You don't know in advance which sources to consult or which sub-questions matter. Here a routing or agentic loop is defensible — the meta-decisions are the task.

**User-driven multi-step workflows** where the user's next intent isn't known until their previous step's result. A chat assistant is intrinsically like this.

For everything else — content pipelines, intelligence briefings, evaluation systems, code-fixing pipelines, data-enrichment loops — the task structure is known in advance. Hide it and you lose more than you gain.

## Practical rules

1. **Prefer a hand-written stage graph over a learned router.** If the task's structure is stable, hardcode it.
2. **Typed shared state over message passing.** Let stages read/write a blackboard rather than serializing handoff payloads. Loses context from prior stages much less often.
3. **Validation gates per stage.** Schema + evidence + calibration. Cheap deterministic checks catch 95% of bad outputs without an LLM auditor.
4. **Graceful degradation on missing capabilities.** Tag `degradedFeatures[]` rather than killing the run.
5. **Observable runs.** Per-span telemetry with cost + model + gate results. Answer production questions with grep.

Determinism sounds boring. On production workloads, boring is the feature. Routing agents are the clever architecture you pay for in mysterious failures; deterministic pipelines are the one where the failures are legible and the wins compound.

## References

- Xia, C.S. et al. (2024). "Agentless: Demystifying LLM-based Software Engineering Agents." arXiv:2407.01489
- Cemri, M. et al. (2025). "Multi-Agent Systems Fail: A Taxonomy of 14 Modes." ICLR. arXiv:2503.13657
- Zhu, K. et al. (2025). "MultiAgentBench: Evaluating the Collaboration Capabilities of LLM Agents." ACL.
- Ong, I. et al. (2024). "RouteLLM: Learning to Route LLMs with Preference Data." arXiv:2406.18665
- Li, Z. et al. (2025). "Blackboard Multi-Agent Systems for Information Discovery." arXiv:2510.01285
