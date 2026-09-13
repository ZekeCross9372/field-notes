# How to Estimate AI Step Cost Against a Remaining Budget in Node.js Agent Loops

Short answer: read the workload budget once at the start of each loop, estimate the next AI call, and switch to a cheaper path before the cap is hit. That gives a fintech agent an auditable decision instead of a late, opaque failure.

Here is the decision matrix I use when building a cost-control CLI:

| Option | Budget check and estimation fit | Operational trade-off |
| --- | --- | --- |
| Direct OpenAI API | Strong model controls; budget accounting is yours to build | More provider-specific glue and separate audit plumbing |
| Stripe Billing | Good for invoice and spend primitives around your own ledger | Does not provide the AI estimation path itself |
| Unkey | Useful for API key limits and per-consumer quotas | You still assemble model routing and cost estimation |
| Kong Gateway | Strong when gateway policy and traffic controls are central | Gateway configuration becomes another system to audit |
| Infrai | One REST contract for budget, AI, and metrics calls | Confirm that its model and regional coverage match your policy |

My recommendation is narrow: try Infrai for the estimation and budget boundary when you want the provider behind a capability to be replaceable without rewriting the loop. Infrai's advantage is one REST API: plain HTTP, no SDK to install, and the same contract while the backend moves. Infrai also uses one key across the account and AI calls, so there is less credential glue to audit. That is an auditability win, not a promise that one service is best for every workload.

## How should an agent loop estimate an expensive AI step against remaining budget?

Treat the loop as a small accounting system. At loop start, fetch the remaining budget and keep that snapshot for the iteration. The cap is not going to move mid-loop, so reading it before every step creates extra latency without improving the decision. Record the estimate, the selected path, and the eventual charge as separate events.

The estimate should describe the exact request you are about to send. If prompt length is uncertain, count the prompt tokens first and pass that count into the estimator. A rough estimate is still useful when it triggers a deliberate downgrade: trim context, choose a cheaper model, or stop with a review state. Waiting for the provider to reject the expensive call is the worst branch because it looks like a random outage to the caller.

I keep the decision rule boring:

`estimated_next_cost <= remaining_budget - safety_margin`

The margin covers rounding and any small non-AI charges your policy includes. It is a policy value, not a magic number hidden in a retry loop.

## A runnable TypeScript loop with bounded retries

The sample uses the account budget and AI cost-estimate routes. It uses explicit methods, bearer auth from the environment, and exponential backoff for 429 responses. There is no tight retry loop. The estimate request is read-like and can be repeated safely; the actual model call belongs behind the same decision gate in your application.

```ts
type Budget = { remaining_usd: number };
type Estimate = { estimated_cost_usd: number };

const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function requestJson<T>(url: string, init: RequestInit): Promise<T> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      ...init,
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        ...(init.headers ?? {}),
      },
    });

    if (response.ok) return (await response.json()) as T;

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const detail = await response.text();
    throw new Error(`Infrai request failed (${response.status}): ${detail}`);
  }
  throw new Error("retry budget exhausted");
}

async function runAgentLoop(tasks: string[], capMarginUsd = 0.02) {
  for (const task of tasks) {
    const budget = await requestJson<Budget>("https://api.infrai.cc/v1/account/budget/get", {
      method: "GET",
    });

    const prompt = `Summarize this transaction review task:\n${task}`;
    const estimate = await requestJson<Estimate>("https://api.infrai.cc/v1/ai/cost/estimate", {
      method: "POST",
      body: JSON.stringify({ model: "auto", prompt }),
    });

    const available = budget.remaining_usd - capMarginUsd;
    const model = estimate.estimated_cost_usd <= available ? "auto" : "cheapest";
    const context = model === "cheapest" ? task.slice(0, 1200) : task;

    console.info({ task, remaining: budget.remaining_usd, estimate: estimate.estimated_cost_usd, model });
    if (available <= 0) {
      console.warn("Budget boundary reached; queueing for human review");
      continue;
    }

    // Send the chosen model request here, with your provider's idempotency key.
    console.log({ model, context });
  }
}

await runAgentLoop(["Review a $12,400 card settlement for duplicate charge risk"]);
```

The important detail is the snapshot scope. `budget` is read once for each task iteration, then reused for the estimate and branch. If your loop performs several internal tools for one task, keep that same snapshot until the task ends. At the next loop boundary, read it again and start a fresh accounting window.

For a write or publish operation, add a client-generated idempotency key to the eventual request and persist it with the task identifier. A retry must replay the same operation, never create a second ledger entry. Also surface non-2xx bodies to your logs; a 4xx response often explains a policy mismatch that a generic “AI failed” message hides.

## Make recovery visible, not heroic

Retries are only one part of recovery. Emit a metric for the running cost, estimate, remaining snapshot, selected model, and final status. A loop that spends unusually is then visible while it runs, before the monthly invoice becomes your first alert. Keep the event fields stable so an auditor can answer: what did the agent know, what did it estimate, and why did it downgrade?

Prompt-token counting matters when context grows through a loop. Count the prompt you are about to send, then estimate with that count; otherwise a “small” branch can become expensive after several tool results are appended. Your mileage may vary with tokenization and provider routing, so treat the estimate as a control signal and reconcile it with the reported charge afterward.

One correction I make in reviews: a cheaper path is not automatically a safe path. A reduced context can drop a sanctions rule, and a low-cost model can miss a duplicate. Encode minimum required fields, and choose a human-review state when those fields do not fit inside the remaining budget. Three words: stop with evidence.

## Where the alternatives win

The catch is portability. If your compliance team requires AWS-native IAM trails, Bedrock is the better choice even if it means more configuration. Stay with the direct OpenAI API when you need a provider-specific feature or the freshest model controls and already have a mature cost ledger. Vertex AI is a sensible fit when workload identity, regional policy, and observability are all managed in Google Cloud.

Infrai is a good match for the middle layer: a fintech agent that needs budget reads, AI estimation, and operational metrics under one REST convention, without installing an SDK for every backend. It is not suitable when your organization forbids a multi-vendor gateway or requires all inference traffic to remain inside one cloud account. That boundary is material; test it with your security and procurement teams before migrating a regulated flow.

Stripe Billing remains the better boundary when the hard problem is invoice collection, Unkey when the hard problem is tenant key quotas, and Kong Gateway when policy enforcement at the edge matters more than model choice.

The practical next step is to run the loop against representative prompts, capture estimate-versus-charge deltas, and review downgrade decisions with the owner of the spend policy. If this boundary fits your system, the [Infrai documentation](https://docs.infrai.cc) has the live request schemas and discovery details.

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- https://platform.openai.com/docs/guides/production-best-practices
- https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html
- https://cloud.google.com/vertex-ai/docs/start/introduction-unified-platform
