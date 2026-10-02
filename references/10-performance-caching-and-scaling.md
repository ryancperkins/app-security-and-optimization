# Performance, Caching, and Scaling

Indexes and query shape, connection pooling, the scaling decision tree, cache policy and staleness, replication and consistency, background jobs for slow work, cascading failures, hosting limits, database selection and migration timing, cost attribution, and multi-region.

Entry format: **Risk** is what goes wrong. **Fix** is the change to make. **Caveat** records where a common version of the advice overstates its case. **Check** marks an item where the risk is clear but the remedy depends on the application. **Test** names a concrete way to prove the fix.

---

### PERF-01: Every request runs a full table scan

**Risk.** The database is slow and expensive because queries filter on unindexed columns and return every column of every row.

**Fix.**
- Add indexes on the columns used for filtering.
- Limit queries to the fields actually needed.
- Enable slow-query logging.
- Build a dashboard that tracks slow and expensive queries.

**Caveat.** Missing indexes do not mean every query scans the whole table, and adding indexes does not resolve every bottleneck. Measure query performance before and after rather than assuming.

---

### PERF-02: Fifty users exhaust the database connections

**Risk.** Login works for one user. A burst of concurrent sign-ups or requests exhausts database or API connections, requests stall, and the app appears broken at tiny scale.

**Fix.** Use appropriate database connection pooling, then test actual concurrency against actual limits.

**Caveat.** User counts in warnings like this are illustrative. Find the real limit by load testing.

---

### PERF-03: The scaling decision tree

**Fix.** Investigate in order: connection constraints first, then query efficiency, then replication needs. Scaling blindly skips the cheap fixes.

---

### PERF-04: Nobody decides to build a caching strategy, so one accretes

**Risk.** A cache gets added because the app got faster, and now there are two versions of the truth. Prices, permissions, and anything else that changes become stale without anyone deciding that was acceptable. The same question is answered a thousand times a day with no policy for when the answer changes.

**Fix.**
- Define cache freshness and invalidation policy explicitly, especially for prices and permissions.
- Cache selectively with explicit staleness tradeoffs. Not everything should be cached.
- Treat a new cache as a new consistency obligation.
- Scope cache keys to tenant and identity (see the access-control catalog).

---

### PERF-05: Two versions of every record

**Risk.** Read replicas lag, a user reads right after writing and sees stale data, and concurrent writes conflict. Downstream consumers fall out of sync with the primary.

**Fix.**
- Handle read-after-write routing, replica lag, and concurrent-write conflicts deliberately. Consistency affects correctness and permission decisions.
- For downstream consistency, use change-data capture or event routing with handling for failed deliveries. Access scope and retry safety still apply.

---

### PERF-06: A twelve-second checkout because everything runs inline

**Risk.** A request does all its work synchronously, including slow steps, and the platform times out at 10 seconds while the work needs 12. Retried submissions do the work twice.

**Fix.**
- Move suitable slow work to a monitored queue and return a response once the job is durably accepted.
- Make submission retry-safe so a repeated request does not duplicate work.
- Check the actual runtime timeouts and upload limits of the platform against the workload before relying on them.

**Caveat.** Specific platform timeout numbers vary by provider and plan; verify them in current documentation.

---

### PERF-07: One failed dependency takes down everything

**Risk.** A single webhook or downstream call hangs, and it takes authentication, the dashboard, and checkout with it because they share connections and have no timeouts.

**Fix.** Apply timeouts to every dependency call, add circuit breakers, and isolate connection pools so one failing dependency cannot starve the others.

---

### PERF-08: Hosting limits and cost cliffs

**Risk.** Serverless was predictable at ten users and the bill climbs steeply at a thousand. Hobby plans cap concurrent executions. The app hits the platform's limits.

**Fix.**
- Compare serverless and container cost and operational tradeoffs against the actual workload.
- Compare managed-service spending against self-hosting effort honestly; neither is free.
- When platform limits are reached, evaluate alternative hosting. Migration is conditional on the evidence, not mandatory.

**Caveat.** Specific plan limits must come from current provider documentation, never from memory, before they drive architecture.

---

### PERF-09: Database selection and migration timing

**Fix.**
- Choose the database by read/write ratio, migration strategy, and ecosystem dependency.
- Store images and other binary assets in object storage rather than database columns.
- When a single database approaches its limits (tens of millions of rows, wide tables, cross-region latency), plan partitioning or migration from measurements, not from the row count alone.
- Decoupled database hosting with branching is an option; review production-data access and branch isolation before using it.
- Evaluate when bundled auth, database, and storage should be separated. There is no mandatory unbundling point.

**Caveat.** Database blobs are not categorically a defect, a wide table is not a defect by itself, and product comparisons without measurable criteria do not decide anything.

---

### PERF-10: Nobody can see where the money goes

**Fix.**
- Attribute infrastructure cost to services and features.
- Track cost per user and per feature so growth stages can be planned: features, then reliability, then architecture.
- Document infrastructure limits and customer requirements before making enterprise commitments; measure specific capacity ceilings.

**Caveat.** Cost figures and user thresholds in examples are illustrative.

---

### PERF-11: Fast in one region, slow everywhere else

**Fix.** Consider multi-region deployment for latency.

**Caveat.** Data residency, consistency, and cost need separate assessment before going multi-region.

---

### PERF-12: Concurrent editing

**Check.** Two users editing the same document must not lose each other's work. The concurrency technique (locking, operational transforms, CRDTs) has to be chosen for the product.
