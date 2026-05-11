# autonomous-project-agent

A Claude Code plugin marketplace containing the **autonomous-project-agent** plugin — a single-skill, end-to-end autonomous project agent. Recommends **Firebase + React + React Context** as the default stack (unless intake says otherwise), runs **free-tier-first cost analysis** during research, enforces a **six-layer security gate** before deployment (five auth layers + a general security audit), and embeds four autonomous subsystems (error-mitigation loop, Firebase CLI bootstrap, reference-mode Anthropic key wiring, multi-agent orchestrator) directly inside the main workflow.

---

## What's in this marketplace

### Plugin: `autonomous-project-agent` (v1.4.0)

**One consolidated skill** at `autonomous-project-agent/skills/autonomous-project-agent/SKILL.md`. Subsystems live as sections inside it:

| Section | Purpose |
|---|---|
| Boot Sequence | First two tool calls every run: load active constraints from prior errors + check for a resumable run state. |
| §0 | Autopilot mode — "you decide" runs all phases continuously. |
| §1 | Intake gate. Captures Firebase project preference, stack override, cost ceiling. |
| §2 | Project detection. |
| **§2a** | **Plan-preview gate** — single one-page plan + one approval moment, then unattended. |
| §3 | Resource-gathering (reinforced) — HAVE/NEED/GAP inventory. |
| §4 + **§4a** | Research gate + **free-tier-first cost analysis**. |
| §5–§9 | Organize → model → workflow → validation → pre-deployment. |
| **§8** | Validation gate with **test-first ordering** in code-mutating phases. |
| **§10** | **Six-layer security gate** (5 auth layers + general security audit: deps, OWASP, HTTPS, logging hygiene, exposed services, file uploads, third-party JS). |
| §11 | Firebase deployment gate. |
| §12 | Reporting gate with full cost + security summary. |
| §19 | **Default stack:** Firebase + React + Vite + TypeScript + React Context; TanStack Query next; Zustand → Redux Toolkit only when measurably needed. |
| §20 | Error mitigation loop — autonomous capture + append-only constraint distillation. |
| §21 | Firebase bootstrap — autonomous CLI verify, project select/create, dev/staging/prod aliasing. |
| §22 | **Reference-mode Anthropic key wiring** — one canonical keystore at `~/.anthropic/api-key`; new projects reference by path, not by value. Includes `git check-ignore` enforcement + pre-commit hook that blocks any commit containing `sk-ant-…`. |
| §23 | Multi-agent orchestrator (honest reality version) — batch-level fanout, isolated git worktrees, sequential merge. |
| **§24** | **Run-state checkpointing** — `<project>/ai_docs/run-state.json` enables resume after laptop sleep / session close / rate limit. |
| **§25** | **Anti-pattern catalog** — 10 named failure modes the skill is designed to prevent, each pointing to the gate that catches it. |

**Slash command:** `/agent-status` — prints the active run's phase, manifests, constraints, errors, and active worktrees. Read-only introspection.

---

## About the "autonomous" claim — read this before installing

This plugin's autopilot mode runs **the skill's** workflow continuously — no per-phase approval pauses inside the skill itself, and the §2a plan-preview gives you exactly **one** informed approval moment at the start.

But Claude Code's permission layer is **separate** from the skill: the harness still gates **tool calls** (Bash, Edit, Write, MCP, etc.) and prompts before each one by default. So on a fresh install, you'll see a permission prompt every time the skill wants to run a command — autopilot doesn't change that.

To get a fully unattended end-to-end run, do **one** of:

1. **Launch with the flag:** `claude --dangerously-skip-permissions` for that session.
2. **Accept the plan in plan-mode** — the harness then executes the whole plan without per-tool prompts.
3. **Set it as your default:** add to `~/.claude/settings.json`
   ```json
   "permissions": { "defaultMode": "bypassPermissions" }
   ```
   (Affects every Claude Code session, not just this plugin. Reversible by removing the key.)

The skill never inserts its own approval pauses — but it can only run as freely as the harness allows.

---

## Install

```
/plugin marketplace add bkcavilee-hue/autonomous-project-agent
/plugin install autonomous-project-agent@bkcavilee
```

Then trigger by describing a project to build:

> "Build me a SaaS for tracking habits …"
> "Spin up a Firebase-backed internal tool that …"
> "I want to make a static landing page for …"

For unattended end-to-end runs, add **"you decide"** — the agent picks defaults at every gate (Firebase + React + Context stack, free-tier-first cost stance), logs assumptions to the Resource Manifest, emits a one-page plan preview, auto-approves in autopilot, then runs continuously until completion (or until a hard security/validation BLOCK).

## See it in action

Three worked examples in [`examples/`](./examples/) show what an end-to-end run actually produces:

- [`examples/saas-habit-tracker/`](./examples/saas-habit-tracker/) — full SaaS run on the default stack. State management escalates from Context to TanStack Query. Cost stays $0/mo until ~25K MAU.
- [`examples/static-marketing-site/`](./examples/static-marketing-site/) — scope discipline in action. A one-page site gets a 4-phase workflow, not a 10-phase one.

Each example contains the captured intake, the plan preview, the Resource Manifest, the Cost Manifest, the per-project `links.md`, and the final §12 report.

---

## Highlights

- **One trigger surface, fully autonomous on activation.** All subsystems run as sections inside one skill.
- **Boot-time institutional memory.** The agent reads `constraints.md` as its first action — past lessons actually apply to future runs.
- **One approval moment, then hands-off.** The §2a plan preview replaces both approval fatigue (12 confirmations) and silent runaway (zero visibility).
- **Resumable runs.** §24 run-state checkpointing means a laptop sleep or rate-limit pause doesn't lose the run.
- **Reference-mode credentials.** API keys live in one canonical keystore; projects reference by path, with `git check-ignore` enforcement and a pre-commit hook as defense in depth.
- **Free-tier-first:** every service evaluated against its free tier. Blaze plan and pay-per-volume treated as zero-cost until usage actually meters.
- **Six-layer security gate:** identity, session integrity, role enforcement, abuse protection, secrets hygiene, **plus general security audit** (dependency CVEs, OWASP Top 10, injection, XSS/CSRF/SSRF, HTTPS/HSTS, logging hygiene, open ports, file uploads, third-party JS).
- **Append-only everything:** error log, constraints, `links.md` all grow without ever rewriting prior entries.
- **Anti-pattern catalog** (§25) documents the 10 failure modes the workflow is designed to prevent, each tied to the gate that catches it.

---

## Author

**Brandon Cavilee** — [@bkcavilee-hue](https://github.com/bkcavilee-hue)

## License

MIT — see [LICENSE](./LICENSE).
