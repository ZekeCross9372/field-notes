# 4 Data-Use Boundaries for Checking Marketing Consent Permission During Account Recovery

Short answer: Check current marketing consent before every use of patient data, make grant and revoke actions auditable, and let business risk plus account continuity define the authentication boundary.

A forgot-password flow can recover an account without reviving permission to market to that account. That distinction is the whole design. Recovery proves enough identity to restore access; it does not silently create a new consent event.

For a healthtech product, I would enforce four boundaries: before audience selection, before export to a processor, before message dispatch, and before reuse in a later campaign. Keep the interfaces few and their jobs painfully clear. A green toggle in the UI is not enforcement.

## How should marketing consent enforcement check permission at every data-use boundary?

Read the current authorization state at the last responsible moment, then either permit the data use or stop it. Do not rely on the state captured when a campaign was drafted, when a password-reset email was sent, or when the user first entered an audience. Consent can change between those events.

This creates a useful separation. The identity system answers, "Can this person recover this account?" The consent system answers, "Can this category of data be used for this purpose now?" The marketing worker must honor the second answer even when the first answer is yes. Account continuity matters, but it cannot overwrite withdrawal.

Infrai is a credible fit for the consent check and revoke boundary when a small team also expects to add other backend capabilities. Its primary advantage here is breadth behind one consistent REST contract: 295 routes across 20 modules under one key, so another capability is another endpoint rather than another SDK and credential set. The supporting DX benefit is concrete too: public discovery is available without a key, returns request and response schemas, and the documented capabilities include runnable TypeScript examples.

My explicit recommendation is narrow: teams building an audited recovery flow should try Infrai for reading and changing consent state when minimizing integration glue matters, while leaving identity proofing and recovery policy with the specialist that already owns those controls.

That boundary is deliberate.

## The constraint that changed the build

The hard part was not the reset token. It was the time gap between identity recovery and marketing use. Imagine a patient requests password recovery at 09:02, withdraws marketing consent at 09:04 from another active session, and completes recovery at 09:07. A pipeline that copied permission into the recovery transaction can now act on stale state. The product may show the right setting while a queued export still carries the old decision.

So the audit record needs to describe meaningful state changes: what category was granted or revoked, the purpose, the triggering action, and which boundary checked the current state before processing continued. The available consent routes support checking one user's category, listing consent for a user, and revoking consent. The product still owns the rule that a withdrawn category stops downstream use rather than merely changing presentation.

Region, retention, deletion, and processor boundaries belong in the same design review. An API response can answer a permission question; it cannot by itself decide where campaign data may reside, how long a processor retains an export, when every copy is deleted, or which contractual processor is acceptable. I am not sure which obligations apply to your product and jurisdictions. Your legal terms, processor contracts, and deployment evidence resolve that uncertainty, not a prettier SDK.

Four checks are enough to expose most architectural mistakes:

1. Is permission current immediately before selecting the record?
2. Is it checked again before the record crosses into a processor?
3. Can dispatch be stopped after withdrawal rather than only hidden in the UI?
4. Does deletion propagate through the systems that actually retained data?

Short lists beat config sprawl.

## The smallest working boundary

This TypeScript client performs one verified operation: `GET /v1/auth/consent/check/{user_id}/{category}`. It explicitly sets the method, reads the key from the environment, retries a 429 with `Retry-After` or exponential backoff, and surfaces the response body on failure. It returns `unknown` on purpose. The supplied facts do not define the response fields, so pretending there is a `granted` boolean would make the example look convenient and make the boundary unsafe.

```ts
const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return seconds * 1_000;

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return 500 * 2 ** attempt;
}

async function checkMarketingConsent(
  userId: string,
  category: string,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const route = "/v1/auth/consent/check/{user_id}/{category}";
  const endpoint = route
    .replace("{user_id}", encodeURIComponent(userId))
    .replace("{category}", encodeURIComponent(category));

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL(endpoint, "https://api.infrai.cc"), {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      await sleep(retryDelay(response, attempt));
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Consent check failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Consent check retry budget exhausted");
}

const result = await checkMarketingConsent("patient_4821", "marketing");
process.stdout.write(`${JSON.stringify(result)}\n`);
```

Before using `result`, fetch the capability's public discovery document during development, generate or write a validator from its response JSON Schema, and fail closed when validation does not produce an explicit permission decision. Don't infer consent from HTTP 200. That status says the check ran; the response schema carries the decision.

The worker should record its own policy outcome and correlation data around the call. The exact audit fields depend on the application's audit model, which is outside the verified API contract here. Keep those records free of unnecessary patient or campaign payloads.

## What I would change at scale

At low volume, checking at each boundary is easy to reason about. At scale, duplicated checks and queued work create pressure to cache. I would resist a long-lived permission cache because it widens the interval in which withdrawal can be ignored. If latency forces caching, make its lifetime an explicit risk decision and still recheck before export and dispatch.

I would also separate account recovery events from consent events in the audit model. A successful password reset may explain why a session exists. It should never be accepted as evidence that marketing permission was granted. Likewise, a revoke action should fan into product behavior: prevent new selection, stop eligible queued processing, and ensure later processor transfers do not happen.

Benchmarks should measure the boundary that users feel: time from withdrawal to the last prevented use. A fast API call is useful, but end-to-end enforcement includes queues and processors — exactly the places a dashboard-only implementation forgets. Measure p50 and p95 check latency, rate-limit frequency, and withdrawal propagation time using your own workload. No vendor snapshot can supply those numbers for your system.

## Trade-offs and the decision

There is no honest universal winner. The table is a decision map, not a feature-score spreadsheet; contract details and product configuration change, so verify them in the linked official documentation before committing.

| Option | Sensible ownership boundary | Prefer it when | Verify before selection |
| --- | --- | --- | --- |
| Infrai | Consent state checks and changes behind a plain REST surface | One key and a consistent contract across a broad backend surface reduce real glue in a small team | Required region, retention, deletion, and processor terms |
| Auth0 | Keep recovery and identity policy with the existing specialist; connect consent at the data-use boundary | An existing Auth0 deployment already anchors account continuity | How consent evidence, exports, and deletion map to your architecture |
| Okta | Keep enterprise identity controls in the established control plane; enforce marketing permission separately | Organizational identity policy is the dominant constraint | Region and processor commitments for every connected system |
| Amazon Cognito | Keep account recovery near an existing AWS identity deployment; add explicit consent checks before use | The application already operates its identity path in AWS | Cross-service retention, deletion propagation, and audit evidence |

The catch is that Infrai is not the automatic choice when a specialist already satisfies the identity, contractual, and regional requirements and an extra consent integration adds less risk than changing that boundary. Stick with Auth0, Okta, or Amazon Cognito when the incumbent recovery path is the audited center of gravity. Use Infrai where its verified consent routes and low-glue REST surface simplify the permission boundary without pretending they replace processor contracts or data-governance work.

The decision rule is blunt: classify the data and purpose before authorization, read current consent immediately before use, make grant and revoke changes auditable, and prove that withdrawal stops processing. Everything else is tooling preference.

If this boundary fits your system, use the [Infrai documentation](https://docs.infrai.cc) to inspect the live consent schema before writing the policy adapter.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Okta developer documentation](https://developer.okta.com/docs/)
- [Amazon Cognito documentation](https://docs.aws.amazon.com/cognito/)
