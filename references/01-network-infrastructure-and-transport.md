# Network, Infrastructure, and Transport

Service-to-service trust, edge protection and firewalls, rate limiting, infrastructure dependencies, self-managed servers, mobile transport, outbound email, dynamic scanning, and cloud storage buckets.

Entry format: **Risk** is what goes wrong. **Fix** is the change to make. **Caveat** records where a common version of the advice overstates its case. **Check** marks an item where the risk is clear but the remedy depends on the application. **Test** names a concrete way to prove the fix.

---

### NET-01: Flat internal trust lets one breached service reach everything

**Risk.** A vulnerability in one service, such as a marketing integration, gives an attacker access to every other service on the network, including the production database, because services trust each other simply for being on the same network.

**Fix.**
- Segment the network so each service can only talk to the services it needs. Marketing cannot reach the database. The admin API cannot reach marketing.
- Require mutual TLS or service tokens for every internal API call. No service trusts another just because the call originated inside the network.
- Apply least privilege to containers, service accounts, and network rules. Each service gets only the access and permissions it needs.
- Design so that one breached service cannot threaten the entire operation.

---

### NET-02: A reverse proxy is not a firewall

**Risk.** An NGINX reverse proxy routes traffic. It does not filter malicious requests. An attacker walks straight through it.

**Fix.**
- Install ModSecurity as a web application firewall on NGINX using the OWASP Core Rule Set, which blocks common attack patterns.
- Run ModSecurity in blocking mode on every route that accepts user input.
- Add Fail2Ban to watch the logs and automatically ban IPs that generate too many blocked requests. Make bans escalate for repeat offenders.
- Alert on spikes in ModSecurity blocks or Fail2Ban bans so an attack is noticed immediately.
- These tools are open source and cost nothing to run.

**Caveat.** A WAF needs tuning and ongoing operational attention, enforcement must be proxy-aware so bans land on the real client address rather than the proxy, and application-level fixes are still required. The WAF is a layer, not a substitute.

---

### NET-03: The origin can be reached around the edge

**Risk.** With a CDN or WAF in front of the app, clients can still bypass every edge control by sending requests directly to the server's IP address.

**Fix.** Restrict origin access so that only the CDN or WAF can reach the application server. Verify that a direct request to the origin IP is refused.

---

### NET-04: Login floods arrive and nothing stops them

**Risk.** Thousands of requests hit the login endpoint overnight and none of them are legitimate. The edge provider in front of the app does not enable authentication rate limits or bot controls by default.

**Fix.** Configure rate limits on authentication endpoints, bot controls, and WAF rules appropriate to the app. Do not assume the hosting or edge provider has turned them on.

---

### NET-05: Rate limiting is an availability and cost control

**Fix.** Apply request limits in three forms: hard limits that cap abusive traffic, adaptive limits that respond to load, and tier-aware limits that give paying or trusted tiers more headroom. The purpose is to protect availability and cost, not to say no to users.

**Caveat.** Thresholds depend on the product and its traffic. Choose them from measured usage.

---

### NET-06: Request floods and bot traffic

**Check.** A single actor with a laptop can take the product offline or run up the bill with a request flood. Put a firewall and rate limiting in front of the app, then size capacity and mitigation for the actual traffic profile. The specific mitigation design must be worked out for the application.

---

### NET-07: Infrastructure dependencies drift out from under you

**Risk.** A caching layer with a known vulnerability sits in production because the dependency was never pinned or reviewed. In the specific case of Redis, a license change led the community to fork it as Valkey, which now receives security patches and updates.

**Fix.**
- Evaluate whether Valkey is a better fit for the setup and migrate if so.
- Do not hard-code the cache client. Use an adapter so the cache implementation can be swapped without rewriting callers.
- Pin every infrastructure dependency to a specific version.
- Review and update infrastructure dependencies on a regular, controlled schedule.

**Caveat.** No specific CVE or version was cited to support a patch gap between Redis and Valkey. Evaluate the migration on its merits for your workload rather than as an emergency.

---

### NET-08: A self-managed server needs hardening

**Risk.** Moving to a VPS for more control also means owning its security.

**Fix.** Review SSH authentication (keys, not passwords), audit exposed ports, and keep security updates applied.

**Caveat.** Moving SSH to a non-default port is not the primary protection. It reduces noise, not risk.

---

### NET-09: System-to-system auth is not user auth

**Fix.** Model service-to-service identity separately from user authentication. Enumerate every trust boundary between services and decide explicitly how each side proves its identity.

---

### NET-10: Mobile clients leak in transit and at rest

**Risk.** A mobile app that sends API calls in plain text exposes user data to anyone on the same public Wi-Fi. If the app does not verify the server, an attacker can impersonate it and hijack API calls. Tokens and credentials stored in insecure local storage on the device are readable.

**Fix.**
- Use HTTPS for all API traffic.
- Add certificate pinning so a fake server cannot intercept requests.
- Encrypt sensitive data inside API payloads rather than relying on transport alone.
- Move authentication tokens into the platform's secure storage.
- Fix the transport layer first. It is the most urgent exposure.

**Caveat.** HTTPS already provides encryption in transit. Pinning is an optional additional trust restriction and carries rotation risk: a pinned certificate that rotates without an app update breaks the app. A web app wrapped as a mobile app is not automatically insecure, but it needs platform secure storage and deep-link validation. Mobile universal links need their own setup and security review.

---

### NET-11: Your transactional email looks like a scam

**Risk.** A customer pays and the confirmation lands in spam or looks forged because the sending domain is not authenticated.

**Fix.** Configure SPF and DKIM, send from a suitable dedicated sending domain, and monitor delivery. Add a DMARC policy where appropriate.

---

### NET-12: Dynamic scanning of your own application

**Fix.** Run OWASP ZAP or Burp Suite against the application, alongside automated testing. The first scan of an application that has never been scanned usually finds a meaningful list of issues.

**Caveat.** Active scans go only against environments you own and have explicitly authorized. Never point a scanner at production systems you do not control.

---

### NET-13: The storage bucket is public and holds more than images

**Risk.** An upload feature needed somewhere to put files, so the bucket was set to public read to make the app serve images without authorization errors. The same bucket also holds database backups, user exports, and configuration files. Anyone who finds the URL downloads all of it. Bucket names built from the company name and a purpose, such as `company-uploads` or `company-backups`, are enumerated and scanned at scale, and without access logging the downloads leave no record: no IP addresses, no timestamps, no file access history.

**Fix.**
- Separate public assets from private data into different buckets. Public assets get public read, ideally served through a CDN with origin access rather than a public bucket. Everything else stays private and is served through pre-signed URLs that expire.
- Keep account-level public-access blocking on, so a single misconfigured bucket cannot become public.
- Use randomized bucket names that do not contain the company name, the environment, or the data purpose.
- Enable server access logging on every bucket and alert on unusual download volume or access from unexpected IP ranges.
- Cloud storage should store data, not serve it to anyone who asks.

**Caveat.** Current provider defaults are private; the exposure comes from someone making the bucket public to fix an authorization error. Fix the authorization, not the bucket. Randomized names reduce enumeration and are not an access control.

**Test.** Request a private object's URL without a signature from outside the app and confirm it is denied. Confirm the access log recorded the attempt.
