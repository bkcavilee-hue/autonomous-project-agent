# autonomous-project-agent

A Claude Code plugin marketplace containing the **autonomous-project-agent** plugin — a single-skill, end-to-end autonomous project agent. Recommends **Firebase + React + React Context** as the default stack (unless intake says otherwise), runs **free-tier-first cost analysis** during research, enforces a **seven-layer security gate** before deployment (five auth layers + general security audit + **connectivity audit**), runs **emulator smoke tests** at pre-deployment, **bootstraps a private GitHub repo with per-phase PRs** into protected `main`, and embeds five autonomous subsystems (error-mitigation loop, Firebase CLI bootstrap, reference-mode Anthropic key wiring, multi-agent orchestrator, GitHub lifecycle) directly inside the main workflow. **Never auto-merges its own PRs** — those are the human review checkpoint.

---

## What's in this marketplace

### Plugin: `autonomous-project-agent` (v1.5.0)

**One consolidated skill** at `autonomous-project-agent/skills/autonomous-project-agent/SKILL.md`. Subsystems live as sections inside it:

| Section | Purpose |
|---|---|
| Boot Sequence | First two tool calls every run: load active constraints from prior errors + check for resumable run state. |
| §0 | Autopilot mode — "you decide" runs all phases continuously. |
| §1 | Intake gate. Captures Firebase project preference, stack override, cost ceiling, **GitHub visibility + user + branch strategy**. |
| §2 | Project detection. |
| §2a | Plan-preview gate — single one-page plan + one approval moment, then unattended. |
| §3 | Resource-gathering (reinforced) — HAVE/NEED/GAP inventory. |
| §4 + §4a | Research gate + free-tier-first cost analysis. |
| §5–§9 | Organize → model → workflow → validation → pre-deployment. |
| §7 | **Scaffolds Playwright smoke specs + MSW handlers per route/API client** during build. |
| §8 | Test-first ordering in code-mutating phases. |
| §9 | **Emulator smoke run** — every route, form, and API call exercised against Firebase emulators before §10. |
| **§10** | **Seven-layer security gate:** identity, session integrity, role enforcement, abuse protection, secrets hygiene, **general security audit, and connectivity audit** (the new Layer 7 catches "compiles fine but doesn't actually work" failures — broken imports, schema mismatches, orphaned routes, dead code). |
| §11 | Firebase deployment gate. |
| §12 | Reporting gate with full cost + security + PR-queue summary. |
| §19 | Default stack: Firebase + React + Vite + TypeScript + React Context; TanStack Query next; Zustand → Redux Toolkit only when measurably needed. |
| §20 | Error mitigation loop — autonomous capture + append-only constraint distillation. |
| §21 | Firebase bootstrap — autonomous CLI verify, project select/create, dev/staging/prod aliasing. |
| §22 | Reference-mode Anthropic key wiring — one canonical keystore at `~/.anthropic/api-key`; new projects reference by path, not by value. Includes `git check-ignore` enforcement + pre-commit hook that blocks any commit containing `sk-ant-…`. |
| §23 | Multi-agent orchestrator (honest reality version) — batch-level fanout, isolated git worktrees, sequential merge. |
| §24 | Run-state checkpointing — `<project>/ai_docs/run-state.json` enables resume after laptop sleep / session close / rate limit. |
| §25 | Anti-pattern catalog — 10 named failure modes the workflow is designed to prevent. |
| **§26** | **GitHub bootstrap & branch lifecycle** — `gh auth status` → pre-flight gitignore + credential scan → private repo with `gh repo create` → PR-only branch protection → per-phase PRs into `main` → Dependabot + secret scanning. **Never auto-merges.** |

**Slash commands:**
- `/agent-status` — prints the active run's phase, manifests, errors, constraints, worktrees, PRs. Read-only introspection.
- `/init-repo` — manual GitHub bootstrap with the same pre-flight safety checks as §26. For retrofitting existing projects.

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

**Two browser prompts are unavoidable** even in autopilot — these are external OAuth flows that genuinely require human interaction:
- `firebase login` (§21) — first time on a machine.
- `gh auth login` (§26) — first time on a machine.

After those one-time logins, subsequent runs are fully hands-off. The skill never inserts its own approval pauses, never auto-merges its own PRs (those stay open for human review), and can only run as freely as the harness permission layer allows.

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

For unattended end-to-end runs, add **"you decide"** — the agent picks defaults at every gate (Firebase + React + Context stack, free-tier-first cost stance, private GitHub repo, per-phase PRs), logs assumptions to the Resource Manifest, emits a one-page plan preview, auto-approves in autopilot, then runs continuously until completion. PRs accumulate against protected `main` for your review.

## See it in action

Three worked examples in [`examples/`](./examples/) show what an end-to-end run actually produces:

- [`examples/saas-habit-tracker/`](./examples/saas-habit-tracker/) — full SaaS run on the default stack. State management escalates from Context to TanStack Query. Cost stays $0/mo until ~25K MAU.
- [`examples/static-marketing-site/`](./examples/static-marketing-site/) — scope discipline in action. A one-page site gets a 4-phase workflow, not a 10-phase one.
- [`examples/internal-tool-with-rejection/`](./examples/internal-tool-with-rejection/) — **the most instructive example.** Demonstrates the §10 security gate actually rejecting a deploy (Layer 6 catches a transitive CVE, Layer 7 catches a Firestore schema mismatch), the agent running the §20 recovery loop autonomously, promoting three new constraints, and shipping only after the gate re-passes.

---

## Highlights

- **One trigger surface, fully autonomous on activation.** All subsystems run as sections inside one skill.
- **Boot-time institutional memory.** The agent reads `constraints.md` as its first action — past lessons actually apply to future runs.
- **One approval moment, then hands-off.** The §2a plan preview replaces both approval fatigue and silent runaway.
- **Resumable runs.** §24 run-state checkpointing means a laptop sleep or rate-limit pause doesn't lose the run.
- **Reference-mode credentials.** API keys live in one canonical keystore; projects reference by path. `git check-ignore` enforcement + pre-commit hook as defense in depth.
- **Free-tier-first.** Every service evaluated against its free tier. Blaze plan and pay-per-volume treated as zero-cost until usage actually meters.
- **Seven-layer security gate:** five auth layers + general security audit + **connectivity audit** that catches broken imports, schema mismatches, orphaned routes, and dead code that compiles fine but doesn't actually work.
- **Emulator smoke tests at pre-deployment.** Five-minute run through every route, form, and API call against Firebase emulators before §10.
- **Append-only everything:** error log, constraints, `links.md`, `run-state.json` all grow without ever rewriting prior entries.
- **GitHub-native lifecycle.** Private repo by default, per-phase PRs into protected `main`, Dependabot + secret scanning enabled. **Never auto-merges its own PRs** — the human review checkpoint stays human.
- **Anti-pattern catalog** (§25) documents 10 named failure modes the workflow prevents, each tied to the gate that catches it.

---

## Author

**Brandon Cavilee** — [@bkcavilee-hue](https://github.com/bkcavilee-hue)

## License

MIT — see [LICENSE](./LICENSE).
