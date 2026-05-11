# Intake — static-marketing-site

## User prompt
> "Build me a one-page landing site for my consulting business. Just needs a headline, three service blurbs, and a contact form. Should be free to host. You decide."

## Captured intake fields

| Field | Value | Source |
|---|---|---|
| Project name | `consulting-landing` | inferred |
| Project type | static website | inferred — "one-page landing site" |
| Project mode | new project | none mentioned |
| Scope | **narrow** | detection — single page, no auth, no DB |
| Must-have features | hero + headline, 3 service blurbs, contact form | explicit |
| Platform | web (responsive) | inferred |
| Budget | free | explicit |
| Firebase project preference | create new | autopilot default |
| Stack override | none | use default |
| Cost ceiling | free-tier-first-strict | explicit |

## Why this example exists

This intake describes a **narrow** project. Even though the user said "you decide," the agent must NOT escalate to the full SaaS workflow. Scope discipline (§15) says: if a project only needs a small tool, the workflow stays small.

The plan preview for this run will have **3 phases**, not 10. The detection gate must catch that there's no auth, no database, no multi-tenancy, and no server logic — just static rendering and a form submission endpoint.

This example is shorter than the SaaS one on purpose. The structure of this directory should reflect that: not every project produces a 15-file ai_docs/ tree.
