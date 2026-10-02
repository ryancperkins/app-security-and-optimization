---
name: app-security-and-optimization
description: Proactive security, hardening, privacy, production-readiness, and performance skill for web, API, mobile, and AI-powered apps. Use it without being asked whenever the project has a login, session, or token flow; user, multi-tenant, or personal data; an API or server actions; payments or webhooks; AI features, RAG, or agents with tools; file uploads; a mobile client; third-party scripts; a database with caches or connection limits; dependency audit findings; or a deploy pipeline. Also use it when asked to audit, harden, secure, threat-model, or make an app production-ready, before a launch or enterprise deal, after AI-generated code was added, when a user can see another user's data, when the app is slow, expensive, or breaking under load, or when the conversation mentions OWASP, GDPR, RLS, CORS, JWT, OAuth, PKCE, CSP, SSRF, XSS, rate limiting, backups, or secrets in git. At the start of work in an applicable project, say the skill applies and offer the triage even if the user only asked for a feature.
---

# App Security and Optimization

## What this skill is

A self-contained security discipline for applications that get built quickly, often with AI assistance, where the defaults fail in the same recurring ways. It has three parts:

1. **Operating rules**: think like an attacker before hardening, and know which security changes you may make alone, which need the user's approval, and which are never acceptable.
2. **The catalog** in `references/`: concrete failure patterns and their fixes, one file per domain, each entry with a stable ID. It says what the attacker or the load does, why the default code fails, and exactly what to change. Where a popular version of the advice overstates its case, the entry carries a caveat.
3. **A workflow** for scoping, auditing, reporting, and fixing, plus a ten-minute triage for when there is no time for the full pass.

This skill depends on nothing outside its own folder.

Entry anatomy in the catalog:

- **Risk** is what goes wrong.
- **Fix** is the change to make, as a checklist.
- **Caveat** is where the common advice overreaches. The caveat is part of the guidance.
- **Check** marks an item where the risk is clear but the remedy depends on the application. Investigate, do not prescribe.
- **Test** names a concrete way to prove the fix holds.

Numeric values in entries (token lifetimes, request limits, hash cost factors, canary percentages) are worked examples. Choose the number from the application's risk and traffic, and say so in the report.

## Operating rules

### Threat model first

Controls bolted on without a threat model are guesses. Before hardening anything, spend five minutes as the attacker:

1. **Map the trust boundaries.** Where does untrusted data enter? HTTP requests, form fields, file uploads, webhooks, third-party APIs, message queues, and model output. Include values that look internal because the operating system handed them over: another process's command line or environment, filenames on a shared volume, a path inside a job payload. Trust follows who wrote a value, not which channel delivered it.
2. **Name the assets.** What is worth stealing or breaking: credentials, personal data, payment data, admin actions, money movement.
3. **Run STRIDE over each boundary**, as a quick lens rather than a ceremony:

| Threat | Ask | Typical mitigation |
|---|---|---|
| Spoofing | Can someone impersonate a user or service? | Authentication, signature verification |
| Tampering | Can data be altered in transit or at rest? | Integrity checks, parameterized queries, HTTPS |
| Repudiation | Can an action be denied later? | Audit logging of security events |
| Information disclosure | Can data leak? | Encryption, field allow lists, generic errors |
| Denial of service | Can it be overwhelmed? | Rate limiting, input size caps, timeouts |
| Elevation of privilege | Can a user gain rights they should not have? | Authorization checks, least privilege |

4. **Write abuse cases next to use cases.** For each feature ask "how would I misuse this?" and make that the first test.

If you cannot name the trust boundaries for a feature, you are not ready to secure it. Most breaches begin in design, not code.

### Three tiers of change

**Always do, without asking:**
- Validate all external input at the system boundary.
- Parameterize every database query.
- Encode output for its context; use the framework's escaping and never bypass it.
- Use HTTPS for all external communication.
- Hash passwords with bcrypt, scrypt, or argon2.
- Set security headers and use HttpOnly, Secure, SameSite cookies for sessions.
- Run the package manager's native audit against the committed lockfile before a release.

**Ask first, because the user owns the risk decision:**
- Adding or changing authentication flows or auth logic.
- Storing a new category of sensitive data (personal, payment, health).
- Adding a new external service integration.
- Changing CORS configuration.
- Adding a file upload handler.
- Modifying rate limiting or throttling.
- Granting elevated permissions or roles.
- Running any active scanner against any environment.

**Never do:**
- Commit secrets to version control.
- Log passwords, tokens, or full personal data.
- Treat client-side validation as a security boundary.
- Disable a security header for convenience.
- Use `eval` or `innerHTML` with user-provided or model-provided data.
- Store auth tokens in client-accessible storage such as `localStorage`.
- Expose stack traces or internal error details to users.
- Delete an existing security control or policy because it is old. Update from evidence.

### Rationalizations to refuse

| Said | Reality |
|---|---|
| "It is an internal tool" | Internal tools get compromised; attackers go for the weakest link. |
| "We will add security later" | Retrofitting costs far more than building it in. |
| "Nobody would try this" | Automated scanners will. Obscurity is not security. |
| "The framework handles it" | Frameworks provide tools, not guarantees. They must be used correctly. |
| "It is just a prototype" | Prototypes become production. |
| "Threat modeling is overkill" | Five minutes of "how would I attack this?" prevents the design flaws no control can patch later. |
| "It is only model output, just text" | That text can be a SQL statement, a script tag, or a shell command. |
| "The audit passed" | Audits match known advisories; they do not detect a newly malicious package or make install scripts safe. |
| "Collect it now, we might need it" | Data you do not hold cannot be breached or subpoenaed. "Might need" is breach scope. |
| "We will handle deletion manually" | Manual erasure misses backups, caches, and analytics copies. Design the schema so data is findable and erasable. |
| "Compliance is legal's problem" | Export, deletion, retention, and consent are schema and code. |

## Proactive use

This skill is meant to be in the room before anyone asks for a security review. The patterns it catalogs are cheap to prevent while a feature is being written and expensive to find after launch.

- **At the start of work in an applicable project**, say in one line that the catalog applies and name the domains the project touches. Offer the ten-minute triage. If the user declines, continue with their task and apply the relevant entries silently as you build.
- **While implementing a feature** that touches any catalog domain (a login flow, an endpoint that takes an ID, a webhook, a checkout, an upload, an agent tool, a cache, a migration), read the matching reference file before writing the code and build to its entries. Cite the entry IDs you applied in your summary.
- **During code review** of changes in those domains, check the diff against the matching file and cite entry IDs for anything missing.
- **Before a deploy or launch**, run the full workflow below for every domain the inventory selects.

Do not turn cosmetic or unrelated work into an audit. The trigger is a real touchpoint with a catalog domain.

## Workflow

### 1. Scope the application

Before opening a reference file, establish what the app is made of. Read the dependency manifest, environment variable names, the route or handler layout, and any infrastructure config. Fill in this inventory:

```
Stack inventory
- Framework / runtime:
- Auth provider or custom auth:            (sessions, JWT, magic links, OAuth providers)
- Database and access path:                (ORM, raw SQL, RLS, service-role keys)
- Hosting / edge / proxy:                  (serverless, VPS, CDN, WAF)
- Payments and webhooks:                   (provider, webhook endpoints, fulfillment path)
- AI features:                             (chat, RAG, agents with tools or fetch, model providers)
- Mobile or wrapped clients:
- File uploads / server-side fetches:
- Third-party scripts on sensitive pages:
- CI/CD, environments, backups:
- Personal or regulated data:              (classes held, where it flows, retention)
```

Everything blank is a domain you can skip. Everything filled in selects reference files in step 2.

### 2. Select reference files

| Signal in the inventory | Read |
|---|---|
| Multiple services, a proxy, a VPS, public endpoints, mobile transport, outbound email | `references/01-network-infrastructure-and-transport.md` |
| Any login, session, token, password reset, magic link, OAuth, or SSO | `references/02-authentication-sessions-and-tokens.md` |
| Resource IDs in routes, admin pages, multi-tenant data, RLS, update handlers | `references/03-authorization-tenancy-and-data-access.md` |
| Form handlers, search, raw SQL, email templates, GraphQL, public API contracts | `references/04-api-input-validation-and-contracts.md` |
| Browser clients, CORS config, React Server Components, Next.js middleware, third-party scripts | `references/05-browser-client-and-frontend-boundaries.md` |
| Checkout, subscriptions, any inbound webhook | `references/06-payments-and-webhooks.md` |
| Package manifests, lockfiles, `.env` files, keys in git, AI instruction or skill files | `references/07-secrets-dependencies-and-supply-chain.md` |
| LLM calls, RAG, agents, tool use, model spend | `references/08-ai-features-agents-and-llm-cost.md` |
| Error pages, logging, alerting, status pages, incident handling | `references/09-errors-logging-and-monitoring.md` |
| Slow queries, cache layers, connection limits, long requests, hosting limits, cloud bill | `references/10-performance-caching-and-scaling.md` |
| Branching, deploys, environments, migrations, backups, rollback, pre-launch testing | `references/11-delivery-environments-backups-and-rollback.md` |
| Health, payment, or personal data obligations; account deletion; enterprise procurement; legal documents | `references/12-compliance-privacy-and-business-readiness.md` |
| Password storage, output encoding, header baseline, SSRF-safe fetch, file uploads, destructive path operations, audit triage, install hygiene, limiter storage, LLM hardening | `references/13-hardening-controls.md` |
| Any personal data: classification, purpose, retention, deletion, consent, vendors, logging | `references/14-privacy-and-data-protection.md` |

Read only the files the inventory selects. For a focused request ("check my Stripe webhook") read one file. For a pre-launch audit read every file the inventory selects, in table order.

### 3. Audit against each entry

For every entry in a selected file:

1. Locate the code path the entry describes. Search for the concrete artifacts it names: the header, the config flag, the SDK call, the route pattern, the storage location.
2. Decide: **present**, **absent**, or **not applicable**. "Not applicable" needs a one-line reason tied to the inventory.
3. For each absent control, record a finding with the file and line, the entry ID, and the fix from the entry.
4. For Check entries, record what you found and what decision the user needs to make.

Audit by reading code and configuration, running the project's own tests, and issuing requests against environments the user controls. Hitting an endpoint directly without the form, submitting a prohibited field, or sending an encoded redirect value are all in scope on a local or staging environment. Active scanners such as OWASP ZAP or Burp Suite run only against environments the user has explicitly authorized (NET-12). Never probe systems the user does not own.

Caveats are findings too. If the code relies on something an entry calls insufficient on its own (an obscured admin path, a non-guessable webhook URL, UUIDs in place of ownership checks, a hidden button in place of API authorization, a wildcard CORS origin "because the browser blocks it"), report that reliance.

### 4. Verify against the actual stack

Entries describe general patterns and sometimes name specific products as worked examples. Before recommending a product-specific change, confirm the behavior against current official documentation for the version in use. Several caveats exist precisely because a general claim did not hold for a specific framework (Server Actions, Server Components, HttpOnly cookies and third-party scripts, hosting plan limits). When the documentation contradicts an entry, follow the documentation and note the discrepancy.

### 5. Prioritize and report

Rank findings by what an attacker or an outage can do with them, not by how many entries mention them:

1. Anyone can read or change another user's data, or act as another user (authorization, tenancy, session, token, webhook forgery).
2. Secrets or credentials are exposed (browser bundle, git history, logs, error pages, shared agent credentials).
3. Untrusted input reaches a sink unvalidated (SQL, templates, redirects, server-side fetch targets, file paths, model output).
4. Money can be manipulated (client-set prices, unverified payment confirmation, duplicate processing).
5. Availability and cost (missing rate limits, missing indexes, cascading failures, unbounded AI spend).
6. Operability (no monitoring for silent failures, no tested restore, no rollback path).
7. Privacy, readiness, and compliance prompts.

Use this report structure:

```
# Security and optimization audit: <app name>

## Scope
Stack inventory, trust boundaries and assets named, environments examined, what was out of scope and why.

## Findings
Most severe first.
### <short title> — <Critical | High | Medium | Low>
- Where: path:line (or config key / route)
- What: one sentence on the failure pattern
- Evidence: what you observed (request, response, code excerpt)
- Fix: the concrete change, taken from the entry
- Entry: <ID>

## Controls confirmed present
Short list; cite the entry each satisfies.

## Decisions needed
Check entries and ask-first items, with what was found and the choice the user has to make.

## Not applicable
Entry IDs skipped, with the inventory reason.

## Caveats and unverified items
Anything you could not confirm, and any entry whose claim the official docs contradicted.
```

### 6. Fix, test, re-audit

When the user asks for fixes, apply them one finding at a time, smallest change first, each with a test that reproduces the failure before the fix and passes after. The catalog names the tests: submit prohibited fields to every update endpoint (ACC-02), call endpoints directly without the form (API-01), submit angle brackets and script tags to template fields (API-03), send encoded and protocol-relative redirect values (AUTH-03), replay a webhook with a reused event ID (PAY-02), run a production build and search the output for secrets (WEB-05), restore the latest backup into a scratch environment (OPS-05), delete a test account and confirm every store is clean (PRIV-03). After the fixes, re-run step 3 for the affected domain.

Keep existing controls unless the evidence says they are wrong (SUP-04). AI agreement is not assurance; keep deterministic security tests and an accountable human decision in the pipeline (AI-07, AI-08).

## Fast path: ten-minute triage

When there is no time for a full audit, check these first. Each line points at the entry with the full guidance.

- Does every endpoint that takes a resource ID verify the authenticated user owns it? (ACC-01, ACC-08)
- Does any update handler write the whole request body to the database? (ACC-02)
- Does any API route use a database service-role key, bypassing row-level security? (ACC-04, ACC-05)
- Are JWT signatures, expiry, and issuer verified with the provider's server-side SDK on every request? (AUTH-05)
- Are passwords hashed with bcrypt, scrypt, or argon2, and are session cookies Secure, HttpOnly, and SameSite? (HARD-01, AUTH-02)
- Are auth tokens in `localStorage`? (AUTH-08)
- Is every redirect parameter in login, logout, magic-link, and OAuth flows checked against an allow list of exact domains? (AUTH-03, AUTH-01, AUTH-07)
- Do reset and magic-link tokens expire quickly and die after first use? (AUTH-04, AUTH-01)
- Is every webhook signature verified, and are processed event IDs stored to reject replays? (PAY-01, PAY-02)
- Does the server set prices from its own data and provision access only after a verified payment event? (PAY-03, PAY-04)
- Does the browser bundle, production build, or git history contain a secret? (WEB-05, WEB-08, SUP-03, HARD-12)
- Does raw SQL anywhere take user input outside ORM parameter methods, or does user or model data reach `innerHTML` or `eval`? (API-02, HARD-02, HARD-13)
- Does the server fetch any user-influenced URL without a host allow list and private-range check? (HARD-05, AI-03)
- Is CORS a wildcard or a reflected origin, especially with credentials enabled? (WEB-01)
- Are framing headers set, and is there a CSP that limits script sources on login, checkout, and admin pages? (WEB-02, WEB-06, HARD-03)
- Do production error responses hide database names, paths, queries, and stack traces? (OBS-01)
- Do agents and AI tools see only the current user's records, with their own identity and credentials, and is model output validated before use? (AI-02, AI-01, AI-04, AI-05)
- Are there indexes on filtered columns, selective column lists, and slow-query logging? (PERF-01)
- Are login and other sensitive endpoints rate limited, with counters in a shared store when more than one instance runs? (NET-04, HARD-11, ACC-03)
- Is personal data classified, collected against a purpose, and deletable from every store? (PRIV-01, PRIV-02, PRIV-03)
- Is the production branch protected, are environments separated, and has a restore from backup been tested? (OPS-01, OPS-02, OPS-05)

## Boundaries

- Compliance entries (health data, EU AI Act, SOC 2, tax, retention, legal documents, breach-notification deadlines) are prompts for a qualified review. State them as questions for the user's counsel, auditor, or accountant, not as determinations.
- Hosting plan limits, product retirements, and vendor behaviors named in entries must be checked against current documentation before anyone relies on them.
- Statistics about how many apps are affected, incident counts, and dollar figures are illustrative. Do not repeat them as facts in a report.
- This skill audits and hardens the user's own applications. It does not produce exploit tooling or guidance for attacking systems the user does not control.

## Reference index

| File | Entries | Covers |
|---|---|---|
| `references/01-network-infrastructure-and-transport.md` | NET-01 to NET-12 | Segmentation, mTLS and service tokens, WAF and Fail2Ban, origin protection, rate limiting, dependency pinning, server hardening, mobile transport, email authentication, scanners |
| `references/02-authentication-sessions-and-tokens.md` | AUTH-01 to AUTH-15 | Magic links, session fixation, redirect abuse, password reset, JWT, PKCE and state, OAuth redirects, token storage and lifetimes, idle sessions, provider choice |
| `references/03-authorization-tenancy-and-data-access.md` | ACC-01 to ACC-11 | Ownership checks, mass assignment, admin surfaces, RLS and service roles, tenant isolation, tenant-scoped caches, RBAC across layers, direct DB access |
| `references/04-api-input-validation-and-contracts.md` | API-01 to API-07 | Server-side schemas, raw SQL, template injection, GraphQL introspection, context-specific encoding, API versioning |
| `references/05-browser-client-and-frontend-boundaries.md` | WEB-01 to WEB-09 | CORS, clickjacking, Server Component payloads, middleware matchers, server-only modules, third-party scripts and CSP, security headers, secrets in bundles, accessibility |
| `references/06-payments-and-webhooks.md` | PAY-01 to PAY-08 | Webhook signatures and replay, server-side pricing, provisioning after verified payment, fulfillment state, webhook monitoring, failed payments, disputes |
| `references/07-secrets-dependencies-and-supply-chain.md` | SUP-01 to SUP-05 | Malicious packages, dependency audits, lockfile integrity, secrets in git, rotation, AI instruction files as supply chain, vendor abstraction |
| `references/08-ai-features-agents-and-llm-cost.md` | AI-01 to AI-14 | Agent identity, tool authorization, SSRF, RAG tenancy, output validation, approval limits, independent review, AI evals in CI, agent memory, model spend |
| `references/09-errors-logging-and-monitoring.md` | OBS-01 to OBS-07 | Safe error responses, structured logs, silent failures, outage detection, incident practice, session replay privacy |
| `references/10-performance-caching-and-scaling.md` | PERF-01 to PERF-12 | Indexes, pooling, scaling order, cache policy, consistency, background jobs, circuit breakers, hosting limits, database selection, cost attribution |
| `references/11-delivery-environments-backups-and-rollback.md` | OPS-01 to OPS-10 | Branch protection, environments, canaries, migrations, backups and restores, CI integrity, testing beyond the happy path, readiness taxonomy |
| `references/12-compliance-privacy-and-business-readiness.md` | GOV-01 to GOV-09 | Regulatory prompts, health data, account deletion, legal documents, SOC 2 and procurement, analytics privacy, business and tax setup |
| `references/13-hardening-controls.md` | HARD-01 to HARD-13 | Password hashing and sessions, output encoding, header baseline, response allow lists, SSRF-safe fetch, boundary schemas, uploads, destructive paths, audit triage, install hygiene, limiter storage, repo secrets, LLM hardening |
| `references/14-privacy-and-data-protection.md` | PRIV-01 to PRIV-08 | Data classification, purpose and minimization, retention and deletion, data-subject rights, consent and vendors, residency, logging, breach clocks |
