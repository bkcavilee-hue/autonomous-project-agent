# Cost Manifest — saas-habit-tracker

Projected monthly cost across usage tiers. Goal: stay free as long as possible.

## Per-service evaluation

| Service | Free tier ceiling | Projected at light (100 MAU) | Projected at moderate (10K MAU) | Projected at heavy (100K MAU) | Free alternative considered | Decision |
|---|---|---|---|---|---|---|
| Firebase Hosting | 10 GB storage, 360 MB/day transfer | well within | within (~20 MB/day) | within (~200 MB/day) | n/a — default | **Adopt (Spark)** |
| Firestore | 50K reads/day, 20K writes/day, 1 GB storage | within | within (~30K reads/day, ~10K writes/day) | **exceeds free** at ~25K MAU | n/a — required | **Adopt (Spark → Blaze at ~25K MAU)** |
| Firebase Auth | 50K monthly active users | within | within | **at the edge** at 50K MAU | n/a — default | **Adopt (Spark)** |
| Firebase Storage | 5 GB | within (icons are optional) | within | within | n/a | **Adopt (Spark)** |
| App Check | free with Firebase | n/a | n/a | n/a | n/a | **Adopt** |
| Cloud Functions | 2M invocations/mo (Blaze free tier) | not used in MVP | not used | not used in MVP | client-side streak compute (substitute) | **Deferred** |
| Cloud Messaging (reminders) | free | not in MVP | not in MVP | not in MVP | n/a | **Deferred (nice-to-have)** |
| Sentry / error tracking | 5K events/mo | within | within | **exceeds** | console.error in MVP, add Sentry post-MVP if needed | **Deferred** |
| Algolia / search | 10K records, 10K ops/mo | n/a | n/a | n/a | client-side filter on per-user habits | **Substituted (free)** |

## Cost ladder

| Usage tier | Projected monthly cost | Notes |
|---|---|---|
| 0 MAU (build/test) | **$0** | Spark tier covers everything during development. |
| 100 MAU (light) | **$0** | Spark tier headroom is generous. |
| 10K MAU (moderate) | **$0** | Still within Spark — Firestore reads ≈ 30K/day vs 50K/day ceiling. |
| 25K MAU (free→paid crossover) | **~$3–5** | First Firestore reads charges begin. Blaze plan required from here on. |
| 100K MAU (heavy) | **~$15–30** | Firestore reads + writes dominate. Auth at 50K MAU ceiling. Bandwidth still cheap. |

## Adopted paid-only items
**None in MVP.** All adopted services are free-tier-sufficient under projected light-to-moderate usage.

## Free alternatives substituted
- **Algolia search → client-side filter** on per-user habits. Tradeoff: no fuzzy match, but per-user habit count is small (typically <50), so a plain array filter is plenty.
- **Cloud Functions streak job → client-side streak compute**. Tradeoff: streak recomputes on every render, but the data is small and React memoization makes this free.
- **Cloud Messaging reminders → deferred**. Reminders are nice-to-have. Adding them requires a service worker + Cloud Messaging setup, both free, but adds a build phase. Deferred to post-MVP.

## Cost ceiling check
User specified `free-tier-first-strict`. Current plan: $0/mo at light usage, first paid charge at ~25K MAU. **Within ceiling intent.**

If user's actual growth pushes past 25K MAU, the workflow should re-run cost analysis and either:
1. Switch to Blaze and accept ~$5–30/mo (recommended; trivial),
2. Add caching / batching on Firestore reads to extend free tier further,
3. Migrate read-heavy paths to a different store.
