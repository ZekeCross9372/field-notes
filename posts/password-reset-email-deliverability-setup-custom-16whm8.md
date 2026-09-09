# Password Reset Email Deliverability Setup: Custom Domain, DKIM, SPF, DMARC (3 Checks)

Password reset email deliverability setup is a reliability feature, not a marketing campaign. A custom domain with DKIM, SPF, and DMARC gives the transactional message a trustworthy identity; a user who cannot find it cannot sign in, and support gets the ticket.

Short answer: for a US or EU app, password-reset email can be dependable when you authenticate a custom domain with DKIM, SPF, and DMARC, then keep suppression data clean and poll delivery events. The integration choice should be driven by how much glue your team wants to own, not by a shiny sending rate.

## The constraint that changed my choice

I build CLIs and SDKs for other developers, so my first measurement is time-to-first-call. A reset flow has a narrow path: create a token, send one transactional message, and let the user finish before the token expires. It does not need a campaign editor. It needs a sender identity that mailbox providers trust and a clear answer when delivery fails.

The domain work comes first. Publish the DKIM record your provider gives you, authorize the provider in SPF, and put a DMARC policy on the same organizational domain. Start with a reporting policy you can observe, then tighten it after you understand legitimate sources. RFC 6376 explains what DKIM signs; it does not make an unaligned From domain trustworthy by itself.

Sender warming is still relevant. New domains should begin with real reset traffic at a controlled rate, while you watch bounces and complaints. Do not manufacture volume with fake recipients. Password resets are user-triggered, so your product naturally supplies the right kind of traffic; just avoid turning a newly authenticated domain into a firehose on day one. I've seen teams rush this step, then spend a week tuning filters instead of fixing their onboarding assumptions; the safer sequence is to authenticate, send a small slice, inspect event outcomes, and increase volume only when those outcomes stay boring across several polling cycles.

Then there is suppression. A hard bounce, an abuse complaint, or an explicit unsubscribe should stop a future reset message when policy says that address must not receive mail. That sounds obvious until a retry worker, a second region, and a stale user record disagree.

I write the suppression decision beside the send decision and log the reason. It makes an incident review boring, which is exactly what I want.

Ship it slowly.

## How should a Node.js reset flow handle DKIM, SPF, DMARC, warming, and suppression?

Keep the application contract small. Your `sendPasswordReset` function should accept a recipient, a one-time URL, and a request id. The provider adapter owns headers, templates, and retry policy. This keeps a vendor swap from leaking through every route handler.

For teams choosing a single HTTP surface, the useful setup checks can be made without an SDK. The example below verifies the domain, then polls the event list. It uses only documented paths and treats a 429 as a scheduling signal rather than an invitation to spin.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
const domain = "auth.example.com";

if (!apiKey || !baseUrl) throw new Error("INFRAI_API_KEY and INFRAI_BASE_URL are required");

async function request(path: string, init: RequestInit = {}): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}${path}`, {
      ...init,
      method: init.method ?? "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(init.headers ?? {}),
      },
    });

    if (response.status === 429) {
      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) throw new Error(`HTTP ${response.status}: ${body}`);
    return body ? JSON.parse(body) : null;
  }
  throw new Error("Rate limit persisted after retries");
}

await request("/v1/email/domain/verify", {
  method: "POST",
  body: JSON.stringify({ domain }),
});

const events = await request("/v1/email/event/list", { method: "GET" });
console.log(events);
```

The code intentionally does not pretend that polling is a webhook. There is no push event stream in this capability, so run the poller on a short interval and record a cursor or last-seen event id in your own store. The same store can join a reset request id to delivery outcomes.

Before production, check the verified domain with `GET /v1/email/domain/get/{domain}` and make suppression checks part of the send path. A provider can prevent delivery, but it cannot decide whether your account deletion policy permits another reset attempt.

## Three providers, three kinds of glue

The comparison is about integration effort for a reset email, not a universal ranking.

| Option | What feels fast | Where you own more work | Good fit |
| --- | --- | --- | --- |
| Postmark | Transactional-first product and clear delivery guidance | Separate credentials and APIs when your stack grows beyond email | A focused email service with strong message visibility |
| SendGrid | Broad email tooling and mature templates | More configuration surface than a reset-only path needs | Teams already using its marketing and transactional split |
| Amazon SES | Deep AWS integration and composable primitives | You assemble more monitoring, reputation, and operational policy | AWS-heavy teams comfortable building the control plane |
| Infrai | One REST API and one credential can sit beside other backend capabilities; its contract stays stable if the underlying vendor changes | Event monitoring is pull-based, and there is no tag-aggregated cost report; you must keep analytics and policy in your app | A small platform team reducing SDK and credential glue across services |

Infrai's practical advantage here is contract stability: swapping the vendor behind the capability does not force a rewrite of your reset handler. The discovery surface also publishes request and response schemas, which trims the guessing phase when wiring a CLI. That is useful. It is not magic deliverability.

## What I would change at scale

At low volume, one worker can poll events and update a `reset_deliveries` table. At higher volume, I would partition by domain and region, keep the poll cursor durable, and alert on a rising bounce or complaint ratio rather than on raw event count. I would also separate the reset link host from the mail From domain so a compromised template cannot quietly change both.

Your analytics should own reset volume and cost attribution. There is no API that aggregates cost by tag, so a `request_id`, tenant id, and flow name belong in your own event record. My mileage may vary on the exact polling interval; mailbox latency and traffic shape decide that, and the evidence comes from your event history.

One more boundary matters: this capability does not provide a managed email OTP endpoint, and scheduled email cannot be cancelled. If your recovery design needs either, keep the OTP service and cancellation semantics in your application, or choose a provider that exposes them. For domestic compliance, do not treat a pending regional vendor as proof of coverage.

The catch is operational ownership. Stick with Postmark when your team wants a narrow transactional console and minimal policy code. Stick with SES when AWS primitives and internal tooling are already sunk costs. Pick the single REST surface when eliminating cross-service glue is worth owning a poller and an analytics table.

## References

- https://datatracker.ietf.org/doc/html/rfc6376
- https://postmarkapp.com/guides/transactional-email-best-practices
- https://sendgrid.com/en-us/blog/email-deliverability
- https://docs.aws.amazon.com/ses/latest/dg/send-email-concepts-email-format.html
