# app-security-and-optimization

A Claude Code skill that keeps security, privacy, production readiness, and performance in the room while an application is being built, not only when someone remembers to ask for an audit.

It has three parts. **Operating rules** make Claude threat-model first and tell it which security changes it may make alone, which need your approval, and which are never acceptable. **The catalog** holds 140 concrete failure patterns with their fixes across fourteen domains: network, authentication, authorization and tenancy, API input, browser boundaries, payments and webhooks, supply chain, AI features and agents, observability, performance, delivery and backups, compliance, hardening controls, and privacy. **The workflow** scopes an app, audits only the domains it has, and reports findings with file-and-line evidence and stable entry IDs.

It is written for the kind of app that gets built quickly, often with heavy AI assistance, where the defaults fail in the same recurring ways: the API trusts the ID in the URL, the webhook is never verified, the price lives in the frontend, the service-role key bypasses every RLS policy, the agent fetches any URL it is given, personal data is collected with no plan to delete it.

The skill is self-contained. It depends on no other skill, plugin, or file outside its own folder.

## Install

Personal, available in every project:

```bash
git clone https://github.com/<you>/app-security-and-optimization.git ~/.claude/skills/app-security-and-optimization
```

Project-scoped, shared with everyone who clones the repository:

```bash
mkdir -p .claude/skills
git clone https://github.com/<you>/app-security-and-optimization.git .claude/skills/app-security-and-optimization
```

Or copy the folder to either location without git. Claude Code picks the skill up on the next session. Invoke it directly with `/app-security-and-optimization`.

To update, pull the repository in place.

## Make it proactive

Claude Code decides whether to consult a skill from its description. This skill's description tells Claude to apply it whenever a project has a login flow, user or personal data, an API, payments, AI features, file uploads, a mobile client, or a deployment pipeline, and to offer the triage pass at the start of work even when the user only asked for a feature. In practice that is enough for most sessions.

To make it a standing rule rather than a judgment call, add this to your `CLAUDE.md` (global at `~/.claude/CLAUDE.md`, or per project):

```markdown
## Application security and optimization

The `app-security-and-optimization` skill is consulted proactively. When a project has
any of: authentication, user-owned, multi-tenant, or personal data, an API, payments or
webhooks, AI features or agents, file uploads, a mobile client, or a production deployment,
read the matching reference file before implementing changes in that area, cite the entry
IDs applied, and offer the ten-minute triage at the start of work. Do not wait to be asked.
```

If you want a reminder injected at the start of every session regardless of project, a `SessionStart` hook in `~/.claude/settings.json` does that:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "echo 'If this project has auth, user data, an API, payments, AI features, or a deploy pipeline, apply the app-security-and-optimization skill proactively.'"
          }
        ]
      }
    ]
  }
}
```

## Use

Say what you want in plain language. Some examples:

- "Audit this app before launch."
- "Check my Stripe webhook and checkout."
- "Users are seeing each other's data. Find out why."
- "Why is this Next.js app slow and expensive at a thousand users?"
- "My agent can fetch URLs. Is that safe?"
- "Triage these npm audit findings."
- "We are about to store health records. What has to change?"
- "Add a password reset flow." (The skill reads the auth catalog before writing it.)

What happens:

1. **Threat model.** Claude names the trust boundaries and assets before touching code.
2. **Scope.** Claude inventories the stack: auth, database access path, hosting, payments, AI features, uploads, clients, CI, personal data.
3. **Select.** Only the reference files that match the inventory are read.
4. **Audit.** Every entry in those files is checked against the code and marked present, absent, or not applicable with a reason.
5. **Verify.** Product-specific claims are confirmed against current official documentation before they become recommendations.
6. **Report.** Findings are ranked by impact, each with file and line, the fix, and the entry ID. Decisions that are yours to make are listed separately.
7. **Fix.** Only when you ask, one finding at a time, each with a test that fails before and passes after. Changes to auth logic, CORS, rate limits, roles, uploads, new integrations, and new categories of sensitive data always get your approval first.

A ten-minute triage covers the highest-impact checks when there is no time for the full pass.

## How the catalog is organized

```
app-security-and-optimization/
├── SKILL.md                  # operating rules, workflow, triage, report format
├── README.md
├── LICENSE
└── references/
    ├── 01-network-infrastructure-and-transport.md       NET-01 … NET-12
    ├── 02-authentication-sessions-and-tokens.md         AUTH-01 … AUTH-15
    ├── 03-authorization-tenancy-and-data-access.md      ACC-01 … ACC-11
    ├── 04-api-input-validation-and-contracts.md         API-01 … API-07
    ├── 05-browser-client-and-frontend-boundaries.md     WEB-01 … WEB-09
    ├── 06-payments-and-webhooks.md                      PAY-01 … PAY-08
    ├── 07-secrets-dependencies-and-supply-chain.md      SUP-01 … SUP-05
    ├── 08-ai-features-agents-and-llm-cost.md            AI-01 … AI-14
    ├── 09-errors-logging-and-monitoring.md              OBS-01 … OBS-07
    ├── 10-performance-caching-and-scaling.md            PERF-01 … PERF-12
    ├── 11-delivery-environments-backups-and-rollback.md OPS-01 … OPS-10
    ├── 12-compliance-privacy-and-business-readiness.md  GOV-01 … GOV-09
    ├── 13-hardening-controls.md                         HARD-01 … HARD-13
    └── 14-privacy-and-data-protection.md                PRIV-01 … PRIV-08
```

Each entry has the same shape:

- **Risk**: what goes wrong.
- **Fix**: the change to make, as a checklist.
- **Caveat**: where the common advice overreaches. Part of the guidance, not a footnote.
- **Check**: the risk is clear but the remedy depends on the app. Investigate rather than prescribe.
- **Test**: a concrete way to prove the fix.

Entry IDs are stable, so reports and commit messages can cite them.

Example entry:

```markdown
### PAY-03: The browser sets the price

**Risk.** Checkout code puts the price in the frontend and the server passes it
through to Stripe. A user edits the request and pays less than intended.

**Fix.**
- Create checkout sessions only on the server.
- Let the client send a product or plan ID, never a dollar amount.
- Look up the price from trusted server-side data, and prefer Stripe Price IDs.
- Never trust an amount sent by the client. The server controls pricing.
```

## Ground rules built into the skill

- Threat model before hardening; abuse cases become the first tests.
- Numeric thresholds in the catalog are worked examples, not requirements.
- Product-specific claims are checked against current documentation before being recommended.
- Active scanners run only against environments you have explicitly authorized.
- Compliance and privacy obligations are surfaced as questions for counsel or an auditor, not as determinations.
- Existing security controls are not removed because they are old.
- The skill hardens your own applications. It does not produce attack tooling.

## Contributing

Pull requests are welcome. For a new entry, keep the shape above, put it in the domain file where an auditor would look for it, give it the next ID in that file, and add a triage line in `SKILL.md` only if it belongs in the top twenty. For a correction, cite the documentation that contradicts the current text.

## License

MIT. See `LICENSE`.
