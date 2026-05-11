# autonomous-project-agent

A Claude Code plugin marketplace containing the **autonomous-project-agent** plugin — a single-skill, end-to-end autonomous project agent. Recommends **Firebase + React + React Context** as the default stack (unless intake says otherwise), runs **free-tier-first cost analysis** during research, enforces a **six-layer security gate** before deployment (five auth layers + a general security audit), and embeds four autonomous subsystems (error mitigation loop, Firebase CLI bootstrap, local-only Anthropic API key locator, multi-agent orchestrator) directly inside the main workflow.

---

## What's in this marketplace

### Plugin: `autonomous-project-agent` (v1.3.0)

**One consolidated skill** at `autonomous-project-agent/skills/autonomous-project-agent/SKILL.md`. Five subsystems live as sections inside it:

| Section | Purpose |
|---|---|
| §0–§18 | Main workflow: Intake → project detection → resource-gathering → research → organize → modeling → workflow construction → validation → pre-deployment → six-layer security gate → Firebase deployment → reporting. |
| §4a | Cost analysis & free-tier-first policy. |
| §10 Layer 6 | General security audit (deps, OWASP, injection, XSS, CSRF, SSRF, HTTPS, logging hygiene, exposed services, uploads, third-party JS). |
| §19 | Default technology stack: Firebase + React + React Context (TanStack Query next, then Zustand, then Redux Toolkit). |
| §20 | Error mitigation loop — autonomous error capture, mitigation rules, append-only constraint distillation. |
| §21 | Firebase bootstrap — autonomous CLI verification, project select/create, dev/staging/prod aliasing. |
| §22 | Anthropic API key locator — local-only key auto-discovery, auto-copy into `.env`, never logs key value. |
| §23 | Multi-agent orchestrator — parallel sub-agent fanout, isolated git worktrees, diminishing-returns cap. |

### Highlights

- **Single skill, truly autonomous on activation.** All subsystems run inside one workflow with one trigger surface.
- **Default stack:** Firebase (Auth, Firestore, Storage, Functions, Hosting) + React + Vite + TypeScript + React Context, escalating to TanStack Query when server state grows.
- **Free-tier-first:** every service is evaluated against its free tier. Blaze plan and pay-per-volume services are treated as zero-cost until usage actually meters. Paid-only features get an auto-substituted free alternative where viable, with the tradeoff documented.
- **Six-layer security gate:** identity, session integrity, role enforcement, abuse protection, secrets hygiene, **plus a general security audit** (dependency vulnerabilities, OWASP Top 10, injection, XSS/CSRF/SSRF, HTTPS/HSTS, logging hygiene, open ports/services, file uploads, third-party JS).
- **Autopilot mode:** say "you decide" / "you decided" to run all phases continuously without per-gate approval pauses.
- **Append-only by design:** error log, constraints, and `<project>/ai_docs/links.md` all grow without ever rewriting prior entries.
- **Per-project resource log:** `<project>/ai_docs/links.md` accumulates Projects / Services / APIs / Credentials Inventory entries as the agent works.

---

## Install

```
/plugin marketplace add bkcavilee-hue/autonomous-project-agent
/plugin install autonomous-project-agent@bkcavilee
```

Then trigger by describing a project to build:

> "Build me a SaaS for tracking …"
> "Spin up a Firebase-backed internal tool that …"
> "I want to make a game where …"

For unattended end-to-end runs, add **"you decide"** to your prompt — the agent will pick reasonable defaults at every gate (including the Firebase + React + React Context stack and the free-tier-first cost stance), log assumptions to the Resource Manifest, fan out to parallel sub-agents where it helps, and run continuously until completion (or until a hard security/validation BLOCK).

---

## Author

**Brandon Cavilee** — [@bkcavilee-hue](https://github.com/bkcavilee-hue)

## License

MIT — see [LICENSE](./LICENSE).
