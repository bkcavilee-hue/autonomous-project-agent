---
name: anthropic-key-locator
description: 'Use this skill autonomously whenever the project being built consumes the Anthropic API (Claude SDK in code, references to anthropic-ai/sdk or the anthropic Python package, or the user mentions needing an Anthropic API key for the new project). Searches local files only — shell rc files, .env files under common dev roots, and standard config locations — for an existing ANTHROPIC_API_KEY. If found, auto-copies the key into the new project .env file and logs only the source path (never the key value) into the per-project links.md Credentials Inventory. If no key is found, treats the project as new and writes a TODO link to console.anthropic.com/settings/keys. Triggers on phrases like "anthropic api key", "i need a claude api key", "wire up claude", or whenever the parent skill detects the Anthropic SDK as a dependency. Autonomous — auto-copies without prompting, never logs the key value.'
---

# Anthropic Key Locator Skill
### Companion skill for autonomous-project-agent — Autonomous local-only API key discovery and .env wiring

---

## Core purpose

When a project the agent is building needs to call the Anthropic API (i.e. the project itself uses Claude, separate from Claude Code the harness), this skill locates an existing `ANTHROPIC_API_KEY` from prior local projects and wires it into the new project's `.env` automatically. It never asks the user to paste a key, never logs the key value, and never queries any remote service.

---

## 0) Autonomy contract

This skill runs **autonomously** during the parent's resource-gathering gate when the project type or stack indicates Anthropic SDK use. Auto-copies on find, no prompts.

### Hard limits
- **Local filesystem reads only.** Never queries the Anthropic console, never makes any network call to discover keys.
- **Never logs the key value.** Logs only file paths, last-4 character previews (e.g. `sk-ant-…AB12`), and discovery timestamps.
- **Never overwrites an existing `.env` line.** If the new project's `.env` already has `ANTHROPIC_API_KEY`, leaves it alone and logs the find as an alternate.
- **Never copies a key to a shared/committed file.** Writes only to `.env` (which is assumed gitignored) and never to `.env.example`, `README.md`, or any tracked config.
- **Never uses or sends the key.** Discovery and wiring only — actual API calls are the project's job.

---

## 1) Activation triggers

Activates when ANY of the following is true:

- The Resource Manifest has `@anthropic-ai/sdk`, `anthropic` (Python), or any Anthropic API client in HAVE or NEED.
- Source code being scaffolded imports those packages.
- The user types phrases like "anthropic api key", "claude api key", "wire up claude", "i need an api key for claude".
- The parent skill detects an `ANTHROPIC_API_KEY` placeholder in any generated `.env.example` or scaffold file.

The skill does **not** activate for:
- Projects that only use Claude Code (the harness) — the harness has its own auth and doesn't need a project-level key.
- Projects where Anthropic is mentioned only in documentation, not in code.

---

## 2) Search locations (read-only)

Walk these locations in order. First match wins, but record all matches for the alternates list.

### 2a) Shell environment files
- `~/.zshrc`
- `~/.zprofile`
- `~/.bashrc`
- `~/.bash_profile`
- `~/.profile`
- `~/.config/fish/config.fish`

Pattern: `export ANTHROPIC_API_KEY=...` or `ANTHROPIC_API_KEY=...`.

### 2b) Project `.env*` files under common dev roots
Recurse (max depth 6) under each of:
- `~/Documents`
- `~/Code`
- `~/Projects`
- `~/Developer`
- `~/Sites`
- `~/Desktop`
- `~/Downloads`
- `~/dev`
- `~/repos`
- `~/workspace`

For files matching `.env`, `.env.local`, `.env.development`, `.env.production`, `.env.*` — search for line `ANTHROPIC_API_KEY=...`.

### 2c) Standard config locations
- `~/Library/Application Support/anthropic/*` (macOS)
- `~/.config/anthropic/*`
- `~/.anthropic/*`

### 2d) Out-of-scope (never read)
- `~/Library/Keychains/*` — never touched. macOS Keychain access requires user interaction.
- Any path inside `node_modules/`, `.git/objects/`, `vendor/`, `dist/`, `build/`, `.next/`, `.cache/`.
- Files larger than 1 MB (avoid scanning binary or generated content).
- Paths the user has not granted access to.

---

## 3) Selection logic

When multiple keys are found:
- Pick the key from the **most recently modified** source file. Modification time is a proxy for "still in active use."
- Skip keys that fail format validation (Anthropic API keys start with `sk-ant-`).
- Skip keys whose source file path contains `example`, `template`, `sample`, or `test` — those are placeholders.
- Record up to 5 alternates in the links log under "alternates."

---

## 4) Wiring action

Once a key is selected:

1. **Confirm the new project has a `.env` file.** If not, create one (this is allowed since `.env` is the standard target and is gitignored by convention).
2. **Confirm `.env` is in `.gitignore`.** If not, append `.env` to `.gitignore` (or create `.gitignore` with `.env` if missing). This is the only edit this skill makes outside its own log.
3. **Append `ANTHROPIC_API_KEY=<value>` to `.env`.** If the line already exists, leave the existing value and treat the located key as an alternate.
4. **Verify** by re-reading `.env` and confirming the line is present. Do not log the value.

---

## 5) Empty-state handling (no key found)

If no key is found anywhere in the search locations, treat this as a **new project** state:
- Write a TODO line to `<project>/ai_docs/links.md` Credentials Inventory:
  ```
  | ANTHROPIC_API_KEY | https://console.anthropic.com/settings/keys | Generate new key | TODO — not yet provisioned |
  ```
- Add an explicit assumption to the Resource Manifest: *"No existing ANTHROPIC_API_KEY found locally. Operator must generate one at console.anthropic.com/settings/keys before the project's API calls will work."*
- Do not block the build — scaffolding can proceed without the key. Block only at the final pre-deployment gate if the key is still missing.

---

## 6) Links log integration

Appends to `<project>/ai_docs/links.md` under the **Credentials Inventory** section. **Never** writes to Projects, Services, or APIs sections (those are owned by other writers).

### Entry format (key found)
```markdown
| ANTHROPIC_API_KEY | <source-file-path> | Auto-copied from existing project | found <date> | preview: sk-ant-…AB12 |
```

### Entry format (alternates, also found)
```markdown
| ANTHROPIC_API_KEY (alternate) | <other-source-path> | Not selected; most recent file won | found <date> | preview: sk-ant-…CD34 |
```

### Strict rule
- Source paths are recorded.
- Last-4 character previews are recorded for disambiguation.
- **Full key values are NEVER recorded in any log file, ever.**

---

## 7) Failure handling

If any of the following occurs, surface to `error-mitigation-loop`:
- A `.env` write fails (permission denied, disk full).
- The `.gitignore` edit cannot be made (read-only file, missing parent dir).
- A located key is malformed (does not start with `sk-ant-` or fails the basic format check).
- The skill discovers a key in a location it should not have read (e.g. inside a `node_modules` path that slipped through). Treat as a hygiene failure and surface immediately.

---

## 8) Cross-skill behavior

- This skill **reads** prior `links.md` Credentials Inventory entries (across projects on this machine) to deduplicate finds. It does **not** modify other projects' links.md.
- Constraint promotion via error-mitigation-loop applies normally — e.g. if multiple projects have repeatedly hit the empty-state, a constraint may be promoted suggesting a default key location.
- Plays nicely with `firebase-bootstrap`: both write to the same `links.md`, but to different sections.

---

## 9) Minimal operating rules

- Local files only.
- Auto-copy on find. No prompts.
- Never log the key value.
- Pick by recency among valid candidates.
- Always confirm `.env` is gitignored.
- Treat empty state as new-project setup — do not block.
- Append-only to `links.md`.
- Failures go to `error-mitigation-loop`.

---

## Short embed version

**Anthropic Key Locator:**
On any project consuming the Anthropic API → search local shell rc files, common dev-root `.env` files, and standard config dirs → pick the key from the most recently modified valid source → append to new project `.env`, ensure `.env` is gitignored → log the source path and last-4 preview to `<project>/ai_docs/links.md` Credentials Inventory (never the key value) → if no key found, write a TODO pointing to console.anthropic.com/settings/keys and continue. Autonomous, local-only, never logs the key value.
