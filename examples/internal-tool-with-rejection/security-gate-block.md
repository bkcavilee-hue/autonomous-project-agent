# Security Gate — first pass result: CONDITIONAL BLOCK

Run: `run-2026-05-12T14:33Z-b8f1`. First attempt at §10 after build phases completed.

## Layer-by-layer results

| Layer | Result | Findings |
|---|---|---|
| Layer 1 — Provider | ✓ PASS | Firebase Auth email + Google OAuth, restricted redirect URIs. |
| Layer 2 — Session | ✓ PASS | Firebase-managed tokens, server-side verification on Cloud Functions. |
| Layer 3 — Roles | ✓ PASS | Firestore rules restrict item writes to authenticated team members. Tested with emulator. |
| Layer 4 — Abuse | ✓ PASS | App Check enabled. Rate limits on signup. |
| Layer 5 — Secrets | ✓ PASS | No secrets in client bundle. `.env.local` gitignored. Pre-commit hook installed (§22.6). |
| **Layer 6 — General audit** | ⚠️ **FAIL** | See below. |
| **Layer 7 — Connectivity** | ⚠️ **FAIL** | See below. |

**Overall: CONDITIONAL BLOCK.** Two layers failed. Deployment paused until resolved.

---

## Layer 6 findings (general audit)

### Critical
- **`npm audit`** found **1 critical CVE** in transitive dependency `lodash@4.17.20` (used via `react-firebase-hooks → lodash`). Advisory: prototype pollution, CVE-2021-23337.
  - **Reachable?** Yes — `react-firebase-hooks` is imported in `src/lib/useUser.ts:3` and uses the affected `lodash.template` path.
  - **Severity:** BLOCK.
  - **Remediation:** upgrade `react-firebase-hooks` to >= 5.1.1 which pulls `lodash@4.17.21` (patched).

### Notes (PASS with conditions)
- 2 moderate CVEs in devDependencies, not reachable at runtime — logged but not blocking.
- HTTPS enforced by Firebase Hosting ✓
- CSP header configured ✓
- No `dangerouslySetInnerHTML` found ✓
- No `console.log` of PII in production build (one in dev was removed pre-build) ✓
- File uploads: not applicable to this project.

---

## Layer 7 findings (connectivity audit)

### Critical
- **Firestore schema mismatch.** `src/lib/items.ts:42` writes documents to `items/` collection with field `assignedTo: <uid>`. `src/components/CurrentHolder.tsx:18` reads from the same collection expecting field `currentHolder: <uid>`. **The reader will always show "no holder" because the field name doesn't match.**
  - **Severity:** BLOCK.
  - **Remediation:** pick one canonical name. `currentHolder` is more readable; update the writer.

- **Route declared without handler.** `src/routes.tsx:24` defines `/audit-log` but no `AuditLogPage` component is imported or exported anywhere in the project.
  - **Severity:** BLOCK.
  - **Remediation:** either implement `AuditLogPage` or remove the route from §7's deferred/nice-to-have queue.

### Warnings
- **Dead export.** `src/utils/formatDate.ts` exports `formatRelative` but nothing imports it.
- **Unused dependency.** `package.json` lists `lodash` as a direct dep; nothing imports it directly (`react-firebase-hooks` brings its own — see Layer 6 above).

---

## Recovery actions taken

The agent, still in autopilot mode, ran the following remediation loop (logged to §20):

1. **Layer 6 fix:** `npm install react-firebase-hooks@^5.1.1`. Re-ran `npm audit` → clean (0 critical, 0 high). Removed unused direct `lodash` dependency.
2. **Layer 7 fix (schema):** updated `src/lib/items.ts:42` to write `currentHolder` instead of `assignedTo`. Wrote a migration script `scripts/migrate-assignedTo-to-currentHolder.ts` for existing dev-environment data. Ran the migration against the Firestore emulator.
3. **Layer 7 fix (route):** removed `/audit-log` from `src/routes.tsx` and added it to the nice-to-have queue in the Resource Manifest (deferred to post-MVP).
4. **Re-run characterization tests:** all green.
5. **Re-run §9 emulator smoke:** all routes load, all forms submit, no console errors.
6. **Re-run §10 security gate:** see `final-report.md` for the second-pass result.

---

## Constraints promoted

Three constraints were promoted from this run's errors:

- **C-012** — Before §11 deployment, run `npm audit --production` and BLOCK on any critical or high severity in a reachable runtime path.
- **C-013** — When scaffolding Firestore writers and readers for the same collection, generate them from a shared TypeScript interface. Schema mismatch is silent until runtime.
- **C-014** — When adding a route to `src/routes.tsx`, the corresponding component must exist and be exported. Layer 7 will catch this, but the build phase should pre-check.

These are now active for all future runs.
