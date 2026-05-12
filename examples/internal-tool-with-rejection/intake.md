# Intake — internal-tool-with-rejection

## User prompt
> "Build me an internal inventory tool for my team. Staff log items in/out, see who has what, search by name. You decide."

## Captured intake fields

| Field | Value | Source |
|---|---|---|
| Project name | `team-inventory-tool` | inferred |
| Project type | Internal tool, web app | inferred — "internal", "my team" |
| Scope | **medium** | detection — has auth, CRUD, search, multi-user |
| Target users | Team members of the operator (~10–50 people) | autopilot |
| Must-have features | item check-in/out log, current-holder display, name search, audit trail | inferred |
| Platform | Web (desktop primary, mobile responsive) | inferred |
| Budget | Free-tier-first | autopilot default |
| Firebase project preference | create new | autopilot default |
| Stack override | none — use default Firebase + React + Context | no override |
| GitHub visibility | **private** | autopilot default (correct for internal tool) |
| Branch strategy | per-phase-pr | autopilot default |

## Why this example exists

This example documents what happens when the §10 security gate **actually rejects** a deploy. Most worked examples show clean passes — but the catch-everything gate has to demonstrate that it catches things, otherwise it's just decorative.

The build phase in this run produces a `team-inventory-tool` that compiles cleanly, passes Layers 1–5, but **fails Layer 6** (an `npm audit` dependency with a known critical CVE in a transitive `lodash` path) **and fails Layer 7** (a Firestore schema mismatch: the writer code writes `assignedTo` but the reader code expects `currentHolder`).

The agent then runs the recovery loop: fix the dependency, fix the schema, re-run the gate, pass, deploy.

The interesting parts of this transcript are `security-gate-block.md` and `final-report.md`.
