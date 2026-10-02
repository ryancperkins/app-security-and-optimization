---
name: app-security-and-optimization
description: Proactive security, production-readiness, and performance catalog for web, API, mobile, and AI-powered apps. Use it without being asked whenever the project has a login, session, or token flow; user-owned or multi-tenant data; an API or server actions; payments or inbound webhooks; AI features, RAG, or agents with tools; a mobile client; third-party scripts; a database with caches or connection limits; or a deploy pipeline. Also use it when asked to audit, harden, secure, or make an app production-ready, before a launch or enterprise deal, after AI-generated code was added, when a user can see another user's data, when the app is slow, expensive, or breaking under load, or when the conversation mentions vibe coding, OWASP, RLS, CORS, JWT, OAuth, PKCE, CSP, SSRF, Stripe, Clerk, Supabase, Next.js, rate limiting, backups, or secrets in git. At the start of work in an applicable project, say the skill applies and offer the triage pass even if the user only asked for a feature.
---

# App Security and Optimization

## What this skill is

A catalog of concrete failure patterns and their fixes for applications that were built quickly, often with AI assistance, where the defaults go wrong in the same recurring ways. Each entry states what the attacker or the load does, why the default code fails, and exactly what to change. Where a popular version of the advice overstates its case, the entry carries a caveat so the fix is applied without repeating the overstatement.

The catalog lives in `references/`, one file per domain. This file tells you how to use it.

Entry anatomy:

- **Risk** is what goes wrong.
- **Fix** is the change to make, as a checklist.
- **Caveat** is where the common advice overreaches. The caveat is part of the guidance.
- **Check** marks an item where the risk is clear but the remedy depends on the application. Investigate, do not prescribe.
- **Test** names a concrete way to prove the fix holds.

Numeric values in entries (token lifetimes, request limits, canary percentages, observation windows) are worked examples. Choose the number from the application's risk and traffic, and say so in the report.

## Proactive use

This skill is meant to be in the room before anyone asks for a security review. The patterns it catalogs are cheap to prevent while a feature is being written and expensive to find after launch.

- **At the start of work in an applicable project**, say in one line that the catalog applies and name the domains the project touches. Offer to run the ten-minute triage. If the user declines, continue with their task and apply the relevant entries silently as you build.
- **While implementing a feature** that touches any catalog domain (a login flow, an endpoint that takes an ID, a webhook, a checkout, an agent tool, a cache, a migration), read the matching reference file before writing the code and build to its entries. Mention the entry IDs you applied in your summary so the user can look them up.
- **During code review** of changes in those domains, check the diff against the matching file and cite entry IDs for anything missing.
- **Before a deploy or launch**, run the full workflow below for every domain the inventory selects.

Do not turn cosmetic or unrelated work into an audit. The trigger is a real touchpoint with a catalog domain, not the mere existence of the skill.

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
- Third-party scripts on sensitive pages:
- CI/CD, environments, backups:
- Regulated data:                          (health, payment, personal data, cross-border users)
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
| Health, payment, or personal data; account deletion; enterprise procurement; legal documents | `references/12-compliance-privacy-and-business-readiness.md` |

Read only the files the inventory selects. For a focused request ("check my Stripe webhook") read one file. For a pre-launch audit read every file the inventory selects, in table order.

### 3. Audit against each entry

For every entry in a selected file:

1. Locate the code path the entry describes. Search for the concrete artifacts it names: the header, the config flag, the SDK call, the route pattern, the storage location.
2. Decide: **present**, **absent**, or **not applicable**. "Not applicable" needs a one-line reason tied to the inventory.
3. For each absent control, record a finding with the file and line, the entry ID, and the fix from the entry.
4. For Check entries, record what you found and what decision the user needs to make.

Audit by reading code and configuration, running the project's own tests, and issuing requests against environments the user controls. Hitting an endpoint directly without the form, submitting a prohibited field, or sending an encoded redirect value are all in scope on a local or staging environment. Active scanners such as OWASP ZAP or Burp Suite run only against environments the user has explicitly authorized (NET-12). Never probe production systems the user does not own.

Caveats are findings too. If the code relies on something an entry calls insufficient on its own (an obscured admin path, a non-guessable webhook URL, UUIDs in place of ownership checks, a hidden button in place of API authorization, a wildcard CORS origin "because the browser blocks it"), report that reliance.

### 4. Verify against the actual stack

Entries describe general patterns and sometimes name specific products as worked examples. Before recommending a product-specific change, confirm the behavior against current official documentation for the version in use. Several caveats exist precisely because a general claim did not hold for a specific framework (Server Actions, Server Components, HttpOnly cookies and third-party scripts, hosting plan limits). When the documentation contradicts an entry, follow the documentation and note the discrepancy.

### 5. Prioritize and report

Rank findings by what an attacker or an outage can do with them, not by how many entries mention them:

1. Anyone can read or change another user's data, or act as another user (authorization, tenancy, session, token, webhook forgery).
2. Secrets or credentials are exposed (browser bundle, git history, logs, error pages, shared agent credentials).
3. Untrusted input reaches a sink unvalidated (SQL, templates, redirects, agent fetch targets, model output shown to users).
4. Money can be manipulated (client-set prices, unverified payment confirmation, duplicate processing).
5. Availability and cost (missing rate limits, missing indexes, cascading failures, unbounded AI spend).
6. Operability (no monitoring for silent failures, no tested restore, no rollback path).
7. Readiness and compliance prompts.

Use this report structure:

```
# Security and optimization audit: <app name>

## Scope
Stack inventory, environments examined, what was out of scope and why.

## Findings
Most severe first.
### <short title> — <Critical | High | Medium | Low>
- Where: path:line (or config key / route)
- What: one sentence on the failure pattern
- Evidence: what you observed (request, response, code excerpt)
- Fix: the concrete change, taken from the entry
- Entry: <ID> (so the reader can look up the full note)

## Controls confirmed present
Short list; cite the entry each satisfies.

## Decisions needed
Check entries, with what was found and the choice the user has to make.

## Not applicable
Entry IDs skipped, with the inventory reason.

## Caveats and unverified items
Anything you could not confirm, and any entry whose claim the official docs contradicted.
```

### 6. Fix, test, re-audit

When the user asks for fixes, apply them one finding at a time, smallest change first, each with a test that reproduces the failure before the fix and passes after. The catalog names the tests: submit prohibited fields to every update endpoint (ACC-02), call endpoints directly without the form (API-01), submit angle brackets and script tags to template fields (API-03), send encoded and protocol-relative redirect values (AUTH-03), replay a webhook with a reused event ID (PAY-02), run a production build and search the output for secrets (WEB-05), restore the latest backup into a scratch environment (OPS-05). After the fixes, re-run step 3 for the affected domain.

Keep existing controls unless the evidence says they are wrong. Do not delete a security policy or instruction merely because it is old (SUP-04). AI agreement is not assurance; keep deterministic security tests and an accountable human decision in the pipeline (AI-07, AI-08).

## Fast path: ten-minute triage

When there is no time for a full audit, check these first. Each line points at the entry with the full guidance.

- Does every endpoint that takes a resource ID verify the authenticated user owns it? (ACC-01, ACC-08)
- Does any update handler write the whole request body to the database? (ACC-02)
- Does any API route use a database service-role key, bypassing row-level security? (ACC-04, ACC-05)
- Are JWT signatures, expiry, and issuer verified with the provider's server-side SDK on every request? (AUTH-05)
- Are auth tokens in `localStorage`, or are session cookies missing Secure, HttpOnly, and SameSite? (AUTH-08, AUTH-02)
- Is every redirect parameter in login, logout, magic-link, and OAuth flows checked against an allow list of exact domains? (AUTH-03, AUTH-01, AUTH-07)
- Do reset and magic-link tokens expire quickly and die after first use? (AUTH-04, AUTH-01)
- Is every webhook signature verified, and are processed event IDs stored to reject replays? (PAY-01, PAY-02)
- Does the server set prices from its own data and provision access only after a verified payment event? (PAY-03, PAY-04)
- Does the browser bundle, production build, or git history contain a secret? (WEB-05, WEB-08, SUP-03)
- Does raw SQL anywhere take user input outside ORM parameter methods? (API-02)
- Is CORS a wildcard or a reflected origin, especially with credentials enabled? (WEB-01)
- Are framing headers set, and is there a CSP that limits script sources on login, checkout, and admin pages? (WEB-02, WEB-06)
- Do production error responses hide database names, paths, queries, and stack traces? (OBS-01)
- Do agents and AI tools see only the current user's records, with their own identity and credentials? (AI-02, AI-01, AI-04)
- Can an agent fetch arbitrary URLs, including internal addresses? (AI-03)
- Is model output validated before it reaches a user? (AI-05)
- Are there indexes on filtered columns, selective column lists, and slow-query logging? (PERF-01)
- Are login and other sensitive endpoints rate limited? (NET-04, NET-05, ACC-03)
- Is the production branch protected, are environments separated, and has a restore from backup been tested? (OPS-01, OPS-02, OPS-05)

## Boundaries

- Compliance entries (health data, EU AI Act, SOC 2, tax, retention, legal documents) are prompts for a qualified review. State them as questions for the user's counsel, auditor, or accountant, not as determinations.
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
