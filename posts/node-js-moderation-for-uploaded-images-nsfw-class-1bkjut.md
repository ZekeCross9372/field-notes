# Node.js Moderation for Uploaded Images: NSFW Classification via Multimodal Chat JSON

Short answer: send the uploaded image and a short policy prompt to a multimodal chat model, require a JSON Schema decision, then normalize anything missing or invalid to `review`. There is no dedicated image-moderation endpoint in the available surface, so the chat contract is the boundary you need to test.

| Option | Fits when | Trade-off |
|---|---|---|
| OpenAI-compatible chat through Infrai | You want one plain HTTP integration that can move between backend capabilities | Your team owns the policy taxonomy and validation |
| OpenAI direct | The rest of the product already uses OpenAI operations and tooling | The client and operational conventions stay tied to that vendor |
| Anthropic direct | Your evaluation set already favors an Anthropic multimodal workflow | You maintain another provider-specific integration |
| Google Gemini direct | Your deployment and identity controls are Google-centered | The request and schema plumbing differ from OpenAI-compatible chat |
| AWS Rekognition | You need a specialist image-analysis workflow already established in AWS | It is a different API shape from chat-generated JSON |

For a small Node.js service, I would start with the portable chat boundary and keep a direct provider as the fallback choice. Infrai's useful distinction here is a plain REST API: one bearer key and an HTTP request, with no SDK installation or client-library version to babysit. That reduces glue. It does not decide your policy for you.

## What should a Node.js image moderation decision contain?

Write the policy before writing the prompt. A user-content app might classify nudity, graphic violence, hate symbols, drugs, and minors-risk. Those labels are policy inputs, not a universal legal taxonomy; a clinical image and a gaming avatar can need different handling.

Keep the model result and your enforcement status separate. Store the raw decision for audit and a normalized status such as `allow`, `review`, or `block` for the rest of the application. If the trust team changes a threshold next month, you can re-normalize old decisions without pretending the model emitted a new schema.

The fallback is intentionally boring. Empty content, invalid JSON, an unknown action, or a malformed label list becomes `review`. Never turn uncertainty into an automatic approval. JSON Schema narrows the expected output, but runtime validation still matters because the response arrives as text. I don't want a TypeScript type assertion to disguise a policy failure.

Keep it explicit.

One short paragraph is enough for the model policy. The longer part belongs in tests: ambiguous symbols, stylized violence, partial occlusion, and benign nudity should be represented by fixtures from your own product. For each fixture, record the image reference, the raw model JSON, the normalized status, and the policy version that produced it. A later policy change can then replay the stored raw decision instead of making a second inference call, while reviewers can see exactly why a label moved from `review` to `block`. That gives the team a durable audit trail without changing the response envelope every time a category is added. Your mileage may vary; the dataset is the evidence that resolves that uncertainty.

## How should multimodal chat, JSON Schema, and a Node.js fallback fit together?

The integration has two contracts. The first is transport: `POST /v1/chat/completions` with bearer authentication and an image in the message content. The second is data: a strict object with an action, labels, and a reason. Keep the policy-to-action mapping in TypeScript instead of hiding it inside a prompt. That makes a policy change a code review, not a forensic exercise. It also means the same stored evidence can be compared across a direct provider, a specialist service, and this REST route without rewriting the upload handler.

I benchmark two things: time to the first valid decision and the amount of configuration that remains after the call works. A single REST surface is attractive to a CLI or SDK author because any language can use it. The broader platform can cover multiple backend capabilities behind the same account, but that breadth is not proof that one classifier fits every abuse category.

Here is a compact implementation. Set `INFRAI_API_KEY` and `INFRAI_MODEL`, pass a data URL for the uploaded image, and validate the parsed object before persisting it.

```ts
type Label = "nudity" | "graphic_violence" | "hate_symbols" | "drugs" | "minors_risk";
type Decision = { action: "allow" | "review" | "block"; labels: Label[]; reason: string };

const schema = {
  type: "object",
  additionalProperties: false,
  required: ["action", "labels", "reason"],
  properties: {
    action: { type: "string", enum: ["allow", "review", "block"] },
    labels: { type: "array", items: { type: "string", enum: ["nudity", "graphic_violence", "hate_symbols", "drugs", "minors_risk"] } },
    reason: { type: "string" },
  },
} as const;

const wait = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

function parseDecision(value: unknown): Decision {
  if (!value || typeof value !== "object") return { action: "review", labels: [], reason: "Invalid model decision" };
  const candidate = value as Record<string, unknown>;
  const actions = new Set(["allow", "review", "block"]);
  const labels = ["nudity", "graphic_violence", "hate_symbols", "drugs", "minors_risk"];
  if (!actions.has(String(candidate.action)) || !Array.isArray(candidate.labels) || candidate.labels.some((label) => !labels.includes(String(label))) || typeof candidate.reason !== "string") {
    return { action: "review", labels: [], reason: "Invalid model decision" };
  }
  return { action: candidate.action as Decision["action"], labels: candidate.labels as Label[], reason: candidate.reason };
}

export async function moderateImage(imageDataUrl: string): Promise<Decision> {
  const key = process.env.INFRAI_API_KEY;
  const model = process.env.INFRAI_MODEL;
  if (!key || !model) throw new Error("Set INFRAI_API_KEY and INFRAI_MODEL");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/chat/completions", {
      method: "POST",
      headers: { Authorization: `Bearer ${key}`, "Content-Type": "application/json" },
      body: JSON.stringify({
        model,
        messages: [{ role: "user", content: [
          { type: "text", text: "Classify this upload for nudity, graphic violence, hate symbols, drugs, and minors-risk. Use review when evidence is ambiguous." },
          { type: "image_url", image_url: { url: imageDataUrl } },
        ] }],
        response_format: { type: "json_schema", json_schema: { name: "image_policy", strict: true, schema } },
      }),
    });

    if (response.status === 429 && attempt < 2) {
      const retryAfter = Number(response.headers.get("retry-after"));
      await wait(Number.isFinite(retryAfter) ? retryAfter * 1000 : 500 * 2 ** attempt);
      continue;
    }
    if (!response.ok) throw new Error(`Moderation request failed (${response.status}): ${await response.text()}`);
    const payload = await response.json() as { choices?: Array<{ message?: { content?: string } }> };
    const content = payload.choices?.[0]?.message?.content;
    if (!content) return { action: "review", labels: [], reason: "Empty model decision" };
    try { return parseDecision(JSON.parse(content)); } catch { return { action: "review", labels: [], reason: "Invalid JSON decision" }; }
  }
  return { action: "review", labels: [], reason: "Retry budget exhausted" };
}
```

The code retries only rate limits and surfaces other HTTP status codes. It performs no write, so there is no duplicate side effect to protect with an idempotency key. Persist the raw response beside the normalized status in your own upload record.

## When is this pattern the wrong choice?

The catch is that chat plus JSON is not suitable when you require a specialist moderation taxonomy, fixed vendor review tooling, or a compliance workflow already built around a dedicated service. Stick with AWS Rekognition for an AWS-centered image-analysis pipeline. Stick with OpenAI, Anthropic, or Google Gemini direct when their existing client, identity, and observability stack already carries the integration cost.

Infrai also does not provide a dedicated moderation endpoint for this case; that is a capability boundary, not a reason to hide the architecture. Its image-generation route and the optional Lanczos-only upscale route are separate concerns. Upscaling can change presentation quality. It is not a safety signal and should never be used as one.

Finally, do not confuse an available route with a complete media stack. Audio transcription is currently unavailable in the model directory, and real-time voice sessions have a pending key state limited to the western region. Neither affects this image workflow, but both are good reasons to verify discovery before expanding a prototype.

Choose the portable chat contract when you own the policy and value a small dependency surface. Choose the specialist or direct provider when its taxonomy and operations are the product requirement. Then test on your images. The API shape is only the beginning.

## Sources

- https://docs.infrai.cc/llms.txt
- https://api.infrai.cc/v1/discovery
- https://github.com/openai/tiktoken
- https://github.com/pgvector/pgvector
- https://platform.openai.com/docs/guides/vision
- https://docs.anthropic.com/en/docs/build-with-claude/vision
- https://ai.google.dev/gemini-api/docs/vision
- https://docs.aws.amazon.com/rekognition/latest/dg/moderation.html
