# Payments and Webhooks

Webhook signature verification, replay protection and idempotency, server-controlled pricing, provisioning only after verified payment, fulfillment state, failed-payment handling, webhook delivery monitoring, and disputes.

Entry format: **Risk** is what goes wrong. **Fix** is the change to make. **Caveat** records where a common version of the advice overstates its case. **Check** marks an item where the risk is clear but the remedy depends on the application. **Test** names a concrete way to prove the fix.

---

### PAY-01: An unverified webhook accepts fake payment confirmations

**Risk.** The Stripe webhook endpoint processes any event it receives. An attacker posts a fabricated payment-succeeded event and the app provisions the order. A webhook endpoint is effectively a command execution interface.

**Fix.**
- Verify the Stripe signature header against your signing secret before processing any event. Reject any event with a missing or mismatched signature.
- Apply the same rule to every webhook provider. Clerk, GitHub, and the rest all sign their payloads. Every webhook endpoint verifies signatures, no exceptions.
- Use a secret, unguessable endpoint path rather than a generic one such as `/webhooks/stripe`, and allow-list the provider's published webhook IP ranges where available.

**Caveat.** A non-guessable URL and an IP allow list reduce noise. They do not authenticate events. The signature check is the control.

---

### PAY-02: Replayed webhooks repeat their effects

**Risk.** Re-sending a captured webhook charges the same card twice, provisions the same user twice, or re-runs any other side effect.

**Fix.**
- Record every processed event ID and reject any event whose ID has been seen.
- Acknowledge duplicates to the provider so it stops retrying.
- Make processing atomic and retries safe, so a crash after recording receipt but before finishing the work does not lose the work.

**Caveat.** A fixed deduplication window, such as 24 hours, is not universal. Size it to the provider's retry behavior and your own.

**Test.** Replay a captured webhook with its original event ID and confirm the side effect does not repeat.

---

### PAY-03: The browser sets the price

**Risk.** Checkout code puts the price in the frontend and the server passes it through to Stripe. A user edits the request and pays less than intended.

**Fix.**
- Create checkout sessions only on the server.
- Let the client send a product or plan ID, never a dollar amount.
- Look up the price from trusted server-side data, and prefer Stripe Price IDs so the price is locked in Stripe. Replace every raw dollar amount in checkout code with a Price ID.
- Never trust an amount sent by the client. The server controls pricing.

---

### PAY-04: A success page is not proof of payment

**Risk.** The app grants access because the user landed on the success URL. Anyone can navigate to that URL.

**Fix.**
- Provision access only after a verified webhook event confirms the payment.
- Validate the paid amount and product against the expected server-side pricing before fulfillment.

---

### PAY-05: Fulfillment and state transitions after the charge

**Risk.** The payment gateway handles the charge correctly, but what happens next, such as granting entitlements, updating subscription state, or sending confirmations, is inconsistent across payment methods and produces duplicate charges or missed activations.

**Fix.** Review every post-payment state transition. Test each supported payment method and each confirmation path end to end, looking for duplicate charges and inconsistent outcomes.

---

### PAY-06: The webhook fails silently for hours

**Risk.** Webhook deliveries fail or processing throws, nothing alerts, and customers discover the problem first.

**Fix.** Monitor webhook delivery and processing failures independently of customer reports, with alerts on failure rate and on silence from a provider that normally sends events.

---

### PAY-07: Failed payments quietly erode revenue

**Fix.** Build bounded retries for failed payments, notify the customer, and define a deliberate grace period before access is removed. Make sure retries can never double-charge.

---

### PAY-08: Disputes and account holds

**Fix.** Publish a refund policy, build a dispute-handling workflow, and alert on chargebacks so the first dispute is handled rather than discovered.

**Caveat.** A first dispute does not inevitably freeze a payment account. Prepare for disputes without assuming the worst outcome.
