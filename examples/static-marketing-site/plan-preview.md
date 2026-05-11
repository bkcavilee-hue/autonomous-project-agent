# Plan preview — static-marketing-site

*(Emitted at §2a. Shorter than the SaaS example because scope discipline §15 kept it small.)*

---

**Project:** `consulting-landing` — a one-page static marketing site with a hero, three service blurbs, and a contact form.

**Detected:** static website / landing page, scope **narrow**, complexity **low**, risk profile **minimal** (no user data, no auth, no DB).

## Recommended stack

| Layer | Choice | Rationale |
|---|---|---|
| Frontend | React + Vite + TypeScript | Default stack, but truly overkill could be plain HTML+CSS. Keeping React because the user said "you decide" and React-based tooling is what the agent ships with the fewest surprises. |
| State | None — site is static | No interactive state beyond form submission. |
| Auth | None | No accounts. |
| Database | None | Form submissions handled via a single Cloud Function or third-party form endpoint. |
| Form submission | Firebase Cloud Function (Spark tier — free up to 125K invocations/mo) → email forwarding | Genuinely free; no third-party signup. |
| Hosting | Firebase Hosting | Free for static assets at this scale. |

## Phase plan (compressed — 4 phases, not 10)

1. **Resource-gathering + Firebase bootstrap** — combined since there's only one external dep.
2. **Build** — single phase:
   - Vite scaffold.
   - Tailwind for styling.
   - One-page layout with hero, services grid, contact form.
   - One Cloud Function: `submitContactForm` (validates input, sends email via Nodemailer + a free SMTP relay).
3. **Six-layer security gate (compressed)** — only Layer 4 (form abuse: captcha or honeypot), Layer 5 (no secrets in client), and Layer 6 (general audit). Layers 1–3 are trivially N/A.
4. **Deploy + report.**

## Cost picture

- All tiers: **$0/mo** — Spark tier covers a one-page site indefinitely unless the contact form gets DDoSed.

## Why the workflow is shorter

The agent applied **§15 scope discipline**. A static landing page does not need:
- A multi-phase build (the page is one component).
- A full six-layer auth gate (no auth exists).
- A cost ladder (cost is $0 at every tier).
- Multi-agent fanout (no independent parallel work).

The §2a plan preview is the proof: it took 3 phases to describe everything, not 10. If the agent had emitted a 10-phase plan for this project, that would be the symptom of §25.1 (overbuilt simple project) failing — the kind of thing the anti-pattern catalog exists to prevent.

---

> **Autopilot is engaged. Proceeding end-to-end.**
