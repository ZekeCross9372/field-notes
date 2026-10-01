# Logistics DNS Orchestration: One Internal Endpoint Sets 2 Sending Domain Modes

Use a customer-owned DNS zone by default, and make one internal endpoint return a durable operation rather than pretend DNS propagation is synchronous. Choose a platform-owned subdomain when the platform must control rollout speed and the customer accepts delegation.

| Decision | Customer-owned zone | Platform-owned zone |
| --- | --- | --- |
| DNS authority | Customer | Platform |
| First setup | Customer applies a record plan | Customer delegates once |
| Ongoing changes | Require customer coordination | Platform applies them |
| Exit path | Records stay with the customer | Delegation and records move |
| Best fit | Brand control and existing governance | Standardized, high-volume onboarding |

**Short answer:** for a logistics system sending dispatch notices from customer brands, keep the zone customer-owned and automate the plan, verification, and mail binding behind one endpoint. Return `pending_dns`, not `ready`, until authoritative DNS answers prove that the records exist.

This is slower than taking over DNS. The trade is deliberate: a late shipment notice is bad, but moving a customer's DNS authority into an application merely to accelerate email onboarding creates a much larger ownership decision.

## Which ownership model matches the blast radius?

The first criterion is control. In a customer-owned model, the logistics platform generates the SPF, DKIM, and DMARC record set while the customer publishes it. That fits organizations with an established DNS change process. It also means the endpoint cannot honestly promise immediate completion.

A platform-owned model usually starts with delegation of a dedicated subdomain such as `notify.example-logistics.com`. Once that delegation is visible, the platform can manage records below it. This reduces repeated coordination, but expands the service boundary to nameserver availability, record lifecycle, and offboarding.

The second criterion is failure isolation. Do not ask for control of an apex zone when the job is authenticated transactional mail. A dedicated sending subdomain separates dispatch traffic from corporate mail and narrows the names the workflow may change.

Narrow scope wins.

No ownership model removes coordination.

The authentication pieces have different jobs. SPF authorizes hosts to use a domain in the SMTP envelope identity and limits evaluation to 10 terms that cause DNS lookups. DKIM signs selected message content and exposes a public key beneath a selector. DMARC evaluates alignment between the visible `From` domain and an authenticated SPF or DKIM domain, then publishes policy and reporting instructions at `_dmarc`. Passing SPF somewhere in the transaction is not enough. Alignment is the detail that breaks otherwise plausible setups.

## Treat readiness as observed state

A single endpoint is useful only if it collapses glue code, not truth. DNS is distributed state. Mail verification is another state machine. The endpoint should create intent, persist the exact expected records, request the mail-side identity, and enqueue verification. It should not hold an HTTP request open while resolvers catch up.

I would expose one command and one operation resource. That is two concepts, not a dozen tiny endpoints. Retrying the command with the same idempotency key must return the same operation; otherwise a client timeout can create multiple selectors or competing identities.

Benchmark the workflow with measurements that can be reproduced: command latency, time from creation to authoritative-record match, time from DNS match to mail verification, and failures grouped by reason. Do not publish a single "setup time" percentile without separating customer change delay from platform processing. It hides the only bottleneck teams can act on.

Keep terminal reasons precise. `spf_lookup_limit`, `dkim_value_mismatch`, `dmarc_missing`, and `mail_verification_rejected` are actionable. `verification_failed` is config-shaped fog.

## How can one internal endpoint set up a sending domain?

The request needs the sending domain, ownership model, and a stable tenant identifier. The server derives record names. Accepting arbitrary DNS names and values from callers turns a small provisioning API into an unreviewed DNS editor.

```ts
type Ownership = "customer" | "platform";
type State =
  | "pending_dns"
  | "pending_mail_verification"
  | "ready"
  | "failed";

interface ProvisionDomainInput {
  tenantId: string;
  sendingDomain: string;
  ownership: Ownership;
  idempotencyKey: string;
}

interface ExpectedRecord {
  name: string;
  type: "TXT" | "CNAME" | "NS";
  values: string[];
}

interface ProvisioningOperation {
  id: string;
  state: State;
  expectedRecords: ExpectedRecord[];
  checkedAt?: string;
  failureReason?: string;
}

interface DomainStore {
  findByKey(key: string): Promise<ProvisioningOperation | null>;
  create(input: ProvisionDomainInput): Promise<ProvisioningOperation>;
  save(operation: ProvisioningOperation): Promise<void>;
}

interface MailControl {
  createIdentity(domain: string): Promise<ExpectedRecord[]>;
}

interface WorkQueue {
  enqueueVerification(operationId: string): Promise<void>;
}

async function provisionSendingDomain(
  input: ProvisionDomainInput,
  store: DomainStore,
  mail: MailControl,
  queue: WorkQueue,
): Promise<ProvisioningOperation> {
  const prior = await store.findByKey(input.idempotencyKey);
  if (prior) return prior;

  const operation = await store.create(input);
  operation.expectedRecords = await mail.createIdentity(input.sendingDomain);
  operation.state = "pending_dns";
  await store.save(operation);
  await queue.enqueueVerification(operation.id);
  return operation;
}
```

For a tenant called `north-harbor`, the caller can propose `mail.north-harbor.example`. The returned plan may include one SPF TXT record, selector-specific DKIM records, and a DMARC TXT record. Exact values come from the selected mail system; inventing them inside a generic DNS layer would couple that layer to a provider contract.

Verification should query authoritative DNS and compare semantic values. This deserves more care than a string equality check. A TXT answer can arrive as multiple character strings even though the publisher and resolver treat it as one record, so the verifier needs to join the chunks in order before comparing the SPF or DMARC value. It must still keep distinct TXT records distinct. For CNAME-based DKIM, compare the normalized target and account for the terminal dot used by fully qualified names. Query the authoritative servers when diagnosing publication, while using normal recursive resolution for the readiness view that resembles an application lookup. Store both observations with timestamps. Otherwise an operator sees only “mismatch,” pastes the same visible value again, and gains no clue that the stale recursive answer or chunk representation caused the disagreement. Preserve the originally expected value as well; normalization is a comparison tool, not permission to rewrite the audit trail.

Representation matters.

Then verify mail-side status. Only after both checks succeed should the operation become `ready`. The message path must refuse a tenant-domain pair that is not ready. Accepting it and silently substituting another `From` domain converts a provisioning defect into a branding and alignment defect.

## The mail example is part of acceptance

DNS presence proves publication. It does not prove that a real message uses the intended identities. Add a post-verification probe addressed to a controlled mailbox, then inspect the received authentication results. Use synthetic shipment data and a non-customer recipient.

```ts
interface MailProbe {
  send(input: {
    from: string;
    to: string;
    subject: string;
    text: string;
  }): Promise<{ messageId: string }>;
}

async function sendAcceptanceProbe(
  domain: string,
  mailbox: string,
  mail: MailProbe,
): Promise<string> {
  const result = await mail.send({
    from: `dispatch@${domain}`,
    to: mailbox,
    subject: "Dispatch authentication probe",
    text: "Synthetic shipment SHP-2048 entered the test network.",
  });
  return result.messageId;
}
```

The acceptance worker checks that the received message has a DMARC pass and records the aligned identifier, selector, timestamp, and message ID. It should not treat inbox placement as a deterministic API response. Authentication and receiver placement are different questions.

Start DMARC policy deliberately. RFC 7489 defines `none`, `quarantine`, and `reject`, plus aggregate and failure reporting mechanisms. A new domain can collect reports before stronger enforcement, but the move to enforcement needs an owner and review date. Leaving `p=none` forever is observation, not protection.

There is a concrete SPF trap too. Appending another `v=spf1` TXT value is not composition. RFC 7208 requires the result to be no more than one SPF record; the workflow must detect an existing policy and demand an explicit merge decision. It must also evaluate DNS-lookup-producing terms against the limit before claiming readiness.

## When is platform ownership the better runner-up?

Platform ownership is better when onboarding volume makes customer-applied changes the dominant delay, every tenant can use a dedicated delegated subdomain, and the operating team is prepared to own DNS availability and migrations. A fleet of regional carriers with the same notification pattern may fit. One delegation can support later selector rotation without another customer ticket.

The limitation is operational weight. Platform ownership is not a fit for a small team that cannot operate authoritative DNS, rehearse zone recovery, and support delegation changes. Customer ownership has the opposite trade-off: it preserves customer control but makes external change queues part of activation time. Neither mode is universally better, and the endpoint should expose that delay instead of disguising it.

Write the exit test first. Can a tenant revoke delegation without losing access to historical DMARC reports? Can record intent be exported? Can a new operator reproduce the zone before nameservers change? If those answers are vague, fast onboarding borrowed time from offboarding.

That boundary is non-negotiable.

Customer ownership remains stronger when a central security team approves every email identity, when the organization already automates DNS through infrastructure as code, or when policy forbids third-party nameserver delegation. Improve the handoff: return machine-readable records, show exact mismatches, and recheck automatically. Do not respond with a screenshot tutorial.

The final architecture is small on purpose: one idempotent command, one persisted operation, an explicit record plan, asynchronous authoritative checks, mail-side verification, and a synthetic acceptance message. **The endpoint coordinates the lifecycle; DNS and received mail supply the evidence.**

## References

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- RFC 6376, DomainKeys Identified Mail (DKIM) Signatures: https://datatracker.ietf.org/doc/html/rfc6376
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
- RFC 1034, Domain Names — Concepts and Facilities: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 1035, Domain Names — Implementation and Specification: https://datatracker.ietf.org/doc/html/rfc1035
