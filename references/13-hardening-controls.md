# Hardening Controls

Implementation-level controls that apply across domains: password hashing and session configuration, output encoding, header baselines, response field allow lists, SSRF-safe fetching, boundary schema validation, file uploads, destructive operations on derived paths, dependency audit triage, supply-chain hygiene, rate-limiter storage, repository secrets, and LLM feature hardening.

Entry format: **Risk** is what goes wrong. **Fix** is the change to make. **Caveat** records where a common version of the advice overstates its case. **Check** marks an item where the risk is clear but the remedy depends on the application. **Test** names a concrete way to prove the fix.

---

### HARD-01: Password storage and session configuration

**Risk.** Plaintext or weakly hashed passwords turn a database leak into every user's credentials. A session cookie without the right flags is readable by scripts, sent over plain HTTP, or attached to cross-site requests.

**Fix.**
- Hash passwords with bcrypt, scrypt, or argon2. For bcrypt use a cost factor of at least 12. Compare with the library's constant-time compare, never with string equality.
- Load the session secret from the environment, never from code.
- Configure the session cookie as `httpOnly: true`, `secure: true`, `sameSite: 'lax'` (or stricter), with an explicit `maxAge`. Twenty-four hours is a worked example.
- Do not save uninitialized sessions or resave unchanged ones.

```ts
import { hash, compare } from 'bcrypt';
const stored = await hash(plaintext, 12);
const ok = await compare(candidate, stored);

app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: { httpOnly: true, secure: true, sameSite: 'lax', maxAge: 24 * 60 * 60 * 1000 },
}));
```

See AUTH-02 for regeneration on login and privilege change, and AUTH-08 for token storage.

---

### HARD-02: Output encoding against cross-site scripting

**Risk.** User-provided text written into the DOM as HTML runs as script in every other user's browser.

**Fix.**
- Rely on the framework's automatic escaping. In React, `{userInput}` inside JSX is encoded; `dangerouslySetInnerHTML` is not.
- Never assign user data to `innerHTML` and never pass it to `eval`.
- If rendering HTML is genuinely required, sanitize it first with a maintained library such as DOMPurify, then render the sanitized result.
- Encode for the context the data lands in. HTML, attribute, URL, and JavaScript contexts each need their own encoding (see API-05).

---

### HARD-03: Security header baseline

**Fix.**
- Use a header middleware such as `helmet` for Express, or the framework equivalent, so the baseline is applied to every response: strict transport security, content type sniffing protection, referrer policy, and framing restrictions.
- Add a Content Security Policy with `default-src 'self'`, `script-src 'self'`, and explicitly listed exceptions. `style-src 'unsafe-inline'` is a common starting concession; tighten it when possible. Allow `data:` and `https:` for images only if the app needs them. Keep `connect-src` to the origins the client actually calls.
- Restrict CORS to a configured list of known origins (see WEB-01).

```ts
import helmet from 'helmet';
app.use(helmet());
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"],
    styleSrc: ["'self'", "'unsafe-inline'"],
    imgSrc: ["'self'", 'data:', 'https:'],
    connectSrc: ["'self'"],
  },
}));
```

**Caveat.** Never disable a security header for convenience. If a header breaks a feature, scope the exception to the route rather than removing the header.

---

### HARD-04: Sensitive fields leave through API responses

**Risk.** A handler returns the whole user record. Password hashes, reset tokens, and internal flags ship to the client.

**Fix.**
- Map records to an explicit public shape before returning them. Strip `passwordHash`, reset tokens, and any internal field; prefer listing the fields that may leave over listing the ones that may not.
- Read secrets from environment variables and fail at startup when a required one is missing, so a misconfigured deploy does not run half-configured.

```ts
function toPublicUser(user: UserRecord): PublicUser {
  const { passwordHash, resetToken, ...publicFields } = user;
  return publicFields;
}
const STRIPE_KEY = process.env.STRIPE_API_KEY;
if (!STRIPE_KEY) throw new Error('STRIPE_API_KEY not configured');
```

See WEB-03 for the same failure through Server Component payloads.

---

### HARD-05: Server-side fetch of user-influenced URLs

**Risk.** Any time the server fetches a URL the user influenced (a webhook target, "import from URL", an image proxy, a link preview, an agent tool), the request can be aimed at internal services: `localhost`, private ranges, and the cloud metadata address `169.254.169.254`, which is the most common SSRF target.

**Fix.**
- Allow only `https:` and only an allow list of hostnames.
- Resolve every DNS record for the host and reject the request if any resolved address is outside public unicast space. That single check covers loopback, link-local, private, and unique-local ranges across IPv4 and IPv6.
- Disable redirects on the fetch, or revalidate every hop.

```ts
import { lookup } from 'node:dns/promises';
import ipaddr from 'ipaddr.js';

const ALLOWED_HOSTS = new Set(['hooks.example.com']);

async function safeUrl(raw: string): Promise<URL> {
  const url = new URL(raw);
  if (url.protocol !== 'https:') throw new Error('https only');
  if (!ALLOWED_HOSTS.has(url.hostname)) throw new Error('host not allowed');
  const records = await lookup(url.hostname, { all: true });
  if (records.some(r => ipaddr.parse(r.address).range() !== 'unicast')) {
    throw new Error('non-public address');
  }
  return url;
}
await fetch(await safeUrl(input), { redirect: 'error' });
```

**Caveat.** This check has a time-of-check to time-of-use gap: the HTTP client resolves DNS again when it connects, so a short-TTL record can rebind to an internal address between validation and connection. For high-risk surfaces, resolve once and connect to the pinned address, or put a filtering agent in front of the client. AI-03 covers the agent-specific version of this control.

---

### HARD-06: Schema validation at the boundary

**Fix.** Validate request bodies at the route handler with a schema, return a structured 422 with field-level details on failure, and pass only the typed, validated result into the service layer. The schema defines allowed fields, lengths, enumerations, and formats.

```ts
import { z } from 'zod';
const CreateTask = z.object({
  title: z.string().min(1).max(200).trim(),
  description: z.string().max(2000).optional(),
  priority: z.enum(['low', 'medium', 'high']).default('medium'),
  dueDate: z.string().datetime().optional(),
});
app.post('/api/tasks', async (req, res) => {
  const parsed = CreateTask.safeParse(req.body);
  if (!parsed.success) {
    return res.status(422).json({ error: { code: 'VALIDATION_ERROR', details: parsed.error.flatten() } });
  }
  return res.status(201).json(await tasks.create(parsed.data));
});
```

See API-01 for the rule and the direct-to-endpoint test.

---

### HARD-07: File upload safety

**Fix.**
- Allow only an explicit list of content types, for example `image/jpeg`, `image/png`, `image/webp`.
- Cap the size. Five megabytes is a worked example.
- Do not trust the file extension or the client-declared type for anything that matters; check the file's magic bytes.
- Store uploads outside the web root, under generated names, and serve them through a controlled path.
- If uploads go to a cloud storage bucket, configure the bucket per NET-13: private by default, pre-signed URLs for private objects, separate buckets for public assets, access logging on.

---

### HARD-08: Destructive operations on derived paths

**Risk.** A delete, move, or overwrite is only as safe as the value that names its target. A path that arrived from a job payload, a config value, a sibling service, or another process's command line is as attacker-controlled as a form field; where it came from proves delivery, not authorship. A shape check ("absolute path, at least one directory deep") proves the string is well-formed and gets mistaken for authorization. That is how a cleanup routine deletes a root instead of a leaf.

**Fix.** Before any destructive call, require all three:
- The resolved target sits under an allow-listed root. Compare after resolving symlinks, never on the raw string.
- The target is at least one level below that root, so a root is never itself the target.
- The target carries evidence that it belongs to this process or tenant, read before the operation and before any teardown that would remove the evidence. Otherwise "absent" and "not mine" look the same.
- On refusal, log the rejected target and stop. A cleanup that falls back to a broader default path is exactly the failure this guards against.

**Caveat.** Two limits. A marker file inside the tree is self-attestation, since anything that can write there can write the marker; the expected owner has to come from authenticated state, and the marker needs integrity protection (restrictive ownership or a MAC) before it counts. And resolving a path then operating on the name is a check/use race wherever an untrusted process can swap an ancestor; on shared volumes hold the target by descriptor, use no-follow and beneath-root operations, or ensure the hierarchy cannot change for the duration.

---

### HARD-09: Triaging dependency audit findings

**Risk.** Package-manager audits report known advisories. They do not prove a package is trustworthy or that the vulnerable code is reachable, and treating every finding as equal either blocks all work or trains the team to ignore the output.

**Fix.** Triage by severity and reachability:
- Critical or high, and the vulnerable code is reachable in runtime, build, test, or deployment paths: fix immediately by updating, patching, or replacing.
- Critical or high, confirmed unreachable across those paths: fix soon, not a blocker.
- No fix available: look for a workaround, consider replacing the dependency, or allow-list it with a documented reason and a review date.
- Moderate: fix in the next release if reachable in production; backlog if dev-only.
- Low: fix during regular dependency updates.

Ask for each finding: is the vulnerable function actually called, is the package runtime or dev-only, and is the issue exploitable in this deployment context (a server-side flaw in a client-only app is not). When a fix is deferred, write down why and when it will be revisited.

See SUP-01 and SUP-02 for the malicious-package and inventory side.

---

### HARD-10: Supply-chain hygiene for installs

**Fix.**
- Find the real installation boundary and manager before doing anything. Use the workspace root that owns the lockfile, or an independent nested project only when it sits outside that workspace. Corroborate the declared package manager, the lockfile, and CI; stop if they disagree or if competing lockfiles exist. Pin the manager version.
- Block dependency install scripts before the first install. Bootstrap with scripts disabled or a documented fail-closed policy, inspect the pending script source, approve only the minimum packages that need scripts, commit the policy, and verify with a clean frozen or immutable install. Never blanket-approve scripts.
- Never apply forced audit remediation automatically. Preview it, read the changelogs, and test each resulting upgrade; forced fixes can cross declared version ranges.
- Verify registry signatures and provenance where the manager supports it. Treat absence as a reason to look closer, not as proof of compromise.
- Review new dependencies, lockfile diffs, and script-policy changes together: ownership, maintenance, release age, provenance, the transitive graph, and typosquats such as `crossenv` for `cross-env`.
- In CI, use the manager's frozen or immutable install against the committed lockfile and run the native audit before every release.

**Caveat.** An audit that passes does not make a newly malicious or typosquatted package safe. The review above is what catches those.

---

### HARD-11: Rate limiters that only count in one process

**Risk.** A rate-limit middleware keeps its counters in process memory by default. Behind a load balancer each instance keeps its own count, so the effective limit is the configured limit multiplied by the instance count. On serverless or edge runtimes every cold invocation starts from zero, so the limit on the login endpoint may never fire at all.

**Fix.**
- Apply a general API limit and a stricter limit on authentication endpoints. One hundred requests per fifteen minutes for the API and ten attempts per fifteen minutes for auth are worked examples.
- Once more than one process serves traffic, back the limiter with a shared store such as Redis, or use an HTTP-based limiter that works where a long-lived connection does not.

```ts
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';
const authLimiter = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '15 m'),
});
const { success } = await authLimiter.limit(`login:${req.ip}`);
if (!success) return res.status(429).end();
```

See NET-04 and NET-05 for where limits belong and how to size them.

---

### HARD-12: Secrets in the repository

**Fix.**
- Commit `.env.example` with placeholder values only. Keep `.env`, `.env.local`, and `.env.*.local` out of version control, along with `*.pem` and `*.key`.
- Before committing, scan the staged diff for secret-shaped content, for example `git diff --cached | grep -i "password\|secret\|api_key\|token"`, and use a pre-commit secret scanner where available.
- If a secret is ever committed, treat it as compromised the moment it reaches a remote. Revoke and reissue first, then purge it from history. Deleting the line or rewriting history alone is not a fix.

See SUP-03 for history scanning and rotation policy.

---

### HARD-13: LLM features inherit a new attack surface

**Risk.** A chatbot, summarizer, agent, or retrieval pipeline adds attack paths that ordinary web controls do not cover. The OWASP Top 10 for LLM Applications (genai.owasp.org) names them; the controls below map to it.

**Fix.**
- Treat all model output as untrusted input. Never pass it straight into `eval`, SQL, a shell, `innerHTML`, or a file path. Parse it defensively, validate it against a schema, then encode it for its destination.
- Assume prompts can be hijacked. Any untrusted text in the context window, including a user message, a fetched page, or a document, can carry instructions. The system prompt is not a security boundary; enforce permissions in code (AI-02).
- Keep secrets and other users' data out of prompts. Anything in the context can be echoed back. Do not place API keys, cross-tenant data, or the full system prompt where the model can repeat them.
- Constrain tool and agent permissions. Scope tools to the minimum, require confirmation for destructive or irreversible actions, and validate every tool argument (AI-01, AI-06).
- Bound consumption. Cap tokens, request rate, and loop or recursion depth so a crafted input cannot run up cost or hang the system (AI-11).
- Isolate retrieval data. Partition embeddings per tenant and validate documents before indexing so poisoned content cannot steer answers (AI-04).

```ts
// Model output is data: parse, validate, then act through an allow list.
let intent;
try {
  intent = CommandSchema.parse(JSON.parse(await llm.replyJson(userMessage)));
} catch {
  throw new ValidationError('unexpected model output');
}
await runAllowlistedAction(intent.action, intent.params);
container.textContent = await llm.reply(userMessage); // never innerHTML
```

See AI-05 for the validate, retry, and fall back pattern on the user-facing side.
