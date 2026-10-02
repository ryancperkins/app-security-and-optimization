# Privacy and Data Protection

Securing data answers "can an attacker read it?" Privacy answers a different question: "should we hold it at all, and for how long?" The cheapest data to protect, breach, and comply over is the data never collected. Treat personal data as a liability to minimize, not an asset to accumulate.

Entry format: **Risk** is what goes wrong. **Fix** is the change to make. **Caveat** records where a common version of the advice overstates its case. **Check** marks an item where the risk is clear but the remedy depends on the application. **Test** names a concrete way to prove the fix.

---

### PRIV-01: Classify data as it is added

**Risk.** Data that nobody classified cannot be protected, exported, or deleted on request, because nobody can find it.

**Fix.** Classify every field when it is introduced, and handle it by class:

| Class | Examples | Handling |
|---|---|---|
| Non-personal | Aggregates, anonymized counts | Normal handling |
| Personal | Name, email, IP address, device and user IDs | Minimize, access-control, include in export and delete |
| Sensitive | Health, finance, precise location, biometrics, government IDs, anything about minors | Extra basis to collect, stricter access, usually encryption and audit logging |

---

### PRIV-02: Collect against a stated purpose

**Risk.** "It might be useful later" is not a purpose. It is breach scope scheduled for later.

**Fix.**
- Collect a field only against a stated use, and record the purpose with the field.
- Keep personal data out of telemetry and logs. Allow-list the fields that may be logged rather than logging whole request bodies.

---

### PRIV-03: Retention is set up front and deletion actually happens

**Risk.** Personal data with no expiry accumulates in the primary store, backups, caches, search indexes, and analytics copies. Deletion that flips a flag leaves all of it in place.

**Fix.**
- Give every personal-data store a retention limit.
- Build a working deletion path that reaches backups, caches, search indexes, and analytics copies, and verify it end to end.

**Caveat.** Soft deletion alone does not satisfy a deletion obligation. Where a documented retention or legal-hold requirement applies, reconcile it explicitly rather than deleting or keeping everything by default (see GOV-03).

**Test.** Delete a test account and confirm its data is gone from every store, including a restored backup after the retention window.

---

### PRIV-04: Data-subject rights are engineering features

**Fix.** Support export, correction, and deletion on request where the jurisdiction requires it (GDPR, CCPA, and similar regimes). Design the schema so one user's data is findable and erasable rather than spread irreversibly across systems. Manual handling misses copies; the schema has to make the request answerable.

---

### PRIV-05: Consent gates collection and third-party sharing

**Risk.** Sending personal data to an analytics, advertising, or model vendor is sharing. Doing it without consent, or without an agreement with the vendor, is a compliance failure and a trust failure.

**Fix.**
- Obtain consent before collecting beyond what the service needs and before sharing with any third party, and make the consent record auditable.
- Put a data-processing agreement in place with every vendor that receives personal data.
- Review which fields flow to each vendor and strip what the vendor does not need.

---

### PRIV-06: Residency and rules differ by user location

**Fix.** Make data-residency and policy a configurable boundary rather than hard-coding one region's law. Apply the rules for where the user is, not where the company is.

---

### PRIV-07: Secrets and personal data in logs

**Risk.** Telemetry pipelines are a classic leak path. A log line with a token, a password, or a full card number is copied to every system that ingests logs.

**Fix.** Never log secrets, tokens, passwords, or full personal data. Allow-list fields at the logging boundary. Redact at the source, not at the dashboard.

---

### PRIV-08: A privacy incident starts a clock

**Fix.** When an incident exposes personal data, breach-notification deadlines become part of the postmortem. Record what data, whose, and when it was exposed, and route the notification decision to the person accountable for it (see OBS-05 for the incident process).

**Caveat.** Which deadline applies depends on jurisdiction and data class; surface it as a decision for counsel rather than deciding it in code.
