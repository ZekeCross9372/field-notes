# CAPTCHA Token Verification: 3 Express Middleware Steps Before Creating a User

Short answer: put CAPTCHA verification in middleware immediately before user creation, fail closed on network or parse errors, and consume the verified result once. For an edtech signup flow, the least complex useful design is a three-check pipeline: validate the request shape, verify the CAPTCHA server-side, then create the user. Do not let a browser callback stand in for server verification.

| Choice | Signup safety | Migration effort | Operational burden |
|---|---|---|---|
| Route-specific middleware | Strong boundary before account creation | Low behind an interface | One timeout and error path |
| Route handler | Works, but ordering is easier to break | Low initially | Logic spreads as routes multiply |
| Browser only | No trustworthy server gate | None | Abuse reaches creation unchecked |

**Recommendation:** use route-specific middleware backed by a tiny provider-neutral verifier interface. It keeps denial ahead of database writes and leaves one narrow boundary to replace during migration off a managed provider.

## How Should Express Middleware Verify a CAPTCHA Token Before Creating a User?

A CAPTCHA token is evidence to be checked, not an authentication credential and not a session. The server sends it to the configured verification service, inspects the response, and rejects the signup before password hashing, welcome-email work, roster creation, or any other side effect. In Node.js, the middleware's job is to make that ordering unavoidable even as the handler changes. Ordering beats abstraction here.

Fail closed.

A malformed request can return `400`; a completed verification that rejects the token can return `403`; and a verification dependency that times out or returns unreadable data should fail closed with `503`. Never echo the submitted token or upstream response into logs.

The first criterion is **side-effect ordering**. Search the complete signup path, including queues and hooks, for work that can happen before the gate. In an education product, even a rejected fake account can trigger guardian email if the boundary sits too late. Put CAPTCHA after basic body validation, so junk does not consume verification calls, but before every durable or externally visible action.

The second criterion is **a small, replaceable contract**. Migration gets ugly when handlers understand provider response fields. Normalize them once into a local result containing only what the application uses. Do not promise risk scores, action names, or hostname semantics unless signup policy actually depends on them. Config bloat starts there.

## A three-check middleware pipeline

This TypeScript example uses Express conventions and the built-in `fetch` API available in current Node.js releases. The endpoint and response shape are local abstractions; map them to the documented contract of the verification service you operate.

```ts
import express, { NextFunction, Request, Response } from "express";

type CaptchaResult = { accepted: boolean };

interface CaptchaVerifier {
  verify(token: string, signal: AbortSignal): Promise<CaptchaResult>;
}

class HttpCaptchaVerifier implements CaptchaVerifier {
  constructor(
    private readonly endpoint: URL,
    private readonly secret: string,
  ) {}

  async verify(token: string, signal: AbortSignal): Promise<CaptchaResult> {
    const response = await fetch(this.endpoint, {
      method: "POST",
      headers: {
        authorization: `Bearer ${this.secret}`,
        "content-type": "application/json",
      },
      body: JSON.stringify({ token }),
      signal,
    });

    if (!response.ok) throw new Error(`verification status ${response.status}`);
    const body: unknown = await response.json();

    if (
      typeof body !== "object" || body === null ||
      !("accepted" in body) || typeof body.accepted !== "boolean"
    ) {
      throw new Error("invalid verification response");
    }
    return { accepted: body.accepted };
  }
}

type SignupBody = {
  email?: unknown;
  password?: unknown;
  captchaToken?: unknown;
};

function requireSignupShape(
  req: Request<unknown, unknown, SignupBody>,
  res: Response,
  next: NextFunction,
): void {
  const { email, password, captchaToken } = req.body;
  if (
    typeof email !== "string" ||
    typeof password !== "string" ||
    typeof captchaToken !== "string" ||
    captchaToken.length === 0
  ) {
    res.status(400).json({ error: "invalid_signup_request" });
    return;
  }
  next();
}

function requireHuman(verifier: CaptchaVerifier) {
  return async (
    req: Request<unknown, unknown, SignupBody>,
    res: Response,
    next: NextFunction,
  ): Promise<void> => {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 2500);

    try {
      const result = await verifier.verify(
        req.body.captchaToken as string,
        controller.signal,
      );
      if (!result.accepted) {
        res.status(403).json({ error: "signup_verification_failed" });
        return;
      }
      next();
    } catch {
      res.status(503).json({ error: "signup_verification_unavailable" });
    } finally {
      clearTimeout(timeout);
    }
  };
}

const app = express();
app.use(express.json({ limit: "16kb" }));

const verifier = new HttpCaptchaVerifier(
  new URL(process.env.CAPTCHA_VERIFY_URL ?? "https://verification.invalid/check"),
  process.env.CAPTCHA_SECRET ?? "",
);

app.post(
  "/signup",
  requireSignupShape,
  requireHuman(verifier),
  async (_req, res) => {
    // Hash the password and create the user only after both gates pass.
    res.status(201).json({ status: "created" });
  },
);
```

The `2500` value demonstrates bounded waiting. It is not a universal target. Benchmark the full middleware path under accepted, rejected, and timed-out responses, then choose a timeout from an explicit availability budget. Track percentiles and concurrent verification requests; a happy-path average hides the failure mode that exhausts sockets while automated signups arrive.

Record a correlation ID, a coarse outcome such as `accepted`, `rejected`, `timeout`, or `invalid_response`, and latency. Do not record the token. Keep email addresses out of these security-event fields unless the incident process genuinely needs them.

Tokens stay secret.

## Test the boundary, not the provider

Unit tests should fake `CaptchaVerifier`; they should not call an external service. Cover five cases: acceptance calls the handler once, rejection never calls it, timeout returns `503`, malformed upstream JSON fails closed, and a missing token is rejected before the verifier runs. Add an integration test with a local HTTP stub to catch serialization and timeout mistakes.

Replay deserves its own test. Submit the same token twice and confirm the behavior promised by the chosen verification service. If its contract does not guarantee single use, consider a short-lived server-side digest cache keyed by the token hash, never the raw token. The cache must be shared across instances or it gives a false guarantee during horizontal scaling. This adds state. Require it only when the documented semantics and threat model justify it.

Treat an upstream `429` as a bounded dependency failure and honor its documented backoff signal; do not loop indefinitely with the same CAPTCHA token. If the client may retry account creation after a timeout, protect the database write with an idempotency key or an equivalent unique signup constraint. CAPTCHA verification does not make the write idempotent.

Measure the count of user-creation attempts that occur without a successful gate. It should stay at zero.

## When is handler-level verification the better runner-up?

A single-purpose service with exactly one signup path may keep verification inside the handler. That runner-up is reasonable while the handler stays short, tests prove no write occurs first, and there is no second entry point such as invitation acceptance or guardian onboarding.

It stops being attractive once the same gate protects several routes. Copy-pasted checks drift: one path forgets the timeout, another maps dependency failure to acceptance, and a third creates a partial student record before checking. Middleware centralizes ordering, but do not install it globally. Login, health checks, and authenticated APIs should not inherit a signup challenge by accident.

A dedicated policy service can fit several applications that need consistent abuse decisions and independent deployment. It also adds a network hop, versioned policy contracts, and another failure domain. For one API, that is usually more machinery than this decision earns.

The middleware approach has a real limitation: it protects only requests that traverse that exact route stack. A background import, administrative enrollment tool, or new router can bypass it. The trade-off is local clarity versus system-wide policy enforcement. Choose a policy service when several independently deployed applications create accounts and the organization can operate another dependency; choose handler-level verification when one tiny transaction is easier to audit in one place. Route middleware is the middle option, not a universal winner.

## Migration and stolen-session revocation are separate controls

The verifier interface makes provider migration mechanical: implement a second adapter, run contract tests against both adapters, switch configuration, and retain the old path only for a bounded rollback window. Compare behavior using synthetic test tokens from each service's documented test facility, not captured production tokens. Never send one provider's token to another.

CAPTCHA does not revoke a session. If a learner reports a stolen session, rotate refresh tokens according to the authorization server's design and invalidate the compromised session in the server-side session store or revocation mechanism. Reject reuse of an invalidated refresh token, and expire the associated session family when the design supports that relationship. OWASP recommends reauthentication after high-risk events and invalidating sessions after reauthentication; those controls belong to session management, not signup middleware.

Keep the two paths observable but distinct. Signup metrics answer whether automation is reaching account creation. Session events answer which session refreshed, reauthenticated, or was revoked. Combining them into one generic `auth_failed` counter destroys the signal needed during an incident.

The decision rule is plain: choose route middleware when account creation has multiple side effects or must survive a provider migration; keep the handler-level runner-up only while one short, well-tested path owns the transaction. Either way, server verification finishes before the first write.

## Further reading

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- https://developer.mozilla.org/en-US/docs/Web/API/AbortController
- https://expressjs.com/en/guide/using-middleware.html
- https://www.rfc-editor.org/rfc/rfc7009
- https://www.rfc-editor.org/rfc/rfc9700
