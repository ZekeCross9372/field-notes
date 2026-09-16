# DMARC Enforcement: How to Progress TXT Policy from Quarantine to Reject

Short answer: publish `p=none` first, move to `p=quarantine` only after aggregate reports account for every legitimate sender, and use `p=reject` only when the same evidence stays clean after the new TXT record is actually observable through DNS.

| Choice | Use it when | Main risk | Exit signal |
|---|---|---|---|
| `p=none` | Sender inventory is incomplete or the DNS move is still being observed | Failing mail is reported but not dispositioned by DMARC | Every legitimate stream is aligned or deliberately excluded |
| `p=quarantine` | The inventory is credible, but false-positive impact still needs containment | Legitimate failures may be treated as suspicious | Aggregate reports show no unexplained legitimate failures |
| `p=reject` | Authentication and alignment are stable across known streams | A missed sender can lose delivery | Continued clean evidence after the enforcement record is visible |

For a B2B SaaS team moving zones away from a registrar-specific API, the first choice is not quarantine versus reject. It is evidence versus hope. Keep policy at `none` during the DNS cutover, record what resolvers can see, and separate propagation delay from mail-authentication failure before tightening disposition.

## What should a DMARC policy rollout monitor before quarantine or reject?

Monitor two different systems: DNS publication and authenticated mail. They fail on different clocks. A control plane may accept a TXT update while the resolver used by an observer still returns the previous value; meanwhile, aggregate DMARC data describes messages that receivers already evaluated. Mixing those signals can make a healthy sender look late or a stale policy look current.

Start with the DMARC record at `_dmarc.<domain>`. RFC 7489 defines `p=none`, `p=quarantine`, and `p=reject`, plus `rua` for aggregate-report destinations. It also defines `pct` as the percentage of messages to which the requested policy applies, with a default of 100. Those are controls, not proof that a rollout is ready.

The proof is narrower: all legitimate mail sources pass DMARC through aligned SPF or aligned DKIM, and the published policy observed by your checks matches the intended stage. DMARC alignment compares the authenticated identifier with the domain in the visible `From` header. A message can pass SPF or DKIM yet still fail DMARC when that identifier is not aligned. This is the quiet failure mode worth chasing.

Don't infer readiness from a single successful test message. A SaaS domain may send account mail, support replies, billing notices, and messages through customer-configured integrations. Low-frequency streams are easy to miss. The exact observation window depends on the business sending cycle, and I'm not sure a generic number can be honest here; close a complete billing, support, and notification cycle instead of borrowing somebody else's day count.

Keep the gates explicit:

1. DNS observation returns one syntactically valid DMARC record with the intended policy.
2. Aggregate reports cover the known sending inventory.
3. Legitimate sources pass through aligned SPF or DKIM.
4. Unknown failures are classified before enforcement grows.
5. The rollback record and the person authorized to publish it are known.

That's the whole control loop.

## Gate policy changes on evidence, not a deployment timestamp

The primary tension is propagation delay versus cutover speed. A fast control-plane write is useful, but it does not establish what a recursive lookup returns. During a zone migration, run the same record check from the deployment job and from independent operational probes. Store the returned TXT value, observation time, domain, and intended stage. If the values disagree, hold the policy stage. Do not interpret the disagreement as an authentication regression.

Treat `pct` carefully. RFC 7489 allows it to stage application of `quarantine` or `reject`, but the RFC also says receivers should apply a policy one step below the requested policy to messages outside the sampled percentage. That means `p=reject; pct=10` is not equivalent to “90 percent monitoring”; the remaining messages can be subjected to quarantine. The tag is a blast-radius control, not a substitute for sender discovery.

My decision rule is deliberately boring — the kind of boring that keeps DNS config small. Move from `none` to `quarantine` after the sender map is complete and unexplained failures are resolved. Increase enforcement only after observing the intended record and reviewing fresh aggregate evidence. Move to `reject` after legitimate traffic remains aligned under quarantine. If any gate fails, retain the current record while investigating; do not combine a zone-provider cutover and a stricter DMARC disposition into one opaque change.

There is no universal failure-rate threshold in RFC 7489. Teams have different mail volumes and different tolerance for delayed or rejected messages, so an absolute percentage can hide a single critical stream inside a large denominator. For a B2B SaaS service, classify by stream and owner first. One failing password-reset source matters even if the overall ratio looks tiny.

## Implement a small TXT progression check

This TypeScript script reads the DMARC TXT record through the configured recursive resolver and validates the only forward sequence this rollout permits. It has no SDK and no provider-specific configuration. Run it before changing policy and again from the deployment environment after publication.

```ts
import { resolveTxt } from "node:dns/promises";

type Policy = "none" | "quarantine" | "reject";

const order: Record<Policy, number> = {
  none: 0,
  quarantine: 1,
  reject: 2,
};

function parsePolicy(chunks: string[][]): Policy {
  const records = chunks
    .map((parts) => parts.join(""))
    .filter((value) => /^v=DMARC1(?:;|$)/i.test(value.trim()));

  if (records.length !== 1) {
    throw new Error(`Expected one DMARC record, found ${records.length}`);
  }

  const match = records[0].match(/(?:^|;)\s*p=(none|quarantine|reject)(?:;|$)/i);
  if (!match) throw new Error("DMARC record has no recognized p tag");
  return match[1].toLowerCase() as Policy;
}

async function verifyStage(domain: string, expected: Policy): Promise<void> {
  const observed = parsePolicy(await resolveTxt(`_dmarc.${domain}`));
  if (observed !== expected) {
    throw new Error(`Expected p=${expected}, observed p=${observed}`);
  }
  process.stdout.write(`Observed p=${observed} for ${domain}\n`);
}

function assertNextPolicy(current: Policy, next: Policy): void {
  const step = order[next] - order[current];
  if (step !== 1) {
    throw new Error(`Refusing policy jump from ${current} to ${next}`);
  }
}

const [domain, current, next] = process.argv.slice(2) as [string, Policy, Policy];
if (!domain || !(current in order) || !(next in order)) {
  throw new Error("Usage: tsx check-dmarc.ts <domain> <current> <next>");
}

assertNextPolicy(current, next);
await verifyStage(domain, current);
```

For example, the pre-change check for the first enforcement stage is:

```bash
npx tsx check-dmarc.ts customer.example none quarantine
```

The script intentionally refuses `none` to `reject`. That restriction is an operational decision, not a requirement imposed by the DMARC grammar. It makes the risky path visible in code review and prevents the DNS migration tool from becoming a hidden enforcement switch.

The check is also intentionally small. It does not pretend that DNS equality proves mail readiness. Feed aggregate reports into a separate inventory keyed by header `From` domain, source, authentication result, and alignment result. XML report handling needs limits on document size and entity processing, since RFC 7489 warns that report consumers must protect themselves against abuse. Keep raw report ingestion away from the credentialed zone-writing process. Fewer privileges. Less glue.

## When should the runner-up policy remain in place?

Stick with `p=none` when a domain still has unowned senders, when a full business sending cycle has not been observed, or when the DNS migration cannot yet produce consistent policy observations. Monitoring is also the right choice when the organization wants visibility but has not accepted receiver disposition of failing messages. The catch is direct: `none` requests no specific action against failing mail, so it does not provide the enforcement outcome of quarantine or reject.

Keep `p=quarantine` instead of advancing to `reject` when legitimate, business-critical mail still fails alignment or the team needs suspicious messages to remain recoverable through receiver handling. RFC 7489 describes quarantine as receiver treatment that may include spam-folder placement, while reject asks receivers to reject failing mail during SMTP. Receiver behavior can vary because DMARC expresses the domain owner's requested handling and preserves receiver discretion.

An immediate `reject` policy is the faster option only when the domain's owners already have strong evidence that every legitimate sender authenticates with alignment and they accept hard failure for anything else. It is not suitable as the first combined change in a registrar API migration. The rollback surface becomes ambiguous: mail evidence, policy publication, delegation, and application configuration all moved at once.

Speed still matters. Measure it at the gate that matters: how soon the intended record is observed and how soon representative aggregate evidence arrives. Don't count how quickly an API returned success. That number is attractive, easy to benchmark, and incomplete.

## References

- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance (DMARC): https://datatracker.ietf.org/doc/html/rfc7489
