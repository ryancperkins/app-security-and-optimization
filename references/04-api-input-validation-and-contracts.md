# API Input Validation, Injection, and Contracts

Server-side schema validation, raw SQL built from user input, template injection, GraphQL introspection, context-specific encoding, and API versioning.

Entry format: **Risk** is what goes wrong. **Fix** is the change to make. **Caveat** records where a common version of the advice overstates its case. **Check** marks an item where the risk is clear but the remedy depends on the application. **Test** names a concrete way to prove the fix.

---

### API-01: The form validates; the API behind it does not

**Risk.** Client-side validation is a user-experience feature. An attacker skips the form and sends raw data straight to the endpoint. If the server does not validate, every rule the form enforced is gone.

**Fix.**
- Validate all incoming data on the server with a schema. Zod is one implementation option.
- Use the same schema on client and server so the rules cannot drift.
- Accept only the fields the schema defines. Reject requests that contain invalid or unknown fields.
- Run validation everywhere JavaScript runs, and especially on the backend.

**Test.** Call every API endpoint directly, without the form, with invalid and extra fields, and confirm the server rejects them.

---

### API-02: Raw SQL built from user input bypasses the ORM

**Risk.** A search field feeds user input directly into a raw SQL string. The ORM's protections do not apply to its escape hatches, and the query exposes data the user should never see.

**Fix.**
- Use the ORM's parameterized methods for every query that includes user input.
- Validate every input for type and length before it reaches the query. Reject anything unexpected at validation rather than trying to sanitize it.
- Audit the codebase for every raw query built from user input. Each one is a risk.

**Caveat.** Raw SQL is not inherently unsafe when it is parameterized correctly. The defect is string-building with user input.

---

### API-03: One unsanitized template field turns email into a phishing platform

**Risk.** A user enters HTML into a form field that is later rendered into an email template. The system sends the attacker's content from your domain with valid SPF, DKIM, and DMARC, so it is indistinguishable from a legitimate message.

**Fix.**
- Sanitize all user input before it enters a template: strip HTML and escape special characters.
- Never render user input as raw HTML in a template. Render it as plain text or safely encoded content.
- Fix any injectable template immediately, because a compromised template destroys the domain's sending reputation.

**Test.** Submit angle brackets and script tags through every field that reaches a template and confirm they arrive as literal text.

**Caveat.** Valid email authentication proves the sender, not the safety of the content.

---

### API-04: GraphQL introspection hands over the whole schema

**Risk.** With introspection enabled in production, anyone can download the entire API schema. Detailed errors and field suggestions leak the same information piece by piece.

**Fix.**
- Disable introspection in production with a config flag.
- Enforce query depth and complexity limits.
- Turn off field suggestions and detailed error messages in production.
- Keep the schema and its documentation available to developers only.

**Caveat.** Schema secrecy is optional hardening. It does not replace resolver-level authorization or query cost limits.

---

### API-05: Universal sanitization is weaker than context-specific handling

**Risk.** An API that trusts every request, or one that runs every string through a single "sanitize" function, misses injection in whichever context the function did not anticipate.

**Fix.** Validate inputs on the server against what each field is allowed to be, encode output for the specific context (HTML, SQL parameter, shell, URL), parameterize database access, and monitor request patterns for abuse.

---

### API-06: Field renames and unsigned mutations break the contract

**Risk.** Renaming a field breaks every integration that depended on it. A forged request to a mutation endpoint succeeds because nothing bound the request to a trusted caller. The API is at once a security surface, a product surface, and a contract.

**Fix.**
- Maintain explicit API contracts and versioning so changes do not break integrations. Monitor usage of deprecated versions before removing them.
- Consider signing mutation requests where the architecture calls for it.

**Caveat.** Signing every request depends on the architecture and does not replace authorization and replay controls.

---

### API-07: An ORM is an abstraction, not a guarantee

**Caveat.** Product stereotypes such as "the ORM protects you from the database" or "direct SQL gives you control" are not security properties. Safety comes from parameterization and validation in whichever path is used, as in API-02.
