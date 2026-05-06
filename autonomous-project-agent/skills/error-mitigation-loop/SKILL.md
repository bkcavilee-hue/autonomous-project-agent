---
name: error-mitigation-loop
description: 'Use this skill autonomously whenever an error, failed test, validation BLOCK, blocked tool call, or recoverable exception occurs during any phase of the autonomous-project-agent workflow (intake, detection, resource-gathering, research, organize, modeling, workflow, validation, pre-deployment, auth security, deployment) or during any general project execution. The skill captures what went wrong, what fixed it, and how to prevent it next time, then distills recurring patterns into append-only constraints that future runs inherit. Trigger on phrases like "got an error", "this failed", "the build broke", "tests are failing", "fix this", or any error/exception output observed in tool results. Runs autonomously — does not pause for confirmation. Append-only behavior — never modifies or deletes existing plugin files, only adds to its own log/constraint/mitigation files.'
---

# Error Mitigation Loop Skill
### Companion skill for autonomous-project-agent — Autonomous, append-only error capture and constraint distillation

---

## Core purpose

Turn every error encountered during a project run into durable institutional knowledge. Each error becomes:
1. A structured **log entry** (what happened, when, in which phase).
2. A **fix record** (the resolution applied).
3. A **mitigation rule** (how to prevent or detect this earlier next time).
4. When a pattern repeats, a **constraint** appended to the plugin's persistent constraints file so future runs inherit the lesson.

This skill is the autonomous-project-agent's memory of its own mistakes.

---

## 0) Autonomy contract

This skill operates **autonomously** by default. It does not ask the user for permission to log errors, write fixes, or distill constraints. It runs as a background companion to the main autonomous-project-agent workflow and surfaces a summary only at phase boundaries or when explicitly asked.

### Autonomous behavior
- Triggers on any observed error, exception, failed test, validation BLOCK, blocked tool call, or recoverable failure.
- Captures, classifies, and writes log entries without confirmation.
- Distills mitigation rules and constraints without confirmation.
- Surfaces a one-line acknowledgment per error (e.g. `[error-mitigation] logged: <short summary> → constraint added`).

### Hard limits on autonomy
- **Append-only.** This skill MUST NOT modify or delete any existing file in the plugin or in the user's project. It may only create new files or append to files it owns (listed in Section 3).
- **No edits to the parent skill.** This skill cannot rewrite `autonomous-project-agent/SKILL.md`, `plugin.json`, or any sibling skill.
- **No deletions, no overwrites, no truncation.** If a write operation would not be a pure append, abort and surface the issue.
- **No execution of remediation commands without user approval.** This skill records the fix that was applied; it does not re-run destructive actions on its own.

---

## 1) Activation triggers

The skill activates whenever ONE of the following is observed:

- A tool call returns a non-zero exit, exception trace, or error string.
- A test runner reports failures.
- A build, compile, lint, type-check, or deploy step fails.
- A validation gate (Section 8 of the parent skill) returns BLOCK.
- An authentication security gate layer returns FAIL or BLOCK.
- A resource manifest item flips from HAVE to NEED/GAP unexpectedly.
- The user types phrases like "got an error", "this failed", "broken", "fix this", "why did this fail", "regression".
- A previously-resolved error pattern recurs.

The skill does NOT activate for:
- Stylistic warnings without functional impact.
- Expected/intentional test failures inside characterization tests.
- User-driven cancellations or interrupts.

---

## 2) Capture format

For every triggering event, write a structured entry. The entry MUST contain all of the following fields:

```jsonl
{
  "id": "err-<utc-timestamp>-<short-hash>",
  "timestamp": "<ISO-8601 UTC>",
  "phase": "<intake|detection|resource-gathering|research|organize|modeling|workflow|validation|pre-deploy|auth-security|deployment|post-deploy|other>",
  "project_name": "<from intake, or 'unknown'>",
  "project_type": "<from detection, or 'unknown'>",
  "severity": "<low|medium|high|critical>",
  "category": "<config|dependency|auth|permissions|env|test|build|deploy|integration|data|logic|other>",
  "what_happened": "<concise, factual description of the failure>",
  "root_cause": "<the actual underlying cause, not the surface symptom>",
  "fix_applied": "<the exact change/command/config that resolved it>",
  "verification": "<how the fix was confirmed to work>",
  "mitigation_rule": "<one sentence: what to check or do BEFORE this can happen again>",
  "constraint_candidate": "<true|false> — whether this should be promoted to a persistent constraint",
  "related_resources": ["<resource manifest items touched>"],
  "tags": ["<freeform tags for grep>"]
}
```

### Field rules
- **root_cause** must go past the symptom. "Build failed" is not a root cause; "missing `firebase-admin` peer dep because Node version is 18 but the SDK requires 20" is.
- **fix_applied** must be specific enough that a future run can re-apply it without rediscovery.
- **mitigation_rule** must be phrased as a check, not a regret. Good: "Verify `node --version >= 20` before installing firebase-admin." Bad: "Should have checked Node version."

---

## 3) Files this skill owns (append-only)

The skill writes to exactly these files. It MUST NOT touch anything else.

### Per-project files (in the active project's working directory)
- `ai_docs/error-log/errors.jsonl` — one JSON object per line, one per error event. **Append-only.**
- `ai_docs/error-log/mitigations.md` — human-readable playbook of fixes and mitigations, grouped by category. **Append-only sections; never rewrite existing entries.**

### Plugin-level files (persistent across all projects)
- `<plugin-root>/skills/error-mitigation-loop/constraints.md` — distilled, deduplicated rules that the parent autonomous-project-agent skill should treat as additional gates. **Append-only.**
- `<plugin-root>/skills/error-mitigation-loop/pattern-index.jsonl` — recurring-pattern index used to detect when an error category has crossed the promotion threshold. **Append-only.**

### Append-only enforcement
Before any write, the skill MUST:
1. Confirm the target file is in the owned-files list above.
2. Read the file (if it exists) and confirm the new content is being added at the end, with no modification to prior bytes.
3. If the file does not exist, create it with a header comment and the first entry.
4. Never use truncating writes, replace operations, or in-place edits on these files. Use append semantics only (`>>` in shell, or read-then-write-with-prior-content-preserved when no append API exists).

---

## 4) Constraint promotion logic

A logged error becomes a **persistent constraint** in `constraints.md` when ANY of these is true:

- The same `category` + `root_cause` pattern has occurred **2 or more times** across runs (use `pattern-index.jsonl` to count).
- The error severity is `critical` (always promote, regardless of recurrence).
- The error involved an authentication security gate FAIL or BLOCK (always promote).
- The error caused data loss, leaked credentials, or required a rollback (always promote).
- The user explicitly says "remember this" or "make sure this doesn't happen again."

### Constraint format

Each constraint appended to `constraints.md` must follow:

```markdown
## C-<NNN> — <short title>

- **Category:** <config|dependency|auth|permissions|env|test|build|deploy|integration|data|logic|other>
- **Trigger phase:** <which gate(s) this constraint runs in>
- **Rule:** <imperative single sentence — what MUST happen or MUST NOT happen>
- **Pre-check:** <concrete check the parent skill should run before the trigger phase>
- **Failure mode if violated:** <what goes wrong>
- **Originating errors:** [<err-id>, <err-id>, ...]
- **Added:** <ISO-8601 date>
```

Constraint IDs are monotonically increasing (`C-001`, `C-002`, ...). Never reuse IDs.

---

## 5) How the parent skill consumes constraints

At the start of every autonomous-project-agent run (right after intake, before resource-gathering), the parent skill SHOULD read `<plugin-root>/skills/error-mitigation-loop/constraints.md` if it exists and treat each active constraint as an additional pre-check that must pass before its trigger phase completes.

This skill does NOT modify the parent skill to add this step. The integration point is documented here for the parent skill's authors to honor — and the parent's existing `Section 0 — Autopilot mode` already runs continuously, so adding a constraint-load step at intake is non-blocking.

If the parent skill has not yet been updated to read constraints, the constraints file still serves as a queryable reference for any operator or future skill version.

---

## 6) Output cadence

- **Per error:** One inline acknowledgment, format: `[error-mitigation] err-<id> | <category> | <severity> | logged → mitigations.md` and, if promoted, append `| constraint C-<NNN> added`.
- **Per phase boundary:** A two-line summary of errors logged in that phase (count, categories, any new constraints).
- **End-of-run report:** A consolidated section appended to the parent skill's final report, listing all errors this run, all new constraints, and any recurring patterns approaching the promotion threshold.

The skill does NOT spam per-line updates during execution beyond the single acknowledgment per error.

---

## 7) Interaction with autopilot mode

When the parent skill is in **Autopilot Mode** (Section 0 of the parent skill):
- This skill remains autonomous and does not insert any approval pauses.
- Constraint promotions happen automatically — no "should I add this?" prompt.
- If a logged error would cause a hard BLOCK (e.g. a critical auth failure), the parent skill's safety floor still halts deployment; this skill records the BLOCK as a `critical` entry and promotes a constraint immediately.
- All logging continues without interrupting the parent skill's flow.

---

## 8) Read-only access guarantees

This skill may **read** any file it needs to understand the error context (logs, source files, config, prior log entries). Reading does not violate the append-only contract.

This skill MUST NOT:
- Re-run the failing command (that's the parent skill's job, with user approval if outside autopilot).
- Rewrite source code to "fix" the error proactively (the parent skill applies fixes; this skill records them).
- Modify environment variables, settings files, or any project artifact other than its own owned files.
- Send error data anywhere outside the local filesystem (no network calls, no telemetry).

---

## 9) Error log hygiene

- Entries are immutable once written. Corrections go in NEW entries that reference the prior `id` via a `corrects: <id>` field.
- If the same exact error recurs (same hash of `category` + `root_cause` + `fix_applied`), still write a new entry — recurrence is signal, not noise. The `pattern-index.jsonl` counter ticks up.
- The log file may grow indefinitely. Rotation is a future concern; do NOT auto-delete.

---

## 10) Minimal operating rules

- Capture every qualifying error.
- Always write a `mitigation_rule` — a log without a forward-looking rule is incomplete.
- Promote to constraint when the threshold is met. Do not over-promote (avoid one-off transient errors becoming permanent rules).
- Append-only, always.
- Run autonomously. Surface concisely.
- Never modify the parent skill or any sibling file.
- Read freely; write only to owned files.

---

## 11) Final skill intent

The autonomous-project-agent learns from every run. This skill is the mechanism. It captures errors, distills mitigations, promotes recurring patterns into constraints, and gives the next run a shorter path to success — without ever changing the parent skill, without ever asking for permission to do its job, and without ever silently overwriting prior knowledge.

---

## Short embed version

**Error Mitigation Loop:**
On any error → capture {what, root cause, fix, mitigation rule} as JSONL → append to per-project `ai_docs/error-log/errors.jsonl` and `mitigations.md` → if pattern recurs ≥2× or severity is critical/auth/data-loss → append a numbered constraint to plugin-level `constraints.md` → parent skill consumes constraints at intake of next run. Append-only. Autonomous. Never modifies the parent skill or any file outside its owned set.
