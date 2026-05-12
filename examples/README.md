# Worked Examples

Three worked examples showing what an `autonomous-project-agent` run actually looks like end-to-end. Each example contains:

- **`intake.md`** — the initial user prompt and what intake captured.
- **`plan-preview.md`** — what the §2a plan preview emitted.
- **`resource-manifest.md`** — the HAVE / NEED / GAP inventory produced at §3.
- **`cost-manifest.md`** — the projected-cost ladder from §4a.
- **`links.md`** — the per-project link log (Projects / Services / APIs / Credentials Inventory) at end of run.
- **`final-report.md`** — the §12 reporting-gate output.
- **`constraints-fired.md`** — any active constraints that fired during the run (from `<plugin>/skills/autonomous-project-agent/constraints.md`).

These are reference transcripts, not runnable code. They exist to give the skill few-shot precedents and to give human operators a clear idea of what to expect when they kick off a run.

## Examples in this directory

| Example | Scope | Stack | Why it's useful |
|---|---|---|---|
| [`saas-habit-tracker/`](./saas-habit-tracker/) | Full SaaS — auth, db, hosting, multi-tenant | Firebase + React + React Context + TanStack Query | Shows the default stack end-to-end. State management escalates from Context to TanStack Query when server state grows. |
| [`static-marketing-site/`](./static-marketing-site/) | Narrow — single landing page | Vite + React + Firebase Hosting | Shows scope discipline — the agent does NOT escalate to a full SaaS workflow when the project is genuinely small. |
| [`internal-tool-with-rejection/`](./internal-tool-with-rejection/) | Medium internal tool with a security-gate rejection + autonomous recovery | Firebase + React + Context | **Demonstrates the §10 security gate actually rejecting a deploy** — Layer 6 catches a transitive CVE, Layer 7 catches a Firestore schema mismatch, the agent runs the §20 recovery loop, promotes three new constraints, and ships only after the gate re-passes. The most instructive of the three examples. |

## Reading these examples

These transcripts are slightly simplified for readability (some tool-call boilerplate elided), but the structure of every artifact (Resource Manifest, Cost Manifest, links.md, etc.) is exactly what the skill produces. The intent is for the skill itself to read these on later runs as anchor cases — and for you to read them to know what you're getting.

If you've just installed the plugin and want to know "what would this thing actually do if I asked it to build me a SaaS," start with `saas-habit-tracker/`.
