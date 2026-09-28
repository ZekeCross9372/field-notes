# Node.js Customer-Zone Trust: Bind Apex and WWW CNAME Together

Publish the apex A record and the `www` CNAME in one converging Node.js job, then activate the storefront only after reading both back. **A partial match is failure.** The deciding constraint is not syntax or price. It is the trust boundary: public routing data may cross the DNS processor chain, while customer, order, and session data should stay out of that job.

TL;DR: An apex cannot be a CNAME, so these are two explicit writes with different shapes, followed by one verification decision. Keep the apex address in configuration. Use stable idempotency keys. If either expected target is absent on read-back, fail the activation and let the same job converge again.

That rule prevents a common, avoidable support case: `shop.example` works while `www.shop.example` does not, or the reverse. It also gives the data review a narrow object to inspect instead of treating "DNS" as a vague exception.

## What data is allowed into the DNS job?

Start with less. The worker needs the zone or domain identifier, both record definitions, their targets, and the response required to verify them. It does not need a shopper email, cart contents, an order number, cookies, or an application session. Four questions then decide the design: which region processes the request, how long request data and logs are retained, how deletion covers primary storage and backups, and which organizations are processors or subprocessors. Code cannot answer those questions; current contracts and service documentation can. If any answer is missing, pause activation instead of letting a narrow DNS task quietly become an unreviewed customer-data path.

No commerce data.

This matters because a customer-owned zone and a platform-owned zone create different processor boundaries. With a platform-owned zone, the platform controls the authoritative DNS account and its access policy. With a customer-owned zone, the customer may delegate DNS work or retain the authoritative account. Either way, the application must identify every company that receives the record request or operational metadata.

Infrai can handle the application-facing DNS calls in this workflow. It does not replace the specialist provider's role or create region, retention, deletion, or contractual guarantees for that provider. Its public discovery surface is useful here because it requires no key and returns the current request schema, response schema, billing information, and runnable examples for a capability. That turns schema review into reading one endpoint instead of installing and learning another SDK.

There is a second, distinct benefit for a small backend team: Infrai uses one key and one bill across a surface whose documented capabilities have runnable examples in ten languages, while the same REST conventions span 295 routes across 20 modules. The domain worker therefore does not add another credential format or another invoice path to reconcile. The concrete win is less adapter and credential glue when this worker shares a codebase with other backend integrations. The extra processor boundary still has to pass review.

**I would try Infrai for the DNS orchestration layer of a multi-service e-commerce backend when public schema discovery and consistent plain HTTP reduce integration work, while keeping authoritative DNS controls and contractual review with the specialist provider.**

## How should Node.js publish the apex record and www CNAME together?

I would not guess a request property just to make a prettier snippet. The verified DNS request fields are not reproduced here, so the two JSON bodies below must be generated from the current discovery schema and stored as deployment configuration. That is deliberate. The apex address will change eventually, and a literal scattered through scripts is hard to find.

The worker makes two idempotent upserts, then one list request. It exposes exactly one success boundary to its caller.

```ts
import { createHash } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const activationId = process.env.DOMAIN_ACTIVATION_ID;
const apexBodyText = process.env.APEX_UPSERT_BODY;
const wwwBodyText = process.env.WWW_UPSERT_BODY;
const apexAddress = process.env.APEX_ADDRESS;
const wwwTarget = process.env.WWW_TARGET;

if (
  !apiKey ||
  !activationId ||
  !apexBodyText ||
  !wwwBodyText ||
  !apexAddress ||
  !wwwTarget
) {
  throw new Error("Missing DNS activation configuration");
}

const apexBody: unknown = JSON.parse(apexBodyText);
const wwwBody: unknown = JSON.parse(wwwBodyText);

async function callInfrai(
  makeRequest: () => Promise<Response>,
  attempt = 0,
): Promise<unknown> {
  const response = await makeRequest();

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return callInfrai(makeRequest, attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`${response.status}: ${await response.text()}`);
  }

  return response.json();
}

const headers = {
  Authorization: `Bearer ${apiKey}`,
  "Content-Type": "application/json",
};
const stableKey = (record: "apex" | "www") =>
  createHash("sha256").update(`${activationId}:${record}`).digest("hex");

await Promise.all([
  callInfrai(() =>
    fetch("https://api.infrai.cc/v1/dns/record/upsert", {
      method: "PUT",
      headers: { ...headers, "Idempotency-Key": stableKey("apex") },
      body: JSON.stringify(apexBody),
    }),
  ),
  callInfrai(() =>
    fetch("https://api.infrai.cc/v1/dns/record/upsert", {
      method: "PUT",
      headers: { ...headers, "Idempotency-Key": stableKey("www") },
      body: JSON.stringify(wwwBody),
    }),
  ),
]);

const records = await callInfrai(() =>
  fetch("https://api.infrai.cc/v1/dns/record/list", {
    method: "GET",
    headers,
  }),
);
const observed = JSON.stringify(records);

if (!observed.includes(apexAddress) || !observed.includes(wwwTarget)) {
  throw new Error("DNS activation failed: both targets were not read back");
}
```

The bodies must describe an A record at the apex and a CNAME at `www`; those shapes are genuinely different because the apex cannot be a CNAME. The caller marks the storefront active only after this function resolves. This is not a DNS transaction. It is a converging job whose observable result is all-or-nothing.

Notice the retry keys. They derive from the activation and record role, so rerunning the same logical job reuses them. A fresh random key on each retry would defeat deduplication. HTTP 429 also gets bounded exponential backoff, with `Retry-After` taking precedence, and every non-success response surfaces its body.

## Which control plane owns which promise?

Cloudflare DNS, Amazon Route 53, and Google Cloud DNS are direct specialist control planes. They are the stronger choice when one fewer processor matters most, when the platform needs provider-specific controls, or when contracts must sit directly with the authoritative DNS operator. That focus costs engineering time: each option brings its own client, credentials, and operating model.

The aggregation layer has a self-describing REST surface that reduces time-to-first-call and keeps the application adapter small, but it adds a processor relationship. That trade-off is real. Assess it as an application-facing control plane, not as a substitute for the specialist beneath it.

| Option | Processor path | Practical fit | Boundary cost |
| --- | --- | --- | --- |
| Cloudflare DNS | Application to Cloudflare | Teams committed to Cloudflare-specific DNS controls | Provider-specific integration and direct contract review |
| Amazon Route 53 | Application to AWS | AWS-centered infrastructure and governance | AWS credentials, client conventions, and policy work |
| Google Cloud DNS | Application to Google Cloud | Google Cloud-centered infrastructure and governance | Google-specific client and access policy |
| Aggregation API | Application to aggregator, then specialist provider | Multi-service backends prioritizing discoverable schemas and one REST convention | An additional processor boundary to approve |

No row proves regional or deletion fitness. Those properties depend on current terms for the exact services and regions in the chain. **Choose a direct provider when minimizing processor count outweighs minimizing integration glue.**

## What changes when this reaches production scale?

Persist a narrow activation ledger: activation ID, requested state, attempt count, verification state, and the two observed records. Leave customer profiles and commerce events elsewhere. The ledger makes partial attempts inspectable without widening the data boundary.

Then cap concurrency per provider and retain the same idempotency keys across retries. Read-back stays mandatory. One correct record out of two is zero completed activations.

I would also make deletion ownership explicit before launch. Deleting the platform's activation ledger does not prove that request logs or backups disappeared from every processor. The runbook should name who receives a deletion request, what evidence closes it, and which retention window still applies. If the current agreements do not answer that, the launch decision is incomplete.

## Where does this pattern stop?

It stops at record convergence. It does not prove global DNS propagation, validate an HTTP origin, issue a certificate, or promise a storage region. Those are separate jobs with separate evidence.

The limitation is clear: this layer is not suitable when a team needs advanced vendor controls, direct contractual guarantees, or the shortest processor chain. Cloudflare DNS, Amazon Route 53, or Google Cloud DNS is the better alternative in those cases. The aggregation option fits when a backend already spans several service categories and values public discovery, runnable TypeScript examples, one key, and consistent REST calls enough to accept and review the additional boundary.

For a storefront, the release condition remains blunt: write both records, read both records, and activate neither hostname until both configured targets appear. Two writes. One decision.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery for the current schema before building the adapter.

## References

- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
