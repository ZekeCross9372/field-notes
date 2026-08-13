# Catalog Gateway API: 5 Node.js One-Key Rate-Limit Fallback Tests

Short answer: choose the gateway shape only after a replay proves that fallback preserves required catalog fields and finishes before the enrichment deadline. One key is useful. It isn't the decision.

| Catalog condition | Default path | Evidence required before launch |
| --- | --- | --- |
| Stable schema, mixed model providers | Gateway API | Field scores, latency percentiles, route reason, region |
| Native model behavior defines the feature | Direct provider API | Native response tests and a separately owned adapter |
| Enrichment can wait | Queued gateway calls | Bounded concurrency and replayable records |
| Human review is immediate | Low-latency route | Explicit abstention and no hidden quality downgrade |

For a B2B catalog, the practical recommendation is a thin Node.js adapter around a gateway, backed by a queue and a golden replay set. Treat OpenAI, Claude, and Gemini as interchangeable only for fields where your own tests show they are interchangeable. The runner-up is direct integration, and it wins when provider-specific behavior matters more than a common surface.

## 1. Which descriptions are safe to admit into the enrichment queue?

The description is the input, but the field is the unit of failure. “Blue cotton shirt, two pack” can produce a correct color and material while still losing the pack count. A single pass/fail score hides that damage. Before comparing gateway APIs, define acceptance separately for brand, SKU, dimensions, quantity, material, and abstention. Exact strings can use exact matching; normalized units need canonical comparison; free-form categories need a reviewed rubric.

This changes the first engineering task. A long model catalog doesn't help if fallback silently changes a required field. Define the maximum tolerated score drop for each route transition, then set a deadline for the full enrichment job. Quality comes first for durable facts such as SKU and pack size. Latency can dominate for suggestions that a user will immediately review.

I benchmark both axes because a fast wrong attribute is still wrong. No mystery metric.

The fixture set should contain scrubbed examples from the actual feed: absent units, duplicated identifiers, comma decimal separators, two plausible brands, and descriptions where “12” could mean size or count. Keep expected abstentions in the set. Don't reward a model for guessing. This is also the first anti-bloat test: if a gateway needs a large configuration tree before it can submit the same fixture to two routes, time-to-first-call is already telling you something about its operational cost.

## 2. How should one gateway API enforce rate limits during fallback routing?

At the worker boundary, use a replay harness that treats transport results and catalog results as different measurements. The transport report records status, elapsed time, attempt count, and the gateway's route metadata. The catalog scorer records field correctness. Run the same frozen inputs against the primary route, the fallback route, and the ordered policy. Then inject a documented `429 Too Many Requests` response in the test double and verify that retry timing honors `Retry-After` when the response includes it, as specified for HTTP 429.

I want the failure to be boring: one injected 429, no duplicate catalog write, and a route reason I can inspect. A green aggregate success rate can't prove any of those properties.

```ts
type CatalogItem = {
  id: string;
  description: string;
};

type Enrichment = {
  brand: string | null;
  color: string | null;
  packCount: number | null;
};

type GatewayReply = {
  output: Enrichment;
  route: string;
  reason: string;
};

async function enrich(item: CatalogItem, signal: AbortSignal): Promise<GatewayReply> {
  const startedAt = performance.now();
  const response = await fetch(process.env.AI_GATEWAY_URL!, {
    method: "POST",
    headers: {
      authorization: `Bearer ${process.env.AI_GATEWAY_KEY}`,
      "content-type": "application/json",
      "x-request-id": item.id
    },
    body: JSON.stringify({
      task: "catalog-enrichment",
      input: item.description
    }),
    signal
  });

  if (response.status === 429) {
    const retryAfter = response.headers.get("retry-after");
    throw new Error(`rate_limit retry_after=${retryAfter ?? "unspecified"}`);
  }
  if (!response.ok) {
    throw new Error(`request_failed status=${response.status}`);
  }

  const reply = (await response.json()) as GatewayReply;
  console.info({
    itemId: item.id,
    elapsedMs: Math.round(performance.now() - startedAt),
    route: reply.route,
    reason: reply.reason
  });
  return reply;
}
```

The endpoint remains an environment value because APIs differ; the example invents no universal route. The adapter is intentionally small — plain HTTP, one credential at the application boundary, a request ID, a cancellation signal, and response metadata. That makes it possible to replace the gateway without leaking its client library through the catalog service.

Rate limits need two tests. First, verify that one busy tenant cannot consume every worker slot. Second, verify that a fallback doesn't multiply traffic beyond the attempt budget. Put jittered backoff in the queue worker, bound concurrency by route, and make the final catalog write idempotent on the item ID plus enrichment version. A gateway may centralize credentials, but your application still owns queue pressure and duplicate prevention.

## 3. Where did each proposed catalog attribute come from?

Transport recovery answers “can another route complete this request?” Quality fallback answers “may another route produce this field?” Those are not the same policy. A timeout before any response may be retryable. Invalid input is not. A completed response with a missing optional color may be acceptable, while a missing SKU should go to review rather than cascade through every available model. Consider a description containing “ACME 12 red”: one route can interpret 12 as pack count, another as size, and both responses can be valid JSON. The write path must retain the input version, schema version, selected route, route reason, and validation outcome so a later correction can target that attribute instead of overwriting the whole product. This is data lineage, not gateway telemetry dressed up with a nicer name.

Write an ordered rule table in application terms: retryable transport class, maximum attempts, time remaining, fields allowed to degrade, and terminal action. Store the chosen route and reason beside the enrichment version. Without that record, an operator cannot distinguish a source-description change from a routing change when an attribute moves.

Keep the attempt count low.

Fallback also interacts with streaming and tool semantics. A common gateway surface is not suitable when the product depends on provider-native event ordering, safety controls, or another response feature the abstraction cannot preserve. I'm not sure a paper comparison can settle that boundary; a captured-response contract test will. Your mileage may vary as provider contracts and account capabilities change, so rerun the contract suite instead of freezing assumptions in a spreadsheet.

## 4. Can a regional queue preserve the record's processing boundary?

“EU” in a control panel is not enough evidence for a catalog containing supplier data. For every candidate path, document where request content, response content, logs, queue payloads, and dead-letter records are processed and retained. The gateway, worker, observability stack, and backup policy all count. Ask for current contractual and technical documentation, then make deployment configuration reviewable in code.

Run synthetic catalog records through each permitted region and confirm that returned route metadata matches the intended policy. Keep region-specific queues when unfinished work must not cross a boundary. Redact descriptions before logs, preserve the request ID, and record the policy version rather than the raw prompt. This test is less glamorous than model sampling, but it catches the glue that dashboards omit.

Simple setup has a measurable definition here: a fresh service can make its first authenticated call with plain HTTP, and an engineer can trace that call through the queue without installing provider-specific SDKs. One key reduces secret distribution across the application. It does not collapse upstream quotas, contractual terms, or regional obligations into one universal rule.

## 5. When is a shadow route ready to write catalog data?

Replay is necessary, but promotion needs production-shaped input. Send a scrubbed sample through a candidate route without committing its attributes, compare the result with the active path, and promote it only after field acceptance and elapsed-time thresholds hold for the agreed observation window. The sample must preserve the ugly distribution of the live feed; a tidy demo set makes shadow traffic theater. Record the schema and policy versions with every comparison so a prompt edit cannot masquerade as a route improvement.

The catch is that a gateway adds a policy boundary and another schema to maintain. Keep a direct adapter as the runner-up when one native capability determines catalog quality, legal review requires a separately controlled vendor boundary, or the workload uses one provider and has no credible fallback need. A direct path also fits when route metadata cannot explain an attribute change. Keep the business schema outside either transport client. That is the exit door — and good DX needs one.

## References

- https://www.rfc-editor.org/rfc/rfc6585#section-4
- https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429
- https://docs.cohere.com/docs/rerank-overview
- https://elevenlabs.io/docs
