# Links — saas-habit-tracker

Auto-maintained, append-only log of every external resource this project uses.

## Projects

| Name | URL | Purpose | Added | Source-of-truth |
|---|---|---|---|---|
| Firebase Project (`saas-habit-tracker-d3f2`) | https://console.firebase.google.com/project/saas-habit-tracker-d3f2 | Active dev + prod deploy target | 2026-05-11 | y |
| GitHub repo | https://github.com/bkcavilee-hue/saas-habit-tracker | Source of truth for code | 2026-05-11 | y |
| Production URL | https://saas-habit-tracker-d3f2.web.app | Deployed app | 2026-05-11 | y |

## Services

| Name | URL | Purpose | Added | Source-of-truth |
|---|---|---|---|---|
| Firebase Auth | https://console.firebase.google.com/project/saas-habit-tracker-d3f2/authentication/users | Identity provider (email/password + Google) | 2026-05-11 | y |
| Firestore | https://console.firebase.google.com/project/saas-habit-tracker-d3f2/firestore | Primary database (habits, check-ins) | 2026-05-11 | y |
| Firebase Storage | https://console.firebase.google.com/project/saas-habit-tracker-d3f2/storage | Optional habit icons | 2026-05-11 | y |
| Firebase Hosting | https://console.firebase.google.com/project/saas-habit-tracker-d3f2/hosting | Static SPA delivery | 2026-05-11 | y |
| App Check | https://console.firebase.google.com/project/saas-habit-tracker-d3f2/appcheck | Anti-abuse layer (prod) | 2026-05-11 | y |

## APIs

| Name | URL | Purpose | Added | Source-of-truth |
|---|---|---|---|---|
| Firebase Auth REST | https://firebase.google.com/docs/reference/rest/auth | Reference for token validation | 2026-05-11 | n |
| Firestore client SDK | https://firebase.google.com/docs/firestore/quickstart | Primary data access path | 2026-05-11 | n |
| Firestore Security Rules language | https://firebase.google.com/docs/firestore/security/rules-conditions | For rules authoring | 2026-05-11 | n |

## Credentials Inventory

| Name | Source | Purpose | Added | Notes |
|---|---|---|---|---|
| Firebase web config | `.env.local` (gitignored) | Client SDK initialization | 2026-05-11 | Public-by-design (not a secret) but kept in env for clean code. |
| Firebase Admin SDK key | NOT USED in MVP | n/a | — | No server-side functions in MVP. |

**No Anthropic API key used in this project** — the agent didn't activate §22.
