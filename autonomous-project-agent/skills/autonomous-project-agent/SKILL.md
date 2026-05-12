---
name: autonomous-project-agent
description: 'Use this skill autonomously whenever the user describes a project they want built — a web app, SaaS, tool, game, automation, mobile app, or anything similar — and wants it researched, planned, and built end-to-end. The skill runs the full workflow (intake, project detection, plan-preview, resource-gathering, research with cost analysis, organize, modeling, workflow construction, validation, pre-deployment with emulator smoke tests, seven-layer security gate including connectivity audit, Firebase deployment with HTTP polling verification, GitHub repo + per-phase PR creation with CI status polling, reporting) and runs the embedded subsystems autonomously on activation: error capture and constraint distillation, Firebase CLI bootstrap with creation-propagation polling, reference-mode Anthropic key wiring (no value copied into project files), multi-agent orchestration with 429 backoff polling, run-state checkpointing, GitHub bootstrap with per-phase PRs, and polling discipline across all external-state waits. Recommends Firebase + React + React Context as the default stack unless intake specifies otherwise, treats free-tier-first as the explicit cost goal, and defaults to private GitHub repos with PR-only branch protection. Trigger on phrases like "build me X", "I want to make Y", "spin up Z", "create a project that", "ship a SaaS", any error/exception observed during a run, any mention of Firebase or Anthropic SDK usage, or any phase boundary with independent parallel work. Runs autonomously throughout — does not pause for confirmation outside the two allowed prompts (Firebase OAuth login, GitHub auth login). Never auto-merges its own PRs and never auto-promotes dev to prod (those are the human review checkpoints). Do NOT trigger for narrow single-step tasks (one bug fix, one function, one query).'
---

# Autonomous Project Research-to-Completion Skill
### Version 6 (v1.6.0) — Adds Polling discipline (§27) with four named helpers, retrofits §9/§11/§21/§23/§26 to use polling for emulator startup, deploy propagation, Firebase project propagation, 429 backoff, and PR CI status

---

## Core purpose

This skill takes a user's initial project articulation, identifies what kind of project it is, researches what it needs, organizes that information into a structured plan, converts the plan into phased execution, audits and tests each phase, runs a seven-layer security gate before final deployment (five auth layers plus a general security audit), and deploys through Firebase only when the correct environment and approval conditions are met.

The skill is **project-specific**, not generic. It must automatically determine which research, structure, logic, execution, and resource-gathering steps apply based on the project type chosen or detected. It must not force irrelevant sections onto the project.

This skill embeds four subsystems that run autonomously throughout every workflow: an append-only error-mitigation loop, an autonomous Firebase CLI bootstrap, a local-only Anthropic API key locator, and a multi-agent orchestrator that fans out independent subtasks under a diminishing-returns cap. All four are sections inside this document (Sections 20–23). They are not separate skills.

---

## Boot sequence *(runs before every other section)*

**The very first tool call when this skill activates MUST be a `Read` of the plugin-level constraints file**, if it exists:

```
Read("<plugin-root>/skills/autonomous-project-agent/constraints.md")
```

If the file exists, load every active constraint into working context. Each constraint is a pre-check that has been promoted from a past error (per Section 20.4) and MUST be honored before its trigger phase completes. This is how the skill consumes its own institutional memory.

If the file does not exist (fresh install, no prior runs have promoted constraints yet), continue normally — there's nothing to load.

**The second tool call MUST check for a resumable run state** in the current working directory:

```
Read("<project>/ai_docs/run-state.json")
```

If `run-state.json` exists and its `status` is `in-progress`, this is a resumed run — pick up at the phase named in `next_phase`, with the prior `resource_manifest_path` and `cost_manifest_path` loaded. See Section 24 for run-state semantics.

If `run-state.json` does not exist or its `status` is `completed` or `aborted`, this is a fresh run — proceed to Section 0.

These two reads are **mandatory** at the very start of every activation. They are not subject to autopilot's "skip approval pauses" rule because they don't require user approval — they're just file reads.

---

## 0) Autopilot mode — "you decide" / "you decided"

If the user says **"you decide"**, **"you decided"**, **"you choose"**, **"go autonomous"**, or any equivalent phrase that hands decision authority to the skill, enter **Autopilot Mode** for the remainder of the run.

### Autopilot mode behavior
- **Run all phases continuously without pausing for user approval between gates.** Do not stop at the end of intake, detection, resource-gathering, research, organize, modeling, workflow construction, validation, pre-deployment, security, or deployment gates to ask "should I continue?" — just continue.
- **Make decisions on behalf of the user for any ambiguous intake field, project-type detection, scope sizing, framework choice, deployment target, or resource gap.** Pick the most reasonable default given context, document the choice in the Resource Manifest as an explicit assumption with stated impact-if-wrong, and proceed.
- **Default to Firebase + React + React Context** (see Section 19) unless intake says otherwise.
- **Default to free-tier-first cost stance** (see Section 4a) when picking services.
- **Do not ask clarification questions during intake.** Infer answers from the user's initial articulation. If a field cannot be inferred, assign a reasonable default and continue.
- **Treat every "PASS WITH NOTES" or "PASS WITH CONDITIONS" outcome as proceed-eligible.** Do not pause to confirm.
- **Skip every interactive checkpoint, confirmation prompt, or "ready for next phase?" gate.** The skill runs end-to-end as one continuous flow.
- **Surface a single consolidated report at the end** rather than per-phase status updates, unless a hard BLOCK occurs.

### What autopilot mode does NOT change
- **Hard BLOCKs in the security gate still block.** A `FULL BLOCK` (e.g. unauthenticated writes in prod rules, exposed Firebase Admin key, no server-side token validation, critical CVE in a runtime dependency) stops deployment. Autopilot does not override security blocks — it only removes user-approval pauses, not safety floors.
- **A phase that fails validation twice still stops** per Section 8.
- **Tool-permission prompts at the Claude Code harness level are NOT controlled by this skill.** If the user wants the harness itself to stop prompting for tool approvals (Bash, Edit, Write, etc.), they must either run with `--dangerously-skip-permissions`, configure `permissions.defaultMode: "bypassPermissions"` in `settings.json`, or accept a plan in plan-mode. The skill cannot grant itself those permissions — but inside whatever permission envelope the harness provides, the skill will run continuously without adding its own approval pauses.
- **The one unavoidable Firebase login pause.** Section 21 may need `firebase login`, which opens a browser OAuth flow. That single interaction is allowed even in autopilot.

### Autopilot mode acknowledgement
On entering autopilot mode, the skill must briefly acknowledge: *"Autopilot mode engaged — running all phases continuously. Decisions logged as assumptions in the Resource Manifest. Hard security blocks still stop deployment."* Then begin immediately.

### Exiting autopilot mode
Autopilot mode persists for the rest of the run unless the user says **"stop"**, **"pause"**, **"wait"**, or asks a direct question. On exit, the skill returns to normal gated behavior at the next phase boundary.

### Embedded subsystems active in autopilot
The four embedded subsystems run autonomously inside autopilot mode without inserting their own approval pauses:
- **Section 20 — Error mitigation loop** captures every error/failed test/validation BLOCK as a structured JSONL entry, distills mitigation rules, and promotes recurring patterns into append-only constraints that future runs inherit.
- **Section 21 — Firebase bootstrap** verifies the CLI, authenticates, selects or creates the Firebase project, sets dev/staging/prod aliases, and logs project + service URLs to `<project>/ai_docs/links.md`.
- **Section 22 — Anthropic API key locator** scans local files only for an existing `ANTHROPIC_API_KEY`, auto-copies into the new project's `.env`, and logs only the source path (never the key value).
- **Section 23 — Multi-agent orchestration** fans out independent subtasks to parallel sub-agents under a diminishing-returns cap (max 6 simultaneous), each in an isolated git worktree.

---

## 1) Intake gate

The skill begins only after the user provides the initial project idea and the system asks the minimum necessary clarification questions.

### Intake must capture
- Project name.
- Project type.
- Project mode.
- One-sentence mission.
- Problem statement.
- Target users.
- Platform.
- Timeline.
- Budget.
- Scope.
- Must-have features.
- Nice-to-have features.
- Integrations.
- Security or compliance concerns.
- Deployment target.
- Existing assets or constraints.
- Firebase project preference (existing project ID, or 'create new'). *(Consumed by Section 21 at the resource-gathering gate.)*
- Stack override (if any). The skill defaults to **Firebase + React + React Context** (see Section 19). If the user wants something different — different framework, different state library, different backend — this is where it gets captured.
- Cost ceiling (optional). Defaults to "free-tier-first" per Section 4a.
- **GitHub visibility preference.** Defaults to `private`. Public requires explicit override. *(Consumed by Section 26.)*
- **GitHub user/org.** Defaults to the user detected by `gh auth status`. Override if pushing to an org.
- **Branch strategy.** Defaults to `per-phase-pr` (each build sub-phase → its own PR into protected `main`). Alternatives: `single-pr` (one PR at end of run) or `no-prs` (push directly to `main` — only allowed for solo private repos with explicit override). *(Consumed by Section 26.)*

### Intake rule
If critical information is missing, the skill must ask targeted follow-up questions before moving forward.
If more than half of the critical information is missing, it must ask all relevant questions for the detected project type.

In autopilot mode, no clarification questions are asked — the skill infers from the initial articulation and logs every inference as an explicit assumption in the Resource Manifest.

---

## 2) Project detection gate

After intake, the skill must classify the project.

### It must detect
- Project category.
- Scope size.
- Complexity level.
- Delivery size.
- Risk profile.
- Approval tier.
- Platform boundaries.
- Whether the project is narrow, medium, broad, or complex.

### Project categories
- Web app.
- SaaS.
- Website.
- Internal tool.
- Automation.
- Workflow system.
- Game.
- Hybrid software.
- Research/analysis.
- Content system.

### Detection rule
The skill must identify what the project actually needs before researching deeply. It must not assume every project needs the same research depth, structure, or execution path.

---

## 2a) Plan-preview gate *(new in v1.4.0)*

Between project detection and resource-gathering, the skill emits a **single one-page plan preview** and asks for one approval. This is the only user-approval moment in the entire workflow. After this, the skill runs end-to-end without pauses (except the unavoidable Firebase OAuth login in §21).

### The plan preview must contain

- **Project summary:** one paragraph restating what's being built, in the skill's own words.
- **Detected category, scope, complexity, risk tier.**
- **Recommended stack** (Section 19) — what the skill will use unless overridden.
- **Phase plan:** the workflow phases that will run (intake done → resource-gathering → research → ... → deploy). Brief.
- **Projected cost summary:** the Cost Manifest's headline numbers — `$0 at projected light usage; first paid charge would land at ~X MAU.`
- **Security posture summary:** which security gate layers will run and which findings would be auto-BLOCK. **All seven layers** (five auth + general audit + connectivity audit, see §10).
- **GitHub plan:** repo URL that will be created (`<user>/<project-name>`), visibility (default `private`), branch strategy (default `per-phase-pr` — one PR per build sub-phase into protected `main`), expected PR count. *(See §26.)*
- **Estimated wall-time and rough effort** ("12–18 tool calls, ~Z minutes if uninterrupted").
- **Known assumptions** that will be logged to the Resource Manifest if not overridden.

### The single approval moment

After emitting the preview, the skill asks **one** question:

> *"Proceed with this plan? Reply OK to run end-to-end. Reply with overrides (e.g. 'use postgres', 'skip auth gate', 'state with zustand') to adjust before kickoff."*

### Approval outcomes

- **"OK" / "yes" / "proceed" / "go"** → run end-to-end with no further pauses. This is the path autopilot mode takes by default.
- **Overrides specified** → apply the overrides to the Resource Manifest as pinned decisions (per §I-style overrides), re-emit the preview once with the changes incorporated, ask again.
- **"stop" / "cancel" / "no"** → abort the run cleanly, no artifacts written, no Firebase project created.

### Autopilot interaction

In autopilot mode (Section 0), the skill **still emits the plan preview** but auto-approves and proceeds after a brief acknowledgment:

> *"Plan preview above. Autopilot mode is engaged — proceeding end-to-end. Reply 'stop' to abort before the next tool call."*

This gives the user a single chance to catch a wildly wrong plan before the run goes silent for the next 30+ minutes. Even autopilot benefits from this checkpoint.

### Why this gate exists

Per-phase approvals (the v1.3.0 default) generate either approval-fatigue (12 confirmations per run) or full silence (autopilot, never asks). Neither is right. One informed plan-time approval, then unattended, is the actually-useful UX. This gate compresses that intent into a single moment.

---

## 3) Resource-gathering gate *(reinforced)*

Before any planning begins, the skill must run a complete resource inventory. This is a **blocking gate** — planning cannot proceed until the inventory is sufficiently resolved for the project's type and risk level.

### Purpose of this gate
The resource-gathering gate exists to prevent planning on top of unknown foundations. It forces the system to answer: *What do we actually have, what do we actually need, and what is still missing?* before a single phase is designed.

### The skill must gather and classify all of the following

**Development tools**
- IDEs, editors, CLI tools, terminal environments.
- Version control systems and repo access.
- Local dev environment and OS constraints.
- Containerization or virtualization requirements.

**Frameworks, libraries, and packages**
- Frontend frameworks and component libraries.
- Backend frameworks.
- State management libraries.
- Testing libraries.
- Build tools and bundlers.
- Linting and formatting tools.
- Package managers.

**APIs and external services**
- First-party APIs with endpoint documentation.
- Third-party APIs with version and auth method.
- Webhooks and event sources.
- Payment processors.
- Messaging and notification services.
- Analytics and tracking services.
- AI/ML model APIs.
- Maps, geo, or location services.

**SDKs and platform-specific tools**
- Mobile SDKs.
- Platform SDKs (e.g. Firebase SDK, Stripe SDK, Twilio SDK).
- Browser extension APIs.
- Native OS APIs.

**Configuration and environment**
- Environment variables and .env structure.
- Secret stores and credential vaults.
- Feature flags.
- Runtime configs.
- CI/CD pipeline configs.
- Build and deploy scripts.
- Dockerfile or container specs.
- Infrastructure-as-code files.

**Design and brand assets**
- Wireframes and mockups.
- Design system or brand guide.
- Typography and font files.
- Icon sets.
- Image assets.
- Video or audio assets.
- Color tokens.
- Spacing and grid specs.

**Data assets**
- Data schemas and ERDs.
- Seed data and test fixtures.
- Migration scripts.
- Sample API payloads.
- Reference data files.
- Source data exports.

**Authentication and security resources**
- Auth provider selection (Firebase Auth by default; Auth0, Clerk, Supabase, custom JWT only if explicitly overridden).
- Session strategy (cookie-based, token-based, stateless, stateful).
- Role and permission definitions.
- OAuth providers required (Google, GitHub, Apple, etc.).
- MFA requirements.
- App Check or equivalent anti-abuse layer.
- Security rules files (Firebase Security Rules, Row-Level Security, etc.).
- Secrets rotation policy.
- Rate limiting configuration.

**Content and copy assets**
- User-facing copy and microcopy.
- Error state messages.
- Onboarding content.
- Legal and policy references.
- Accessibility references.
- Localization or i18n resources.

**Operational resources**
- Logging configs and log destinations.
- Monitoring and alerting tools.
- Observability dashboards.
- Analytics tags and tracking IDs.
- Rollback artifacts.
- Release notes templates.
- Support and incident runbooks.

**Documentation and reference materials**
- API documentation.
- Architecture decision records.
- README files.
- Compliance documentation.
- Third-party terms of service references.

**Permissions, licenses, and compliance**
- API keys and access tokens.
- OAuth credentials.
- License agreements for libraries and assets.
- GDPR, HIPAA, SOC2, or other compliance constraints.
- Data residency requirements.

### Resource classification rule
Every identified item must be sorted into exactly one of three states:

| State | Meaning |
|-------|---------|
| **HAVE** | Already available and confirmed accessible. |
| **NEED** | Required but not yet obtained or configured. |
| **GAP** | Unknown, unclear, or blocking — cannot proceed without resolution. |

### NEED resolution rule
Every item in NEED must be mapped to:
- what permission or access is required to obtain it,
- what environment it belongs to,
- what other items depend on it being available first (dependency order).

### GAP resolution rule
Every item in GAP must become one of two things before planning proceeds:
- A **question** directed at the user with a specific answer needed.
- An **explicit assumption** with a clearly stated impact if the assumption turns out to be wrong.

### Resource sufficiency rule
The skill must not move to the research gate or planning gate until:
- All NEED items have a resolution path.
- All GAPs are converted to answered questions or accepted assumptions.
- Authentication and security resource requirements are at minimum identified, even if not fully configured.

### Resource manifest output
At the end of this gate, the skill must produce a **Resource Manifest** — a structured inventory that shows every item, its state (HAVE / NEED / GAP), its dependency chain, and any outstanding questions or assumptions. This manifest becomes a living reference for the rest of execution.

### Section 21 hand-off
At this gate the skill also runs **Section 21 — Firebase bootstrap** to verify the Firebase CLI, authenticate, and select/create the active Firebase project. Section 21 writes its findings (project ID, console URL, aliases) back into the Resource Manifest.

### Section 22 hand-off
If the Resource Manifest contains any Anthropic SDK (`@anthropic-ai/sdk`, `anthropic` Python package, or equivalent) in HAVE or NEED, the skill also runs **Section 22 — Anthropic API key locator** to discover an existing local key and wire it into the new project's `.env`.

---

## 4) Research gate

The skill researches only what the project needs.

### Research should cover
- Problem space.
- Comparable solutions.
- User roles.
- User journeys.
- Functional requirements.
- Non-functional requirements.
- Data and state model.
- Technical options.
- Dependencies.
- Constraints.
- Risks.
- Security concerns.
- Deployment and environment needs.
- **Cost** (see Section 4a).

### Research rule
The research depth must scale to project scope and complexity.

- Narrow project → lean, focused research.
- Medium project → moderate research and structure.
- Broad project → recommend narrowing or splitting.
- Complex project → deeper architecture, dependency, and risk analysis.

### Research output must exclude
- Irrelevant sections.
- Universal templates that do not fit the project.
- Unnecessary depth for simple projects.
- Shallow treatment of complex systems.

---

## 4a) Cost analysis & free-tier-first policy *(new in v1.3.0)*

The skill must run cost analysis as part of the research gate. The default stance is **as free as we can get it** — keep the project's running cost as close to zero as possible while still meeting the must-have features.

### Cost evaluation
For every candidate service, API, library, or platform considered, the skill must evaluate:

1. **Free tier limits.** What's the free quota? (e.g. Firestore: 50K reads/20K writes per day, Firebase Hosting: 10GB storage / 360MB/day transfer, Cloud Functions: 2M invocations/month on Blaze with free tier, etc.)
2. **Paid tier shape.** Pay-per-use, flat monthly, seat-based, volume-tiered.
3. **Likely project usage.** Based on target users and scope captured at intake, estimate whether the project will stay within free tier under realistic usage.
4. **Cost ceiling estimate.** What does the project cost per month at:
   - Zero users (build/test phase)
   - Light usage (e.g. 100 monthly actives)
   - Moderate usage (e.g. 10K monthly actives)
   - Heavy usage (e.g. 100K monthly actives)

### Selection policy

In order of preference:

1. **Fully free service that meets requirements** — pick it.
2. **Pay-per-volume service that's free under projected usage** — pick it. Firebase Blaze plan, Anthropic API metered billing, Stripe transactional pricing, Twilio per-message pricing, etc. all fall here. These stay free until usage actually crosses the meter, so they're treated as zero-cost during build and light usage.
3. **Paid-only service with a free alternative** — **substitute the free alternative automatically** and note the tradeoff in the Resource Manifest. Example: managed search → Algolia paid vs. Firestore + client-side full-text. Example: custom domain TLS → managed cert via Firebase Hosting (free) vs. external CDN paid plan. Surface the substitution and its capability/limit difference.
4. **Paid-only with no viable free alternative** — adopt it, but loudly flag the cost line in the Resource Manifest and in the final report. Continue without pausing in autopilot, but make sure the operator sees the line item.

### Blaze plan note
Firebase Blaze is pay-as-you-go and includes the same free tier as Spark. The skill treats Blaze as a free-by-default choice when the project needs Functions outbound network access, larger Storage, or higher Firestore quotas. It surfaces the standing cost as "$0 at current projected usage; first paid charge would land at ~$X/mo at heavy-usage projection."

### Cost manifest output
The research gate's output must include a **Cost Manifest** appended to the Resource Manifest, with columns:
| Service | Free tier ceiling | Projected usage | Projected monthly cost | Free alternative considered | Decision |

The Cost Manifest is also surfaced in the final report (Section 12).

### When the operator overrides the free-first policy
If intake captured a cost ceiling or a specific paid tool preference, that wins. The skill still documents the free alternative it would have picked, for future reference.

---

## 5) Organize gate

The skill must sort all findings into three buckets.

### HAVE
Assets, tools, repo access, environments, services, credentials, libraries, references, or existing systems already available.

### NEED
Missing services, modules, permissions, approvals, environments, integrations, resources, or decisions.

### GAPS
Blocking unknowns that prevent safe planning.

### Organize rule
Every NEED must be mapped to required permissions, environment, and dependency chain.
Every GAP must be turned into either a question to the user or an explicit assumption with impact if wrong.

---

## 6) Modeling gate

The skill must model the people and flow involved.

### Required models
- Primary user.
- Secondary actors.
- User journey.
- Drop-off risks.
- Interaction points.
- Permissions model.
- Operator needs where relevant.

### System model must include
- Inputs.
- Outputs.
- States.
- Transitions.
- Persistence.
- Sync behavior.
- Data sensitivity.
- Failure modes.

This is mandatory because it prevents shallow planning and forces the system to understand what is being built before execution.

---

## 7) Workflow construction gate

The skill must convert research into a phased build workflow.

### Default progression
1. Smallest useful function.
2. Core workflow.
3. Dependent modules.
4. Integrated system.
5. Validation layer.
6. **Seven-layer security gate** *(see Section 10)*.
7. Deployment layer.

### Workflow rule
Each phase must build on the previous one and become more complex only after the simpler version is validated.

### The workflow must include
- Feature map.
- System map.
- Dependency map.
- State map.
- Test map.
- Auth and security map.
- Cost map.
- Deployment map.
- **Connectivity map** *(new in v1.5.0)* — the graph of which components import which, which routes have which handlers, which collections are read/written by which code paths. Consumed by §10 Layer 7 (connectivity audit).
- **Smoke-test map** *(new in v1.5.0)* — the list of routes, forms, and API client calls that will get scaffolded Playwright specs and MSW contract handlers during the build phase. Consumed by §9 pre-deployment emulator smoke run.

### Smoke-test scaffolding (built during build phases, run at §9) *(new in v1.5.0)*

During build sub-phases that produce routes, forms, or API clients, the skill auto-generates:

- **`e2e/smoke.spec.ts`** — one Playwright test per route: load the page, assert no console errors, assert no network 4xx/5xx. For auth-gated routes, log in as a seeded emulator user first.
- **`src/mocks/handlers.ts`** — one MSW handler per external API client call (each `httpsCallable`, `fetch`, SDK call). Each handler validates the request shape and returns a typed fixture response. Mismatches between client expectation and handler shape become §10 Layer 7 BLOCK findings.
- **`firebase.json` updates** — Auth, Firestore, Functions, Storage emulators configured so smoke tests can run end-to-end without burning real-service quota.

The smoke suite is **emulator-only**. It is NOT a contract test against production APIs and NOT a load/security/accessibility audit. It is a five-minute "do the wires actually connect?" verification.

### Scope rule
The workflow must be project-specific. If a project only needs a small tool, the workflow should stay small. If it is a large SaaS or internal platform, the workflow should deepen accordingly.

---

## 8) Validation gate

Every phase must be validated before the next phase begins.

### Each phase must have
- a deliverable,
- a test,
- an audit,
- a success criterion,
- a rollback or recovery path when needed.

### Validation must check
- correctness,
- completeness,
- security,
- scope fit,
- dependency fit,
- readiness for the next step.

### Validation outcomes
- PASS.
- PASS WITH NOTES.
- BLOCK.

If a phase fails twice, the skill must stop and surface the failure with remediations.

### Test-first ordering within code-mutating phases *(new in v1.4.0)*

For any phase that mutates code (scaffolding, build, integration, fixes), the **default order is test-first**:

1. **Write the characterization test FIRST.** Describe the expected behavior as a failing test.
2. **Run the test.** Confirm it fails for the expected reason (not for an unrelated setup error). A test that passes immediately means the test isn't actually exercising the behavior.
3. **Write the minimum code to make it pass.** Resist the urge to add adjacent functionality.
4. **Run the test again.** Confirm it now passes.
5. **Run the broader suite.** Confirm nothing else broke.
6. **Refactor if useful.** Keep the test green throughout.

This is the default. Exceptions where test-first is skipped:

- Pure UI scaffolding with no behavior to test yet (still write a render test).
- One-line config changes.
- Generated boilerplate (e.g. `vite create`) where the framework provides its own scaffold validation.

When test-first is skipped, log the reason as an assumption in the Resource Manifest.

---

## 9) Pre-deployment validation gate

Before the security gate and deployment gate are reached, a final pre-deployment validation pass must be run on the integrated system.

### This pass must confirm
- All phases validated individually have been re-confirmed in integration.
- No new regressions introduced during integration.
- All resource manifest items marked NEED are now resolved.
- All GAPs marked as assumptions have been reviewed and accepted.
- Environment is confirmed as the correct target (dev, staging, or prod).
- Cost Manifest is up to date and within ceiling (if a ceiling is set).
- **Smoke test suite (built during §7) runs green against the Firebase emulator suite** *(new in v1.5.0)*.

### Emulator smoke run *(new in v1.5.0, polling-aware as of v1.6.0)*

1. **Start the Firebase emulator suite in the background:** `firebase emulators:start --only auth,firestore,functions,storage,hosting &`.
2. **Wait for emulators to actually be listening** via `poll_until_port_open` (§27) for each emulator's default port:
   - Firestore: `localhost:8080`
   - Auth: `localhost:9099`
   - Functions: `localhost:5001`
   - Storage: `localhost:9199`
   - Hosting: `localhost:5000`
   Default timeout: 30s per port. If any port fails to open, BLOCK and surface to §20 — do not run the smoke spec against half-started emulators.
3. **Build the app:** `npm run build`.
4. **Run the smoke spec:** `npx playwright test e2e/smoke.spec.ts` against the built app pointed at emulators.
5. **Verify:** every route loads with no console errors, every form submits successfully, every API client call hits the expected emulator endpoint with the expected request shape and receives the expected response shape.
6. **Tear down emulators** when done (`kill` the background process).
7. PASS → continue to §10. FAIL → BLOCK, surface failing route/form/call to operator, log to §20.

**Scope of the smoke run:**
- Emulator-only. Does NOT hit production or staging Firebase projects.
- Five minutes wall-time budget. Anything longer means the smoke suite has overgrown — refactor to keep it lean.
- Smoke covers connectivity, not behavior coverage. Full integration testing is the team's job post-MVP.

---

## 10) Seven-layer security gate

This is a **mandatory blocking gate** that runs immediately before deployment. Nothing deploys until this gate passes.

### Purpose
The security gate exists to verify that the system's identity, access control, trust model, and general security posture are correctly implemented and cannot be bypassed, misconfigured, or exploited before real users encounter it.

### Gate structure

This gate has **seven layers**. All seven must pass before deployment proceeds. Layers 1–5 are authentication-focused; Layer 6 is a broader security audit; Layer 7 is a connectivity audit (introduced in v1.5.0).

---

### Layer 1 — Identity and provider verification

Verify that the authentication provider is correctly configured and operational.

**Checks include:**
- Auth provider is initialized in the correct Firebase project (not a dev project accidentally pointed at prod, or vice versa).
- Sign-in methods are explicitly enabled only for methods the product requires (no accidental open OAuth providers).
- OAuth redirect URIs are allowlisted correctly and do not include localhost in production.
- API keys scoped to the auth provider are restricted to allowed domains/IPs.
- Firebase Auth emulator is confirmed NOT running in production environments.
- Email link, phone, or passwordless flows (if used) are tested end-to-end.
- Social auth providers (Google, Apple, GitHub, etc.) are tested in the production OAuth app, not development credentials.

**Gate outcome:** PASS / FAIL with specific provider and config listed.

---

### Layer 2 — Session and token integrity

Verify that sessions and tokens are issued, validated, and expired correctly.

**Checks include:**
- ID tokens are verified server-side on every protected request. Client-side trust alone is a FAIL.
- Token expiration is set and enforced. Tokens that never expire are a FAIL.
- Refresh token rotation is confirmed if applicable.
- Session fixation risks are assessed and mitigated.
- Token claims (UID, email, custom claims, roles) are correct and not injectable.
- Signed-out sessions are invalidated on the server, not just the client.
- JWTs (if custom) use a strong algorithm (RS256 or ES256). HS256 with a weak or hardcoded secret is a FAIL.
- Session cookies (if used) have Secure, HttpOnly, and SameSite flags set.

**Gate outcome:** PASS / FAIL per token or session mechanism with finding listed.

---

### Layer 3 — Role and permission enforcement

Verify that access control rules are correctly applied at every layer.

**Checks include:**
- Every protected route, API endpoint, and data query requires a valid authenticated session before returning data.
- Role assignments (admin, editor, viewer, etc.) are enforced server-side and in security rules — not just client-side.
- Firebase Security Rules are audited for any rule that allows unauthenticated reads or writes where they should not.
- No rule uses `allow read, write: if true` in production. This is an automatic BLOCK.
- Firestore and Storage rules are tested with both authenticated and unauthenticated test users to confirm correct behavior.
- Privilege escalation paths are identified and tested — can a regular user claim an admin role?
- Multi-tenant isolation is verified if applicable — can User A access User B's data?
- Admin-only operations require re-authentication or elevated verification before execution.

**Gate outcome:** PASS / FAIL per role or permission layer with specifics listed.

---

### Layer 4 — Attack surface and abuse protection

Verify that authentication flows cannot be abused, brute-forced, enumerated, or bypassed.

**Checks include:**
- Rate limiting is applied to sign-in, sign-up, password reset, and OTP endpoints.
- Account enumeration via error messages is mitigated (login errors should not confirm whether an email exists).
- Password reset flows use short-lived tokens and invalidate after use.
- Firebase App Check (or equivalent) is enabled and enforced in production to block unauthorized API access.
- CORS policies are restrictive — only known origins are allowed.
- Referrer policies are set correctly.
- Auth-related endpoints are not exposed via debug routes or admin panels without their own auth layer.
- Captcha or bot protection is implemented on public-facing auth forms where appropriate.
- Auth bypass via parameter tampering (e.g. `?admin=true` in URL) is tested and blocked.

**Gate outcome:** PASS / FAIL per attack surface with remediation required before proceeding.

---

### Layer 5 — Secrets, keys, and credential hygiene

Verify that no credentials, secrets, or keys are exposed or misconfigured.

**Checks include:**
- No Firebase config object, API key, secret, or credential is hardcoded in client-side code.
- Environment variables are confirmed loaded from a secrets store or .env file that is not committed to the repo.
- `.env` is listed in `.gitignore` and confirmed absent from version control history.
- Firebase Admin SDK private key is stored server-side only. It must never appear in client-side bundles.
- API keys have domain or IP restrictions applied in the Firebase console and Google Cloud console.
- Service account keys follow least-privilege principle — no key has broader access than it needs.
- Key rotation dates are recorded and a rotation schedule exists.
- Any previously exposed key (committed to git, logged, or leaked) is considered compromised and must be rotated before deployment.
- Secret scanning has been run on the full repo before deployment (e.g. truffleHog, git-secrets, or equivalent).

**Gate outcome:** PASS / FAIL with specific credential or config listed.

---

### Layer 6 — General security audit *(new in v1.3.0)*

Verify the project's broader security posture beyond authentication. This layer catches the issues that don't fit under auth-specific layers but can still ship a vulnerable product.

**Checks include:**

**Dependency vulnerabilities**
- Run the language's standard vulnerability scanner: `npm audit --production`, `yarn audit`, `pnpm audit`, `pip-audit`, `cargo audit`, `bundle audit`, `mix deps.audit`, etc.
- Any dependency with a **critical** or **high** severity vulnerability is a FAIL until updated or patched.
- Vulnerabilities in transitive dependencies are evaluated by whether the vulnerable path is actually reachable; surface but do not auto-BLOCK if unreachable.

**Input validation and injection**
- All user-supplied input that reaches a database query, a shell command, an HTTP request, an HTML render, or a file path is validated and/or parameterized.
- SQL injection: parameterized queries / prepared statements only. Concatenated SQL is a FAIL.
- NoSQL injection (Firestore where applicable): query operators are not user-controllable in unsafe ways.
- Command injection: no shelling out with user-controlled strings; `child_process.exec` with template literals is a FAIL.
- Path traversal: file paths derived from user input are normalized and bounded to expected directories.

**XSS**
- React JSX usage is reviewed; any `dangerouslySetInnerHTML` is justified and the content is sanitized.
- Server-rendered HTML escapes by default.
- Content Security Policy (CSP) headers are present in production.
- No inline event handlers in server-rendered HTML.

**CSRF**
- State-changing endpoints require a CSRF token or rely on SameSite cookies plus origin checks.
- GET endpoints never mutate state.

**SSRF and external request safety**
- Any outbound HTTP request from the server side that uses a user-supplied URL is filtered (no internal IPs, no metadata endpoints, no localhost).

**HTTPS / TLS**
- All public endpoints serve over HTTPS only.
- HSTS header is set in production.
- Insecure cookie flags are not used.

**Sensitive data logging**
- No PII, no secrets, no full session tokens, no full credit card numbers, no passwords appear in logs.
- Error messages surfaced to clients do not include stack traces, internal paths, or DB schema details in production.

**Open ports and exposed services**
- Cloud project exposes only the services that need to be public.
- No development services (databases, admin UIs, debug consoles) are publicly reachable in production.
- Firewall / Firebase Functions ingress rules are reviewed.

**File upload safety (if applicable)**
- Uploaded files are content-type validated, size-limited, and stored outside the web root or in object storage with restricted access.
- Filenames are sanitized; original filenames are not used as paths.
- Image uploads are re-encoded server-side where feasible to strip metadata and embedded payloads.

**Third-party JS**
- Any third-party script loaded in the frontend is reviewed: known vendor, served over HTTPS, ideally with Subresource Integrity (SRI) hash.

**Gate outcome:** PASS / FAIL per check group, with specific findings listed. Critical findings (auto-BLOCK): unparameterized SQL, exposed dev services in prod, unsanitized `dangerouslySetInnerHTML` of user content, missing HTTPS on public endpoints, secrets in logs, dependency with known active exploitation.

---

### Layer 7 — Connectivity audit *(new in v1.5.0)*

Verify the project's wiring graph: every export is imported somewhere it matters, every import resolves, every route has a handler, every collection read has a corresponding writer with a matching schema. This layer catches "compiles fine but doesn't actually work" failures that pass Layers 1–6 but break at runtime.

This is a **static** graph check over the source code. Tools used (auto-installed during resource-gathering, auto-detected by language):

| Language | Primary tools |
|---|---|
| TypeScript / JavaScript | `knip` (exports, files, dependencies), `madge` (cycles), `dependency-cruiser` (custom rules) |
| Python | `vulture` (dead code), `pyflakes` (undefined names), `import-linter` (architectural rules) |
| Go | `staticcheck`, `go vet` |
| Rust | `cargo clippy --warn dead_code` |

**Checks include:**

**Dead code & unused exports**
- Functions, classes, components exported but never imported anywhere.
- Files that exist but are imported by nothing.
- Dependencies in `package.json` / `requirements.txt` not actually used.

**Broken imports**
- `import { foo } from './bar'` where `bar` doesn't export `foo`.
- Imports resolving to `undefined`.
- Type imports that don't match runtime imports.

**Route / handler mismatches** *(Firebase-stack-specific extension)*
- React Router routes defined with no corresponding component.
- Cloud Functions defined with no callers, or callers to non-existent functions.
- Firestore collection paths referenced in security rules but never written by any code path.
- API client calls (e.g. `httpsCallable("foo")`) where no function named `foo` exists.

**Schema consistency**
- Firestore reads expecting fields that no write path produces.
- TypeScript interfaces used in client code that don't match server return types.
- Form field names that don't match the schema they POST to.
- MSW handler shapes (from §7's smoke-test scaffolding) that don't match the client call shape they're matched against.

**Cyclic dependencies**
- Detected via `madge` for JS/TS or equivalent. Cycles WARN unless they cross architectural boundaries (e.g. UI imports from data layer that imports back to UI), which BLOCK.

**Severity ladder:**

| Finding | Severity |
|---|---|
| Missing import (resolves to nothing) | **BLOCK** |
| Route declared without handler | **BLOCK** |
| Handler declared without route | **WARN** (might be called dynamically) |
| Firestore reader expects field never written | **BLOCK** |
| Firestore writer produces field never read | **WARN** |
| Schema mismatch between client and server types | **BLOCK** |
| MSW handler shape mismatch with client call | **BLOCK** |
| Dead export | **WARN** |
| Unused file | **WARN** |
| Unused dependency in `package.json` / `requirements.txt` | **WARN** |
| Architectural cycle | **BLOCK** |
| Non-architectural cycle | **WARN** |

**Gate outcome:** PASS / FAIL per check group with specific findings listed. Any BLOCK → CONDITIONAL BLOCK at the overall gate level. Multiple BLOCKs or critical patterns (e.g. all routes missing handlers) → FULL BLOCK.

**Integration with §20:**
Every Layer 7 finding becomes a structured entry in `errors.jsonl` with `category: connectivity`. Recurring patterns (e.g. "schema mismatch between Firestore writer and reader" hitting on multiple projects) get promoted to constraints — likely as pre-checks in §7 requiring shared TypeScript types between server and client.

---

### Seven-layer gate outcomes

| Result | Meaning |
|--------|---------|
| **FULL PASS** | All seven layers passed. Deployment may proceed. |
| **PASS WITH CONDITIONS** | Minor findings noted across one or more layers. Deployment may proceed with documented exceptions and a remediation deadline. |
| **CONDITIONAL BLOCK** | Critical findings in one or more layers. Deployment is paused until specific items are resolved. |
| **FULL BLOCK** | Severe findings (e.g. unauthenticated writes in prod rules, exposed Firebase Admin key, no server-side token validation, critical CVE in a runtime dependency, unparameterized SQL on user input, route declared with no handler, schema mismatch between Firestore writer and reader, all routes missing handlers). Deployment is stopped. No exceptions. |

### Security gate rule
If the gate returns CONDITIONAL BLOCK or FULL BLOCK, the skill must surface the exact findings, the specific remediation steps required, and what must be re-tested before the gate can be re-run. It must not suggest workarounds or proceed anyway.

---

## 11) Firebase deployment gate

The deployment step must happen only after validation, the seven-layer security gate passes, and only through the approved Firebase environment.

### Firebase requirements
- Distinct dev, test/staging, and prod environments.
- No mixing of debug and production resources.
- Use emulators locally when useful.
- Use the correct Firebase project for the correct environment.
- Respect security rules and App Check where applicable.
- Keep deployment separate from discovery and planning.

### Deployment rule
Do not deploy until:
- the phase is validated,
- the audit passes,
- the seven-layer security gate returns PASS or PASS WITH CONDITIONS,
- the environment is confirmed correct,
- and the deployment path is approved.

### Deploy verification (polling-aware as of v1.6.0)

`firebase deploy` returns quickly but the deployed URL is not necessarily live yet — CDN propagation takes seconds to minutes, and Cloud Functions cold starts add more delay. Do not declare deploy success on the basis of the CLI return alone.

After `firebase deploy` returns:

1. **Determine the deployed URL** from the CLI output (typically `https://<projectId>.web.app` for Hosting, or function-specific URLs).
2. **Run `poll_until_http_ok` (§27)** against the deployed URL. Default timeout 300s, interval 10s.
3. If polling times out → BLOCK, surface to §20 with `category: deploy`, `severity: high`, `root_cause: "deploy returned but URL did not serve 200 within 300s"`.
4. If polling succeeds → **run a subset of the §9 smoke spec against the deployed dev URL** (not against emulators). This is the production-style verification — same routes, same forms, same API calls, but against the actually-deployed environment.
5. Only after the deployed smoke passes is the deploy considered successful.

For Cloud Functions specifically:
- Each function's HTTPS trigger URL is also polled with `poll_until_http_ok` (typically returns 200 on a no-arg call, or 400 with a "missing parameters" message — both indicate the function is warm and serving).

### Production promotion

Promotion from `dev` to `prod` requires:
- All §10 layers still PASS on the dev deploy.
- Deploy verification (above) PASSED on dev.
- Explicit operator confirmation. Autopilot does NOT auto-promote to prod even with the autonomy flags set — prod deploys are the final human checkpoint, in the same spirit as §26's "never auto-merge PRs."

---

## 12) Reporting gate

After execution, the skill must report:
- what was done,
- what passed,
- what failed,
- what security findings (across all seven layers) were surfaced and resolved,
- what the Cost Manifest looks like at the end,
- what remains,
- what was learned,
- what should be improved next.

The report also includes:
- **Error log summary** (from Section 20) — total errors, categories, new constraints promoted.
- **Multi-agent activity summary** (from Section 23) — agents spawned, throttle events, phases that fell back to serial.
- **Firebase project links** (from Section 21) — console URLs and aliases.
- **Per-project `ai_docs/links.md`** path so the operator can find the full link log.

This report should also capture user corrections and repeated preferences for future runs.

---

## 13) Field purpose logic

Every field must exist for a reason.

### Every required field must answer at least one:
- What is it?
- Why does it matter?
- What does it change?
- What happens if it is wrong or missing?

If a field does not affect planning, structure, validation, deployment, resource gathering, or cost, it should not be mandatory.

---

## 14) Decision logic

The skill must make decisions based on project need, not template habit.

### It must:
- detect the project type,
- identify scope and risk,
- research only relevant needs (including cost),
- inventory required resources in full before planning,
- classify every resource as HAVE / NEED / GAP,
- resolve all NEEDs and GAPs before proceeding,
- produce a Resource Manifest and a Cost Manifest,
- organize into have/need/gaps across the broader research phase,
- build a phased workflow,
- validate every phase,
- run the seven-layer security gate before deployment,
- deploy only through Firebase correctly.

### It must not:
- force universal all-in-one structure,
- overbuild simple projects,
- underbuild complex projects,
- skip audit or testing,
- skip any layer of the seven-layer security gate,
- deploy without environment discipline,
- continue with unresolved critical gaps,
- ignore the free-tier-first cost stance unless explicitly overridden.

---

## 15) Execution priorities

The skill should prioritize:
1. Project fit.
2. Scope discipline.
3. Research relevance.
4. Resource sufficiency *(confirmed before planning)*.
5. Cost discipline *(free-tier-first)*.
6. Structural clarity.
7. Validation.
8. Security *(including all six gate layers)*.
9. Deployment correctness.
10. Reusability only where helpful.

---

## 16) Minimal operating rules

- Ask questions first (or infer in autopilot).
- Research only what is needed.
- Run cost analysis as part of research; default to free-tier-first.
- Inventory all tools, materials, assets, APIs, configs, credentials, and dependencies before planning.
- Produce a Resource Manifest and a Cost Manifest before the first plan is written.
- Default to Firebase + React + React Context (Section 19) unless intake says otherwise.
- Keep output project-specific.
- Use HAVE / NEED / GAPS.
- Build from simple to complex.
- Audit every phase.
- Test every phase.
- Run the seven-layer security gate before any deployment.
- Deploy only through Firebase correctly.
- Report what happened, including all security findings and the final cost picture.
- Run all embedded subsystems (Sections 20–23) autonomously throughout.

---

## 17) Optional expansion rules

If the project is larger or riskier, the skill may also include:
- threat modeling,
- SAST or policy checks,
- penetration testing simulation,
- rollout planning,
- rollback planning,
- observability,
- artifact cataloging,
- advanced secrets management (Vault, Google Secret Manager),
- and more detailed environment controls.

These should only appear when the project actually needs them.

---

## 18) Final skill intent

The skill's intent is not to be a generic planner.

Its intent is to be a **project-specific autonomous research and execution system** that understands what a project needs, inventories the full set of resources required to make it real before any plan is written, structures it correctly, runs cost analysis under a free-tier-first stance, validates each phase, enforces a seven-layer security gate before deployment, and only then completes the deployment path through the correct Firebase environment — all while autonomously logging errors, bootstrapping Firebase, locating Anthropic credentials, and orchestrating parallel sub-agents in the background.

---

## 19) Default technology stack *(new in v1.3.0)*

Unless intake specifies otherwise, the skill recommends and scaffolds against this stack:

### Frontend
- **React** (latest stable; functional components and hooks).
- **Vite** as the dev server and bundler (free, fast, zero infra cost).
- **TypeScript** by default. JavaScript only if intake explicitly opts out.

### State management
The skill picks the **lightest viable** state pattern for the project:

1. **React Context** — default for most state. Authentication state, theme, locale, current user, feature flags. Use it until it's actually insufficient.
2. **TanStack Query (React Query)** — next preference when the project has *server state* (data fetched from APIs / Firestore) that needs caching, background revalidation, and request deduplication. Add this the moment the project has more than two endpoints being consumed across more than one component.
3. **Zustand** — only if Context becomes a performance bottleneck (frequent global updates causing wide re-renders) or the state tree is genuinely complex and benefits from a non-Context store.
4. **Redux Toolkit** — only if the project explicitly needs Redux for team familiarity, dev tooling, or middleware ecosystem reasons. Not the default.

The escalation order is deliberate: start with Context, add TanStack Query for server state, only reach for Zustand/Redux when measurably needed. Document the choice and trigger in the Resource Manifest.

### Backend / data / hosting / auth
- **Firebase** is the default platform.
  - **Firebase Auth** for identity (email/password + Google OAuth as minimum).
  - **Firestore** for primary database (NoSQL, serverless, generous free tier).
  - **Firebase Storage** for file uploads.
  - **Firebase Cloud Functions** for server-side handlers (Node.js runtime; Python optional).
  - **Firebase Hosting** for static frontend.
  - **App Check** enabled in production.
- **Blaze plan** is used when Functions need outbound network, larger Storage, or higher Firestore quotas. Blaze stays free until usage actually crosses the meter, so it's treated as zero-cost during build (see Section 4a).

### When the default stack does NOT apply
The skill must override the default when ANY of the following is true in intake or detection:

- The project is a native mobile app where Firebase web SDK is the wrong fit (use Firebase Native SDKs, plus React Native or the user's preferred native framework).
- The project explicitly requires a relational database (use Cloud SQL on GCP, or note the override).
- The project explicitly requires self-hosting / on-prem deployment.
- The project explicitly requires a non-Firebase backend (Supabase, AWS Amplify, Convex, etc.).
- The project is a game, CLI tool, or other non-web/non-mobile build where this stack doesn't apply.
- The user's intake field "Stack override" specifies different choices.

### Recording the stack decision
The chosen stack (default or overridden) goes into the Resource Manifest under `selected_stack` with the rationale for any deviation from the default. This record is referenced in the final report.

---

## 20) Error mitigation loop *(embedded subsystem, was a separate skill in v1.2.0)*

Turn every error encountered during a project run into durable institutional knowledge. Each error becomes:
1. A structured **log entry** (what happened, when, in which phase).
2. A **fix record** (the resolution applied).
3. A **mitigation rule** (how to prevent or detect this earlier next time).
4. When a pattern repeats, a **constraint** appended to the plugin's persistent constraints file so future runs inherit the lesson.

### 20.0) Autonomy contract for this subsystem

This subsystem operates **autonomously** by default. It does not ask the user for permission to log errors, write fixes, or distill constraints. It runs as a background process throughout the main workflow and surfaces a summary only at phase boundaries or when explicitly asked.

**Autonomous behavior:**
- Triggers on any observed error, exception, failed test, validation BLOCK, blocked tool call, or recoverable failure.
- Captures, classifies, and writes log entries without confirmation.
- Distills mitigation rules and constraints without confirmation.
- Surfaces a one-line acknowledgment per error (e.g. `[error-mitigation] logged: <short summary> → constraint added`).

**Hard limits:**
- **Append-only.** This subsystem MUST NOT modify or delete any existing file in the plugin or in the user's project. It may only create new files or append to files it owns (listed in 20.3).
- **No deletions, no overwrites, no truncation.** If a write operation would not be a pure append, abort and surface the issue.
- **No execution of remediation commands without user approval.** This subsystem records the fix that was applied; it does not re-run destructive actions on its own.

### 20.1) Activation triggers

Activates whenever ONE of the following is observed:

- A tool call returns a non-zero exit, exception trace, or error string.
- A test runner reports failures.
- A build, compile, lint, type-check, or deploy step fails.
- A validation gate (Section 8) returns BLOCK.
- Any layer of the seven-layer security gate returns FAIL or BLOCK.
- A resource manifest item flips from HAVE to NEED/GAP unexpectedly.
- The user types phrases like "got an error", "this failed", "broken", "fix this", "why did this fail", "regression".
- A previously-resolved error pattern recurs.

Does NOT activate for: stylistic warnings without functional impact, expected/intentional test failures inside characterization tests, user-driven cancellations or interrupts.

### 20.2) Capture format

For every triggering event, write a structured entry. The entry MUST contain all of:

```jsonl
{
  "id": "err-<utc-timestamp>-<short-hash>",
  "timestamp": "<ISO-8601 UTC>",
  "phase": "<intake|detection|resource-gathering|research|organize|modeling|workflow|validation|pre-deploy|security|deployment|post-deploy|other>",
  "project_name": "<from intake, or 'unknown'>",
  "project_type": "<from detection, or 'unknown'>",
  "severity": "<low|medium|high|critical>",
  "category": "<config|dependency|auth|permissions|env|test|build|deploy|integration|data|logic|orchestration|other>",
  "what_happened": "<concise, factual description of the failure>",
  "root_cause": "<the actual underlying cause, not the surface symptom>",
  "fix_applied": "<the exact change/command/config that resolved it>",
  "verification": "<how the fix was confirmed to work>",
  "mitigation_rule": "<one sentence: what to check or do BEFORE this can happen again>",
  "constraint_candidate": "<true|false>",
  "related_resources": ["<resource manifest items touched>"],
  "tags": ["<freeform tags for grep>"]
}
```

**Field rules:**
- **root_cause** must go past the symptom. "Build failed" is not a root cause; "missing `firebase-admin` peer dep because Node version is 18 but the SDK requires 20" is.
- **fix_applied** must be specific enough that a future run can re-apply it without rediscovery.
- **mitigation_rule** must be phrased as a check, not a regret.

### 20.3) Files this subsystem owns (append-only)

**Per-project files** (in the active project's working directory):
- `ai_docs/error-log/errors.jsonl` — one JSON object per line. **Append-only.**
- `ai_docs/error-log/mitigations.md` — human-readable playbook. **Append-only.**

**Plugin-level files** (persistent across all projects):
- `<plugin-root>/skills/autonomous-project-agent/constraints.md` — distilled rules. **Append-only.**
- `<plugin-root>/skills/autonomous-project-agent/pattern-index.jsonl` — recurrence counter for promotion. **Append-only.**

**Append-only enforcement:** before any write, confirm the target file is in the owned-files list, confirm the new content is being added at the end (no modification to prior bytes), and never use truncating writes.

### 20.4) Constraint promotion logic

A logged error becomes a **persistent constraint** in `constraints.md` when ANY of:

- The same `category` + `root_cause` pattern has occurred **2 or more times** across runs.
- Severity is `critical` (always promote).
- The error involved any security gate layer FAIL or BLOCK (always promote).
- The error caused data loss, leaked credentials, or required a rollback (always promote).
- The user explicitly says "remember this" or "make sure this doesn't happen again."

**Constraint format:**
```markdown
## C-<NNN> — <short title>

- **Category:** <config|dependency|auth|permissions|env|test|build|deploy|integration|data|logic|orchestration|other>
- **Trigger phase:** <which gate(s) this constraint runs in>
- **Rule:** <imperative single sentence>
- **Pre-check:** <concrete check the workflow should run before the trigger phase>
- **Failure mode if violated:** <what goes wrong>
- **Originating errors:** [<err-id>, <err-id>, ...]
- **Added:** <ISO-8601 date>
```

Constraint IDs are monotonically increasing (`C-001`, `C-002`, ...). Never reuse IDs.

### 20.5) Constraint consumption

At the start of every run (right after intake, before resource-gathering), the workflow MUST read `<plugin-root>/skills/autonomous-project-agent/constraints.md` if it exists and treat each active constraint as an additional pre-check that must pass before its trigger phase completes.

### 20.6) Output cadence

- **Per error:** one inline acknowledgment.
- **Per phase boundary:** two-line summary of errors logged that phase.
- **End-of-run report:** consolidated section appended to the Section 12 report.

### 20.7) Read/write hygiene

- May **read** any file needed to understand error context.
- MUST NOT re-run failing commands, rewrite source code "proactively," modify env/settings outside owned files, or send error data anywhere outside the local filesystem.

### 20.8) Hygiene rules

- Entries immutable once written. Corrections go in NEW entries that reference prior `id` via a `corrects: <id>` field.
- Same exact error recurring still writes a new entry — recurrence is signal.
- Log file may grow indefinitely. Rotation is a future concern; do NOT auto-delete.

---

## 21) Firebase bootstrap *(embedded subsystem, was a separate skill in v1.2.0)*

Make sure that by the time the workflow reaches the Firebase deployment gate, the Firebase CLI is installed, authenticated, and pointed at the correct project for the correct environment. Capture every relevant link into the per-project `ai_docs/links.md` log.

### 21.0) Autonomy contract

This subsystem runs **autonomously** as part of the resource-gathering gate (Section 3) and again at the pre-deployment gate (Section 9). It does not pause for confirmation in autopilot mode.

**Hard limits:**
- **Never** rotates or modifies an existing Firebase project's settings beyond `firebase use --add` aliasing.
- **Never** deploys — deployment is the parent workflow's job.
- **Never** writes the Firebase CI token, service account JSON, or any credential to `links.md` or any log. Only writes paths and console URLs.
- Hard-blocks the deployment gate if any check FAILs after one auto-remediation attempt.

### 21.1) Activation triggers

Activates when ANY of:
- Intake captured a deployment target containing "firebase", "firestore", "firebase auth", "cloud functions", "firebase hosting", or "firebase storage".
- The default stack (Section 19) was selected.
- Resource Manifest contains any Firebase SDK in HAVE or NEED.
- The user types phrases like "deploy to firebase", "set up firebase", "use firestore", "firebase project".
- The workflow enters Section 3 (resource-gathering) or Section 9 (pre-deployment).

### 21.2) Bootstrap sequence

**Step 1 — CLI presence.** Run `firebase --version`. If missing → in autopilot, attempt `npm install -g firebase-tools`. Otherwise surface install command.

**Step 2 — Authentication.** Run `firebase login:list`. If not logged in → run `firebase login` (browser-based OAuth flow; this is the one allowed prompt even in autopilot).

**Step 3 — Project list.** Run `firebase projects:list --json` and parse.

**Step 4 — Project selection.** Resolution order:
1. Explicit user choice from intake "Firebase project preference".
2. Project name match (case-insensitive, dash/underscore tolerant).
3. Autopilot create: `firebase projects:create <slug>-<short-hash> --display-name "<project name>"`.
4. Non-autopilot ambiguity: present list and ask.

**Step 4a — Wait for project propagation (polling, new in v1.6.0).** After `firebase projects:create`, the project takes 10–30 seconds to propagate before subsequent CLI calls can find it. Run `poll_firebase_project_exists` (§27.3) against the new project ID — default timeout 60s, interval 5s. Do not proceed to Step 5 until propagation completes. Without this, `firebase use --add` will sometimes fail with "project not found" on a freshly-created project.

**Step 5 — Environment aliasing.** `firebase use --add <projectId> --alias <dev|staging|prod>` for each env. Confirm via `firebase use`.

**Step 6 — Service enablement check.** `firebase apps:list`, `firebase functions:list`. Services in NEED but not enabled → surface console URL for enablement; do not auto-enable (billing implications).

**Step 7 — Links log write.** Append to `<project>/ai_docs/links.md` under Projects and Services.

### 21.3) Autopilot interaction

All steps run without confirmation **except** Step 2's `firebase login` browser flow. Step 4 creates a new project when no match exists, using the project name from intake. All decisions logged as explicit assumptions in the Resource Manifest.

### 21.4) Links log integration

Appends to `<project>/ai_docs/links.md` under **Projects** and **Services**. Never writes to Credentials Inventory.

**Entry format under Projects:**
```markdown
| Firebase Project | https://console.firebase.google.com/project/<projectId> | Active <env> deploy target | added <date> | source-of-truth: y |
```

**Entry format under Services:**
```markdown
| Firebase Auth | https://console.firebase.google.com/project/<projectId>/authentication/users | Identity provider | added <date> | source-of-truth: y |
| Firestore | https://console.firebase.google.com/project/<projectId>/firestore | Primary database | added <date> | source-of-truth: y |
| Cloud Functions | https://console.firebase.google.com/project/<projectId>/functions | Server-side handlers | added <date> | source-of-truth: y |
```

Append-only contract per Section 20.

### 21.5) Failure handling

Every failed check surfaces a structured event to Section 20 (error mitigation loop) with `category: config`, `phase: pre-deploy`, and a stated mitigation rule.

### 21.6) Hard BLOCK conditions

Deployment cannot proceed if ANY of the following is true after one auto-remediation attempt:
- Firebase CLI not installed.
- Not authenticated.
- No project selected.
- Selected project is in a different Firebase organization than specified at intake.
- Active alias resolves to `prod` but workflow is still in `dev` deployment phase (or vice versa).

---

## 22) Anthropic API key locator *(rewritten in v1.4.0 — reference-mode, not copy-mode)*

When a project the agent is building needs to call the Anthropic API (i.e. the project itself uses Claude, separate from Claude Code the harness), this subsystem locates an existing `ANTHROPIC_API_KEY` from prior local projects and wires it into the new project **by reference, not by value**. The key value lives in exactly one place on disk; every project that needs it references that one path. Never asks for paste, never logs the key value, never queries any remote service.

### 22.0) Design principle — reference, not copy

The v1.2.0/v1.3.0 design copied the key VALUE into each project's `.env`. That works, but it scatters the credential across every project on disk and creates as many leakage paths as there are repos. v1.4.0 changes the model:

**One canonical keystore.** `~/.anthropic/api-key`, mode `0600` (owner read/write only). The key value lives here, once.

**Every project references it.** New projects get `ANTHROPIC_API_KEY_FILE=$HOME/.anthropic/api-key` in `.env.local` and a tiny resolver script that reads from that path at runtime.

**Multiple safety walls.** `.gitignore` enforcement, `git check-ignore` verification (abort if not ignored), pre-commit hook that blocks any commit containing an `sk-ant-` pattern, and production deploys that use Firebase Functions secrets rather than bundled values.

**Effect:** rotating the key means editing one file. Leaking a project's `.env.local` leaks a *path*, not a credential. Even an accidental `git add .env.local` would be caught by the pre-commit hook before the value could escape — and even if it did, the value isn't in `.env.local`, only the path is.

### 22.1) Autonomy contract

Runs **autonomously** during Section 3 (resource-gathering) when the project type or stack indicates Anthropic SDK use. Auto-canonicalizes on find. Auto-wires the new project by reference. No prompts.

**Hard limits:**
- **Local filesystem reads only.** Never queries the Anthropic console, never makes any network call to discover keys.
- **Never logs the key value.** Logs only file paths, last-4 character previews (e.g. `sk-ant-…AB12`), and discovery timestamps.
- **Never writes the key value to any project file.** Project files contain only the *path* to the canonical keystore.
- **The single exception:** writes the key value once to `~/.anthropic/api-key` (mode 0600) if it isn't already there. That file is the keystore; the discovered key originated from another file on the same machine, so this is canonicalization, not duplication.
- **Abort if `.gitignore` enforcement fails.** If `git check-ignore .env.local` does not return success after the gitignore is updated, abort the entire wire-up and surface the failure to Section 20.
- **Cross-platform:** macOS and Linux only in v1.4.0. Windows is unsupported (different credential paradigm — surfaces a note in the Resource Manifest if detected).

### 22.2) Activation triggers

Activates when ANY of:
- Resource Manifest has `@anthropic-ai/sdk`, `anthropic` (Python), or any Anthropic API client in HAVE or NEED.
- Source code being scaffolded imports those packages.
- User types phrases like "anthropic api key", "claude api key", "wire up claude".
- An `ANTHROPIC_API_KEY` placeholder appears in any generated `.env.example` or scaffold file.

Does **not** activate for projects that only use Claude Code (the harness has its own auth).

### 22.3) Search locations (read-only)

**Canonical store first (fast path):**
- `~/.anthropic/api-key` — if this exists and contains a valid `sk-ant-…` key, use it directly and skip the full scan.

**Shell environment files:**
- `~/.zshrc`, `~/.zprofile`, `~/.bashrc`, `~/.bash_profile`, `~/.profile`, `~/.config/fish/config.fish`

**Project `.env*` files under common dev roots** (max depth 6):
- `~/Documents`, `~/Code`, `~/Projects`, `~/Developer`, `~/Sites`, `~/Desktop`, `~/Downloads`, `~/dev`, `~/repos`, `~/workspace`

For files matching `.env`, `.env.local`, `.env.development`, `.env.production`, `.env.*` — search for `ANTHROPIC_API_KEY=...`.

**Standard config locations:**
- `~/Library/Application Support/anthropic/*` (macOS), `~/.config/anthropic/*`, `~/.anthropic/*`

**Out-of-scope (never read):**
- `~/Library/Keychains/*` — never touched.
- Inside `node_modules/`, `.git/objects/`, `vendor/`, `dist/`, `build/`, `.next/`, `.cache/`.
- Files larger than 1 MB.

### 22.4) Selection logic

When multiple keys found across non-canonical sources:
- Pick the key from the **most recently modified** source file.
- Skip keys that fail format validation (Anthropic keys start with `sk-ant-`).
- Skip keys whose source path contains `example`, `template`, `sample`, `test` — those are placeholders.
- Record up to 5 alternates in the links log.

### 22.5) Canonicalization step

Once a key is selected (or the canonical store is found to already exist):

1. **Ensure `~/.anthropic/` exists.** `mkdir -p ~/.anthropic && chmod 700 ~/.anthropic`.
2. **If `~/.anthropic/api-key` does not exist**, write the discovered key value to that file with mode `0600`. Use `umask 077` semantics — only the owner may read.
3. **If `~/.anthropic/api-key` exists and matches the discovered key**, no-op.
4. **If `~/.anthropic/api-key` exists and differs from the most-recent discovered key**, treat the existing canonical store as source of truth. Record the discovered alternate in the links log under "alternates" but do NOT overwrite the canonical store. (Rotation is a manual user action, not an autonomous one.)

### 22.6) Wiring action (reference-mode)

For the new project:

1. **Ensure `.env.local` exists.** Create if not. (`.env.local` is preferred over `.env` because every standard React/Vite/Next.js/Node.js scaffolder gitignores `.env.local` by default — fewer footguns than `.env`.)
2. **Append the reference line** to `.env.local`:
   ```
   ANTHROPIC_API_KEY_FILE=/Users/<user>/.anthropic/api-key
   ```
   (Expand `$HOME` to the absolute path so cross-shell sourcing works.)
3. **Update `.gitignore`** to ensure ALL of the following are present (append any that are missing):
   ```
   .env
   .env.local
   .env.*.local
   .env.development.local
   .env.production.local
   secrets/
   *.pem
   *.key
   ```
4. **Verify gitignore enforcement** by running:
   ```
   git check-ignore .env.local
   ```
   If this command does NOT exit 0 (meaning `.env.local` is NOT actually ignored), **abort the entire wire-up immediately**. Surface to Section 20 as a `severity: critical` event. Do not proceed under any circumstances.
5. **Generate a tiny resolver script** based on detected project language. Write it to `scripts/load-anthropic-key.{ts|py|sh}`:

   **TypeScript / Node** (`scripts/load-anthropic-key.ts`):
   ```typescript
   import { readFileSync } from "node:fs";
   export function loadAnthropicKey(): string {
     const direct = process.env.ANTHROPIC_API_KEY;
     if (direct) return direct;
     const path = process.env.ANTHROPIC_API_KEY_FILE;
     if (!path) throw new Error("ANTHROPIC_API_KEY_FILE not set");
     return readFileSync(path, "utf8").trim();
   }
   ```

   **Python** (`scripts/load_anthropic_key.py`):
   ```python
   import os
   def load_anthropic_key() -> str:
       direct = os.environ.get("ANTHROPIC_API_KEY")
       if direct:
           return direct
       path = os.environ.get("ANTHROPIC_API_KEY_FILE")
       if not path:
           raise RuntimeError("ANTHROPIC_API_KEY_FILE not set")
       with open(path, "r") as f:
           return f.read().strip()
   ```

   **Shell** (`scripts/load-anthropic-key.sh`):
   ```sh
   #!/usr/bin/env sh
   if [ -n "$ANTHROPIC_API_KEY" ]; then echo "$ANTHROPIC_API_KEY"; exit 0; fi
   if [ -z "$ANTHROPIC_API_KEY_FILE" ]; then echo "ANTHROPIC_API_KEY_FILE not set" >&2; exit 1; fi
   cat "$ANTHROPIC_API_KEY_FILE"
   ```

6. **Install a pre-commit hook** at `.git/hooks/pre-commit` (or via Husky/lefthook if either is in deps). The hook scans staged content for `sk-ant-` substrings and aborts the commit if any are found:

   ```sh
   #!/usr/bin/env sh
   if git diff --cached -U0 | grep -qE 'sk-ant-[A-Za-z0-9_-]{20,}'; then
     echo "ERROR: staged content contains what looks like an Anthropic API key (sk-ant-...). Commit aborted." >&2
     echo "If this is intentional (e.g. a documentation example), remove the real prefix." >&2
     exit 1
   fi
   ```
   `chmod +x .git/hooks/pre-commit`.

### 22.7) Production deployment wiring

For production runtime environments, **never bundle the key into client-side code** and **never bake it into a built artifact**. Use the platform's secret store:

- **Firebase Cloud Functions:** `firebase functions:secrets:set ANTHROPIC_API_KEY`, then access via `defineSecret("ANTHROPIC_API_KEY").value()` in function code.
- **Cloud Run / GCP:** Secret Manager + secret-as-env mounting.
- **Vercel / Netlify / Render / Railway:** the platform's encrypted environment variable UI or CLI.
- **AWS Lambda:** Parameter Store SecureString or Secrets Manager.

The skill logs the chosen production-secret path in `links.md` under Credentials Inventory.

### 22.8) Empty-state handling

If no key is found anywhere (including the canonical store), treat as new-project state:
- Write TODO line to `<project>/ai_docs/links.md` Credentials Inventory:
  ```
  | ANTHROPIC_API_KEY | https://console.anthropic.com/settings/keys | Generate new key | TODO — not yet provisioned |
  ```
- Add explicit assumption to Resource Manifest.
- Do not block the build — scaffolding can proceed. The reference line in `.env.local` is still added; it will fail loudly at first API call until the canonical store is populated. Block only at final pre-deployment if still missing.

### 22.9) Links log integration

Appends to `<project>/ai_docs/links.md` under **Credentials Inventory** only. Never writes to Projects, Services, or APIs sections.

**Entry format (key wired by reference):**
```markdown
| ANTHROPIC_API_KEY | $HOME/.anthropic/api-key | Reference-wired into .env.local (key value never copied to project) | wired <date> | preview: sk-ant-…AB12 |
```

**Entry format (canonical store source):**
```markdown
| ANTHROPIC_API_KEY (canonical store) | <original-source-path> | Source of canonicalization for ~/.anthropic/api-key | discovered <date> | preview: sk-ant-…AB12 |
```

**Strict rule:** source paths recorded; last-4 previews for disambiguation; **full key values NEVER recorded.**

### 22.10) Failure handling

Surfaces to Section 20 on any of:
- `git check-ignore .env.local` returns non-zero — `severity: critical`, BLOCK the run.
- `.env.local` write fails (permissions, disk full).
- `.gitignore` edit fails.
- Pre-commit hook installation fails.
- Malformed key (does not match `sk-ant-` format).
- Key discovered in a disallowed location (e.g. inside `node_modules`).
- Canonical store has unsafe permissions (anything other than 0600).

### 22.11) Rotation playbook

If the user needs to rotate the Anthropic key:
1. Generate a new key at console.anthropic.com/settings/keys.
2. Replace contents of `~/.anthropic/api-key` (mode 0600).
3. No project file changes needed — every project picks up the new value on next process restart.
4. Revoke the old key in the Anthropic console.

The skill documents this in the per-project `links.md` Credentials Inventory as a note.

---

## 23) Multi-agent orchestration *(rewritten in v1.4.0 — grounded in Claude Code reality, not aspirational)*

Speed up research-heavy and independent-subtask phases by spawning parallel sub-agents via the Claude Code `Agent` tool. This section describes what the orchestrator **actually does** in the harness today, separated from the **aspirational** features that depend on capabilities the harness doesn't yet expose mid-flight.

### 23.0) Reality check: what the harness actually supports

The Claude Code `Agent` tool spawns sub-agents in parallel when invoked with multiple tool-use blocks in one message. The harness handles isolation per agent. **What the harness does NOT expose to the orchestrator** is per-agent live token counters, mid-flight rate-limit headroom, or measured-efficiency feedback during a spawn batch. Those are visible only after each batch returns.

So the orchestrator works at the **batch level**, not the live-agent level. A "phase" emits one or more **fanout batches**, each batch returns, and the orchestrator decides what the next batch (or whether to fall back to serial) looks like based on what came back.

### 23.1) Autonomy contract

Runs **autonomously** at every phase boundary that qualifies for fanout. No per-batch confirmation in autopilot mode. Reports a single summary line per phase.

**Hard limits:**
- **Never** spawns agents in batches larger than 6.
- **Never** spawns agents that would write to overlapping file sets.
- **Never** parallelizes phases tiered as serial below.
- **Always** uses git worktrees for isolation when sub-agents write code.

### 23.2) Phase tiering

| Phase | Tier | Typical batch size |
|---|---|---|
| Intake | serial | 1 |
| Project detection | serial | 1 |
| Plan preview (§2a) | serial | 1 |
| Resource-gathering | parallel | 3–6 (one batch, one agent per resource category cluster) |
| Research | parallel | 3–6 (one batch, one agent per research dimension) |
| Organize | serial | 1 |
| Modeling | parallel | 2 (user model + system model) |
| Workflow construction | serial | 1 |
| Build / scaffolding | parallel | 2–4 (one batch, one agent per independent module) |
| Validation | parallel | 2–5 (one batch, one agent per test suite or audit) |
| Pre-deployment | serial | 1 |
| Security gate | parallel | up to 6 (one agent per layer in §10) |
| Deployment | serial | 1 |
| Reporting | serial | 1 |

### 23.3) Independence check (real, applied at batch-plan time)

Before emitting a batch, verify:
1. **Disjoint file write sets.** Two sub-agents about to write to the same path means a merge conflict waiting to happen — split them serially.
2. **No producer/consumer pair in the same batch.** If agent B needs agent A's output, they can't be in the same batch.
3. **No shared API rate-limit pool exhaustion.** If two sub-agents will both hammer the same external API, stagger them across batches.

If any check fails for a candidate batch, downgrade those subtasks to serial.

### 23.4) Git worktree isolation (for code-writing phases)

For build, scaffolding, integration phases where sub-agents write code:
- The orchestrator creates a worktree per sub-agent: `git worktree add <project>/.worktrees/agent-<id> -b agent/<phase>/<id>`.
- Each sub-agent is invoked with `Agent({ isolation: "worktree", ... })` — the harness handles the worktree lifecycle.
- After the batch returns, the orchestrator merges each agent's worktree back to the integration branch sequentially.
- Merge conflicts → log to §20, surface the conflict to the user, drop the offending subtasks to serial for retry.

For research, reading, and audit phases (no writes), no worktree needed.

### 23.5) Batch-level cap (the realistic version)

```
batch_cap = min(
  independent_subtask_count,
  hard_ceiling_6,
  non_overlapping_file_sets,
  prior_batch_throttle_signal
)
```

**`prior_batch_throttle_signal`:** if the most recent batch in this phase returned rate-limit errors, merge conflicts, or two agents producing redundant output, the next batch is capped at 2 or downgraded to serial.

### 23.6) What was aspirational in v1.2.0 / v1.3.0 — now demoted to "future direction"

The earlier specs described:
- Per-agent live `tokens_in` / `tokens_out` / efficiency tracking mid-spawn.
- A rolling-median efficiency cap that kills spawning when efficiency drops below 50% of median.
- Real-time throughput observability across worktrees.

The Claude Code harness today exposes none of those mid-flight. **The orchestrator can only observe batch-level signals after the fact.** The earlier spec language is preserved here for honesty but is not a hard requirement — it becomes operative only when the harness exposes the needed observability hooks.

### 23.7) Communication and merge protocol

Parallel sub-agents do not communicate with each other. Each gets at spawn:
- A scoped task description.
- A scoped read budget (paths / docs they may consult).
- A scoped write budget (worktree-enforced).
- An expected return shape (structured summary or file paths).

Each sub-agent returns artifacts (files in its worktree) or a structured summary (for research). The orchestrator merges artifacts sequentially in a deterministic order (alphabetical by agent ID), runs §8 validation on the merged result, records the pass to the Resource Manifest.

### 23.8) Throttling triggers (batch-level, observed after return)

After each batch returns, throttle the next batch if:
- Any agent returned a 429 / rate-limit / quota error.
- Any merge produced a conflict.
- Two or more agents produced obviously redundant work (same conclusion, same files touched).
- The phase budget has been exceeded (token or wall-time).

**Throttle action for the next batch in this phase:**
- On 429 / rate-limit: invoke `poll_with_backoff_on_429` (§27) on the call that hit the limit. Honor `Retry-After` if present, exponential backoff otherwise, capped at 300s total. After recovery, cap the next batch at 2 sub-agents. If `poll_with_backoff_on_429` itself times out (300s total wait without successful retry), the rate-limit is structural — fall back to serial for the rest of this phase and surface to §20 with `severity: high`.
- On merge conflict or redundant work: cap next batch at 2, or fall back to serial entirely.

### 23.9) Reporting

Per phase, one line:
```
[orchestrator] phase=<phase-name> tier=<serial|parallel> batches=<B> agents_total=<N> merge_conflicts=<C> throttled=<T> serial_fallback=<true|false>
```

End-of-run section appended to §12 report:
```
## Multi-agent activity
- Total batches emitted: <B>
- Total agents spawned: <N>
- Phases that fell back to serial: [list]
- Merge conflicts encountered: <C>
- Throttle events: <T>
```

### 23.10) Cross-subsystem behavior

- **§20 (error mitigation)** receives all merge conflicts and rate-limit/redundancy throttles as logged errors with `category: orchestration`. Recurring patterns (e.g. always-conflicting on a specific module) get promoted to constraints that tighten the offending phase to serial in future runs.
- **§21 (Firebase bootstrap)** is always serial.
- **§22 (Anthropic key locator)** is always serial.
- **§10 (Security gate)** Layer 6 dependency scan, OWASP review, and HTTPS check are each independent and can run as a parallel batch within the security gate.

### 23.11) Autopilot interaction

Runs without per-batch confirmation. Cap and tier decisions are autonomous and logged as assumptions in the Resource Manifest. Hard BLOCKs from §10 still halt deployment regardless of how the orchestrator routed work.

---

## 24) Run-state checkpointing *(new in v1.4.0)*

Long autonomous runs die for boring reasons: laptops sleep, sessions close, rate limits hit, the user steps away. This subsystem makes runs resumable.

### 24.1) The checkpoint file

`<project>/ai_docs/run-state.json` — written at every phase boundary, read at the start of every activation (see Boot Sequence).

**Schema:**
```json
{
  "run_id": "run-<utc-timestamp>-<short-hash>",
  "started_at": "<ISO-8601 UTC>",
  "last_checkpoint_at": "<ISO-8601 UTC>",
  "status": "in-progress | completed | aborted",
  "current_phase": "<intake|detection|plan-preview|resource-gathering|research|organize|modeling|workflow|build|validation|pre-deploy|security|deployment|reporting>",
  "phases_completed": ["intake", "detection", "..."],
  "next_phase": "<phase name or null>",
  "resource_manifest_path": "<project>/ai_docs/resource-manifest.md",
  "cost_manifest_path": "<project>/ai_docs/cost-manifest.md",
  "links_md_path": "<project>/ai_docs/links.md",
  "intake_captured": { /* the full intake answers */ },
  "stack_decision": { "frontend": "react", "state": "context", "backend": "firebase", "overrides": [] },
  "active_constraints_loaded": ["C-001", "C-002"],
  "pending_assumptions": [/* assumptions not yet resolved */],
  "active_worktrees": [/* if mid-fanout, list of worktree paths */],
  "github_repo_url": "https://github.com/<user>/<project>",
  "github_visibility": "private",
  "current_branch": "feat/phase-5c-auth-flow",
  "open_prs": [
    { "number": 1, "title": "Phase 5a — Vite scaffold", "url": "https://github.com/.../pull/1", "branch": "feat/phase-5a-scaffold", "status": "open" }
  ],
  "aborted_reason": null
}
```

### 24.2) When to write

The skill writes the checkpoint:
- After every phase completes (`phases_completed` grows, `current_phase` advances).
- After any constraint is promoted by §20.
- After any orchestrator batch returns (so a crash mid-fanout can roll back to a known state).
- When the user types "stop" / "pause" — `status: aborted`, `aborted_reason: user-paused`.

### 24.3) When to read (resume protocol)

At the start of every activation (per Boot Sequence):
1. Read `<project>/ai_docs/run-state.json` if it exists.
2. If `status: in-progress`:
   - Restore `resource_manifest_path`, `cost_manifest_path`, `links_md_path` into working context.
   - Restore `intake_captured` and `stack_decision` — do not re-ask intake questions.
   - Jump directly to `next_phase`.
   - Announce briefly: `Resuming run <run_id> from phase <next_phase> (started <ago>).`
3. If `status: completed`:
   - This is either a re-run on a finished project (start a fresh run, write a new file) or the user is asking for a status report. Surface the file path and ask which is intended.
4. If `status: aborted`:
   - Surface the abort reason. Ask whether to resume from `next_phase`, start fresh, or just inspect.

### 24.4) Worktree recovery

If `active_worktrees` is non-empty on resume:
- Each worktree's branch is checked for uncommitted changes.
- Worktrees with no useful changes are pruned (`git worktree remove --force`).
- Worktrees with changes are surfaced — the user decides merge / discard / inspect.

### 24.5) Cleanup on completion

When the run finishes (deployment succeeds or user accepts a partial-completion endpoint), the skill writes `status: completed` and leaves the file in place. Future runs on the same project see it and start a new `run_id`. The completed `run-state.json` is renamed to `run-state-<run_id>.json` to keep history without overwriting.

### 24.6) Privacy

The checkpoint contains intake answers, manifest paths, and stack decisions — **no credentials, no key values**. It is safe to commit to the repo if desired, but defaults to gitignored (`<project>/.gitignore` gets `ai_docs/run-state*.json` appended by the workflow).

---

## 25) Anti-patterns this skill is designed to prevent *(new in v1.4.0)*

A self-test catalog. Each anti-pattern lists the failure mode, the gate or section that prevents it, and the symptom to watch for if the prevention fails.

### 25.1) Overbuilt simple project
- **Failure mode:** asked for a simple form, got a SaaS-style monorepo with microservices.
- **Prevented by:** §2 detection gate (scope sizing) + §15 scope discipline.
- **Symptom of prevention failing:** the workflow plan in §2a has more than 5 phases for a project the user described in under 3 sentences.

### 25.2) Deployed without authentication
- **Failure mode:** project ships to production with `allow read, write: if true` Firestore rules.
- **Prevented by:** §10 Layer 3 (automatic BLOCK on that exact rule).
- **Symptom of prevention failing:** deployment proceeded with a PASS WITH CONDITIONS that included an unauthenticated-write finding — this should be a BLOCK, not a condition.

### 25.3) Leaked API key
- **Failure mode:** `.env` (or `.env.local`) committed to a public repo with a real Anthropic or Firebase key.
- **Prevented by:** §22.6 `git check-ignore` enforcement, §22.6 pre-commit hook, §22 reference-mode design.
- **Symptom of prevention failing:** a commit lands containing an `sk-ant-…` pattern. The pre-commit hook should have blocked this — investigate why it didn't fire.

### 25.4) Stuck in research forever
- **Failure mode:** the agent loops in §4 research, never advancing to §5.
- **Prevented by:** §4 research-rule scope-cap + §2a plan-preview which time-boxes the run.
- **Symptom of prevention failing:** 30+ tool calls inside research with no phase advancement.

### 25.5) All-or-nothing failure with no rollback
- **Failure mode:** something breaks at §11 deployment, no clean state to return to.
- **Prevented by:** §8 rollback paths required per phase + §24 run-state checkpointing.
- **Symptom of prevention failing:** a failed deploy leaves the project in an unknown state with no checkpoint to resume from.

### 25.6) Cost runaway in production
- **Failure mode:** project hits 100K MAU and the user sees a $4,000 Firebase bill.
- **Prevented by:** §4a free-tier-first cost stance + cost ladder projection.
- **Symptom of prevention failing:** the Cost Manifest at end of run doesn't include heavy-usage projection lines.

### 25.7) Silently ignored constraint
- **Failure mode:** a constraint was promoted last run, but this run repeated the exact same mistake.
- **Prevented by:** Boot Sequence mandatory read of `constraints.md`.
- **Symptom of prevention failing:** a new error in this run matches the `root_cause` of an existing constraint — the constraint either wasn't loaded or wasn't applied.

### 25.8) Multi-agent merge chaos
- **Failure mode:** parallel sub-agents write conflicting changes that get clobbered at merge.
- **Prevented by:** §23.3 disjoint-file-set check + §23.4 worktree isolation + sequential merge.
- **Symptom of prevention failing:** a file appears in the final tree containing content from only one of multiple agents that should have all contributed.

### 25.9) Approval fatigue OR silent runaway
- **Failure mode:** either the user has to OK 12 things per run (fatigue) or the agent runs for 2 hours with zero visibility (silent runaway).
- **Prevented by:** §2a plan-preview gate (one approval, then unattended) + §23.9 batch-level summary lines.
- **Symptom of prevention failing:** the run had more than one approval pause, OR the run had zero status lines emitted between phases.

### 25.10) Treating a paid-only feature as free
- **Failure mode:** scaffolded SMS via Twilio without surfacing that it's metered from the first message.
- **Prevented by:** §4a per-service free-tier evaluation.
- **Symptom of prevention failing:** Cost Manifest shows `Twilio: $0/mo at projected light usage` without noting the per-message charge.

---

## 26) GitHub bootstrap & branch lifecycle *(new in v1.5.0)*

Make sure every project the agent builds lives in a reviewable, version-controlled place — not in a laptop-resident directory that disappears with the next reboot. This subsystem creates the GitHub repo, enforces credential safety before any push, opens per-phase PRs for human review, and never auto-merges.

### 26.0) Autonomy contract

Runs **autonomously** as part of the build phase (§7) and again at end-of-run. Auto-creates the repo, auto-pushes, auto-opens PRs. **Never auto-merges PRs** — those are explicitly the human review checkpoint.

**Hard limits:**
- **Never** creates a public repo unless intake captured `github_visibility: public` explicitly.
- **Never** pushes before `git check-ignore` enforcement (per §22.6) and credential scan pass.
- **Never** auto-merges its own PRs.
- **Never** overwrites an existing repo. If `<user>/<project-name>` already exists on GitHub, abort and surface the conflict.
- **Cross-platform:** requires `gh` CLI installed. If missing, surfaces the install command (`brew install gh`, `apt install gh`, etc.) and stops.

### 26.1) Activation triggers

Activates when ANY of:
- The intake captured `github_visibility` and `github_user` (default behavior — happens on every new project run).
- The build phase (§7) completes its first sub-phase and a commit is ready.
- The user types phrases like "push to github", "open a PR", "create a repo".
- The `/init-repo` slash command is invoked.

Does **not** activate for runs explicitly marked `branch_strategy: no-prs` (solo private repos that don't want GitHub at all).

### 26.2) Bootstrap sequence (runs once per project)

**Step 1 — CLI presence.**
- Run `gh --version`. If missing, surface install command and BLOCK.

**Step 2 — Authentication.**
- Run `gh auth status`. If not authenticated, surface `gh auth login` (one allowed prompt — second alongside `firebase login` in §21). Capture the authenticated user and write to Resource Manifest as `github_user`.

**Step 3 — Pre-flight gitignore check (mandatory, BLOCKING).**
- Verify `.gitignore` exists and contains: `.env`, `.env.local`, `.env.*`, `.env.*.local`, `secrets/`, `*.pem`, `*.key`.
- Run `git check-ignore .env.local` — must exit 0. Same for any file matching the patterns above that currently exists.
- If ANY check fails, **abort the entire §26 sequence** and surface to §20 with `severity: critical`. Do not push to GitHub.

**Step 4 — Pre-flight credential scan (mandatory, BLOCKING).**
- Scan the working tree (excluding `node_modules/`, `.git/`, `dist/`, `build/`, `.next/`, `.cache/`) for these patterns OUTSIDE of `.env*` files:
  - `sk-ant-[A-Za-z0-9_-]{20,}` (Anthropic)
  - `sk-[A-Za-z0-9]{40,}` (OpenAI-style)
  - `AIza[0-9A-Za-z_-]{35}` (Google API keys including Firebase web config — flag but don't BLOCK unless found in a tracked file outside `.env*`)
  - High-entropy strings >= 32 chars matching `[A-Za-z0-9_-]+` patterns in source files
- If any match is found outside `.env*` files, **BLOCK** and surface the offending file + line to the operator.

**Step 5 — Repo creation.**
- Run `gh repo create <github_user>/<project-name> --<visibility> --source=. --remote=origin --description "<one-line project description from intake>"`.
- `<visibility>` defaults to `--private`. Only `--public` if intake captured `github_visibility: public`.
- If the repo already exists, abort and surface — never overwrite.

**Step 6 — Initial commit.**
- `git add -A`
- `git commit -m "Initial scaffold from autonomous-project-agent v1.5.0\n\nProject: <name>\nStack: <stack summary>\nGenerated by autopilot."`

**Step 7 — Push `main`.**
- `git push -u origin main`

**Step 8 — Branch protection on `main` (PR-only, per intake default `branch_protection: pr-only`).**
- Via `gh api -X PUT repos/<user>/<repo>/branches/main/protection -f required_pull_request_reviews=null -f enforce_admins=true -f required_status_checks=null -f restrictions=null` (configured to require PRs but not approving reviews — appropriate for solo accounts; teams override via intake to `pr-plus-review`).
- Confirm protection is active via `gh api repos/<user>/<repo>/branches/main/protection`.

**Step 9 — Dependabot + secret scanning.**
- Write `.github/dependabot.yml` for the detected package manager (npm/pip/cargo/gomod/etc.). Weekly schedule by default.
- For private repos: enable secret scanning via `gh api -X PATCH repos/<user>/<repo> -f security_and_analysis[secret_scanning][status]=enabled` if the account tier supports it. If not (free private), skip and log a note.
- For public repos: secret scanning is auto-enabled by GitHub. Just verify.

**Step 10 — Set integration branch.**
- Create `dev` branch: `git checkout -b dev && git push -u origin dev`. All subsequent build sub-phases work on `dev`-derived feature branches; PRs target `main`.

**Step 11 — Log to `links.md`** under Projects:
```markdown
| GitHub repo | https://github.com/<user>/<repo> | Source of truth (visibility: private) | added <date> | source-of-truth: y |
| GitHub Actions | https://github.com/<user>/<repo>/actions | CI runs | added <date> | source-of-truth: y |
| Dependabot | https://github.com/<user>/<repo>/security/dependabot | Vulnerability alerts | added <date> | source-of-truth: y |
| Secret scanning | https://github.com/<user>/<repo>/security/secret-scanning | Credential leak detection | added <date> | source-of-truth: y |
```

### 26.3) Per-phase PR pattern (default `branch_strategy: per-phase-pr`)

For each build sub-phase in §7:

1. **Start fresh from `dev`:**
   ```
   git checkout dev && git pull
   git checkout -b feat/phase-<id>-<slug>
   ```

2. **Do the phase's work** (scaffolding, code, tests).

3. **Commit per phase:**
   ```
   git add -A && git commit -m "Phase <id>: <title>\n\n<bullet list of what changed>"
   ```

4. **Push the branch:**
   ```
   git push -u origin feat/phase-<id>-<slug>
   ```

5. **Open the PR:**
   ```
   gh pr create --base main --head feat/phase-<id>-<slug> --title "Phase <id> — <title>" --body "<auto-populated body, see below>"
   ```

6. **PR body template** (auto-populated from Resource Manifest entries and test results):
   ```markdown
   ## Summary
   <one paragraph describing what this phase delivered>

   ## Files changed
   - <list of files touched>

   ## Tests added
   - <list of new tests>

   ## Resource Manifest entries
   - <newly resolved NEEDs>
   - <newly accepted assumptions>

   ## Smoke / connectivity status
   - Layer 7 audit (this branch): PASS / WARN / FAIL
   - Emulator smoke (this branch): PASS / FAIL / not yet run

   ## How to review
   <one paragraph guiding the reviewer to the key files and what to verify>

   ---
   🤖 Opened autonomously by autonomous-project-agent v1.5.0. Not auto-merged. Review and merge on your schedule.
   ```

7. **Log the PR** to `links.md` under a new **PRs** section (created on first PR if missing):
   ```markdown
   ## PRs
   | # | Title | URL | Branch | Status | CI | Opened |
   |---|---|---|---|---|---|---|
   | <N> | Phase <id> — <title> | <pr-url> | feat/phase-<id>-<slug> | open | pending | <date> |
   ```

8. **Continue building on `dev`** for the next phase. `dev` rebases onto its own previous state, not onto `main` — `main` stays clean until operator merges PRs.

9. **Poll PR checks (polling-aware as of v1.6.0).** Invoke `poll_pr_checks` (§27) against the new PR. Default timeout 600s, interval 30s. The skill does NOT block waiting for PR checks during the run — `poll_pr_checks` is invoked **at end of run** (per phase 12) so the §12 final report shows real CI status, not "pending."
   - On `poll_pr_checks` success → update the PR's `links.md` row with the final CI status (`pass` / `fail` / `partial`) and update the PR body's "Smoke / connectivity status" section via `gh pr edit`.
   - On `poll_pr_checks` timeout (CI exceeded 10 minutes) → mark the PR's CI status as `timeout` in `links.md`, surface in §12 report. Do not BLOCK the deploy — CI on the PR is independent of the workflow's own §9 emulator smoke.
   - On any CI check returning FAIL → surface in §12 report as `PR #<N> has failing checks: <names>`. The operator decides whether to fix-and-push or merge anyway.

### 26.4) End-of-run behavior

When the run completes (deployment to dev alias succeeded, §12 reporting gate reached):
- **Do not auto-merge any PR.** They stay open.
- The §12 final report includes a **PR Queue** section listing all open PRs with one-line descriptions and review guidance.
- Surface to the operator: *"N PRs open against `main`. Review at your convenience: <list of URLs>. The agent will not merge any of them."*

### 26.5) Alternative branch strategies

If intake captured `branch_strategy: single-pr`:
- All build phases commit to a single `dev` branch.
- One PR opened at end of run: `gh pr create --base main --head dev --title "<project> initial build"` with all phases summarized in the body.

If intake captured `branch_strategy: no-prs`:
- Only allowed for solo private repos.
- Build phases commit directly to `main` (no feature branches, no PRs).
- Branch protection on `main` is **NOT** enabled in this mode.
- Surface a warning: *"branch_strategy: no-prs disables PR review and branch protection. Continue?"* In autopilot, this warning is logged but doesn't pause.

### 26.6) Failure handling

Every failure surfaces to §20 with `category: github`. Common failures:
- `gh` CLI not installed → BLOCK with install instructions.
- Not authenticated → BLOCK with `gh auth login`.
- Repo already exists → BLOCK, ask for different name.
- Pre-flight gitignore check fails → CRITICAL BLOCK, no push attempted.
- Pre-flight credential scan finds keys outside `.env*` → CRITICAL BLOCK, surface offending files.
- Push fails (auth expired, network) → retry once, then BLOCK.
- PR creation fails (rate limit, permissions) → retry once with backoff, then continue without that PR but flag in final report.

### 26.7) Autopilot interaction

All steps autonomous **except** `gh auth login` if the user isn't already authenticated. That's the one allowed prompt, same shape as `firebase login` in §21.

In autopilot mode:
- Visibility defaults to `private`.
- Branch strategy defaults to `per-phase-pr`.
- Branch protection defaults to PR-only (no required reviews).
- All defaults logged as assumptions in the Resource Manifest.

### 26.8) Integration with other sections

- **§22 (Anthropic key):** §22's `git check-ignore` check is a prerequisite for §26 Step 3. They share the same gitignore enforcement floor.
- **§24 (run state):** `github_repo_url`, `github_visibility`, `current_branch`, `open_prs` all written to `run-state.json` at every phase boundary. Resume on a different machine picks up the lifecycle.
- **§10 Layer 7:** Layer 7 results are reported in each PR's body (`Smoke / connectivity status` section).
- **§12 (reporting):** end-of-run report includes PR Queue with review guidance.

---

## 27) Polling discipline *(new in v1.6.0)*

Several sections of this skill wait on external state — emulator startup, deploy propagation, CI checks, rate-limit recovery, project creation. A one-shot check immediately after the triggering command is unreliable because the *triggering* command returns fast while the *resulting* state takes seconds to minutes to settle. This section codifies the right way to wait.

### 27.0) When polling is appropriate

Use polling when:
- The state you want is produced by an asynchronous process you don't control directly.
- No webhook, event, or push channel is available (or the available one is unreliable).
- The wait is bounded — there's a reasonable upper limit on how long it could legitimately take.

Don't poll when:
- A synchronous return is available (use it).
- A push/event channel exists (use it).
- The condition you're checking is on a local file system event (use file watchers, not polling).

### 27.1) Three rules of good polling

1. **Bounded.** Every poll has a maximum wait. Infinite polling is a bug. Default behavior on timeout: surface to §20 with `severity: high` and abort the surrounding action.
2. **Appropriate interval.** Match the natural rhythm of what you're waiting on. Emulator port-open ≈ 500ms. CDN propagation ≈ 10s. CI check completion ≈ 30s. Cron-style jobs ≈ minutes. Don't poll a 5-minute thing every 100ms; don't poll a 100ms thing every 30s.
3. **Distinguish errors from not-ready.** A 500 from the endpoint you're polling is different from a 404 saying "not deployed yet." Treat them differently. Errors should fail fast; not-ready should keep polling.

### 27.2) Named polling helpers

This skill uses four standard polling helpers. Other sections invoke them by name with their specified defaults. The helpers are conceptual — actual implementation is via the Claude Code `Bash` tool with the relevant `curl` / `gh` / `firebase` commands inside an `until` loop, or via `Monitor` with a stream-aware until-condition when watching a background process.

#### `poll_until_http_ok(url, timeout=300, interval=10, expected_status=200)`

Polls an HTTP endpoint until it returns the expected status code.

- **Default timeout:** 300 seconds (5 minutes) — appropriate for CDN propagation and Cloud Functions cold starts.
- **Default interval:** 10 seconds — most propagation completes within 30s; 10s gives 3 checks within the first window without burning quota.
- **Distinguishes:** connection refused / DNS failure / 5xx → keep polling (server still coming up). 4xx other than 404 → fail immediately (the request itself is wrong). Expected status → success.
- **Used by:** §11 (Firebase deploy verification).

Implementation pattern:
```sh
end=$(($(date +%s) + 300))
while [ $(date +%s) -lt $end ]; do
  status=$(curl -s -o /dev/null -w "%{http_code}" --max-time 5 "$URL" || echo "000")
  case "$status" in
    200) echo "ready"; exit 0 ;;
    000|404|5*) sleep 10 ;;  # not-ready / transient — keep polling
    4*) echo "client error: $status"; exit 1 ;;  # request is wrong, fail fast
  esac
done
echo "timeout"; exit 1
```

#### `poll_until_port_open(host="localhost", port, timeout=30, interval=0.5)`

Polls a TCP port until it accepts connections.

- **Default timeout:** 30 seconds — emulators and dev servers all start within this budget.
- **Default interval:** 0.5 seconds — tight because the wait is short and the check is local + cheap.
- **Distinguishes:** connection refused → keep polling. Connection reset → keep polling. Successful connect → success.
- **Used by:** §9 (Firebase emulator startup), any local dev server health-check.

Implementation pattern:
```sh
end=$(($(date +%s) + 30))
while [ $(date +%s) -lt $end ]; do
  if nc -z localhost "$PORT" 2>/dev/null; then exit 0; fi
  sleep 0.5
done
exit 1
```

#### `poll_pr_checks(pr_number, timeout=600, interval=30)`

Polls GitHub PR status checks until all checks have completed (PASS or FAIL).

- **Default timeout:** 600 seconds (10 minutes) — most CI runs complete within this budget; longer means investigate.
- **Default interval:** 30 seconds — CI status updates aren't more frequent than this anyway.
- **Distinguishes:** any check still pending → keep polling. All checks PASS → success. Any check FAIL → success-with-failure (return the failing check names, don't abort).
- **Used by:** §26 (PR lifecycle).

Implementation pattern:
```sh
end=$(($(date +%s) + 600))
while [ $(date +%s) -lt $end ]; do
  state=$(gh pr checks "$PR" --json state -q '[.[]] | map(.state) | unique')
  if ! echo "$state" | grep -q "PENDING\|IN_PROGRESS\|QUEUED"; then
    echo "$state"; exit 0
  fi
  sleep 30
done
echo "timeout"; exit 1
```

#### `poll_with_backoff_on_429(call, max_wait=300)`

Honors `Retry-After` on HTTP 429 responses and retries with exponential backoff if no `Retry-After` is provided.

- **Default max wait:** 300 seconds total (sum of all backoff intervals) — beyond this, the rate-limit is structural and the run should pause for operator action.
- **Backoff algorithm:** if `Retry-After: N` is present, sleep N seconds. If absent, exponential — 1s, 2s, 4s, 8s, 16s, 32s, capped at 60s per attempt.
- **Distinguishes:** 429 → backoff + retry. Other errors → propagate immediately.
- **Used by:** §23 (orchestrator throttling), §20 (any 429-triggered error).

Implementation pattern (pseudo):
```sh
attempt=0
total_wait=0
while [ $attempt -lt 10 ] && [ $total_wait -lt 300 ]; do
  response=$($CALL)
  status=$(echo "$response" | extract_status)
  if [ "$status" != "429" ]; then echo "$response"; exit 0; fi
  retry_after=$(echo "$response" | extract_header "Retry-After")
  if [ -n "$retry_after" ]; then
    sleep "$retry_after"; total_wait=$((total_wait + retry_after))
  else
    backoff=$(( (1 << attempt) > 60 ? 60 : (1 << attempt) ))
    sleep "$backoff"; total_wait=$((total_wait + backoff))
  fi
  attempt=$((attempt + 1))
done
exit 1
```

### 27.3) Project creation propagation polling

In addition to the four named helpers, §21 uses a Firebase-specific polling check:

#### `poll_firebase_project_exists(project_id, timeout=60, interval=5)`

After `firebase projects:create <project_id>`, poll `firebase projects:list --json` until `<project_id>` appears in the output. This handles the 10–30 second propagation delay between project creation and the project being usable for `firebase use --add`.

- **Default timeout:** 60 seconds.
- **Default interval:** 5 seconds.
- **Used by:** §21.

### 27.4) Polling and the error mitigation loop

Every polling helper that times out surfaces to §20 with:
- `category: polling`
- `severity: high` (timeout) or `severity: medium` (recovered after retry)
- `root_cause`: the specific external state that didn't settle (e.g. "Firebase Hosting CDN did not serve 200 for `<url>` within 300s")
- `mitigation_rule`: typically extends the timeout for the offending operation OR investigates an underlying infrastructure issue

Recurring polling timeouts on the same operation (≥2× across runs) get promoted to constraints — for example, a constraint adjusting the default `poll_until_http_ok` timeout for slow regional Firebase deployments.

### 27.5) Polling and autopilot

All polling is autonomous. Autopilot does not pause on polling waits — the run continues silently until either success or timeout. On timeout, autopilot surfaces the failure and either retries the surrounding action (if §20's mitigation rule says to) or BLOCKs (if the operation is critical and the timeout indicates real failure).

### 27.6) Why this section exists

Without polling discipline, the most common failure mode in a multi-tool autonomous run is **intermittent flakiness from race conditions** — emulators not yet listening, deploys not yet propagated, CI not yet started, rate limits hit on a retry storm. These failures are invisible at design time and infuriating at runtime. Codifying polling as a named pattern with named helpers and named timeouts turns "the run sometimes fails for no obvious reason" into "the run waited 300 seconds for the deploy to propagate and the propagation did not complete, which is a real signal."

---

## Short embed version

**Autonomous Project Research-to-Completion Skill (v1.6.0, consolidated single-skill):**
Take an initial project idea → clarify only needed information → detect project type and scope → research only project-specific requirements **including cost analysis under a free-tier-first stance** → **default to Firebase + React + React Context unless overridden** → **inventory all tools / materials / assets / APIs / libraries / configs / credentials / permissions / dependencies and classify every item as HAVE / NEED / GAP before any planning begins** → produce a Resource Manifest **and a Cost Manifest** → emit a one-page **plan preview** with one approval moment → organize broader findings into HAVE / NEED / GAPS → convert to a phased workflow from simple to complex (test-first within each code-mutating phase) → **scaffold smoke specs and MSW handlers during build, push per-phase PRs to a private GitHub repo with branch protection** → **run §9 emulator smoke with port-open polling** → run the **seven-layer security gate** (identity, session, roles, abuse protection, credential hygiene, general security audit, connectivity audit) → deploy through the correct Firebase environment only after all checks pass → **verify deploy via HTTP polling against the deployed URL before declaring success** → final report includes a PR Queue with real CI status (polled at end of run) for human review (the agent never auto-merges). Throughout: autonomously capture errors and distill constraints (§20), bootstrap Firebase CLI and project with creation-propagation polling (§21), locate and reference-wire Anthropic API key from local files (§22), fan out independent subtasks to parallel sub-agents under a diminishing-returns cap with 429 backoff polling (§23), checkpoint run state for resumability (§24), bootstrap GitHub with per-phase PRs (§26), and apply polling discipline throughout (§27 — four named helpers: `poll_until_http_ok`, `poll_until_port_open`, `poll_pr_checks`, `poll_with_backoff_on_429`). Single skill. Truly autonomous on activation, with one human approval at plan-time and PRs as the team review checkpoint.
