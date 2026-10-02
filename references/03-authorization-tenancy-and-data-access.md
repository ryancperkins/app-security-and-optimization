# Authorization, Tenancy, and Data Access

Object ownership checks, mass assignment, admin surfaces, row-level security and service-role bypass, tenant isolation, tenant-scoped caches, role-based access across layers, and direct browser-to-database access.

Entry format: **Risk** is what goes wrong. **Fix** is the change to make. **Caveat** records where a common version of the advice overstates its case. **Check** marks an item where the risk is clear but the remedy depends on the application. **Test** names a concrete way to prove the fix.

---

### ACC-01: The API trusts the ID in the URL instead of the session

**Risk.** An endpoint loads a record by the user or resource ID supplied in the URL, query string, or request body without checking that the authenticated user owns it. Any user can read or change any other user's data by changing the ID. Sequential integer IDs make the enumeration trivial.

**Fix.**
- On every endpoint that returns or mutates a user-specific resource, check that the requesting user owns the resource.
- Add middleware that compares the authenticated session principal against the actual resource owner on the server.
- Return a denial, such as `403`, when a user asks for another user's resource.
- Audit every endpoint that accepts a resource identifier, wherever the identifier appears, and verify an ownership check is present. Flag any endpoint where users can reach others' resources.
- Replace sequential IDs with UUIDs to make enumeration harder.
- The API trusts the user session, never the URL.

**Caveat.** UUIDs reduce enumeration. They do not enforce authorization. The ownership check is the control.

---

### ACC-02: Mass assignment lets users set fields the form never sends

**Risk.** An update handler writes the entire request body to the database. A user adds a field such as `is_admin`, `role`, `plan`, `permissions`, or account status to the request and the server saves it. Automated code review can pass while this remains.

**Fix.**
- Validate every write endpoint against an explicit list of allowed fields, such as name, email, and avatar. Reject or strip everything outside the list.
- Keep profile updates separate from role changes. Create dedicated admin endpoints with elevated authorization for privileged fields, and remove admin-level fields from user-facing handlers entirely.

**Test.** For every update endpoint, submit requests containing prohibited fields and verify the server rejects them or strips them completely.

---

### ACC-03: The admin dashboard has no authentication

**Risk.** Admin routes sit at common paths such as `/admin` or `/dashboard` with no authentication, no role check, and no audit log. Anyone who finds the URL can view or change sensitive user and order data.

**Fix.**
- Add strong authentication to every admin route and allow only users with an admin role.
- Add rate limiting on admin login attempts.
- Record audit logs for every admin action and every admin login.
- Move admin routes off guessable paths.

**Caveat.** A hidden or unpredictable admin path is not an access-control boundary. Authentication and authorization are the controls; the path change only reduces noise.

---

### ACC-04: A service-role key bypasses every row-level security policy

**Risk.** Row-level security policies are written and tested, but API routes connect to the database with a service-role key. The service role ignores RLS, so every one of those routes exposes all data. Attackers look for exactly these routes. This pattern shows up in a large share of reviewed applications.

**Fix.**
- Audit every path that reaches the database with elevated credentials.
- Use the user-scoped connection wherever a request acts on behalf of a user, so RLS applies.
- Treat security as protecting every path to the data, not as writing policies.

---

### ACC-05: Row-level security enforces tenancy where application code cannot

**Risk.** Filtering by tenant in application code is a hope that no query anywhere forgets the filter. One missed filter leaks another tenant's data.

**Fix.**
- Enable row-level security so the database itself filters every query through a policy that matches tenant IDs to the authenticated user.
- In Supabase, RLS is enabled per table and applies to every query automatically.
- Use RLS wherever users must never see each other's information. Protect data at the source.

**Caveat.** RLS is defense in depth. Correct application-level authorization is still required, and RLS has role and policy caveats of its own, including the service-role bypass in ACC-04.

---

### ACC-06: Tenant isolation is a launch requirement

**Risk.** Customer A logs in and sees customer B's data. This is a breach with notification and contractual consequences, not a bug to schedule.

**Fix.**
- Treat tenant isolation as a condition of launch and verify it with cross-tenant tests before any customer data is loaded.
- Choose the isolation architecture (shared schema with RLS, schema per tenant, database per tenant) from business requirements and risk, not only from technical convenience.

**Caveat.** Claims of zero-leak multi-tenant platforms built quickly cannot be taken on faith. The architecture has to be inspected.

---

### ACC-07: A cache that ignores tenancy serves one customer's data to another

**Risk.** A customer opens their dashboard and sees someone else's revenue because the cache key did not include the tenant or user.

**Fix.** Scope every cache key and cached result to tenant and identity so caching can never bypass database access controls.

---

### ACC-08: Role-based access must hold at every layer

**Risk.** The frontend hides the admin button but the API still accepts the admin request. Authentication confirms who the user is; it says nothing about which objects they may touch.

**Fix.**
- Enforce access control in the API and data layer. Hiding a control in the UI is not authorization.
- Apply role-based authorization consistently across every layer of the system rather than treating it as a UI feature.
- Separate authentication from object-level authorization in the design.

**Test.** Log in as one user and request another user's objects directly through the API.

---

### ACC-09: The frontend is a display layer, not a trust layer

**Fix.** Keep trust decisions, authoritative validation, business rules, and secrets on trusted server boundaries. Anything in the browser can be changed by the user.

---

### ACC-10: The browser talks to the database directly

**Risk.** A frontend that queries the database directly exposes whatever the database credentials permit. Publicly accessible tables are a common consequence.

**Fix.** Inspect the actual table permissions and policies rather than assuming them.

**Caveat.** A deliberately designed public API with safe, scoped credentials, RLS, and authorization can be legitimate. The failure is unexamined access, not direct access as such.

---

### ACC-11: Per-customer customization without forking

**Risk.** A column added for one client slows or complicates every other tenant. A developer forks the whole codebase for each new client and the forks drift.

**Fix.**
- Prefer tenant-aware configuration and feature flags over per-customer forks. Feature flags still require server-side authorization checks.
- Where tenants need real extensions, use tenant-specific extension points, compute isolation, and proper migrations.

**Caveat.** Claimed slowdowns and mandatory architecture choices must be supported by workload evidence.
