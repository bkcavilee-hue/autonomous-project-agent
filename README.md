# autonomous-project-agent

A Claude Code plugin marketplace containing the **autonomous-project-agent** plugin — an end-to-end project agent that classifies, researches, plans, and ships projects through gated phases with audits between each, a five-layer authentication security gate, an autopilot mode for unattended runs, an append-only error-mitigation loop, autonomous Firebase CLI bootstrap, local-only Anthropic API key auto-discovery, and multi-agent orchestration governed by a diminishing-returns heuristic.

---

## What's in this marketplace

### Plugin: `autonomous-project-agent` (v1.2.0)

Five skills:

| Skill | Purpose |
|---|---|
| `autonomous-project-agent` | The main workflow. Intake → project detection → resource-gathering → research → organize → modeling → workflow construction → validation → pre-deployment → five-layer auth security gate → Firebase deployment → reporting. |
| `error-mitigation-loop` | Captures every error encountered during a run (what went wrong, what fixed it, how to mitigate next time), and promotes recurring patterns into append-only constraints that future runs inherit. |
| `firebase-bootstrap` | Verifies Firebase CLI install and auth, lists or creates the Firebase project, sets dev/staging/prod aliases, logs project + service URLs to `<project>/ai_docs/links.md`. Hard-blocks deployment if Firebase isn't usable. |
| `anthropic-key-locator` | When the project consumes the Anthropic API, searches local files only for an existing `ANTHROPIC_API_KEY`, auto-copies it into the new project's `.env`, and logs only the source path (never the key value). |
| `multi-agent-orchestrator` | Fans out independent subtasks to parallel sub-agents inside each phase, capped by a diminishing-returns heuristic (max 6, throttled on rate limits / merge conflicts / redundant work). Each parallel agent runs in an isolated git worktree. |

Highlights:

- **Project-specific, not generic** — the workflow scales depth to scope and category.
- **Reinforced resource-gathering gate** — full HAVE / NEED / GAP inventory before any planning.
- **Five-layer authentication security gate** — identity verification, session/token integrity, role enforcement, abuse protection, secrets/credential hygiene. Hard BLOCKs stop deployment.
- **Autopilot mode** — say "you decide" / "you decided" to run all phases continuously without per-gate approval pauses.
- **Append-only by design** — error log, constraints, and links log all grow without ever rewriting prior entries.
- **Per-project resource log** — `<project>/ai_docs/links.md` accumulates Projects / Services / APIs / Credentials Inventory entries as the agent works.

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

For unattended end-to-end runs, add **"you decide"** to your prompt — the agent will pick reasonable defaults at every gate, log assumptions to the Resource Manifest, fan out to parallel sub-agents where it helps, and run continuously until completion (or until a hard security/validation BLOCK).

---

## Author

**Brandon Cavilee** — [@bkcavilee-hue](https://github.com/bkcavilee-hue)

## License

MIT — see [LICENSE](./LICENSE).
