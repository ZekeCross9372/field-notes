# Compliance Evidence for Urgent Event Notifications with SMS and Itinerary Polling

Short answer: send an urgent school-trip change by SMS first, but treat provider acceptance as evidence of an attempt, not proof that a parent saw it. Poll only while the message can still affect the trip. If no terminal success arrives by that deadline, send an email containing the same change plus the generated itinerary report as an attachment. Store every transition in an append-only notification record.

| Choice | Speed | Evidence quality | Attachment | Use here |
| --- | --- | --- | --- | --- |
| SMS only | Fast path | Delivery events, not acknowledgement | No | No |
| Email only | Slower attention path | Submission and delivery events | Yes | No |
| SMS, then email | Fast path plus detail | One correlated timeline | Yes | Yes |
| Both together | Fast, but noisy | Overlapping attempts | Yes | Highest-risk policy only |

**Recommendation:** use a deadline-driven state machine, not a generic retry loop. The useful output is a compact evidence bundle showing what changed, who was targeted, which channel was attempted, what the provider reported, when fallback began, and which exact report was attached.

## What does compliance evidence actually prove?

A provider message ID proves that a provider accepted a request. It does not prove delivery, reading, comprehension, or consent. Keep those claims separate. Inflating one status into another makes an audit log look cleaner and less truthful.

Acceptance is not delivery.

For a school trip, bind the evidence record to an immutable event revision. Record the trip ID, recipient ID, normalized destination, template version, locale, change-summary hash, report hash, channel, provider message ID, timestamps, raw provider status, normalized status, and the policy decision that caused fallback. Do not store an attachment and later regenerate its hash from current trip data. The itinerary may have changed again by then.

Urgent operational mail is not automatically marketing mail. RFC 8058 defines a one-click unsubscribe mechanism for mailing-list messages using `List-Unsubscribe-Post`; it is not a receipt mechanism for a transactional alert. Classify the message before applying list controls, and never interpret an unsubscribe request as evidence that an earlier update was read.

Can an auditor reconstruct the decision without application logs scattered across three systems? If not, the notification pipeline is unfinished.

## How should urgent travel event notifications poll SMS delivery?

The first clock is the **attention deadline**: the last moment when an SMS can still change what the family does. The second is the poll schedule: when a worker asks for a newer delivery state. They deserve different configuration values. Config bloat begins when one `retryDelay` quietly controls provider errors, status polling, and business fallback.

Map provider states into a small internal vocabulary: `pending`, `delivered`, `failed`, and `unknown`. Preserve the raw status beside it. Business policy should not depend on every spelling in an external enum. `Delivered` remains a network delivery signal. If the workflow requires human confirmation, add an explicit acknowledgement action and record it separately.

Retries need identities. Reusing one logical attempt key prevents a worker restart from producing duplicate SMS requests. The email fallback gets its own key because it is a distinct attempt. A database lease or compare-and-set transition should grant one worker the right to act; an in-memory flag will not survive a restart.

Keep polling bounded. A policy might poll at 15, 45, and 105 seconds after acceptance, then evaluate the deadline. Those numbers are policy examples, not delivery guarantees. Benchmark them against the trip operator's escalation window and status cadence observed in test traffic. Three reads are easy to reason about. Thirty usually mean the state machine lacks a real deadline.

Stop there.

## A small TypeScript state machine

The adapters stay narrow. They expose only what the workflow needs, while evidence construction remains inside the application boundary. An injected clock makes deadline tests deterministic.

```ts
type Channel = "sms" | "email";
type DeliveryState = "pending" | "delivered" | "failed" | "unknown";

type Receipt = {
  providerId: string;
  state: DeliveryState;
  rawState: string;
  observedAt: string;
};

type Alert = {
  alertId: string;
  tripId: string;
  phone: string;
  email: string;
  summary: string;
  report: Uint8Array;
  reportSha256: string;
  attentionDeadline: string;
};

interface SmsPort {
  send(input: { to: string; body: string; idempotencyKey: string }): Promise<Receipt>;
  inspect(providerId: string): Promise<Receipt>;
}

interface EmailPort {
  send(input: {
    to: string;
    subject: string;
    body: string;
    attachment: { filename: string; contentType: "application/pdf"; bytes: Uint8Array };
    idempotencyKey: string;
  }): Promise<Receipt>;
}

interface EvidenceStore {
  append(event: {
    alertId: string;
    channel: Channel;
    action: string;
    receipt: Receipt;
    reportSha256?: string;
  }): Promise<void>;
}
```

The orchestration is deliberately blunt. Good. Hidden cleverness is expensive during an incident review.

```ts
const pollOffsetsMs = [15_000, 45_000, 105_000] as const;

async function notify(
  alert: Alert,
  deps: {
    sms: SmsPort;
    email: EmailPort;
    evidence: EvidenceStore;
    now: () => Date;
    waitUntil: (time: Date) => Promise<void>;
  },
): Promise<"sms-delivered" | "email-fallback"> {
  const startedAt = deps.now();
  let receipt = await deps.sms.send({
    to: alert.phone,
    body: alert.summary,
    idempotencyKey: `${alert.alertId}:sms`,
  });
  await deps.evidence.append({ alertId: alert.alertId, channel: "sms", action: "submitted", receipt });

  for (const offset of pollOffsetsMs) {
    const pollAt = new Date(startedAt.getTime() + offset);
    if (pollAt >= new Date(alert.attentionDeadline)) break;
    await deps.waitUntil(pollAt);
    receipt = await deps.sms.inspect(receipt.providerId);
    await deps.evidence.append({ alertId: alert.alertId, channel: "sms", action: "polled", receipt });
    if (receipt.state === "delivered") return "sms-delivered";
    if (receipt.state === "failed") break;
  }

  const emailReceipt = await deps.email.send({
    to: alert.email,
    subject: `School trip update ${alert.tripId}`,
    body: alert.summary,
    attachment: {
      filename: `trip-${alert.tripId}-update.pdf`,
      contentType: "application/pdf",
      bytes: alert.report,
    },
    idempotencyKey: `${alert.alertId}:email-fallback`,
  });
  await deps.evidence.append({
    alertId: alert.alertId,
    channel: "email",
    action: "fallback-submitted",
    receipt: emailReceipt,
    reportSha256: alert.reportSha256,
  });
  return "email-fallback";
}
```

This code does not retry arbitrary exceptions. An adapter should classify a timeout with an unknown outcome differently from a confirmed rejection. After a timeout, query by attempt identity before sending again. Blind resubmission is how one itinerary change becomes four texts.

The limitation is deliberate: this state machine coordinates channel attempts, but it cannot establish that a human understood the change. It is also a poor fit for a document-led notice with no short-lived action window, or for a destination where the school lacks valid consent for SMS. In those cases, start with email or require an explicit acknowledgement workflow. The trade-off is extra workflow state and recipient friction in exchange for evidence of a human action.

## Test decisions and evidence together

A happy-path unit test is cheap and insufficient. Exercise delivered-before-deadline, terminal failure, status remaining pending, deadline already passed, inspection timeout with unknown outcome, worker restart after SMS submission, concurrent workers, and email submission failure. Verify the event sequence as well as the return value.

Attachment tests deserve byte-level checks. Assert the PDF content type, filename, stored SHA-256 value, and event revision used to generate it. MIME defines body parts and content types in Internet messages; the mail adapter should construct a valid multipart message and avoid logging attachment bytes.

I benchmark notification code in two layers: adapter latency separately from end-to-end time-to-first-attempt. Mixing them hides queue delay. Record counts for attempts, transitions, fallback decisions, and unknown outcomes, plus latency distributions for queue age and provider calls. Avoid recipient addresses and message bodies in metric labels. They create privacy exposure and ruin cardinality.

The release check is practical: replay contract fixtures for every raw provider state being mapped. A new external status should become `unknown`, generate an observable event, and wait for an explicit mapping change. Never guess it into `delivered`.

## When is the runner-up better?

Email first is the runner-up when the attachment is the primary object and urgency is measured in hours rather than minutes. A generated assessment report needs layout, context, and a durable file more than a terse interruption. Send one email, then use SMS only under a later escalation policy if acknowledgement is required.

Sending both channels immediately fits only when policy says the missed-deadline cost outweighs duplicate attention and the recipient has consented to both destinations. It produces broader attempt coverage but noisier evidence: two concurrent timelines can obscure which message triggered a response. Record correlation IDs and acknowledgement independently if this mode is necessary.

The decision rule is narrow. Choose SMS-first fallback for a time-sensitive change with a short summary, a valid mobile destination, and a report that supplies supporting detail. Choose email-first for document-led communication. In either case, compliance evidence comes from immutable inputs and explicit transitions, not from calling an accepted request “delivered.”

## Further reading

- RFC 8058, Signaling One-Click Functionality for List Email Headers: https://datatracker.ietf.org/doc/html/rfc8058
- RFC 2045, Multipurpose Internet Mail Extensions Part One: https://datatracker.ietf.org/doc/html/rfc2045
- RFC 3464, An Extensible Message Format for Delivery Status Notifications: https://datatracker.ietf.org/doc/html/rfc3464
- Twilio SMS documentation, one example of provider delivery-status documentation: https://www.twilio.com/docs/sms
