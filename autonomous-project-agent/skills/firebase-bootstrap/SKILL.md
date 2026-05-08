---
name: firebase-bootstrap
description: 'Use this skill autonomously whenever a project targets Firebase deployment or the user mentions Firebase, Firestore, Firebase Auth, Cloud Functions, or Firebase Hosting. Verifies the Firebase CLI is installed and authenticated, lists available Firebase projects, sets the active project with dev/staging/prod aliases, or creates a new project when no obvious match exists and the user is in autopilot. Logs the chosen project, console URL, and aliases into the per-project links.md. Triggers on phrases like "deploy to firebase", "use firestore", "set up firebase", "firebase project", or whenever the parent autonomous-project-agent enters its resource-gathering or pre-deployment gate with Firebase as the deployment target. Runs autonomously — does not pause for confirmation. Hard-blocks deployment if Firebase is not usable.'
---

# Firebase Bootstrap Skill
### Companion skill for autonomous-project-agent — Autonomous Firebase CLI verification, project selection, and aliasing

---

## Core purpose

Make sure that by the time the parent autonomous-project-agent reaches the Firebase deployment gate, the Firebase CLI is installed, authenticated, and pointed at the correct project for the correct environment. Capture every relevant link (console URL, project ID, aliases) into the per-project `links.md` log so that the operator never has to dig through `firebase use` history later.

---

## 0) Autonomy contract

This skill runs **autonomously** as part of the parent's resource-gathering gate (Section 3) and again at the pre-deployment gate (Section 9). It does not pause for confirmation in autopilot mode.

### Hard limits
- **Never** rotates or modifies an existing Firebase project's settings beyond `firebase use --add` aliasing.
- **Never** deploys, even if it could — deployment is the parent skill's job.
- **Never** writes the Firebase CI token, service account JSON, or any credential to `links.md` or any log. Only writes paths and console URLs.
- Hard-blocks the parent skill's deployment gate if any of the checks below FAIL after one auto-remediation attempt.

---

## 1) Activation triggers

Activates when ANY of the following is true:

- The parent skill's intake captured a deployment target containing "firebase", "firestore", "firebase auth", "cloud functions", "firebase hosting", or "firebase storage".
- The Resource Manifest contains any Firebase SDK in HAVE or NEED.
- The user types phrases like "deploy to firebase", "set up firebase", "use firestore", "firebase project".
- The parent skill enters its resource-gathering gate (Section 3) and Firebase appears anywhere in the project's tech stack.
- The parent skill enters its pre-deployment validation gate (Section 9).

---

## 2) Bootstrap sequence

Run these checks in order. On any FAIL, attempt the auto-remediation. If remediation fails, log to error-mitigation-loop and BLOCK.

### Step 1 — CLI presence
- Run `firebase --version`.
- If not installed → install instructions are surfaced (`npm install -g firebase-tools` or `brew install firebase-cli`). In autopilot, attempt `npm install -g firebase-tools` directly. If npm is unavailable, BLOCK with a clear message.

### Step 2 — Authentication
- Run `firebase login:list`.
- If no logged-in user → run `firebase login` (this opens a browser-based OAuth flow that requires user interaction; in autopilot mode, surface the URL and pause only for this single step — login cannot be automated and is treated as the one allowed prompt).
- Capture the authenticated email; record in the Resource Manifest under `auth_identity`.

### Step 3 — Project list
- Run `firebase projects:list --json`.
- Parse to a list of `{projectId, displayName, projectNumber}`.

### Step 4 — Project selection
Resolution order:
1. **Explicit user choice** captured at intake (Section 1 of parent skill, "Firebase project preference"). Use it as-is.
2. **Project name match** — if the project name from intake matches an existing `displayName` (case-insensitive, dash/underscore tolerant), use that.
3. **Autopilot create** — if no match and autopilot is engaged, run `firebase projects:create <generated-id> --display-name "<project name>"`. The generated ID follows pattern `<slug>-<short-hash>` to avoid collisions.
4. **Non-autopilot ambiguity** — present the list and ask which to use. (This is the only ask outside autopilot.)

### Step 5 — Environment aliasing
- Run `firebase use --add <projectId> --alias dev` (or `staging` / `prod`) for each environment the project plans to ship to.
- Record alias mappings in `<project>/.firebaserc` (managed by the CLI, not by this skill directly).
- Confirm aliases by running `firebase use` and capturing the output.

### Step 6 — Service enablement check
For each Firebase service the project will use (Auth, Firestore, Storage, Functions, Hosting, App Check, Remote Config, Analytics):
- Run `firebase apps:list` and `firebase functions:list` (if applicable).
- If a service is in NEED but not enabled in the project, surface the console URL for enablement. Do not attempt to enable services automatically — service enablement often involves billing changes which require explicit user action.

### Step 7 — Links log write
Append a structured entry to `<project>/ai_docs/links.md` under the **Projects** and **Services** sections (see Section 4).

---

## 3) Autopilot interaction

When the parent skill is in autopilot mode (Section 0 of parent):
- All steps run without confirmation **except** Step 2's `firebase login` browser flow, which is the one unavoidable interactive moment.
- Step 4 (project selection): when no match exists, **create a new project** rather than asking. The project name comes from the intake field added in v1.2.0 ("Firebase project preference — existing project ID or 'create new'").
- All decisions logged as explicit assumptions in the Resource Manifest with stated impact-if-wrong.

---

## 4) Links log integration

This skill is one of the writers of `<project>/ai_docs/links.md`. It appends to the **Projects** and **Services** sections only. **Never** writes to **Credentials Inventory**.

### Entry format

Under **Projects**:
```markdown
| Firebase Project | https://console.firebase.google.com/project/<projectId> | Active <env> deploy target | added <date> | source-of-truth: y |
```

Under **Services** (one row per enabled service):
```markdown
| Firebase Auth | https://console.firebase.google.com/project/<projectId>/authentication/users | Identity provider | added <date> | source-of-truth: y |
| Firestore | https://console.firebase.google.com/project/<projectId>/firestore | Primary database | added <date> | source-of-truth: y |
| Cloud Functions | https://console.firebase.google.com/project/<projectId>/functions | Server-side handlers | added <date> | source-of-truth: y |
```

### Append-only rule
Same contract as `error-mitigation-loop`: never modify or remove existing rows. Recurrences and aliasing changes get NEW rows referencing prior entries.

---

## 5) Failure handling

Every check that fails surfaces a structured event to `error-mitigation-loop`:
- `category: config`, `phase: pre-deploy`
- `root_cause`: the actual reason (CLI missing, not authenticated, no project, billing not enabled, etc.)
- `fix_applied`: what auto-remediation did, or "blocked — operator must resolve"
- `mitigation_rule`: how to verify before this can recur

Recurring failures (≥2×) get promoted to constraints in the plugin-level `constraints.md` (handled by error-mitigation-loop, not this skill).

---

## 6) Hard BLOCK conditions

Deployment cannot proceed if ANY of these is true after one auto-remediation attempt:
- Firebase CLI not installed.
- Not authenticated.
- No project selected (in non-autopilot, after the user has been asked).
- The selected project is in a different Firebase organization than the one specified at intake (cross-org accidents are common and dangerous).
- The active alias resolves to a `prod` project but the parent skill is still in `dev` deployment phase (or vice versa).

---

## 7) Minimal operating rules

- Run autonomously inside the parent's resource-gathering and pre-deployment gates.
- Auto-remediate where safe (install CLI, create project in autopilot).
- Never auto-deploy.
- Never log credentials.
- Append-only to `links.md`.
- BLOCK if any check fails after remediation.
- Surface failures to `error-mitigation-loop` for constraint promotion.

---

## Short embed version

**Firebase Bootstrap:**
On any Firebase-targeted project → check CLI installed → check authenticated → list projects → pick by intake preference, fuzzy name match, or autopilot-create → set dev/staging/prod aliases → confirm services enabled → append project & service URLs to `<project>/ai_docs/links.md` (Projects + Services sections, never Credentials) → BLOCK deployment if any check fails. Autonomous, never deploys, never logs secrets.
