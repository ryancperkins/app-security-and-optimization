# Errors, Logging, and Monitoring

Safe production error responses, structured logging, silent-failure detection, business-level monitoring, outage detection, incident process, status communication, and session replay.

Entry format: **Risk** is what goes wrong. **Fix** is the change to make. **Caveat** records where a common version of the advice overstates its case. **Check** marks an item where the risk is clear but the remedy depends on the application. **Test** names a concrete way to prove the fix.

---

### OBS-01: Error pages show the database name, file paths, and queries

**Risk.** An unhandled error in production renders the stack trace, database name, server file paths, the failing query, or even credentials to the user.

**Fix.**
- Separate error handling by environment: detailed diagnostics in development, generic messages in production.
- Catch errors at boundaries. Route every error to centralized logging with full context, restricted to the team, while the user sees only a clean message.
- Build branded custom error pages for common status codes that give simple guidance and never reveal server details.

---

### OBS-02: Logs say everything and tell you nothing

**Risk.** When the app breaks, the logs are either absent or an unstructured wall of text that cannot answer what failed, for whom, or when. Silent failures, timeouts, and slow endpoints leave no trace.

**Fix.**
- Emit structured logs with consistent fields so failures can be queried.
- Make sure operational logs exist for silent failures, timeouts, and slow endpoints, and that they are useful during an incident, not only after one.

---

### OBS-03: The error tracker only catches thrown errors

**Risk.** Revenue drops by a large fraction while every HTTP response is 200. Payment failures, stalled queues, and wrong results do not throw, so the error tracker and the infrastructure status page both stay green while customers screenshot broken screens.

**Fix.**
- Monitor business metrics such as orders, payments, and sign-ups, not only exceptions.
- Run synthetic transactions through critical flows.
- Watch dead-letter queues.
- Classify support tickets by root cause so user-observed failures feed back into monitoring.
- Define endpoint error budgets and burn-rate alerts, and express incidents in business impact.

---

### OBS-04: Users notice the outage before you do

**Risk.** The app crashes at 2 AM and nobody knows until the morning's complaints.

**Fix.**
- Use external health checks, correlated traces, defined service-level objectives, and alerts that are actionable.
- Combine error tracking, user analytics, and independent uptime monitoring. Error tracking is one layer of the production stack, not the whole of it.

**Caveat.** Specific vendors, free tiers, and setup-time estimates are not guarantees. Choose tools for the stack and confirm they are actually configured.

---

### OBS-05: The first production incident is coming

**Fix.**
- Keep a status page hosted separately from the app, with maintenance notices and an incident communications workflow, so an outage does not also look like theft.
- Write a postmortem for every incident, keep an incident library, and review on a regular cadence.

**Caveat.** A fixed postmortem deadline such as 48 hours is a proposed practice, not a rule.

---

### OBS-06: Three gaps that recur in quickly built apps

**Fix.** Close the three most common operational gaps: error handling, separated environments, and audit trails for sensitive actions.

**Caveat.** Claims about how many apps share these gaps are unverified. The gaps are worth checking regardless.

---

### OBS-07: Session replay for bug reports needs privacy controls

**Fix.** Link session replay to error events and interaction signals so a bug report comes with context.

**Caveat.** Add data minimization, input masking, access controls, retention limits, and appropriate consent before recording sessions.
