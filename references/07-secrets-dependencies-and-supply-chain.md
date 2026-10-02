# Secrets, Dependencies, and Supply Chain

Malicious and risky packages, dependency inventory and audit, lockfile integrity, secrets in git history, credential rotation, AI instruction files as supply-chain inputs, and vendor lock-in.

Entry format: **Risk** is what goes wrong. **Fix** is the change to make. **Caveat** records where a common version of the advice overstates its case. **Check** marks an item where the risk is clear but the remedy depends on the application. **Test** names a concrete way to prove the fix.

---

### SUP-01: A package installed last week is exfiltrating your environment

**Risk.** A malicious or compromised npm package, direct or transitive, reads environment variables on install or at runtime and sends API keys and credentials to an attacker. The dependency list is a major attack surface, and the code is only as safe as its weakest package.

**Fix.**
- Run a full dependency audit across all direct and transitive packages, looking for suspicious behavior.
- Flag packages with fewer than 100 downloads, recent maintainer changes, or post-install scripts.
- Restrict environment variable access so each module receives only the secrets it needs.
- Verify lockfile integrity in CI and block the build if checksums do not match.
- Never trust dependencies blindly.

**Caveat.** Modules running in the same process normally share environment access. Real isolation requires an execution boundary such as a separate process or service, not just discipline about which module reads which variable.

---

### SUP-02: Hundreds of dependencies nobody has reviewed

**Risk.** First-party code is small, but the app pulls in hundreds of packages, some deprecated, unmaintained, or carrying known vulnerabilities.

**Fix.**
- Inventory every dependency and review it, even when first-party code is small.
- Run `npm audit` (or the package manager's equivalent), lock dependency versions, and review on a regular schedule. Monthly is a workable starting cadence.
- Review deprecated and unmaintained packages for replacement.

**Caveat.** Advisory-based scans are incomplete, and a package's age alone does not prove it is exploitable. The scan is the start of the review, not the end.

---

### SUP-03: A deleted API key is still in git history

**Risk.** Committing a key to the repository leaves it in history even after the file is edited. Bots scan public repositories for keys continuously. Exposed payment keys have caused real financial loss.

**Fix.**
- Scan git history for leaked secrets, not just the current tree.
- Revoke and rotate any exposed credential. Rewriting history does not un-expose a key that was already public.
- Treat environment variables and `.env` files as configuration, not as secrets management. Use a secrets manager for encryption, access control, and audit trails.
- Rotate keys on a schedule, before a breach rather than after.
- Plan for the possibility of repository exposure as part of the credential lifecycle instead of assuming a key stays private indefinitely.

**Caveat.** Reports that most audited repositories contain committed keys are anecdotal. Inspect the actual repository rather than assuming either way.

---

### SUP-04: Prompts, instruction files, and skills are supply-chain inputs

**Risk.** The supply chain is no longer only packages. AI instruction files, skill files, and prompts shape what an assistant does with code and credentials. Stale or conflicting instructions degrade behavior; a malicious instruction injected through content the assistant reads, such as a pull-request description, has produced remote code execution in an AI coding assistant.

**Fix.**
- Treat prompts, instruction files, and skills as dependencies: inventory them, review them for stale or conflicting instructions, and assign trust levels to their sources.
- Rebuild AI instructions periodically from current evidence.
- Keep the assistant's exposure to untrusted content bounded, and keep human review on actions that touch production.

**Caveat.** Do not delete a security policy or instruction merely because it is old. Update from evidence; calendar-based removal can strip valid controls and institutional knowledge. The specific CVE and affected version of the cited assistant vulnerability must be verified for the tools in use.

---

### SUP-05: A vendor can retire the product you built on

**Risk.** A cloud provider renames, closes to new customers, or retires a managed AI or agent service, and the integration is written directly against its SDK.

**Fix.** Abstract vendor SDKs behind your own interfaces, keep a migration plan for critical services, and reassess agreements when a provider changes direction.

**Caveat.** Any specific claim that a named product was retired should be verified against the provider's current announcements before acting on it.
