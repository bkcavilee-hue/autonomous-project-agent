# Intake — saas-habit-tracker

## User prompt
> "Build me a habit tracker SaaS. Users sign up, create habits, check them off daily, see streaks. Should work on mobile and desktop. Goal is to keep it free to run for as long as possible. You decide on everything else."

## Captured intake fields

| Field | Value | Source |
|---|---|---|
| Project name | `saas-habit-tracker` | inferred from prompt |
| Project type | SaaS, web app | inferred — "SaaS", "users sign up" |
| Project mode | new project | no existing repo mentioned |
| One-sentence mission | "Help users build daily habits by tracking check-ins and streaks." | inferred |
| Problem statement | "Habit-tracking is high-friction in spreadsheets and most apps overcharge for basic streak/calendar features." | inferred from market context |
| Target users | "Individuals 18–45 building daily habits — fitness, study, mindfulness." | autopilot assumption (logged) |
| Platform | Web (mobile + desktop responsive) | inferred — "mobile and desktop" |
| Timeline | Not specified — autopilot assumes "MVP in one session" | autopilot assumption |
| Budget | "Free as long as possible" — `cost_ceiling: free-tier-first-strict` | explicit |
| Scope | Medium | detection |
| Must-have features | sign-up/sign-in, create habit, daily check-off, streak count, list view | inferred from prompt |
| Nice-to-have features | reminders, shareable streaks, habit categories, dark mode | autopilot assumption (logged) |
| Integrations | None required for MVP | autopilot — none mentioned |
| Security or compliance concerns | None special — standard user-data privacy | autopilot |
| Deployment target | Firebase Hosting | default stack §19 |
| Existing assets or constraints | None | none mentioned |
| Firebase project preference | `create new` | autopilot — no project mentioned |
| Stack override | None — use default | no override given |
| Cost ceiling | free-tier-first-strict | from explicit "free as long as possible" |

## Autopilot mode
Engaged — user said "you decide on everything else."

## Assumptions logged to Resource Manifest
1. **Target users 18–45.** Impact-if-wrong: visual design and copy may need to skew younger/older. Reversible.
2. **MVP in one session.** Impact-if-wrong: if user wanted a multi-week plan, the agent shipped too fast.
3. **No reminders in MVP.** Reminders require push notification infra (Cloud Messaging + service worker setup) which adds 1–2 phases. Logged as nice-to-have, deferred.
4. **No shareable streaks in MVP.** Sharing requires public read paths in Firestore rules which expands the auth gate surface. Deferred.
