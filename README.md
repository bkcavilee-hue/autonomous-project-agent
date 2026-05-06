# autonomous-project-agent

A Claude Code plugin marketplace containing the **autonomous-project-agent** plugin — an end-to-end project agent that classifies, researches, plans, and ships projects through gated phases with audits between each, a five-layer authentication security gate, an autopilot mode for unattended runs, and an append-only error-mitigation loop that learns from past failures.

---

## What's in this marketplace

### Plugin: `autonomous-project-agent` (v1.1.0)

Two skills:

| Skill | Purpose |
|---|---|
| `autonomous-project-agent` | The main workflow. Intake → project detection → resource-gathering → research → organize → modeling → workflow construction → validation → pre-deployment → five-layer auth security gate → Firebase deployment → reporting. |
| `error-mitigation-loop` | Companion skill. Captures every error encountered during a run (what went wrong, what fixed it, how to mitigate next time), and promotes recurring patterns into append-only constraints that future runs inherit. |

Highlights:

- **Project-specific, not generic** — the workflow scales depth to scope and category.
- **Reinforced resource-gathering gate** — full HAVE / NEED / GAP inventory before any planning.
- **Five-layer authentication security gate** — identity verification, session/token integrity, role enforcement, abuse protection, secrets/credential hygiene. Hard BLOCKs stop deployment.
- **Autopilot mode** — say "you decide" / "you decided" to run all phases continuously without per-gate approval pauses.
- **Append-only error log + constraints** — the agent learns across runs without ever rewriting prior entries or modifying its own skill files.

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

For unattended end-to-end runs, add **"you decide"** to your prompt — the agent will pick reasonable defaults at every gate, log assumptions to the Resource Manifest, and run continuously until completion (or until a hard security/validation BLOCK).

---

## Author

**Brandon Cavilee** — [@bkcavilee-hue](https://github.com/bkcavilee-hue)

## License

MIT — see [LICENSE](./LICENSE).
