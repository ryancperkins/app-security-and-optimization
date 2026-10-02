# Compliance, Privacy, and Business Readiness

Regulatory applicability, health data, account deletion and retention, legal documents, SOC 2 and enterprise procurement, analytics privacy, business and tax setup, internationalization, and financial tracking.

Every entry in this file is a prompt for a qualified review by counsel, an auditor, or an accountant. The skill surfaces the question; it does not make the determination.

Entry format: **Risk** is what goes wrong. **Fix** is the change to make. **Caveat** records where a common version of the advice overstates its case. **Check** marks an item where the risk is clear but the remedy depends on the application. **Test** names a concrete way to prove the fix.

---

### GOV-01: Regulatory obligations arrive with the users

**Check.** AI features may carry disclosure obligations under the EU AI Act, and users in other jurisdictions can change which rules apply. Applicability depends on jurisdiction, activity, and role, not on user location alone. Keep a compliance review on the launch checklist.

**Caveat.** Not every AI app is subject to every rule, and claims that an app has "become illegal" are overstated without analysis.

---

### GOV-02: Health data

**Check.** A healthcare app needs protected-health-information safeguards: access controls, audit logs, encryption, and the applicable agreements with vendors and hosts. Multi-tenancy with row-level security, defined roles, and MFA are the usual building blocks.

**Caveat.** Penalty figures and blanket statements about which agreements are required need qualified assessment. A quickly built system described as compliant has to be verified, not believed.

---

### GOV-03: Account deletion

**Fix.**
- Map deletion cascades and downstream copies before implementing account removal. Deletion must cover related records and copies in other systems.
- Reconcile deletion with documented retention obligations and legal holds.

**Caveat.** Soft deletion alone may not satisfy a deletion obligation. Blanket retention periods, such as seven years for everything, are not universal.

---

### GOV-04: Legal documents and policies

**Check.** Before the first paying user: privacy policy, terms of service, data-processing agreement where applicable, platform terms review, cyber insurance, and a quarterly review of all of them. Make sure the privacy disclosures accurately describe what the app does.

**Caveat.** AI-drafted documents are a starting point, not legal review. An insurance quote is not an executed policy. Documents alone do not secure software, and no checklist makes a product lawsuit-proof.

---

### GOV-05: Enterprise procurement and SOC 2

**Check.** Enterprise buyers send procurement checklists and often ask for SOC 2. Architecture reviews and penetration-test reports answer some questions. Compliance automation can shorten readiness time.

**Caveat.** Neither a report nor a shortcut guarantees a deal, a timeline, or acceptance, and some buyers do not require SOC 2 at all. Assurance needs depend on the customer.

---

### GOV-06: Analytics and user behavior data

**Fix.** Use cohort, usage, and drop-off analysis to understand customer behavior, with privacy controls on collection and retention.

---

### GOV-07: Business entity, banking, and tax

**Check.** A product that collects revenue needs an entity, banking, and agreements. Revenue from many states or countries can create sales-tax obligations; assess exposure, collect at checkout where required, and keep a filing calendar.

**Caveat.** These are prompts for an accountant and counsel, not determinations.

---

### GOV-08: Built for one country

**Check.** Review international payments, time zones, date formats, and language assumptions before serving users outside the first market.

---

### GOV-09: Is the build making money

**Fix.** Track per-feature cost, per-user profitability, and financial outcomes.

**Caveat.** This is business measurement, distinct from security assurance.
