# AI Features, Agents, and LLM Cost

Agent identity and credentials, authorization between AI tools and data, SSRF through agent fetches, retrieval across tenants, validating model output, human approval limits, independent review, AI evaluation in CI, agent memory and long tasks, integration protocols, and model spend.

Entry format: **Risk** is what goes wrong. **Fix** is the change to make. **Caveat** records where a common version of the advice overstates its case. **Check** marks an item where the risk is clear but the remedy depends on the application. **Test** names a concrete way to prove the fix.

---

### AI-01: Agents run on borrowed human credentials

**Risk.** Production agents commonly use a shared human login. A compromised or misbehaving agent then acts as that person, with no way to tell which agent did what.

**Fix.**
- Give each agent a unique identity and its own credential. Never share human logins.
- Issue short-lived keys and require approval before an agent touches production or sensitive data.
- Keep separate logs for every agent session so it is always clear who did what and when.

**Caveat.** Industry anecdotes about agent incidents support the direction but were not verified in detail. Agent control and logging tooling is maturing; the controls above do not depend on any particular tool.

---

### AI-02: The AI feature treats every message the same

**Risk.** An assistant built into the app accepts instructions from any user message. An attacker overrides its instructions and it returns records belonging to other users, or reveals internal logic.

**Fix.**
- Put an authorization layer between the AI and the data so the model can only reach records belonging to the authenticated user.
- Limit data access to the current user's records, never the whole database. Scope the database connection the AI uses to the current user.
- Add output validation that filters responses, hiding internal logic and other users' data.
- Prompts alone cannot enforce access. The boundary must be in code.

---

### AI-03: An agent that fetches any URL is an SSRF proxy

**Risk.** An agent with internal network access fetches whatever URL a user submits. Attackers use it to bypass the firewall, discover internal services, and map the network from error messages.

**Fix.**
- Restrict agent fetches to an approved list of external domains and block requests to internal address ranges.
- Pin every URL to its resolved address and revalidate on every redirect so a safe-looking URL cannot bounce to an internal one.
- Return only generic errors to the user on failed fetches. Log the detailed error internally for the team.
- Make sure the agent cannot help map internal services or leak network structure.

---

### AI-04: Retrieval answers one customer with another customer's documents

**Risk.** A retrieval-augmented system embeds documents from every customer into one index. A query from customer A retrieves chunks from customer B's documents and the answer leaks them.

**Fix.**
- Permission-scoped retrieval: tag every document with ownership and access context and filter retrieval by the requesting user's access before ranking.
- Prompt-injection filtering: scan and sanitize documents for embedded instructions before embedding them.
- Output verification: after retrieval, confirm that only authorized source chunks are included in the answer.
- Enforce user and tenant boundaries inside the retrieval path. An AI answer must never expose another customer's documents.

---

### AI-05: Raw model output should never reach a user

**Risk.** Model responses can be malformed, too long, off-policy, or wrong. Showing them directly produces errors, blank screens, and embarrassing content.

**Fix.**
- Validate every model response before display: schema, length, and prohibited content.
- If validation fails, retry with feedback describing the failure. Limit retries to 2 or 3 attempts.
- If retries fail, degrade gracefully: fall back to a simpler model, a cached response, or a human handoff.
- Never show a raw error or a blank screen. Always give the user something usable.
- Build this validation-retry-fallback layer once and reuse it everywhere a model is called.

---

### AI-06: Human approval alone does not make an agent action safe

**Risk.** An agent action that a human approved still breached a production database. Approval is a checkpoint, not a control.

**Fix.**
- Connect agents to production only with explicitly scoped access and technical safeguards that limit what an approved action can do.
- For support automation, build runbooks, automated or assisted triage, and clear escalation to humans.

**Caveat.** Products designed to be discovered and purchased by agents are not permission for an agent to make purchases or install software autonomously. Those remain human decisions.

---

### AI-07: The same model builds and reviews the code

**Fix.**
- Use independent, adversarial review of generated code. Rotate the builder and reviewer roles, and consider a different model or a different context for review.
- Include AI review in the pipeline, but keep an accountable human decision at the gate. AI agreement is not assurance.

**Caveat.** Review by the same model can still find defects, and using different models does not guarantee correctness. Independence improves the odds; it does not replace tests.

---

### AI-08: AI features need evaluation gates in CI, not eyeballing

**Fix.**
- Add evaluation suites, cost checks, and canary quality gates to CI for AI features.
- Where a model is used as a judge in CI, calibrate it against human-labeled examples.

**Caveat.** Probabilistic evaluations do not replace deterministic security assertions or human review where it matters. Keep both.

---

### AI-09: Agent memory and long-running tasks

**Risk.** An agent forgets what it was doing halfway through a job, or carries one undifferentiated context dump that grows without bound.

**Fix.**
- Maintain explicit task state, split long work into manageable steps, and validate progress between steps.
- Separate short-term working context from long-term memory stored in a vector database. Both need scoped access and a retention policy.
- Multi-agent designs with an orchestrator or conductor are an option, not an automatic delegation of authority. Define what each agent may do.

---

### AI-10: Protocol choice is not a security property

**Caveat.** Whether tools are wired through an integration protocol such as MCP or through direct API integrations the agent builds itself, both paths need the same authorization and scope controls. Choosing one over the other settles nothing about security.

---

### AI-11: An AI endpoint without a gateway is an open wallet

**Risk.** A public endpoint that calls a paid model, with no gateway in front of it, lets anyone run up the bill. Even a small per-call cost multiplies across users and calls. Per-model spend is often invisible until the invoice arrives.

**Fix.**
- Put a gateway in front of AI endpoints with authentication, per-user limits, and spend controls.
- Track token and cost usage per model and per feature.
- Route tasks to models by complexity and cost rather than sending everything to the largest model. Keep security boundaries consistent across providers.
- Cache repeated inputs and use prompt caching and batching where the provider supports them. Cache selectively and preserve tenant boundaries in the cache.

**Caveat.** Cost arithmetic in motivational examples is often inconsistent; measure your own usage. Self-hosting open-weight models for cost or IP control is an option to evaluate against the actual workload, not a universal saving.

---

### AI-12: Wrap the provider, design the fallback

**Fix.** Wrap each model provider integration behind your own interface, design fallback behavior for outages, and decide where generated data lives.

**Caveat.** A fallback must preserve authentication, privacy, and correctness. Accepting any response from any provider because the primary is down is not a fallback.

---

### AI-13: Detecting prompt theft

**Caveat.** One proposed approach plants per-prompt callback markers to detect when a system prompt has been copied. Callbacks are unreliable for attribution, can expose data, and must never turn into covert tracking of users. Treat prompt confidentiality as a secrets-handling problem instead.

---

### AI-14: Automated support has a boundary

**Fix.** Let automation handle routine support, and route nuanced disputes and escalations to humans by design.

**Caveat.** Percentages claimed for how much support automation can absorb are illustrative, not targets.
