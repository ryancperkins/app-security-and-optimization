# Delivery, Environments, Backups, and Rollback

Branch protection and pull-request gates, separated environments, staged rollouts, migrations with rollback, backups with tested restores, CI integrity, testing beyond the happy path, threat-model-first review, and the production-readiness taxonomy.

Entry format: **Risk** is what goes wrong. **Fix** is the change to make. **Caveat** records where a common version of the advice overstates its case. **Check** marks an item where the risk is clear but the remedy depends on the application. **Test** names a concrete way to prove the fix.

---

### OPS-01: Forty-seven files pushed straight to production in one commit

**Risk.** An AI assistant pushes a large change directly to the production branch with no review and no tests. Payments break and revenue stops until someone notices.

**Fix.**
- Never push directly to the production branch. Every change goes through a pull request.
- Enable branch protection so all changes require a pull request and an approval.
- Run automated checks on every pull request: tests, linting, and security scanning. Allow a merge only when all checks pass.
- Make small, clear commits that can be traced and rolled back quickly. One change per pull request.
- Tell the AI assistant to set these safeguards up, and verify that it did.

**Caveat.** Whether the main branch directly represents production depends on the release model; protect whichever branch deploys.

---

### OPS-02: Development, staging, and production are all the same place

**Risk.** Every push goes straight to production, or the laptop serves as every environment. Testing breaks live customers.

**Fix.** Separate development and testing from production. Gate production releases on tests passing in a staging environment, and verify rollback works before it is needed.

---

### OPS-03: Ship to a small slice first

**Risk.** A deploy that works in the demo breaks for real users, and the whole user base gets it at once. Deployment results are inconsistent from one release to the next.

**Fix.**
- Use preview deployments for every change.
- Release as a canary to a small group first, observe, then widen. Five percent with a fifteen-minute observation window is a worked example.
- Use feature flags to decouple deploy from release.
- Keep changes small and rehearse rollback so it is fast when needed.

**Caveat.** Choose the canary size and observation window from risk and traffic. Rollback time targets are contextual. Deployment quality comes from the process, not the day of the week.

---

### OPS-04: The database schema never planned for change

**Risk.** A schema change cannot be undone, and it was applied straight to production.

**Fix.** Use additive migrations, design the rollback before the rollout, and test every migration in staging first.

---

### OPS-05: There is a backup, and it has never been restored

**Risk.** The database loses hours of customer data, or the backup exists but nobody has ever restored from it. Backup existence is not recovery evidence.

**Fix.**
- Decide backup cadence, keep backups in a separate storage and failure domain from the database, and set retention.
- Use point-in-time recovery where the data warrants it.
- Define explicit recovery-time and data-loss objectives.
- Write and rehearse the restore runbook. A restore that has not been tested is not a backup.
- Treat backups and outage recovery as part of application delivery, not an afterthought.

**Test.** Restore the latest backup into a scratch environment and confirm the application runs against it.

---

### OPS-06: CI that quietly stops testing

**Risk.** The CI minute budget runs out, tests stop running, and merges continue. Pipelines that take forty-five minutes get skipped.

**Fix.** Keep the test gates intact regardless of CI quota; add usage alerts. Self-hosted runners are an option for cost, provided job isolation stays in place. Treat slow pipelines as something to optimize, not bypass.

**Caveat.** Provider limits, costs, and "unlimited" claims should be verified rather than assumed.

---

### OPS-07: The app only knows the happy path

**Risk.** A feature generated in twenty minutes works in the demo and fails for the first unfamiliar user. The users become the QA team.

**Fix.**
- Write feature tests, and separate unit from integration checks.
- Deliberately exercise failure paths, not only successful operation.
- Test with unfamiliar users before treating a demo as a product.
- Add clear error, loading, and recovery behavior.
- Review production error handling, security headers, endpoint validation, and rate limits instead of trusting defaults.

**Caveat.** A coverage threshold such as 60% is a working target, not a security-assurance level.

---

### OPS-08: Map the whole application before chasing fixes

**Fix.**
- Before fixing isolated bugs, map the whole application and its gaps. This is the start of a threat model.
- Repair broken existing behavior before adding new features.
- For generated code, make sure someone understands what was built before extending it.

---

### OPS-09: Production readiness is more than frontend and backend

**Check.** A full production stack also covers identity, hosting, infrastructure limits, delivery and CI/CD, security, rate limits, caching, load balancing, error tracking, and outage recovery. Walk this list before launch and confirm each layer has an owner and a decision. It is a taxonomy, not a standard; the domain files in this catalog hold the specifics. A short pre-deploy checklist at minimum covers secrets, fallback behavior, validation, and rollback. Tool selection alone, however good the tools, does not establish readiness.

---

### OPS-10: Customization and documentation debt

**Fix.**
- Manage customer-specific requests with a cost model, configuration over code, and a threshold at which a customization becomes a product feature.
- Make documentation survive the person who wrote the feature.

**Caveat.** These are maintainability practices, not security controls on their own.
