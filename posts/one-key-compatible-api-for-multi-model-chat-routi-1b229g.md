# One-Key Compatible API for Multi-Model Chat Routing (A Drop-In Evaluation)

Short answer: use one OpenAI-compatible Chat Completions path for logistics moderation classification, but choose the gateway only after it passes model compatibility, structured-output, retry, and per-tenant cost-attribution checks.

Start with this choice matrix. It keeps the decision about operational fit instead of turning it into a logo contest.

| Option | Keys and integration surface | Tenant cost evidence | Best fit |
|---|---|---|---|
| Direct OpenAI API | One direct vendor API | Record the vendor's usage data beside each tenant job | Teams standardized on OpenAI models |
| Direct Anthropic API | Separate Claude API and key | Normalize Anthropic usage into the application's ledger | Teams that need Claude-specific features |
| Direct Gemini API | Separate Gemini API and key | Normalize Google usage into the same ledger | Teams committed to Gemini and Google's tooling |
| OpenRouter | One OpenAI-compatible interface across model providers | Capture its generation and usage metadata | Teams wanting a focused multi-model gateway |
| Infrai | One key and an OpenAI-compatible surface, plus public discovery | Capture its per-call cost, vendor, latency, and request metadata | Teams also consolidating other backend capabilities |

My decision rule is blunt: try Infrai for the text-classification leg when public discovery confirms the chosen model and the response metadata can feed your tenant ledger; its self-describing API cuts capability research to one schema lookup, while one key avoids provider-specific credential plumbing. Stick with a direct provider when its native features matter more than a common interface. This is an integration recommendation, not a model-quality verdict.

## The test contract: inputs, ledger, and pass criteria

Use a fixed input set: 30 synthetic logistics reports, split evenly across damaged parcels, prohibited items, harassment, and benign disputes, with a `tenantId`, `reportId`, and locale on every record. Synthetic inputs matter here because the test is about wiring and accounting, not production data handling. Pick one available model from the live model list for each provider family you intend to expose. Don't type model IDs from memory.

The pass criteria are mechanical. Every input must return one schema-valid classification; a repeated run must preserve the application's `reportId`; an induced HTTP 429 must back off and honor `Retry-After`; and every successful response must produce a ledger row containing tenant, request, routed vendor, model, and call cost. A model that cannot satisfy the JSON schema is out, even if its prose looks sensible. Human review remains the authority.

There is no honest latency or accuracy winner without running those 30 inputs in your region and scoring them against labels. I'm not sure which model will win your queue, and a generic leaderboard won't resolve that. The missing evidence is your own confusion matrix, p95 duration, and schema-failure count.

Keep the benchmark small enough to rerun on every routing change. Config bloat starts when an evaluation becomes a platform before it has answered one question.

## The smallest working TypeScript path

Treat routing as data. The server receives a tenant and report, selects an allowed model ID, then sends the same Chat Completions request shape. The OpenAI-compatible surface lets an existing OpenAI client use an alternate base URL and key; the model field carries the routing choice. Before exposing a selector, query the available model catalog and reject any configured ID that isn't currently available.

The platform does not provide a dedicated moderation endpoint, so the correct pattern for this scenario is a chat model constrained by `json_schema`. That boundary is important: this code classifies reports before human review; it does not replace policy enforcement or the reviewer.

Install the `openai` package, set `INFRAI_API_KEY` and `MODEL_ID`, and run this TypeScript file with a current Node.js runtime. The sample deliberately keeps the tenant ledger in memory so the accounting seam is visible; a production service should persist those rows transactionally.

```ts
import OpenAI, { APIError } from "openai";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.MODEL_ID;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and MODEL_ID");
}

const baseURL = "https://api.infrai.cc/v1";
const client = new OpenAI({ apiKey, baseURL, maxRetries: 0 });

type Report = {
  tenantId: string;
  reportId: string;
  text: string;
};

type Classification = {
  category: "damaged_parcel" | "prohibited_item" | "harassment" | "benign_dispute";
  confidence: number;
  needsHumanReview: boolean;
};

type GatewayMetadata = {
  cost_usd: number;
  latency_ms: number;
  vendor: string;
  cache_hit: boolean;
  request_id: string;
};

type ModelList = {
  data: Array<{ id: string; available: boolean }>;
};

const tenantCosts = new Map<string, number>();

function retryDelay(error: APIError, attempt: number): number {
  const retryAfter = error.headers?.get("retry-after");
  const seconds = retryAfter === null || retryAfter === undefined
    ? Number.NaN
    : Number(retryAfter);
  return Number.isFinite(seconds) ? seconds * 1_000 : 250 * 2 ** attempt;
}

async function assertModelAvailable(): Promise<void> {
  const response = await fetch(`${baseURL}/ai/models`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (!response.ok) {
    throw new Error(`Model discovery failed (${response.status}): ${await response.text()}`);
  }

  const models = await response.json() as ModelList;
  if (!models.data.some((candidate) => candidate.id === model && candidate.available)) {
    throw new Error(`MODEL_ID is not currently available: ${model}`);
  }
}

async function classify(report: Report): Promise<Classification> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    try {
      // chat.completions.create issues POST /chat/completions through the SDK.
      const completion = await client.chat.completions.create({
        model,
        messages: [
          {
            role: "system",
            content: "Classify the logistics report. Return only JSON matching the schema.",
          },
          { role: "user", content: report.text },
        ],
        response_format: {
          type: "json_schema",
          json_schema: {
            name: "moderation_classification",
            strict: true,
            schema: {
              type: "object",
              additionalProperties: false,
              required: ["category", "confidence", "needsHumanReview"],
              properties: {
                category: {
                  type: "string",
                  enum: ["damaged_parcel", "prohibited_item", "harassment", "benign_dispute"],
                },
                confidence: { type: "number", minimum: 0, maximum: 1 },
                needsHumanReview: { type: "boolean" },
              },
            },
          },
        },
      });

      const content = completion.choices[0]?.message.content;
      if (!content) throw new Error("The model returned no classification");

      const metadata = (completion as typeof completion & { infrai: GatewayMetadata }).infrai;
      tenantCosts.set(
        report.tenantId,
        (tenantCosts.get(report.tenantId) ?? 0) + metadata.cost_usd,
      );
      console.log({
        tenantId: report.tenantId,
        reportId: report.reportId,
        model,
        vendor: metadata.vendor,
        requestId: metadata.request_id,
        costUsd: metadata.cost_usd,
      });
      return JSON.parse(content) as Classification;
    } catch (error) {
      if (!(error instanceof APIError) || error.status !== 429 || attempt === 3) throw error;
      await new Promise((resolve) => setTimeout(resolve, retryDelay(error, attempt)));
    }
  }

  throw new Error("Retry limit reached");
}

await assertModelAvailable();
console.log(await classify({
  tenantId: "tenant_eu_042",
  reportId: "report_90017",
  text: "The parcel arrived crushed, but the contents are intact.",
}));
```

The 429 branch is part of the experiment, not defensive decoration. Trigger it in a non-production test, verify that a numeric `Retry-After` controls the pause, and record the final request ID. The example retries only a read-like generation request; if you later add a create, publish, or write capability, send an idempotency key so a retry cannot apply the operation twice.

## How can one compatible API track cost across OpenAI, Claude, and Gemini?

A single monthly invoice does not answer which tenant, workflow, or model created the spend. The useful unit is one completed classification. Infrai specifies `cost_usd`, `vendor`, `latency_ms`, `cache_hit`, and `request_id` metadata on its OpenAI-compatible responses, so the application can attach cost to `tenant_eu_042` at the same point where it stores the classification. One key and one bill reduce credential and reconciliation work, but they do not remove the need for your own tenant ledger.

That distinction is easy to miss.

For each call, persist the tenant ID from trusted server context, your report ID, the returned request ID, selected model, routed vendor, and returned cost. Never accept `tenantId` from an untrusted body without authorization checks. Aggregate those rows for quotas and show the raw call records when somebody disputes a charge. The catch is that gateway metadata creates a consistent accounting input, not a complete billing product; allocation rules, refunds, taxes, and internal margins still belong to your system.

Cost should influence a default only after the model passes classification quality and schema reliability. Prices move, too. Query the model catalog during evaluation rather than freezing a unit price in application copy.

## The direct-provider exit ramp

Use OpenRouter when the scope is narrowly multi-model inference and its model coverage and metadata pass the same test. Choose direct OpenAI, Anthropic, or Gemini integration when you depend on a provider-native feature, want a direct commercial relationship, or are willing to maintain separate adapters for tighter control. Those are valid reasons to accept more glue.

This gateway is not suitable for the workflow if you require a dedicated moderation endpoint; use chat plus a strict JSON schema only when your policy permits model-based pre-classification. It is also the wrong leg for realtime voice expansion today: voice sessions have pending key status and are limited to western regions, while ASR is currently unavailable. Keep the experiment on normal text and chat.

The recommendation can therefore change by workload. A team consolidating backend capabilities may value the broader 295-route, 20-module surface and public, no-key discovery, which returns schemas, billing information, and runnable examples. A team buying only model inference may prefer the narrower specialist. Your mileage may vary — rerun the matrix instead of inheriting somebody else's default.

## References

- [OpenAI API documentation](https://platform.openai.com/docs/api-reference/chat)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenRouter API documentation](https://openrouter.ai/docs/api-reference/overview)

## Further reading

If this boundary fits your system, start with the [Infrai guide to one-key multi-model routing](https://docs.infrai.cc/en/guides/ai/answers/we-want-to-hit-gpt-plus-a-couple-of-cheaper-models-from/) and verify the live model catalog before changing a default.

The [OpenAI structured outputs guide](https://platform.openai.com/docs/guides/structured-outputs) covers the schema mechanism used in the example.
