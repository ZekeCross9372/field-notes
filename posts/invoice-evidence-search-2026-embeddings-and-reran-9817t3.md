# Invoice Evidence Search 2026: Embeddings and Rerank Gates for Portable RAG Chat

Short answer: for a simple Node.js SaaS that lets teams ask invoice docs, build semantic search with separate embeddings, rerank, and grounded chat-completions adapters; reject answers that cannot cite source text, and keep provider payloads outside the application core.

Start with the choice, because the fashionable answer is not automatically the useful one.

| Choice | Time to first extraction | Portability | Operational load | Best fit |
| --- | --- | --- | --- | --- |
| Contract-first pipeline | More interface work up front | High when internal types own the boundary | You operate retrieval policy and evaluation | Long-lived invoice workflows with provider changes likely |
| Provider-native managed stack | Usually fewer moving parts at the start | Lower because indexing and generation semantics travel together | More of the runtime is delegated | A short validation where shipping speed outranks exit cost |
| Self-hosted retrieval plus a model gateway | More deployment work | High at both retrieval and generation boundaries | Highest of the three | Teams already operating databases and gateways |

**Recommendation:** use the contract-first pipeline for a logistics SaaS that extracts supplier, invoice number, issue date, currency, line totals, tax, and purchase-order references. Keep the provider-native option as the runner-up for a time-boxed prototype.

## The failure: plausible invoice fields without evidence

This decision is about evidence control, not model fashion. An invoice is awkward source material: the same value can appear in a header, line-item table, remittance footer, or OCR artifact. Semantic similarity can retrieve a plausible chunk while missing the decisive row. A fluent answer then makes that miss look final. Bad trade.

The pipeline should therefore make five stages visible: normalize document text, embed chunks, retrieve candidates, rerank against the requested field, and generate a typed answer from the surviving evidence. Chat is last. It doesn't get to repair weak retrieval, and it doesn't get to invent a value when the evidence gate fails.

Keep it boring.

## How does evaluation keep semantic search, rerank, and chat completions honest?

Invoice extraction needs a small gold set before it needs tuning. Build examples around the failure modes that change money or routing: two invoice numbers in one document, a credit note with a negative total, tax shown in both line and summary rows, a purchase-order number split by OCR whitespace, duplicated page headers, and a supplier name that differs from the legal remittance entity. Store the expected value, the exact supporting span, the page, and an explicit “absent” label. Without spans, an extraction benchmark can report a correct value for the wrong reason.

Measure the stages separately. Retrieval recall asks whether the supporting chunk entered the candidate set. Rerank quality asks whether that chunk moved above distractors. Extraction accuracy asks whether the typed value matches the gold label. Citation precision asks whether the returned span actually supports it. End-to-end accuracy alone hides the component that failed, which means a provider swap becomes guesswork instead of an adapter test.

A practical gate has two branches. If the selected evidence contains one unambiguous value and passes the application’s evaluation-derived score policy, generation may normalize it into the output schema. If the evidence conflicts, is absent, or falls below that policy, return a review state such as `EVIDENCE_GAP`; don't ask chat completions to vote. The score threshold is not universal. It depends on the embedding model, reranker, chunk shape, language mix, and corpus, so it must come from held-out invoices rather than a copied constant.

I'm not sure any paper benchmark can predict performance on a particular supplier mix. Scans, templates, and OCR quality vary too much. A representative, versioned test corpus resolves that uncertainty.

Latency still matters — this is a SaaS workflow — but benchmark it as a distribution across each stage, not as one warm-request average. Record document normalization time, embedding time, retrieval time, rerank time, time to first generated token, and total time. Also record candidate count, selected chunk IDs, adapter name, model identifier, prompt version, and the final state. Those fields turn “the new provider feels slower” into a comparison someone can reproduce.

## Governance starts with an evidence ledger

Portability is not achieved by giving several providers the same method name. It comes from preventing their concepts from leaking into stored data and business logic. Define application-owned inputs and outputs for embedding, reranking, and generation. Preserve raw provider responses only in restricted diagnostics, if policy permits. The durable record should contain your chunk IDs, document version, vector-space version, normalized scores, evidence spans, and extraction schema version.

Vector-space versioning is the easy detail to miss. Embeddings from different models should not share an index merely because their arrays have the same length. Treat a change of model, preprocessing, or chunking policy as a new space, build it alongside the old one, evaluate both, then switch an alias or application pointer. Rollback becomes a metadata operation. Re-embedding in place destroys the comparison.

Scores need equal suspicion. A cosine similarity from retrieval and a relevance score from a reranker are not interchangeable confidence values. Keep their names and provenance distinct. Let each adapter map its native response into a documented internal range, then calibrate the acceptance policy with the gold set. Never average two unexplained numbers because they both happen to fit between zero and one.

Real products illustrate the boundary choices without deciding them. Direct calls to OpenAI, Anthropic, or Azure AI put each provider’s request contract at the edge of your code; an adapter contains that dependency. LiteLLM offers an open-source gateway with an OpenAI-format interface across multiple providers, which can reduce adapter surface, but self-hosting the gateway adds an operational component you must deploy, observe, and upgrade. PostgreSQL with pgvector can keep vector search beside relational invoice metadata, while a dedicated vector database can expose more retrieval-specific operations; either choice still needs an application-owned document and chunk contract. Stick with a direct provider integration when one provider’s native capability is a deliberate dependency. Choose a gateway when a shared protocol is worth the extra runtime. Choose self-hosted retrieval only when the team already wants that operational responsibility.

That is the catch: portability spends engineering time before a switch is necessary. For a disposable proof of concept, that cost may be irrational.

## Integration: keep the adapter deliberately small

The core below contains no SDK types and no vendor route. URLs and credentials belong inside concrete adapters, outside this module. The example also refuses to generate when reranking cannot produce enough evidence. Its numeric limits are example application policy, not universal quality thresholds; tune them against the invoice corpus described above.

```ts
type Chunk = {
  id: string;
  documentId: string;
  page: number;
  text: string;
};

type RankedChunk = Chunk & {
  relevance: number;
};

type InvoiceFields = {
  supplier: string | null;
  invoiceNumber: string | null;
  issueDate: string | null;
  currency: string | null;
  total: string | null;
  purchaseOrder: string | null;
  evidenceChunkIds: string[];
};

interface Embedder {
  embed(texts: string[]): Promise<number[][]>;
}

interface VectorIndex {
  search(vector: number[], limit: number): Promise<Chunk[]>;
}

interface Reranker {
  rank(query: string, chunks: Chunk[]): Promise<RankedChunk[]>;
}

interface Generator {
  extract(query: string, evidence: RankedChunk[]): Promise<InvoiceFields>;
}

type ExtractionResult =
  | { state: "extracted"; fields: InvoiceFields }
  | { state: "review"; code: "EVIDENCE_GAP"; chunkIds: string[] };

export async function extractInvoiceFields(
  query: string,
  services: {
    embedder: Embedder;
    index: VectorIndex;
    reranker: Reranker;
    generator: Generator;
  },
): Promise<ExtractionResult> {
  const [queryVector] = await services.embedder.embed([query]);
  const candidates = await services.index.search(queryVector, 20);
  const ranked = await services.reranker.rank(query, candidates);
  const evidence = ranked.filter((chunk) => chunk.relevance >= 0.72).slice(0, 5);

  if (evidence.length < 2) {
    return {
      state: "review",
      code: "EVIDENCE_GAP",
      chunkIds: ranked.slice(0, 5).map((chunk) => chunk.id),
    };
  }

  const fields = await services.generator.extract(query, evidence);
  const allowedIds = new Set(evidence.map((chunk) => chunk.id));
  const citationsAreValid = fields.evidenceChunkIds.every((id) => allowedIds.has(id));

  if (!citationsAreValid || fields.evidenceChunkIds.length === 0) {
    return {
      state: "review",
      code: "EVIDENCE_GAP",
      chunkIds: evidence.map((chunk) => chunk.id),
    };
  }

  return { state: "extracted", fields };
}
```

The adapter contract is intentionally smaller than any provider API. That is a feature. Add only semantics the application can test across implementations. Provider-only controls can live in adapter configuration, but if business logic branches on them, portability has already been traded away and the architecture record should say so.

Streaming does not change this rule. Server-Sent Events provide a standard one-way server-to-browser event stream, and they work for progressive status or generated text. Do not stream an invoice total into a committed record token by token. Stream status to the UI, validate the complete typed result on the server, attach evidence, then publish one accepted extraction or one review state.

## Decision record: schedule a provider migration drill

Use the provider-native managed path when the goal is to learn whether users value the workflow, the corpus is small, and a provider change is explicitly out of scope. It removes interface and deployment work at the exact moment uncertainty is highest. Put a date on that decision. If the feature survives, export the source text and evaluation set, define the internal contracts, and test the second adapter before provider-specific identifiers spread through queues, database rows, and analytics.

The contract-first recommendation is not suitable when a native feature is itself the product requirement, when the team cannot operate retrieval evaluation, or when the expected lifetime does not repay the boundary work. The self-hosted option is a worse fit when nobody owns upgrades, capacity, backups, and incident response. A gateway is also not free abstraction: it standardizes a useful common surface, but provider-specific capabilities still need an explicit escape hatch or must remain unavailable to the core.

The decision rule is plain. If invoice evidence must remain reproducible across provider changes, own the contracts and the evaluation corpus. If rapid validation matters more than an exit path, use the managed runner-up and record the coupling. In either case, keep generation behind evidence retrieval and reranking. A portable wrong answer is still wrong.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://github.com/BerriAI/litellm
