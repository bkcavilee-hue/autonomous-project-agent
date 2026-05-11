---
name: autonomous-project-agent
description: 'Use this skill autonomously whenever the user describes a project they want built — a web app, SaaS, tool, game, automation, mobile app, or anything similar — and wants it researched, planned, and built end-to-end. The skill runs the full workflow (intake, project detection, resource-gathering, research with cost analysis, organize, modeling, workflow construction, validation, pre-deployment, six-layer security gate, Firebase deployment, reporting) and also runs the four embedded subsystems autonomously on activation: error capture and constraint distillation, Firebase CLI bootstrap, local-only Anthropic API key auto-discovery, and multi-agent orchestration capped by a diminishing-returns heuristic. Recommends Firebase + React + React Context as the default stack unless intake specifies otherwise, and treats free-tier-first as the explicit cost goal. Trigger on phrases like "build me X", "I want to make Y", "spin up Z", "create a project that", "ship a SaaS", any error/exception observed during a run, any mention of Firebase or Anthropic SDK usage, or any phase boundary with independent parallel work. Runs autonomously throughout — does not pause for confirmation outside the one unavoidable Firebase OAuth login. Do NOT trigger for narrow single-step tasks (one bug fix, one function, one query).'
---

# Autonomous Project Research-to-Completion Skill
### Version 3 — Single-skill consolidation: full workflow + error mitigation + Firebase bootstrap + Anthropic key locator + multi-agent orchestration + default stack + cost analysis + general security audit

---

## Core purpose

This skill takes a user's initial project articulation, identifies what kind of project it is, researches what it needs, organizes that information into a structured plan, converts the plan into phased execution, audits and tests each phase, runs a six-layer security gate before final deployment (five auth layers plus a general security audit), and deploys through Firebase only when the correct environment and approval conditions are met.

The skill is **project-specific**, not generic. It must automatically determine which research, structure, logic, execution, and resource-gathering steps apply based on the project type chosen or detected. It must not force irrelevant sections onto the project.

This skill embeds four subsystems that run autonomously throughout every workflow: an append-only error-mitigation loop, an autonomous Firebase CLI bootstrap, a local-only Anthropic API key locator, and a multi-agent orchestrator that fans out independent subtasks under a diminishing-returns cap. All four are sections inside this document (Sections 20–23). They are not separate skills.

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
6. **Six-layer security gate** *(see Section 10)*.
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

---

## 10) Six-layer security gate

This is a **mandatory blocking gate** that runs immediately before deployment. Nothing deploys until this gate passes.

### Purpose
The security gate exists to verify that the system's identity, access control, trust model, and general security posture are correctly implemented and cannot be bypassed, misconfigured, or exploited before real users encounter it.

### Gate structure

This gate has **six layers**. All six must pass before deployment proceeds. Layers 1–5 are authentication-focused; Layer 6 is a broader security audit.

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

### Six-layer gate outcomes

| Result | Meaning |
|--------|---------|
| **FULL PASS** | All six layers passed. Deployment may proceed. |
| **PASS WITH CONDITIONS** | Minor findings noted across one or more layers. Deployment may proceed with documented exceptions and a remediation deadline. |
| **CONDITIONAL BLOCK** | Critical findings in one or more layers. Deployment is paused until specific items are resolved. |
| **FULL BLOCK** | Severe findings (e.g. unauthenticated writes in prod rules, exposed Firebase Admin key, no server-side token validation, critical CVE in a runtime dependency, unparameterized SQL on user input). Deployment is stopped. No exceptions. |

### Security gate rule
If the gate returns CONDITIONAL BLOCK or FULL BLOCK, the skill must surface the exact findings, the specific remediation steps required, and what must be re-tested before the gate can be re-run. It must not suggest workarounds or proceed anyway.

---

## 11) Firebase deployment gate

The deployment step must happen only after validation, the six-layer security gate passes, and only through the approved Firebase environment.

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
- the six-layer security gate returns PASS or PASS WITH CONDITIONS,
- the environment is confirmed correct,
- and the deployment path is approved.

---

## 12) Reporting gate

After execution, the skill must report:
- what was done,
- what passed,
- what failed,
- what security findings (across all six layers) were surfaced and resolved,
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
- run the six-layer security gate before deployment,
- deploy only through Firebase correctly.

### It must not:
- force universal all-in-one structure,
- overbuild simple projects,
- underbuild complex projects,
- skip audit or testing,
- skip any layer of the six-layer security gate,
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
- Run the six-layer security gate before any deployment.
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

Its intent is to be a **project-specific autonomous research and execution system** that understands what a project needs, inventories the full set of resources required to make it real before any plan is written, structures it correctly, runs cost analysis under a free-tier-first stance, validates each phase, enforces a six-layer security gate before deployment, and only then completes the deployment path through the correct Firebase environment — all while autonomously logging errors, bootstrapping Firebase, locating Anthropic credentials, and orchestrating parallel sub-agents in the background.

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
- Any layer of the six-layer security gate returns FAIL or BLOCK.
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

## 22) Anthropic API key locator *(embedded subsystem, was a separate skill in v1.2.0)*

When a project the agent is building needs to call the Anthropic API (i.e. the project itself uses Claude, separate from Claude Code the harness), this subsystem locates an existing `ANTHROPIC_API_KEY` from prior local projects and wires it into the new project's `.env` automatically. Never asks for paste, never logs the key value, never queries any remote service.

### 22.0) Autonomy contract

Runs **autonomously** during Section 3 (resource-gathering) when the project type or stack indicates Anthropic SDK use. Auto-copies on find, no prompts.

**Hard limits:**
- **Local filesystem reads only.** Never queries the Anthropic console, never makes any network call to discover keys.
- **Never logs the key value.** Logs only file paths, last-4 character previews (e.g. `sk-ant-…AB12`), and discovery timestamps.
- **Never overwrites an existing `.env` line.** If the new project's `.env` already has `ANTHROPIC_API_KEY`, leaves it alone and logs the find as an alternate.
- **Never copies a key to a shared/committed file.** Writes only to `.env` (assumed gitignored).
- **Never uses or sends the key.** Discovery and wiring only.

### 22.1) Activation triggers

Activates when ANY of:
- Resource Manifest has `@anthropic-ai/sdk`, `anthropic` (Python), or any Anthropic API client in HAVE or NEED.
- Source code being scaffolded imports those packages.
- User types phrases like "anthropic api key", "claude api key", "wire up claude".
- An `ANTHROPIC_API_KEY` placeholder appears in any generated `.env.example` or scaffold file.

Does **not** activate for projects that only use Claude Code (the harness has its own auth).

### 22.2) Search locations (read-only)

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

### 22.3) Selection logic

When multiple keys found:
- Pick the key from the **most recently modified** source file.
- Skip keys that fail format validation (Anthropic keys start with `sk-ant-`).
- Skip keys whose source path contains `example`, `template`, `sample`, `test` — those are placeholders.
- Record up to 5 alternates in the links log.

### 22.4) Wiring action

1. Confirm `.env` exists; create if not.
2. Confirm `.env` is in `.gitignore`. Append `.env` to `.gitignore` if not (the only edit this subsystem makes outside its own log).
3. Append `ANTHROPIC_API_KEY=<value>` to `.env`. If the line already exists, leave it and treat the located key as alternate.
4. Verify by re-reading `.env`. Do not log the value.

### 22.5) Empty-state handling

If no key found, treat as new-project state:
- Write TODO line to `<project>/ai_docs/links.md` Credentials Inventory:
  ```
  | ANTHROPIC_API_KEY | https://console.anthropic.com/settings/keys | Generate new key | TODO — not yet provisioned |
  ```
- Add explicit assumption to Resource Manifest.
- Do not block the build — scaffolding can proceed. Block only at final pre-deployment if still missing.

### 22.6) Links log integration

Appends to `<project>/ai_docs/links.md` under **Credentials Inventory** only. Never writes to Projects, Services, or APIs sections.

**Entry format (key found):**
```markdown
| ANTHROPIC_API_KEY | <source-file-path> | Auto-copied from existing project | found <date> | preview: sk-ant-…AB12 |
```

**Entry format (alternates):**
```markdown
| ANTHROPIC_API_KEY (alternate) | <other-source-path> | Not selected; most recent file won | found <date> | preview: sk-ant-…CD34 |
```

**Strict rule:** source paths recorded; last-4 previews for disambiguation; **full key values NEVER recorded.**

### 22.7) Failure handling

Surfaces to Section 20 on any of: `.env` write fail, `.gitignore` edit fail, malformed key, key discovered in disallowed location (e.g. inside `node_modules`).

---

## 23) Multi-agent orchestration *(embedded subsystem, was a separate skill in v1.2.0)*

Speed up phases by fanning work out to parallel sub-agents, while preventing edit conflicts, redundant work, runaway token spend, rate-limit lockouts, and coordination overhead that exceeds the speedup it was supposed to deliver.

### 23.0) Autonomy contract

Runs **autonomously** at every phase boundary. No per-spawn confirmation. Reports a single summary line per phase.

**Hard limits:**
- **Never** spawns agents that would write to overlapping file sets.
- **Never** exceeds hard ceiling of 6 simultaneous agents.
- **Never** spawns parallel agents in serial-tier phases (integration, state-shared edits, deployment).
- **Never** continues spawning when measured throughput-per-token drops below threshold.
- **Always** uses git worktrees for isolation when agents write code, never plain branches.

### 23.1) Activation triggers

Activates automatically at every phase boundary with ≥2 independent subtasks.

| Phase | Tier | Typical fanout |
|---|---|---|
| Intake | serial | 1 |
| Project detection | serial | 1 |
| Resource-gathering | high | up to 6 |
| Research | high | up to 6 |
| Organize | medium | 2–3 |
| Modeling | medium | 2 |
| Workflow construction | serial | 1 |
| Build / scaffolding | medium | 3–4 |
| Validation | medium | up to 5 |
| Pre-deployment | serial | 1 |
| Security gate | medium | up to 6 (one per layer) |
| Deployment | serial | 1 |
| Reporting | serial | 1 |

User overrides via "do this in parallel" / "do this serially" are honored.

### 23.2) Independence detection

Before spawning:
1. **File-set overlap** — disjoint write sets only.
2. **Dependency order** — no consumer/producer in parallel.
3. **Resource contention** — stagger calls to the same rate-limited API.
4. **Shared state** — mutations to Resource Manifest / `links.md` / parent memory serialize after fanout.

If any check fails, drop the offending subtasks back to serial.

### 23.3) Worktree isolation

For code-writing phases:
- `git worktree add <project>/.worktrees/agent-<id>` on branch `agent/<phase>/<id>`.
- Agent operates only in its worktree.
- Orchestrator merges back sequentially, deterministic order.
- Merge conflict → throttle this phase down next run, log to Section 20, surface conflicting files.
- Worktrees cleaned up after merge; abandoned worktrees pruned at phase start.

For read-only phases (research, reading), no worktree needed.

### 23.4) Diminishing-returns cap

```
cap = min(
  independent_subtask_count,
  hard_ceiling_6,
  non_overlapping_file_sets,
  rate_limit_budget,
  measured_throughput_cap
)
```

**Measured throughput cap:** track per-agent `tokens_in`, `tokens_out`, `useful_output_units` (files written, tests passing, sections produced). Compute `efficiency = useful_output / tokens_total`. Maintain rolling median across last N agents. Most recent agent < 50% of median → stop spawning. Two consecutive below threshold → drop cap permanently for this run.

### 23.5) Throttling triggers

Reduce active count immediately on:
- API rate-limit error (HTTP 429, "rate limit exceeded", "quota exceeded").
- Merge conflict at merge time.
- Detected redundant work across agents.
- Host system resource pressure.

Throttle action: cancel youngest agent without output, refund its work to the queue.

### 23.6) Communication and merge protocol

Parallel agents do **not** communicate with each other. Each agent gets at spawn: scoped task, scoped read budget, scoped write budget (worktree-enforced), token budget, expected output schema.

Each agent returns: produced artifacts, self-reported confidence, flagged dependencies.

Orchestrator merges sequentially in deterministic order, runs Section 8 validation on merged result, records orchestrator pass to Resource Manifest.

### 23.7) Reporting

Per phase:
```
[orchestrator] phase=<phase-name> tier=<serial|medium|high> spawned=<N> completed=<K> throttled=<M> cap=<C> reason=<why-cap>
```

End-of-run section in Section 12 report:
```
## Multi-agent activity
- Total agents spawned: <N>
- Total worktrees created: <W>
- Merge conflicts encountered: <C>
- Throttle events: <T>
- Phases that fell back to serial: [list]
- Estimated wall-time saved vs serial: <minutes>
```

### 23.8) Cross-subsystem behavior

- **Section 20 (error mitigation)** receives all merge conflicts and throttle events as logged errors with `category: orchestration`. Recurring patterns get promoted to constraints that tighten the offending phase to serial.
- **Section 21 (Firebase bootstrap)** is always serial.
- **Section 22 (Anthropic key locator)** is always serial.

### 23.9) Autopilot interaction

Runs without per-phase confirmation. Cap decisions autonomous and logged as assumptions. Hard throttle-back recorded but does not pause. Hard BLOCKs from Section 10 still halt deployment.

---

## Short embed version

**Autonomous Project Research-to-Completion Skill (v1.3.0, consolidated single-skill):**
Take an initial project idea → clarify only needed information → detect project type and scope → research only project-specific requirements **including cost analysis under a free-tier-first stance** → **default to Firebase + React + React Context unless overridden** → **inventory all tools / materials / assets / APIs / libraries / configs / credentials / permissions / dependencies and classify every item as HAVE / NEED / GAP before any planning begins** → produce a Resource Manifest **and a Cost Manifest** → organize broader findings into HAVE / NEED / GAPS → convert to a phased workflow from simple to complex → validate and audit each phase → **run the six-layer security gate (identity verification, session integrity, role enforcement, abuse protection, credential hygiene, general security audit) before deployment** → deploy through the correct Firebase environment only after all checks pass. Throughout: autonomously capture errors and distill constraints (Section 20), bootstrap Firebase CLI and project (Section 21), locate and auto-wire Anthropic API key from local files (Section 22), and fan out independent subtasks to parallel sub-agents under a diminishing-returns cap (Section 23). Single skill. Truly autonomous on activation.
