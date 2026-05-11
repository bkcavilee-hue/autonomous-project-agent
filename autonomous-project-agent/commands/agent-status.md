---
description: Print a status report for the active autonomous-project-agent run — current phase, Resource Manifest, Cost Manifest, active constraints, errors logged this run, and active worktrees.
---

# /agent-status

Print a concise status report for the active autonomous-project-agent run in the current working directory. This is a read-only introspection command — it does NOT mutate any state.

## Steps

1. **Find the run state.** Read `ai_docs/run-state.json` in the current working directory. If it does not exist, output:
   > *"No active autonomous-project-agent run in this directory. Start one by describing a project."*
   and stop.

2. **Read companion files** if present:
   - `ai_docs/resource-manifest.md`
   - `ai_docs/cost-manifest.md`
   - `ai_docs/links.md`
   - `ai_docs/error-log/errors.jsonl` (just count entries; do not dump)
   - `ai_docs/error-log/mitigations.md` (just count entries)

3. **Read plugin-level constraints** that are currently in effect:
   - `<plugin-root>/skills/autonomous-project-agent/constraints.md` — list active constraint IDs and one-line titles only.

4. **Check for active worktrees.** Run `git worktree list` in the project root. Note any under `.worktrees/agent-*`.

5. **Emit a structured report** in this exact layout:

```
# autonomous-project-agent — status

**Run:** <run_id>
**Started:** <started_at> (<relative time ago>)
**Status:** <status>
**Current phase:** <current_phase>
**Next phase:** <next_phase>
**Phases completed:** <count> of <total> — [<list>]

## Stack decision
- Frontend: <frontend>
- State: <state>
- Backend: <backend>
- Overrides: [<list or "none">]

## Resource Manifest summary
- HAVE: <count>
- NEED: <count> (<count> resolved)
- GAP: <count> (<count> resolved as assumptions)
- Path: <resource_manifest_path>

## Cost Manifest summary
- Projected at light usage: <$/mo>
- Projected at moderate usage: <$/mo>
- Projected at heavy usage: <$/mo>
- Paid-only items adopted: <count>
- Free alternatives substituted: <count>
- Path: <cost_manifest_path>

## Errors this run
- Total logged: <count>
- By category: <breakdown>
- New constraints promoted this run: <list of IDs or "none">

## Active constraints (loaded at boot)
<one line per constraint: C-NNN — title>

## Active worktrees
<one line per worktree, or "none">

## Links log
- Projects: <count>
- Services: <count>
- APIs: <count>
- Credentials Inventory: <count>
- Path: <links_md_path>

## What's next
<one-line description of what the next phase will do>
```

6. **If autopilot mode is engaged**, append at the bottom:
   > *Autopilot mode is ON. The run will continue end-to-end unless you reply "stop" or "pause".*

## Notes

- This command never modifies state. Read-only.
- If the run state is `aborted` or `completed`, surface the reason / completion time prominently at the top.
- If multiple `run-state-*.json` files are present (historical), only the active `run-state.json` is reported — historicals are summarized at the bottom as "Prior runs: <count>".
