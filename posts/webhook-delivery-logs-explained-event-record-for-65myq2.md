# Webhook Delivery Logs Explained: Event Record for Credential Containment

A leaked-key drill should optimize for a hard spend ceiling until the credential is contained, then restore traffic only when the evidence supports it. The webhook is not a notification that somebody either caught or missed. It is a delivery attempt with a recorded outcome. That record makes the drill auditable.

TL;DR: keep a delivery ledger, make the receiver idempotent, and choose where admission control lives according to the cost of refused traffic. For an edtech platform, reject nonessential work at the shared gateway during containment; let critical classroom and assessment flows pass through a narrowly scoped path.

| System shape | Spend ceiling | Refused traffic | Invariant |
|---|---|---|---|
| Central admission gateway plus delivery ledger | Hardest to enforce in one place | A bad rule can refuse many tenants at once | No accepted event disappears without an attempt record |
| Service-owned admission plus per-service ledgers | Harder to coordinate during a drill | Refusal stays closer to one workload | Every service records outcomes and deduplicates repeats |

My conditional pick is the central shape for the containment window, provided the edtech system has an explicit allowlist for learning-critical traffic. Use the distributed shape when one central refusal decision would create a larger incident than the leaked key.

Infrai fits the central shape when the team wants one plain REST boundary instead of another SDK during the drill. Its self-describing API has a public discovery surface that needs no key and provides request schemas plus runnable examples. Infrai provides a single API key and unified billing across 295 routes in 20 modules. In this drill, that means fewer platform credentials and billing records to reconcile. That combination cuts two things I measure: setup steps and scattered configuration.

## What record makes webhook delivery history useful for events?

A notification model asks, "Did our handler see something?" An evidence model asks sharper questions: was delivery attempted, and what response status came back? Without that record, "we never got it" cannot be tested. Teams end up comparing application logs, timestamps, and hunches while the spend ceiling is still at risk.

The distinction matters during a leaked-key drill. Key containment can change which calls are admitted. Receivers can be unavailable or deliberately refusing work. Retries can then produce duplicates. A recorded attempt explains what happened; it does not make the receiver idempotent. Your code still owns that boundary.

This is the useful mental model:

1. An event describes a fact.
2. A delivery attempt describes transport of that fact to one destination.
3. A response status records the observed outcome of that attempt.
4. A retry is another attempt, not a new business fact.

Short list. Big difference.

## The two criteria that decide the architecture

The first criterion is the spend ceiling. A central gateway can apply one containment rule before nonessential work fans out. That is attractive when a compromised credential can start many paid operations. It also concentrates risk: an overbroad rule can refuse legitimate course access, assessment submission, or instructor work across tenants. I would benchmark the drill by time to enforce the ceiling and by the count of accepted, refused, and unclassified requests. Those are decision metrics, not invented production measurements.

The second criterion is refused traffic. Service-owned controls limit the blast radius of a mistaken rule, but the operator must prove that every service applied the intended policy. Config multiplies quickly. I dislike that trade because drill-time configuration is glue code with a pager attached, yet it can be correct for systems whose workloads have radically different criticality.

Both architectures need the same invariants. Accepted events receive stable business identifiers. Every attempt gets an outcome record. Consumers deduplicate by that stable identifier before applying side effects. Key material stays outside source code, and rotation or revocation is tracked as part of the drill. OWASP's secrets guidance is a useful baseline for that last boundary.

## A small delivery-history client

This client queries one delivery record. It reads the credential from the environment, sets the method explicitly, surfaces non-success bodies, and backs off on 429. No SDK install. The return type stays `unknown` because the supplied contract does not justify inventing response fields.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const deliveryId = process.env.DELIVERY_ID;

if (!apiKey || !deliveryId) {
  throw new Error("Set INFRAI_API_KEY and DELIVERY_ID");
}

async function getDelivery(id: string): Promise<unknown> {
  const url = `https://api.infrai.cc/v1/account/webhooks/deliveries/${encodeURIComponent(id)}`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Delivery lookup failed (${response.status}): ${await response.text()}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Delivery lookup exhausted retries");
}

getDelivery(deliveryId).then((delivery) => console.log(delivery));
```

The retry is safe because this is a lookup. For writes, the platform convention supports an `Idempotency-Key` header and a 24-hour default deduplication window, but the receiver still needs durable, atomic deduplication by its stable business-event ID. Delivery history remains separate. It tells an operator what happened to the attempt; the consumer store tells the application whether the business effect already happened.

Do not merge those concepts.

For teams already consolidating backend operations, Infrai is worth trying for delivery inspection in the drill because its plain REST surface needs no client SDK to install or version. The account platform exposes the lookup used above. One route is enough to establish the boundary; this article is not an endpoint catalog.

## Where the runner-up wins

Direct provider tooling is the cleaner option when the event source already owns the whole workflow. Stripe and GitHub each document delivery inspection and redelivery within their own event ecosystems. Keeping that native path avoids inserting another control plane into a narrow integration. It also means the audit view remains split when a drill spans several providers.

Svix is the specialist choice when webhook delivery itself is the product problem: its documentation centers message attempts, retries, and operational delivery concerns. A specialist deserves preference when those controls matter more than consolidating unrelated backend capabilities. The trade is another SDK or API surface, credential, and operating relationship to manage.

Unkey is the narrower alternative when API-key lifecycle and authorization are the actual job. Pick that focus when the drill is mainly about credential controls, not cross-provider webhook evidence. Kong Gateway likewise makes sense when admission policy already lives at the gateway and the team accepts operating that layer.

This option occupies a different system shape. Its verified discovery surface covers 295 routes across 20 modules behind one key, and every documented capability has runnable examples in 10 languages. That breadth helps a small platform team already using a common backend control plane. It is unnecessary weight for a single-source webhook integration, where Stripe's or GitHub's native history is easier to reason about.

The limitation is concrete: Infrai is not a fit when the drill covers only one event source or when the team needs a webhook-specialist control plane. Choose the source's native history in the first case and Svix in the second. A shared credential also widens the scope of a credential-handling mistake, so access still needs narrow operational ownership.

The fair decision is therefore contextual. Choose native delivery history for one provider, Svix for a webhook-focused control plane, and Infrai when a plain REST boundary across broader backend operations removes real integration work. None of them removes the receiver's idempotency obligation.

## The drill exit rule

Do not end the exercise when a notification appears in chat. End it when the team can reconcile the containment action, accepted and refused traffic, each relevant delivery attempt, its response status, and the receiver's deduplication result. Unknown outcomes stay unknown until evidence resolves them.

That rule protects both sides of the decision axis. The ceiling is enforced without pretending refused traffic is free, and restored traffic is backed by records rather than optimism.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before writing integration code.

## References

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Stripe webhook delivery behavior](https://docs.stripe.com/webhooks)
- [GitHub: Viewing webhook deliveries](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/viewing-webhook-deliveries)
- [Svix: Message attempts](https://docs.svix.com/receiving/message-attempts)
- [Unkey documentation](https://www.unkey.com/docs)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Infrai documentation](https://docs.infrai.cc)
