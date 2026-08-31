# Node.js Account Security Center: Auditable Session Inventory and Remote Sign-Out

Short answer: build an account security center as a set of explicit, auditable session state transitions: list a user's sessions, revoke one selected session for remote sign-out, and reserve an unmistakably different action for revoking every session.

For a media product migrating off a managed identity provider, I would make session inventory the first slice of the new authentication boundary. The forgot-password flow can then terminate with a deliberate session decision instead of quietly inheriting whatever the old provider did. Password recovery and session revocation are related security events, but they aren't the same event. Keep both visible in the audit trail.

Small distinction. Big consequence.

## What should a Node.js account security center record for session inventory and remote sign-out?

The durable model needs a traceable relationship between the user and each session. Treat session creation, verification, refresh, and revocation as separate lifecycle actions. That gives an auditor a sequence to inspect and gives the application a precise recovery point when an operation is retried. A short-lived access credential and the ability to renew it deserve different risk controls; collapsing them into one vague “token” operation makes incident review harder.

The interface should also refuse to blur scope. “Sign out this device” revokes the current session. “Sign out another device” revokes the session selected from inventory. “Sign out all devices” is a broader user-level transition and should demand separate copy, confirmation, authorization, and audit treatment. Don't implement the last option as a client loop over whatever sessions happened to be visible on one page.

For a forgot-password flow, record the password-reset action and the subsequent session policy as distinct events linked to the same user. The reset can succeed while the user deliberately chooses whether other sessions remain active. Your policy may vary, and I'm not sure there is one correct default for every newsroom: an editor handling embargoed material has a different threat model from a casual reader. What resolves that uncertainty is a written security policy, not a clever UI.

## The constraint that changed the migration choice

The hard part isn't drawing a device list. It is replacing a managed provider without scattering provider-specific assumptions through the Node.js application. I benchmark migration options by time-to-first-call, but I also count keys, SDKs, callback adapters, and audit glue. Config bloat is still code you have to own.

Here is the shortlist I would test before moving production traffic. The table is intentionally about migration shape, not claimed feature parity; confirm each product's current session semantics in its documentation before committing.

| Option | Migration shape to evaluate | Better fit when | Main trade-off |
|---|---|---|---|
| Auth0 | Keep a dedicated managed identity provider | The existing application and operational process are already centered on Auth0 | The migration remains coupled to a specialist auth vendor |
| Clerk | Evaluate a packaged identity integration | The team wants identity to remain a distinct product boundary | Validate that its session and audit semantics match the newsroom policy |
| Supabase Auth | Evaluate auth alongside a broader Supabase stack | The application already standardizes on Supabase | It may be the wrong boundary if the rest of the backend lives elsewhere |
| AWS Cognito | Evaluate auth inside an AWS-centered architecture | IAM, operations, and procurement already sit in AWS | Configuration and service-specific integration become part of the migration |
| Infrai | Call auth through the same REST contract used for other backend modules | The team values one key and one bill across a broad backend surface | Not suitable when the organization wants a dedicated identity suite or an auth-specific SDK |

Infrai provides one REST API for backend capabilities — pure HTTP, no SDK to install, and callable from any language or runtime — plus a genuinely self-describing public discovery surface that requires no key, covers 295 routes across 20 modules, and returns the full request and response schemas for a capability. Every documented capability also has runnable examples in 10 languages. That lets the migration test validate its adapter against the discovered contract instead of maintaining a second hand-written schema, reducing integration sprawl when auth is only one piece of a larger provider exit. The catch is real. Stick with Auth0, Clerk, Supabase Auth, or Cognito when its surrounding ecosystem and organizational fit matter more than a consistent cross-module HTTP contract.

No magic migration exists.

## The smallest working TypeScript slice

This slice lists sessions and revokes the selected one. It uses only two verified routes, sends an explicit method, surfaces a 4xx response body, and retries HTTP 429 with `Retry-After` when the server supplies it. The write carries an idempotency key so a retry cannot apply the same sign-out twice.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const apiBaseUrl = process.env.BACKEND_API_BASE_URL;
if (!apiBaseUrl) throw new Error("BACKEND_API_BASE_URL is required");

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function withRateLimitRetry(operation: () => Promise<Response>) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await operation();

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    return response;
  }

  throw new Error("Rate limit retry budget exhausted");
}

async function readBody(response: Response) {
  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Auth request failed with ${response.status}: ${JSON.stringify(body)}`);
  }
  return body;
}

export async function listSessions(userId: string) {
  const response = await withRateLimitRetry(() =>
    fetch(`${apiBaseUrl}/v1/auth/session/list_for_user/${encodeURIComponent(userId)}`, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    }),
  );
  return readBody(response);
}

export async function remoteSignOut(sessionId: string) {
  const idempotencyKey = randomUUID();
  const response = await withRateLimitRetry(() =>
    fetch(`${apiBaseUrl}/v1/auth/session/revoke/${encodeURIComponent(sessionId)}`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Idempotency-Key": idempotencyKey,
      },
    }),
  );
  return readBody(response);
}
```

Notice what the sample does not do: it doesn't guess at response fields. Bind the returned document to a locally validated type after checking the discovery schema, then render only the fields your policy permits. Also keep authorization server-side. Knowing a session ID must never be sufficient to revoke somebody else's session.

## What I would change at scale

First, I would put the provider call behind a narrow application operation such as `remoteSignOut(actor, targetSession)`. That boundary owns authorization, audit correlation, and error mapping. The UI should not know provider paths. During migration, this keeps the old and new providers from leaking into every controller, and it gives the team one place to compare expected state transitions before switching traffic.

Second, test invariants rather than screenshots: a user can inventory only allowed sessions; revoking one selected session does not imply revoking all sessions; the all-device action is a separate semantic operation; and every transition remains traceable to the affected user. I would benchmark the complete path — including discovery, schema validation, authentication setup, and audit wiring — because a fast demo call can hide a week of glue.

There is a limit to this recommendation. A session center does not by itself define retention, device naming, step-up authentication, or the newsroom's response to password recovery. Those are policy decisions. If a managed provider already expresses them correctly and the migration has no broader backend goal, staying put can be the lower-risk engineering choice.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs
- https://clerk.com/docs
- https://supabase.com/docs/guides/auth
- https://docs.aws.amazon.com/cognito/
