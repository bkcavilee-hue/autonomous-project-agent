# Plan preview — saas-habit-tracker

*(Emitted at §2a after intake and detection completed. In autopilot mode this is shown and the run auto-proceeds after a brief acknowledgment.)*

---

**Project:** saas-habit-tracker — a habit-tracking SaaS where users sign up, define habits, check them off daily, and see streaks. Web (responsive mobile + desktop).

**Detected:** SaaS / web-app, scope **medium**, complexity **medium**, risk profile **standard user-data**, approval tier **standard**.

## Recommended stack (default per §19, no override)

| Layer | Choice | Rationale |
|---|---|---|
| Frontend | React + Vite + TypeScript | Default stack. |
| State (local) | React Context | For auth state, theme. |
| State (server) | TanStack Query | Habits + check-ins are server state. Escalation from Context is appropriate here. |
| Auth | Firebase Auth (email/password + Google OAuth) | Default. |
| Database | Firestore | Habit and check-in data fit document model cleanly. |
| Storage | Firebase Storage | For optional habit icons (nice-to-have). |
| Server logic | Firebase Cloud Functions | Streak recomputation cron, only if streaks can't be computed client-side cheaply. *Skipped in MVP* — streaks compute from check-in timestamps on the client. |
| Hosting | Firebase Hosting | Static SPA. |
| Anti-abuse | App Check | Enabled in production. |

## Phase plan

1. **Resource-gathering** — confirm Node 20+, Firebase CLI, npm. Run §21 Firebase bootstrap. Run §22 Anthropic key locator (skipped — no Anthropic SDK in this project). Produce Resource Manifest.
2. **Research** — Firestore schema for habits + check-ins; client-side streak computation; auth UX patterns. Run §4a cost analysis.
3. **Organize / model** — HAVE/NEED/GAP inventory; user journey for signup → habit-create → check-off; data model.
4. **Workflow construction** — phased build plan.
5. **Build** — six sub-phases:
   - 5a. Vite + React + TS scaffold, Tailwind for styling.
   - 5b. Firebase project init, SDK install, env wiring.
   - 5c. Auth flow (signup, signin, signout, password reset).
   - 5d. Habit CRUD (Firestore + TanStack Query).
   - 5e. Daily check-off flow + streak display (client-side compute).
   - 5f. Responsive layout pass.
6. **Validation** — characterization tests per sub-phase (test-first per §8).
7. **Pre-deployment** — integration sanity, Firestore rules tested with emulator.
8. **Six-layer security gate** — full §10 pass including Layer 6 audit (deps, OWASP, HTTPS, etc.).
9. **Firebase deployment** — `firebase deploy` to dev alias first; if green, prompt for prod.
10. **Reporting** — §12 final report + Cost Manifest summary.

## Projected cost summary

- **At 0 users (build/test):** $0 / month — Spark tier covers everything.
- **At 100 MAU:** $0 / month — well within Spark free tier (50K Firestore reads/day, 20K writes/day, 10GB Hosting, 5GB Storage).
- **At 10K MAU:** $0 / month — still within Spark, assuming ~5 check-ins/user/day and ~20 reads/user/day.
- **At 100K MAU:** **first paid charge would land around ~25K MAU** as Firestore reads cross the free tier. Switch to Blaze pay-per-use at that point. Projected cost at 100K MAU: ~$15–30 / month.

**Free alternatives considered and substituted:**
- Cloud Messaging (for reminders) — **deferred to nice-to-have**, not in MVP.
- Algolia search — **substituted** with client-side filter (project has small per-user dataset).

**Adopted paid-only (none in MVP).**

## Security posture summary

- **Layer 1 (provider):** Firebase Auth with email/password and Google OAuth. OAuth redirect URIs restricted to deployed domain.
- **Layer 2 (session):** Firebase-managed ID tokens; server-side verification on Cloud Functions if any are added later.
- **Layer 3 (roles):** Firestore rules — users can read/write only their own habits; admin rules omitted (single-tenant per user).
- **Layer 4 (abuse):** App Check enabled in prod. Rate limiting on signup via Firebase quota.
- **Layer 5 (secrets):** No client-side secrets. `.env.local` referenced (no values in repo).
- **Layer 6 (general):** `npm audit` clean before deploy. HTTPS enforced by Firebase Hosting. CSP headers configured. No `dangerouslySetInnerHTML`.

**Auto-BLOCK conditions watched:** Firestore `allow read, write: if true`, hardcoded Firebase Admin key in client, missing HTTPS, high/critical dependency CVE.

## Estimated effort

- **Wall-time:** ~25–40 minutes if uninterrupted.
- **Tool calls:** ~80–120.
- **Files written:** ~30–40.

## Known assumptions (will be in Resource Manifest)

1. Target users 18–45.
2. MVP scope = signup + habit CRUD + daily check-off + streak display.
3. No reminders, no sharing, no categories in MVP (nice-to-have queue).
4. Client-side streak computation (no Cloud Functions in MVP).
5. Free-tier-strict — any service that would cost > $0 at projected light usage gets substituted or deferred.

---

> **Autopilot is engaged. Proceeding end-to-end. Reply 'stop' to abort before the next tool call.**
