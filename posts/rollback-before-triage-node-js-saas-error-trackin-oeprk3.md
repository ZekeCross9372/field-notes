# Rollback Before Triage: Node.js SaaS Error Tracking, Logging, and Correlation IDs

Short answer: for rollback-safe notification delivery, make structured logs the release scoreboard, reserve error tracking for unexpected exceptions, and carry the same correlation ID through both.

| Decision | Structured logs | Error tracking | Rollback consequence |
| --- | --- | --- | --- |
| Did accepted notifications reach a terminal outcome? | Primary record | Incomplete view | Use the log-derived outcome ratio |
| Did a release introduce a code defect? | Release context and affected deliveries | Grouped exception and stack | Correlate both before reverting |
| Did one notification retry three times? | Full attempt sequence | Only unexpected failures | Count the notification once at terminal state |
| Did a provider reject a valid request? | Classified operational event | Usually no exception | Do not treat the rejection alone as a bad release |

**Recommendation:** define the rollback contract before choosing either destination. Record one terminal outcome for every accepted notification, attach `release`, `notification_id`, and `correlation_id`, then send only unexpected code failures to an error tracker with those same fields. This is the least complex design that gives a B2B SaaS team both a denominator and a useful stack trace.

Don't let exception volume become the release scoreboard.

## What should Node.js SaaS error tracking, structured logs, and correlation IDs prove?

A rollback signal must distinguish a deployment regression from ordinary delivery trouble. Exception counts cannot do that alone. Successful notifications usually do not throw, while one failed notification may throw once per retry. A count without a denominator can rise because traffic rose, retries changed, or code actually regressed. Those are different decisions.

The operational contract starts with the notification state machine. Record events for acceptance, attempt completion, retry scheduling, and the terminal outcome. Keep the vocabulary small: `delivered`, `retryable_failure`, `permanent_failure`, and `internal_error` are enough for the example here. The exact labels can differ because provider contracts differ, but their meanings must be reviewed like an API. A terminal event answers whether the job ultimately completed; an attempt event explains how it got there.

Rollback safety comes from comparing terminal outcomes by release. Keep old and new release labels visible during a staged deployment, set the decision threshold before rollout, and require enough completed notifications for the comparison to mean something. I'm not sure what threshold is useful without a service's baseline traffic and failure distribution. A low-volume tenant and a high-volume shared queue cannot usefully inherit the same number. Replay representative cases, observe the distribution, and decide the gate from evidence.

This is the key distinction: structured logging records what the delivery system did, including expected rejection and retry paths; error tracking collects unexpected exceptions that an engineer can fix, such as an invariant violation or a rejected promise crossing a worker boundary. Sending every permanent provider rejection to the exception channel destroys that distinction. Soon the alert queue is just another inbox.

Correlation IDs make the two records navigable. Generate one when work enters the service, or accept a valid upstream identifier at a trusted boundary, and propagate it through queue metadata. Keep `notification_id` separate. The first joins an execution across components; the second identifies the domain job across attempts. One notification can have several attempts under the same correlation context, and a terminal query must still count it once.

No scavenger hunt.

The identifier must not encode a tenant, destination, message body, or credential. OWASP's Logging Cheat Sheet advises excluding or masking access tokens, passwords, sensitive personal data, and connection strings, and it recommends sanitizing event data to prevent log injection. For notification systems, an allowlist is easier to reason about than an expanding redact list. Keep destinations, authorization headers, provider credentials, and message bodies out of both channels.

## Put the rollback contract in code

The implementation below uses Node.js request context so correlation metadata does not have to travel through every function signature. The logger and tracker remain tiny interfaces. Business logic knows the event contract, not a vendor SDK or its configuration tree.

```ts
import { AsyncLocalStorage } from "node:async_hooks";
import { randomUUID } from "node:crypto";

type Context = {
  correlationId: string;
  release: string;
};

type DeliveryOutcome =
  | "delivered"
  | "retryable_failure"
  | "permanent_failure"
  | "internal_error";

interface Logger {
  info(fields: Record<string, unknown>, message: string): void;
  error(fields: Record<string, unknown>, message: string): void;
}

interface ErrorTracker {
  captureException(
    error: Error,
    context: Record<string, string | number>,
  ): void;
}

const deliveryContext = new AsyncLocalStorage<Context>();

function currentContext(): Context {
  const context = deliveryContext.getStore();
  if (!context) throw new Error("Notification context is missing");
  return context;
}

function safeErrorFields(error: unknown): Record<string, string> {
  if (error instanceof Error) {
    return { error_name: error.name, error_message: error.message };
  }
  return { error_name: "NonErrorThrown", error_message: String(error) };
}

async function deliverNotification(
  notificationId: string,
  notificationKind: string,
  attempt: number,
  send: () => Promise<DeliveryOutcome>,
  logger: Logger,
  tracker: ErrorTracker,
): Promise<DeliveryOutcome> {
  const { correlationId, release } = currentContext();
  const base = {
    event: "notification.delivery_attempt",
    notification_id: notificationId,
    notification_kind: notificationKind,
    correlation_id: correlationId,
    release,
    attempt,
  };

  try {
    const outcome = await send();
    logger.info({ ...base, outcome }, "Notification attempt completed");
    return outcome;
  } catch (cause) {
    const error =
      cause instanceof Error ? cause : new Error("Non-error value thrown");
    const outcome: DeliveryOutcome = "internal_error";

    logger.error(
      { ...base, outcome, ...safeErrorFields(cause) },
      "Notification attempt failed internally",
    );
    tracker.captureException(error, {
      correlation_id: correlationId,
      release,
      notification_kind: notificationKind,
      attempt,
    });
    throw error;
  }
}

export async function runDelivery<T>(
  release: string,
  work: () => Promise<T>,
  incomingCorrelationId?: string,
): Promise<T> {
  const correlationId = incomingCorrelationId ?? randomUUID();
  return deliveryContext.run({ correlationId, release }, work);
}
```

There is a deliberate limit in this sample: it records attempts, while the queue or orchestration layer must emit the single terminal notification event after delivery or retry exhaustion. Keeping those concepts separate prevents an attempt from masquerading as a final result. It also makes duplicate terminal events a testable contract violation instead of a dashboard surprise.

Typed fields matter more than polished messages. Dashboards and gates should query `event`, `outcome`, `release`, and identifiers; prose belongs in `message` for a human scanning one record. Parsing sentences in production is config debt with punctuation.

The adapter boundary deserves scrutiny too. If changing a destination requires edits across handlers, workers, and provider clients, the interface is too shallow. If a backend dictates domain event names, it owns part of the delivery model. Keep initialization boring: one context adapter, one logger adapter, one tracker adapter.

## Rehearse the failure before trusting the rollback

A rollback dashboard that has never seen a controlled failure is decoration. Test the evidence path in layers, then run the exact release query used by automation.

Unit tests should cover a successful attempt, an expected retryable outcome, a permanent failure, and an unexpected throw. Assert that every attempt emits one structured record and that only `internal_error` triggers exception capture. Queue-boundary tests should serialize and restore `correlation_id`, `notification_id`, `release`, and `attempt`; otherwise a retry can silently break the join that on-call relies on.

The deployment test is longer because it should be. Send known outcomes through old and candidate release labels. Confirm that terminal counts use notifications rather than attempts, that an expected retry does not inflate the internal-error ratio, and that a thrown error can be located in both systems using the same correlation ID. Then roll back the candidate and verify that events continue to carry an unambiguous release value during the transition. The test is about evidence continuity, not a particular dashboard.

Benchmark the instrumentation in the same harness. Compare p50 and p99 handler latency, serialized bytes per notification, and CPU with logging disabled, structured events enabled, and exception capture exercised separately. No universal overhead number is honest here; runtime, transport, sampling, payload shape, and traffic all affect it. Publish the harness beside the service so a future config change can be measured instead of debated.

Operational review goes beyond speed. Set retention by purpose, restrict access, sanitize untrusted fields, and make failures in the telemetry path visible without allowing them to change delivery semantics. The delivery service should not mark a customer notification successful because a log was written, nor failed merely because diagnostic transport was unavailable. That separation is easy to state and easy to blur in a rushed integration.

## When is one signal enough?

The combined design has a real cost: two data paths, two access-control surfaces, two retention policies, and a correlation contract that can drift. It is not suitable for every service.

Stick with structured logging alone when a small service has low traffic, no staged releases, and an engineer can diagnose rare failures from one event stream. It preserves the delivery timeline and the rollback denominator. Add error tracking when reconstructing stack traces and grouping repeated code failures becomes recurring work, not because a maturity checklist demands it.

Error-tracking-first is the runner-up when the immediate risk is an unexpected crash in template rendering or provider-client code and the team has not defined useful delivery states yet. The catch is that it cannot answer how many accepted notifications ultimately delivered. Treat it as a triage choice, not a rollback metric, and add terminal outcomes before automating a release gate.

Logging-first also fits systems where long-lived event queries and auditability dominate. Error tracking earns its place where exception grouping and engineering ownership dominate. Neither choice permits sensitive payloads in diagnostics, and neither replaces a tested correlation contract.

The decision rule stays short. Roll back on a release-correlated regression in terminal outcomes. Investigate unexpected code failures through exception tracking. Use the correlation ID to cross the boundary, and reject every extra field or config layer until it proves operational value.

## Further reading

- OWASP Logging Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html
