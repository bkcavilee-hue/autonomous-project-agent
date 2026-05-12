# Final Report — team-inventory-tool

Run: `run-2026-05-12T14:33Z-b8f1` — autopilot, end-to-end with one mid-run recovery loop.

## Overview

**Status:** PASS WITH NOTES on second security gate pass. Deployed to dev alias.
**Duration:** 47 minutes (33 build + 14 recovery & re-verify).
**Phases:** all completed.

## Phase summary

| Phase | First pass | Notes |
|---|---|---|
| Intake + detection + plan preview | ✓ | |
| Resource-gathering | ✓ | Firebase project created. GitHub repo created (private). |
| Research (incl. cost) | ✓ | $0/mo at projected light usage. |
| Organize + model | ✓ | |
| Workflow construction | ✓ | |
| Build (5 sub-phases, 5 PRs opened) | ✓ | All 5 PRs into protected `main`. None auto-merged. |
| Validation per-phase | ✓ | 18 characterization tests, all green. |
| Pre-deployment | ✓ | Emulator smoke: all routes loaded, all forms submitted, no console errors. |
| **Security gate (first pass)** | ⚠️ **CONDITIONAL BLOCK** | Layer 6 + Layer 7 both flagged. See `security-gate-block.md`. |
| **Recovery loop** | ✓ | 4 fixes applied autonomously, re-verified. |
| **Security gate (second pass)** | ✓ **PASS WITH NOTES** | All 7 layers cleared. 2 moderate devDep CVEs noted (not blocking). |
| Firebase deploy (dev) | ✓ | https://team-inventory-tool-7c2a.web.app |
| Reporting | ✓ | This document. |

## What got rejected and recovered

See `security-gate-block.md` for the detailed first-pass failure.

Summary: the build phase produced a tree that compiled, passed unit tests, and looked correct — but had two latent issues that only Layers 6 and 7 caught:

1. A transitive dependency CVE (`lodash@4.17.20` via `react-firebase-hooks@5.0.x`).
2. A Firestore schema mismatch — writer used `assignedTo`, reader used `currentHolder`. Would have shipped silently broken.

Both were caught at §10. Both were fixed autonomously per the §20 mitigation loop. Three constraints were promoted (C-012, C-013, C-014) that will run as pre-checks in all future Firebase-stack runs.

## Cost picture

| Tier | Monthly cost |
|---|---|
| Build phase | $0 |
| 50 team users (target) | $0 |
| 500 team users (growth) | $0 |
| 5,000 users (would require scope rethink) | ~$3–8 |

Internal tools rarely cross free tier at their projected user counts. Strict free-tier compliance maintained.

## GitHub state at end of run

| # | PR | Branch | Status |
|---|---|---|---|
| #1 | Phase 5a — Vite + React + TS scaffold | `feat/phase-5a-scaffold` | open, awaiting review |
| #2 | Phase 5b — Firebase init + Auth | `feat/phase-5b-firebase-auth` | open, awaiting review |
| #3 | Phase 5c — Items CRUD (Firestore) | `feat/phase-5c-items-crud` | open, awaiting review |
| #4 | Phase 5d — Search + current-holder view | `feat/phase-5d-search` | open, awaiting review |
| #5 | Phase 5e — Recovery fixes (deps + schema) | `feat/phase-5e-recovery-fixes` | open, awaiting review |

Repo: https://github.com/<operator>/team-inventory-tool (private). Branch protection on `main`: PR-only. Dependabot enabled. Secret scanning enabled (private repo with GHAS).

**No PR was auto-merged.** Operator reviews and merges at their own pace.

## Errors logged this run

8 errors logged to `ai_docs/error-log/errors.jsonl`:
- 5 build-phase issues (typos, missing imports during scaffolding) — fixed and logged, none promoted (single-occurrence).
- 1 Layer 6 critical (the `lodash` CVE) — **promoted to C-012**.
- 1 Layer 7 critical (schema mismatch) — **promoted to C-013**.
- 1 Layer 7 critical (orphaned route) — **promoted to C-014**.

## Multi-agent activity

- 4 batches emitted (resource-gathering, research, modeling, security gate).
- 11 sub-agents spawned across batches.
- 0 merge conflicts.
- 0 throttle events.

## What this example demonstrates

1. **The security gate actually rejects.** Layer 6 and Layer 7 both fired on real findings. Without those layers, this project would have shipped with a CVE in production code AND a silently-broken feature.

2. **Recovery is autonomous.** The agent didn't stop and ask "what should I do?" It ran the §20 mitigation loop, applied the four fixes, re-verified, and re-ran the gate. Operator visibility came at end-of-run, not mid-run.

3. **Constraints propagate.** Three constraints (C-012, C-013, C-014) were promoted. Future runs will run `npm audit --production` as a pre-check, will generate Firestore writer/reader pairs from shared types, and will verify routes have handlers before §10 is even reached. This run paid the cost of the discovery; future runs benefit.

4. **PRs preserve human oversight.** Even with full autonomy on the build + recovery, five PRs were opened against protected `main` and **none were auto-merged**. The operator gets to review what shipped before it ships.

5. **Polling caught two race conditions silently** *(noted as of v1.6.0)*. Pre-v1.6.0 this run would have produced two intermittent flaky failures that looked random:
   - **§9 emulator smoke** would have started running the Playwright spec before the Firestore emulator was listening on `:8080`, producing a "Connection refused" failure on roughly 1 in 4 runs. `poll_until_port_open` now waits for all five emulator ports before the spec runs. **Zero flakiness here.**
   - **§11 deploy verification** would have hit the deployed `web.app` URL during the 20–40 second CDN propagation window, getting a 404 and declaring deploy failure even though the deploy itself succeeded. `poll_until_http_ok` now waits up to 300s for the URL to actually serve 200. **In this run, the URL was live at 38 seconds — well within the polling window.**

   Neither catch generated a constraint (these aren't errors — they're expected propagation delays handled correctly). But without §27, they'd have surfaced as intermittent failures that the operator would have had to debug manually.
