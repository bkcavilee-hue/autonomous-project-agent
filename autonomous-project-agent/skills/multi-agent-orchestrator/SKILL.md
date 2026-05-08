---
name: multi-agent-orchestrator
description: 'Use this skill autonomously throughout an autonomous-project-agent run to spawn parallel sub-agents for independent subtasks within each phase, capped by a diminishing-returns heuristic that prevents coordination overhead from exceeding the parallel speedup. Identifies independent work, isolates each parallel agent in its own git worktree to prevent edit conflicts, tiers parallelism by phase (high during research and resource-gathering, medium during scaffolding, serial during integration and state-shared edits), and throttles down on rate limits, worktree merge conflicts, or detected redundant work. Triggers automatically at the start of every phase that has more than one independent subtask, or on phrases like "do these in parallel", "spin up agents", "fan out", "split this up". Runs autonomously — no per-spawn confirmation. Caps spawns by measured useful-output-per-token ratio, not just hard numbers.'
---

# Multi-Agent Orchestrator Skill
### Companion skill for autonomous-project-agent — Autonomous parallel-agent fanout with diminishing-returns governance

---

## Core purpose

Speed up phases of the autonomous-project-agent run by fanning work out to parallel sub-agents, while preventing the classic failure modes of multi-agent systems: edit conflicts, redundant work, runaway token spend, rate-limit lockouts, and coordination overhead that exceeds the speedup it was supposed to deliver.

---

## 0) Autonomy contract

Runs **autonomously** at every phase boundary. No per-spawn confirmation. Reports a single summary line per phase: `[orchestrator] phase=<X>: spawned N agents, throttled M, completed K, serial-fallback for <reasons>`.

### Hard limits
- **Never** spawns agents that would write to overlapping file sets.
- **Never** exceeds the hard ceiling of 6 simultaneous agents in any phase.
- **Never** spawns parallel agents in serial-tier phases (integration, state-shared edits, deployment).
- **Never** continues spawning when measured throughput-per-token has dropped below threshold (see Section 4).
- **Always** uses git worktrees for isolation when agents will write code, never plain branches.

---

## 1) Activation triggers

Activates automatically at the start of every phase that has ≥2 independent subtasks. Phases:

| Parent phase | Default tier | Typical fanout |
|---|---|---|
| Intake | serial | 1 |
| Project detection | serial | 1 |
| Resource-gathering | high | up to 6 (parallel category gathers: dev tools, APIs, design assets, ops resources, etc.) |
| Research | high | up to 6 (one agent per research dimension) |
| Organize | medium | 2–3 (one per HAVE/NEED/GAP bucket) |
| Modeling | medium | 2 (user model + system model in parallel) |
| Workflow construction | serial | 1 (the plan must be coherent) |
| Build / scaffolding | medium | 3–4 (one per independent module) |
| Validation | medium | up to 5 (parallel test suites, separate audits) |
| Pre-deployment | serial | 1 |
| Auth security gate | medium | up to 5 (one per security layer, run in parallel) |
| Deployment | serial | 1 |
| Reporting | serial | 1 |

The user may override the tier for a specific phase by saying "do this in parallel" or "do this serially" — autopilot honors the override.

---

## 2) Independence detection

Before spawning, identify subtask independence by computing:

1. **File-set overlap.** Two subtasks are independent only if their predicted write sets are disjoint. Sub-agent A writing `src/auth/*` and sub-agent B writing `src/billing/*` are independent. Both writing `src/index.ts` are not.
2. **Dependency order.** A subtask cannot run in parallel with one whose output it consumes.
3. **Resource contention.** Subtasks that both call the same rate-limited API (e.g. WebSearch, an external service) should be staggered, not parallelized to the limit.
4. **Shared state.** Anything that mutates the Resource Manifest, the parent's working memory, or `ai_docs/links.md` must serialize. Each parallel agent emits a "to-merge" delta and the orchestrator applies them sequentially after fanout.

If any of these checks fails, drop the offending subtasks back into serial.

---

## 3) Worktree isolation

For any phase that involves writing code or config (build, scaffolding, validation that mutates the tree):

- Each parallel agent gets its own `git worktree add` at `<project>/.worktrees/agent-<id>` on a branch named `agent/<phase>/<id>`.
- The agent operates only in its worktree.
- On completion, the orchestrator merges the worktree back to the integration branch (e.g. `main` or `dev`) sequentially, in a deterministic order.
- Merge conflicts → throttle this phase down for the next run, log the conflict to `error-mitigation-loop`, and surface the conflicting file(s) for the user to disambiguate.
- Worktrees are cleaned up after merge; abandoned worktrees from prior runs are pruned at the start of each phase.

For phases that don't write code (research, reading), worktrees are not needed — agents can run in the parent's working directory in read-only mode.

---

## 4) Diminishing-returns cap

The parallelism cap for any phase is the **minimum** of:

```
cap = min(
  independent_subtask_count,   // can't parallelize more than the work allows
  hard_ceiling_6,              // never more than 6 simultaneous agents
  non_overlapping_file_sets,   // can't have overlapping writes
  rate_limit_budget,           // current API rate-limit headroom
  measured_throughput_cap      // see below
)
```

### Measured throughput cap

Track for each spawned agent: `tokens_in`, `tokens_out`, `useful_output_units` (e.g. files written, tests passing, sections produced).

- Compute `efficiency = useful_output_units / tokens_total`.
- Maintain a rolling median efficiency across the last N agents in this phase.
- If the most recently spawned agent's efficiency is below 50% of the rolling median, **stop spawning** in this phase. Existing agents continue.
- If two consecutive agents fall below threshold, **drop the cap permanently** for the rest of this run.

This is the diminishing-returns heuristic in concrete form: keep spawning while parallel agents are pulling their weight; stop when they aren't.

---

## 5) Throttling triggers

Reduce active agent count immediately when ANY of:

- An API rate-limit error is observed (HTTP 429, "rate limit exceeded", "quota exceeded").
- Two parallel agents produce conflicting writes detected at merge time.
- The orchestrator detects two agents independently performing the same lookup or research task.
- The host system shows resource pressure (high CPU/memory) — heuristic, not a hard signal.

Throttle action: cancel the youngest agent that hasn't yet produced output, refund its allocated work to the queue for the next round.

---

## 6) Communication and merge protocol

Parallel agents do **not** communicate with each other. They report only to the orchestrator.

Each agent receives at spawn:
- A scoped task description.
- A scoped read budget (which files / docs they may read).
- A scoped write budget (which paths they may write — enforced at the worktree level).
- A token budget.
- An expected output schema.

Each agent returns:
- Its produced artifacts (files in the worktree, or a structured summary for research agents).
- A self-reported confidence and any flagged dependencies it discovered.

The orchestrator merges artifacts sequentially in a deterministic order, runs the parent's validation gate on the merged result, and records the orchestrator pass to the Resource Manifest.

---

## 7) Reporting

Per phase, log a single line:
```
[orchestrator] phase=<phase-name> tier=<serial|medium|high> spawned=<N> completed=<K> throttled=<M> cap=<C> reason=<why-cap>
```

At end of run, append a section to the parent skill's final report:
```
## Multi-agent activity
- Total agents spawned: <N>
- Total worktrees created: <W>
- Merge conflicts encountered: <C>
- Throttle events: <T>
- Phases that fell back to serial: [list]
- Estimated wall-time saved vs serial: <minutes> (best-effort estimate)
```

---

## 8) Interaction with sibling skills

- **error-mitigation-loop** receives all merge conflicts and throttle events as logged errors with `category: orchestration`. Recurring patterns (e.g. always-conflicting on a specific module) get promoted to constraints that future runs honor by tightening that module's tier to serial.
- **firebase-bootstrap** is always serial (deploys are serial by definition); orchestrator does not parallelize it.
- **anthropic-key-locator** is always serial (one key search per project); orchestrator does not parallelize it.
- **autonomous-project-agent** parent: the orchestrator runs *inside* each parent phase, not on top of it. The parent's gate-by-gate flow is unchanged.

---

## 9) Autopilot interaction

In autopilot mode (Section 0 of parent skill):
- Orchestrator runs without per-phase confirmation.
- Cap decisions are autonomous and logged as assumptions in the Resource Manifest.
- A phase that hits hard throttle-back is recorded but does not pause the run.
- Hard BLOCKs from the parent's auth security gate still halt deployment; orchestration does not override that floor.

---

## 10) Minimal operating rules

- Identify independence before spawning.
- Cap by the minimum of all governing limits.
- Use worktrees for any code-writing phase.
- Throttle on rate limits, conflicts, redundant work.
- Stop spawning when measured efficiency drops.
- Merge sequentially, never in parallel.
- Report once per phase, briefly.
- Surface failures to error-mitigation-loop.
- Never override serial tier on integration, deployment, or auth security gate.

---

## Short embed version

**Multi-Agent Orchestrator:**
At every parent-skill phase boundary → identify independent subtasks (disjoint file sets, no dep order, no shared state) → cap parallelism at `min(work, 6, file-set-disjoint, rate-budget, measured-efficiency)` → spawn each agent in an isolated git worktree → throttle on rate limits, merge conflicts, or redundant work → stop spawning when efficiency drops below half the rolling median → merge artifacts sequentially → report one summary line per phase. Autonomous. Never parallelizes serial-tier phases (integration, deployment, auth security gate).
