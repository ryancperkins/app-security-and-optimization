# Authentication, Sessions, and Tokens

Magic links, session fixation, redirect abuse in auth flows, password reset, JWT verification, OAuth state and PKCE, token storage, token lifetimes and revocation, idle sessions, and identity provider selection.

Entry format: **Risk** is what goes wrong. **Fix** is the change to make. **Caveat** records where a common version of the advice overstates its case. **Check** marks an item where the risk is clear but the remedy depends on the application. **Test** names a concrete way to prove the fix.

---

### AUTH-01: Magic links can be intercepted, redirected, reused, and flooded

**Risk.** Passwordless login becomes a serious vulnerability when a magic link can be intercepted, when its redirect parameter can be pointed at an attacker's site, when the token never expires, or when an attacker can flood an inbox with requests.

**Fix.**
- Validate the redirect parameter against an allow list of your own domains before redirecting.
- Expire magic-link tokens within 10 minutes and invalidate them after first use.
- Rate limit magic-link requests to 3 per email address per hour and throttle by IP.

**Caveat.** Ten minutes and three per hour are worked examples. Set the numbers from the application's risk and traffic.

---

### AUTH-02: Session fixation lets an attacker log in as your user

**Risk.** If the session ID is not regenerated at login, an attacker who planted a session ID before authentication owns the session after the user logs in. If the session does not change when the user's privileges change, it can be hijacked to keep elevated access.

**Fix.**
- Regenerate the session ID after every login. In Express this is `req.session.regenerate`.
- Regenerate the session on every privilege escalation, not only at login.
- Set Secure, HttpOnly, and an appropriate SameSite value on the session cookie.

**Caveat.** The Express method is framework-specific. Every session library has an equivalent; use it.

---

### AUTH-03: Redirect parameters in auth flows send users to phishing sites

**Risk.** Login URLs commonly carry a redirect or return-to parameter. If it is not validated, users sign in on your real page and are then sent to a phishing site the attacker chose, at the moment they trust you most. Logout flows have the same exposure.

**Fix.**
- Allow redirects only to URLs you trust, using an allow list of your own domains.
- Parse and resolve every redirect URL properly before comparing it. Reject anything outside the application's domains and block protocol-relative URLs.
- Expect bypass attempts using encoded or tricky relative forms such as `//`, backslashes, and percent-encoding.
- Audit every redirect in every authentication flow, including logout, not just login.

**Test.** Send encoded, protocol-relative, and backslash variants to each redirect parameter and confirm each is rejected.

---

### AUTH-04: Old password reset links are permanent keys

**Risk.** A reset link issued months ago still works because tokens never expire and are never invalidated after use. Unlimited reset requests let an attacker abuse the flow.

**Fix.**
- Expire reset tokens within 15 minutes.
- Invalidate each reset token immediately after first use.
- Limit password reset attempts to 3 per email per hour.
- A secure reset flow needs all three: expiration, single use, and rate limiting.

**Caveat.** Fifteen minutes and three per hour are worked examples. Assess them for the application.

---

### AUTH-05: Unverified JWT claims let anyone promote themselves to admin

**Risk.** If the server decodes a JWT and trusts its claims without checking the signature, expiration, and issuer, a user can edit the token to set their role to admin and the server accepts it. Every hand-rolled JWT implementation risks this mistake.

**Fix.**
- Verify the JWT signature on every request using the identity provider's key pair.
- Reject tokens that are expired or carry the wrong issuer.
- Do not parse tokens manually. Use the identity provider's maintained server-side SDK for signature and claim checks (Clerk's server SDK is the worked example). The SDK handles verification and key rotation automatically.

---

### AUTH-06: OAuth authorization codes can be intercepted on mobile networks

**Risk.** An attacker who intercepts the authorization code from a Google sign-in on a mobile device can exchange it and log in as the user before the app does.

**Fix.** Implement both the state parameter and PKCE.
- The state parameter prevents forgery of the authorization response.
- PKCE makes an intercepted code useless: generate a code verifier and store it server-side, hash it and send the hash as the code challenge, then send the original verifier with the token request. The provider checks that the verifier matches the hash. If an attacker intercepted the code without the verifier, the exchange fails.

**Caveat.** Missing PKCE does not by itself mean any observer can read TLS traffic. PKCE guards the code exchange; it does not replace transport security.

---

### AUTH-07: An open OAuth redirect lets an attacker collect login tokens

**Risk.** If the OAuth redirect URL is not locked to exact registered values, an attacker can route the login response to a site they control.

**Fix.**
- Register exact redirect URLs with the provider. Never use wildcards or patterns.
- Restrict every OAuth redirect to your own domain through the provider's allow list.
- Send a unique, random state parameter on every login and reject mismatches.
- Request only the OAuth scopes the app actually needs.
- Audit every OAuth integration against these points.

---

### AUTH-08: Auth tokens in localStorage are readable by any script

**Risk.** A token stored in `localStorage` is readable by every script on the page, including third-party widgets. A stolen bearer token is the user's identity for as long as it stays valid.

**Fix.**
- Move tokens out of `localStorage` into cookies marked HttpOnly, Secure, and an appropriate SameSite value so JavaScript cannot read them.
- Keep access tokens short-lived. Fifteen minutes is the worked example.
- Use refresh-token rotation so each refresh issues a new token pair instead of extending one long-lived credential.
- Revoke tokens when the user's security state changes, such as a password change or a compromised-account report. Two workable designs: a token revocation list, or a per-user token version that invalidates everything issued before it.

**Caveat.** Cookie-based designs need CSRF defenses that bearer-token designs did not. Add them when moving to cookies.

---

### AUTH-09: Tokens and credentials that never expire are not authentication

**Risk.** A user who logged in six months ago still has a live session. A static API key that has been the same for six months is a permanent door, and if it is shared, its blast radius is everything it can reach.

**Fix.**
- Give every session and token a finite lifetime and a revocation path, sized to the credential type. This covers abandoned sessions and lost devices.
- Prefer appropriately scoped short-lived or dynamically issued credentials over secrets that stay valid indefinitely.
- Review every long-lived shared API key for rotation and scope reduction.

---

### AUTH-10: Social sign-in needs a refresh-token lifecycle

**Fix.** After adding a provider such as Google sign-in, implement refresh-token issuance, rotation, and graceful handling of expiration so users are not silently stranded or silently kept alive forever.

---

### AUTH-11: Idle-session policy should not punish active users

**Risk.** A fixed timer that logs users out every 15 minutes regardless of activity reads as broken security and loses user work.

**Fix.** Base idle expiry on meaningful user activity, warn before expiry, and preserve safe in-progress work across re-authentication.

---

### AUTH-12: Unusual access needs context, not a single signal

**Risk.** A login at 3 AM from another country may be an attacker, or a traveling user.

**Fix.** Use contextual authentication signals in combination to identify unusual access and step up verification.

**Caveat.** Geography or time alone must not establish that a login is malicious, nor that it is legitimate.

---

### AUTH-13: Token forgery through algorithm handling

**Check.** Token libraries have had vulnerabilities where the algorithm field in the token is trusted, letting anyone forge a token. Confirm the library in use pins accepted algorithms and rejects `none`. The specific affected library and version must be verified for your stack.

---

### AUTH-14: Choosing an identity provider

**Fix.** Evaluate providers against what customers will demand: enterprise SSO and SAML, assurance level, and documentation quality. Enterprise customers often need tenant-specific SAML configuration. OIDC is also an option and a Google sign-in may satisfy SSO for some buyers.

**Caveat.** The cheapest or free provider is not automatically insecure, and self-hosted authentication is a risk decision rather than a philosophy. Any maintained provider still requires correct integration.

---

### AUTH-15: Quickly built apps usually ship broken auth

**Fix.**
- Use a maintained authentication library or provider rather than custom auth.
- Test logout: confirm protected data is no longer reachable after logging out.
- Enforce row-level access to data in addition to authenticating the user.

**Caveat.** A public page shell may still load after logout without being a defect; the test is whether protected data loads. Filtering by user ID in application code is not the same as database row-level security.

**Check.** If login works for one user but users land in each other's accounts at scale, the root cause is likely session or cache keying. Investigate before adding users.
