# Browser, Client, and Frontend Boundaries

CORS, clickjacking, React Server Component payloads, Next.js middleware coverage, server-only modules, third-party scripts and Content Security Policy, security headers, secrets in browser bundles, and accessibility.

Entry format: **Risk** is what goes wrong. **Fix** is the change to make. **Caveat** records where a common version of the advice overstates its case. **Check** marks an item where the risk is clear but the remedy depends on the application. **Test** names a concrete way to prove the fix.

---

### WEB-01: Wildcard or reflected CORS lets other sites read your API

**Risk.** Browsers ask which origins the API trusts. If the server answers with a wildcard, or reflects whatever origin was sent without checking it, the API trusts every website, including an attacker's. With credentials allowed, the attacker's page can make requests with the victim's cookies and read the responses.

**Fix.**
- Allow only an explicit list of trusted origins.
- Never combine a wildcard origin with credentials.
- Do not reflect the request's origin header unless it matches a hard-coded allow list.
- Set credentials to true only when the origin matches the trusted list.
- Constrain allowed methods and headers along with origins, and keep cookies scoped tightly.

**Caveat.** A wildcard origin together with credentials is blocked by browsers, so it does not by itself disclose authenticated data. Unvalidated origin reflection with credentials is the configuration that does. Treat the two as distinct cases, and fix both.

---

### WEB-02: Invisible iframes let another site click your buttons

**Risk.** An attacker embeds your app in an invisible iframe on their page and tricks users into clicking actions in your app while they think they are interacting with the attacker's site.

**Fix.**
- Send `X-Frame-Options: DENY`.
- Send `Content-Security-Policy: frame-ancestors 'self'` for modern browsers.
- Use both headers for full coverage, on every response.
- For widgets or integrations that legitimately need framing, set `frame-ancestors` per route to the specific embedding origins.

---

### WEB-03: Server Components ship the whole database row to the browser

**Risk.** A React Server Component queries all fields of a record but displays a few. The entire row, including password hashes, roles, or billing tokens, is serialized into the RSC payload and sent to the browser. The payload is structured and easy to parse. Nested child components inherit every parent prop, so a leak at the parent reaches all children. One leaking component can expose many records quickly.

**Fix.**
- Select and pass only the exact fields needed for rendering.
- Create minimal data objects at every component boundary so props never carry more than the child needs.
- Audit every Server Component regularly to confirm no sensitive field reaches the client.
- Treat the UI as a display window: anything in the payload is accessible, not only what is visible.

**Caveat.** Fetching a full row in a Server Component does not automatically serialize the whole row; what crosses the boundary is what is passed as props to Client Components or otherwise emitted. Inspect the actual payload to see what crosses.

---

### WEB-04: Middleware authentication misses routes and path variants

**Risk.** Next.js middleware guards protected pages, but the matcher excludes some routes, often API endpoints, or fails on path variations such as trailing slashes and encoded characters. Unauthorized requests reach every protected page.

**Fix.**
- Include every route that needs authentication in the middleware matcher, including API endpoints.
- Test path variants: trailing slashes, encoded characters, and case differences.
- Do not trust cookies blindly in middleware. Perform server-side authorization checks on API routes, server components, and data access.
- Treat middleware as a convenience layer. The strongest security is the server-side check at each entry point.

---

### WEB-05: Server secrets end up in the client bundle

**Risk.** Database credentials or API keys written into code that is imported by client components get bundled and shipped to the browser.

**Fix.**
- Install the `server-only` package and import it in every file that touches the database or secrets. The build then fails if that file is ever pulled into client code.
- Move server functions into separate files from client components, and confirm no server module is imported by browser code.
- Run a production build and search the output for secrets. Any match means the secret is exposed.

**Caveat.** Next.js Server Actions do not inherently ship server credentials to the browser. The defect is mixing server and client modules; the fixes above are the controls.

---

### WEB-06: Third-party scripts can read what your pages show

**Risk.** A chat widget, analytics tag, or review script added to the site runs with full access to the page. On a login, checkout, settings, or admin page, that includes what the user types and sees. Apps commonly load scripts from a dozen or more domains without anyone reviewing them.

**Fix.**
- Inventory every external script and resource. For each, document what data it can access and which pages load it.
- Remove third-party scripts from sensitive pages. No analytics, chat, or review script belongs on a page with passwords or payment data.
- Implement page-level script exclusions for sensitive areas.
- Build a Content Security Policy that allows only trusted script sources and blocks inline script injection. Start in report-only mode, observe the reports, then enforce after validation.

**Caveat.** Scripts cannot read HttpOnly cookies or the contents of cross-origin frames. The exposure is what is rendered and typed on the page, which is still enough.

---

### WEB-07: Browser security headers

**Check.** Browsers enforce protections the server has to request through headers. Confirm the response headers for framing (WEB-02), content security policy (WEB-06), strict transport security, content type sniffing, and referrer policy are set deliberately rather than left to framework defaults.

---

### WEB-08: A secret key in the frontend is public

**Risk.** An API key or other secret in frontend code is visible to anyone who opens the browser's developer tools on the live site.

**Fix.** Remove secret keys from client code. Route calls that need the secret through a server endpoint that holds it.

**Test.** Open the live app in the browser, inspect the delivered JavaScript and network responses, and search for key-shaped strings.

**Caveat.** Some keys, such as publishable payment or analytics keys, are designed to be public. A text search is a starting point, not a complete leak audit.

---

### WEB-09: Accessibility is part of production readiness

**Fix.** Support screen readers, full keyboard navigation, and sufficient color contrast. An app that quickly built a polished UI frequently fails all three.

**Caveat.** Lawsuit statistics and compliance consequences cited for accessibility vary by jurisdiction and were not verified; treat the controls as the point.
