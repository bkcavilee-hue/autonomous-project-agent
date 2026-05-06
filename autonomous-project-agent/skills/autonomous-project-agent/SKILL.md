---
name: autonomous-project-agent
description: Use this skill when the user describes a project they want built — a web app, SaaS, tool, game, automation, mobile app, or anything similar — and wants it researched, planned, and built end-to-end with reinforced resource-gathering and authentication security gates before Firebase deployment. Trigger on requests like "build me X", "I want to make Y", "let's spin up Z", or any high-level project description that implies multi-phase work. Do NOT trigger for narrow single-step tasks (one bug fix, one function, one query). v2 — adds reinforced resource-gathering gate and five-layer authentication security gate before deployment. Supports "you decide" / "you decided" autopilot mode for continuous unattended execution.
---

# Autonomous Project Research-to-Completion Skill
### Version 2 — With Reinforced Resource-Gathering Gate + Authentication Security Gate

---

## 0) Autopilot mode — "you decide" / "you decided"

If the user says **"you decide"**, **"you decided"**, **"you choose"**, **"go autonomous"**, or any equivalent phrase that hands decision authority to the skill, enter **Autopilot Mode** for the remainder of the run.

### Autopilot mode behavior
- **Run all phases continuously without pausing for user approval between gates.** Do not stop at the end of intake, detection, resource-gathering, research, organize, modeling, workflow construction, validation, pre-deployment, authentication security, or deployment gates to ask "should I continue?" — just continue.
- **Make decisions on behalf of the user for any ambiguous intake field, project-type detection, scope sizing, framework choice, deployment target, or resource gap.** Pick the most reasonable default given context, document the choice in the Resource Manifest as an explicit assumption with stated impact-if-wrong, and proceed.
- **Do not ask clarification questions during intake.** Infer answers from the user's initial articulation. If a field cannot be inferred, assign a reasonable default and continue.
- **Treat every "PASS WITH NOTES" or "PASS WITH CONDITIONS" outcome as proceed-eligible.** Do not pause to confirm.
- **Skip every interactive checkpoint, confirmation prompt, or "ready for next phase?" gate.** The skill runs end-to-end as one continuous flow.
- **Surface a single consolidated report at the end** rather than per-phase status updates, unless a hard BLOCK occurs.

### What autopilot mode does NOT change
- **Hard BLOCKs in the authentication security gate still block.** A `FULL BLOCK` (e.g. unauthenticated writes in prod rules, exposed Firebase Admin key, no server-side token validation) stops deployment. Autopilot does not override security blocks — it only removes user-approval pauses, not safety floors.
- **A phase that fails validation twice still stops** per Section 8. Autopilot does not paper over real failures.
- **Tool-permission prompts at the Claude Code harness level are NOT controlled by this skill.** If the user wants the harness itself to stop prompting for tool approvals (Bash, Edit, Write, etc.), they must either run with `--dangerously-skip-permissions`, configure allowlists in `settings.json`, or accept a plan in plan-mode. The skill cannot grant itself those permissions — but inside whatever permission envelope the harness provides, the skill will run continuously without adding its own approval pauses.

### Autopilot mode acknowledgement
On entering autopilot mode, the skill must briefly acknowledge: *"Autopilot mode engaged — running all phases continuously. Decisions logged as assumptions in the Resource Manifest. Hard security blocks still stop deployment."* Then begin immediately.

### Exiting autopilot mode
Autopilot mode persists for the rest of the run unless the user says **"stop"**, **"pause"**, **"wait"**, or asks a direct question. On exit, the skill returns to normal gated behavior at the next phase boundary.

---

## Core purpose

This skill takes a user's initial project articulation, identifies what kind of project it is, researches what it needs, organizes that information into a structured plan, converts the plan into phased execution, audits and tests each phase, runs an authentication security gate before final deployment, and deploys through Firebase only when the correct environment and approval conditions are met.

The skill is **project-specific**, not generic. It must automatically determine which research, structure, logic, execution, and resource-gathering steps apply based on the project type chosen or detected. It must not force irrelevant sections onto the project.

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

### Intake rule
If critical information is missing, the skill must ask targeted follow-up questions before moving forward.
If more than half of the critical information is missing, it must ask all relevant questions for the detected project type.

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

**Authentication and security resources** *(new emphasis)*
- Auth provider selection (Firebase Auth, Auth0, Clerk, Supabase, custom JWT, etc.).
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
6. **Authentication security gate** *(new — see Section 10)*.
7. Deployment layer.

### Workflow rule
Each phase must build on the previous one and become more complex only after the simpler version is validated.

### The workflow must include
- Feature map.
- System map.
- Dependency map.
- State map.
- Test map.
- Auth and security map *(new)*.
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

Before the authentication security gate and deployment gate are reached, a final pre-deployment validation pass must be run on the integrated system.

### This pass must confirm
- All phases validated individually have been re-confirmed in integration.
- No new regressions introduced during integration.
- All resource manifest items marked NEED are now resolved.
- All GAPs marked as assumptions have been reviewed and accepted.
- Environment is confirmed as the correct target (dev, staging, or prod).

---

## 10) Authentication security gate *(new)*

This is a **mandatory blocking gate** that runs immediately before deployment. Nothing deploys until this gate passes.

### Purpose
The authentication security gate exists to verify that the system's identity, access control, and trust model are correctly implemented and cannot be bypassed, misconfigured, or exploited before real users encounter it.

### Gate structure

This gate has five layers. All five must pass before deployment proceeds.

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

### Authentication gate outcomes

| Result | Meaning |
|--------|---------|
| **FULL PASS** | All five layers passed. Deployment may proceed. |
| **PASS WITH CONDITIONS** | Minor findings noted. Deployment may proceed with documented exceptions and a remediation deadline. |
| **CONDITIONAL BLOCK** | Critical findings in one or more layers. Deployment is paused until specific items are resolved. |
| **FULL BLOCK** | Severe findings (e.g. unauthenticated writes in prod rules, exposed Firebase Admin key, no server-side token validation). Deployment is stopped. No exceptions. |

### Authentication gate rule
If the gate returns CONDITIONAL BLOCK or FULL BLOCK, the skill must surface the exact findings, the specific remediation steps required, and what must be re-tested before the gate can be re-run. It must not suggest workarounds or proceed anyway.

---

## 11) Firebase deployment gate

The deployment step must happen only after validation, the authentication security gate passes, and only through the approved Firebase environment.

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
- the authentication security gate returns PASS or PASS WITH CONDITIONS,
- the environment is confirmed correct,
- and the deployment path is approved.

---

## 12) Reporting gate

After execution, the skill must report:
- what was done,
- what passed,
- what failed,
- what authentication findings were surfaced and resolved,
- what remains,
- what was learned,
- what should be improved next.

This report should also capture user corrections and repeated preferences for future runs.

---

## 13) Field purpose logic

Every field must exist for a reason.

### Every required field must answer at least one:
- What is it?
- Why does it matter?
- What does it change?
- What happens if it is wrong or missing?

If a field does not affect planning, structure, validation, deployment, or resource gathering, it should not be mandatory.

---

## 14) Decision logic

The skill must make decisions based on project need, not template habit.

### It must:
- detect the project type,
- identify scope and risk,
- research only relevant needs,
- inventory required resources in full before planning,
- classify every resource as HAVE / NEED / GAP,
- resolve all NEEDs and GAPs before proceeding,
- produce a Resource Manifest,
- organize into have/need/gaps across the broader research phase,
- build a phased workflow,
- validate every phase,
- run the authentication security gate before deployment,
- deploy only through Firebase correctly.

### It must not:
- force universal all-in-one structure,
- overbuild simple projects,
- underbuild complex projects,
- skip audit or testing,
- skip the authentication security gate,
- deploy without environment discipline,
- continue with unresolved critical gaps.

---

## 15) Execution priorities

The skill should prioritize:
1. Project fit.
2. Scope discipline.
3. Research relevance.
4. Resource sufficiency *(confirmed before planning)*.
5. Structural clarity.
6. Validation.
7. Security *(including authentication gate)*.
8. Deployment correctness.
9. Reusability only where helpful.

---

## 16) Minimal operating rules

- Ask questions first.
- Research only what is needed.
- Inventory all tools, materials, assets, APIs, configs, credentials, and dependencies before planning.
- Produce a Resource Manifest before the first plan is written.
- Keep output project-specific.
- Use HAVE / NEED / GAPS.
- Build from simple to complex.
- Audit every phase.
- Test every phase.
- Run the authentication security gate before any deployment.
- Deploy only through Firebase correctly.
- Report what happened, including all auth findings.

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

Its intent is to be a **project-specific autonomous research and execution system** that understands what a project needs, inventories the full set of resources required to make it real before any plan is written, structures it correctly, validates each phase, enforces an authentication and security gate before deployment, and only then completes the deployment path through the correct Firebase environment.

---

## Short embed version

**Autonomous Project Research-to-Completion Skill:**
Take an initial project idea → clarify only needed information → detect project type and scope → research only project-specific requirements → **inventory all tools / materials / assets / APIs / libraries / configs / credentials / permissions / dependencies and classify every item as HAVE / NEED / GAP before any planning begins** → produce a Resource Manifest → organize broader findings into HAVE / NEED / GAPS → convert to a phased workflow from simple to complex → validate and audit each phase → **run the five-layer authentication security gate (identity verification, session integrity, role enforcement, abuse protection, credential hygiene) before deployment** → deploy through the correct Firebase environment only after all checks pass.
