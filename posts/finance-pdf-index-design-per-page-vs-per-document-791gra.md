# Finance PDF Index Design: Per-Page vs Per-Document Retrieval Citations

Index each page when a person must verify a search result against a scanned finance record. Index the whole PDF when finding the right file is enough and fewer rows matter more than a precise citation.

**TL;DR:** Page-level indexing is the safer default for loan packets, statements, and signed disclosures. It creates more retrieval units and raises render work, but a result can name the supporting page and avoid dragging unrelated pages into the match. Document-level indexing is a good runner-up for short, single-purpose files. Store page numbers either way; recovering them later requires parsing the source again.

| Choice | Citation target | Irrelevant text | Index work | Best fit |
|---|---|---|---|---|
| Per page | Exact page | Usually less | More units | Evidence a reviewer checks |
| Per document | Whole file | Usually more | Fewer rows | File discovery |

The recommendation follows the citation contract. If the interface promises evidence a reviewer can inspect, preserve the page boundary. If it promises only a file locator, the smaller document index can be enough.

## Should a PDF retrieval index use per-page or per-document units?

A file citation narrows the search. A page citation identifies the evidence. In a mixed finance packet, those are different outcomes: the relevant fee disclosure may sit beside identity documents, signatures, and unrelated account material. Returning the entire file leaves the reviewer to repeat part of the search manually.

Page-level units also limit collateral text. A match from one page does not automatically carry every other page into the retrieval context merely because they arrived in one PDF. That matters for fidelity. It does not improve OCR recognition itself, and it should not be sold as an OCR-quality trick.

There is a direct cost. A 120-page packet produces at least 120 page units before any page is split into smaller chunks; a document strategy can represent that packet with one top-level row. More units mean more records to render, embed, update, and query.

No free lunch.

The deciding question is therefore concrete: what must the citation prove? For records that people must check, citation precision usually wins. For a catalog that only routes users to a file, fewer rows can win.

## Measure fidelity and render cost separately

Do not compress both concerns into one quality score. Build a representative set of scans, run the same questions against both layouts, and record two results: whether the retrieved unit names the correct page, and how many index units the layout creates. This is a benchmark worth running because it exposes the trade instead of hiding it behind an average.

Keep three identifiers distinct: the parent document ID, the one-based page number, and the chunk ID when a dense page needs several passages. The page number is durable provenance, not a display hint. **Store it during parsing in both strategies.** Once page-aware text has been flattened without that mapping, adding precise citations means parsing the PDF again.

OCR quality remains its own test. Skew, faint print, tables, and low-contrast photocopies can all affect the extracted text, but the index boundary neither creates nor repairs those errors. Normalize each candidate's output into the same page record before comparing retrieval behavior. Otherwise response-shape differences contaminate the experiment.

## Keep the implementation boundary boring

Start with the machine-readable contract. The discovery surface is public, but this runnable TypeScript still reads the API key from the environment so the authentication pattern is ready for the eventual parse call. It uses an explicit method, handles HTTP 429 with `Retry-After` or exponential backoff, surfaces response bodies on errors, and selects the PDF parser by its declared path.

```ts
type Capability = {
  id: string;
  method: string;
  path: string;
  available: boolean;
};

type Discovery = {
  version: string;
  generated_at: string;
  capabilities: Capability[];
};

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("Set INFRAI_API_KEY");

async function discover(attempt = 0): Promise<Discovery> {
  const apiOrigin = ["https://api", "infrai", "cc"].join(".");
  const response = await fetch(`${apiOrigin}/v1/discovery`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = response.headers.get("retry-after");
    const delayMs = retryAfter && /^\d+$/.test(retryAfter)
      ? Number(retryAfter) * 1_000
      : 2 ** attempt * 1_000;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return discover(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }
  return response.json() as Promise<Discovery>;
}

const manifest = await discover();
const parser = manifest.capabilities.find(
  (item) => item.method === "POST" && item.path === "/v1/pdf/parse",
);
if (!parser?.available) throw new Error("PDF parsing is unavailable");
console.log({ capability: parser.id, method: parser.method, path: parser.path });
```

The capability response supplies the full request and response schemas, billing information, and a runnable example. Use those live fields for the parse request instead of guessing a body from prose.

Guessing loses.

The useful application abstraction is a local page record, not a vendor response object. This second TypeScript block turns the same parsed pages into either strategy while retaining page provenance. It is deliberately small enough to test without a remote service.

```ts
type OcrPage = {
  pageNumber: number;
  text: string;
};

type RetrievalUnit = {
  id: string;
  documentId: string;
  text: string;
  pageNumbers: number[];
};

function buildUnits(
  documentId: string,
  pages: OcrPage[],
  mode: "page" | "document",
): RetrievalUnit[] {
  const ordered = [...pages].sort((a, b) => a.pageNumber - b.pageNumber);

  if (mode === "document") {
    return [{
      id: documentId,
      documentId,
      text: ordered
        .map((page) => `[Page ${page.pageNumber}]\n${page.text}`)
        .join("\n\n"),
      pageNumbers: ordered.map((page) => page.pageNumber),
    }];
  }

  return ordered.map((page) => ({
    id: `${documentId}:page:${page.pageNumber}`,
    documentId,
    text: page.text,
    pageNumbers: [page.pageNumber],
  }));
}

const pages: OcrPage[] = [
  { pageNumber: 1, text: "Application cover sheet" },
  { pageNumber: 2, text: "Authorized amount and signatures" },
  { pageNumber: 3, text: "Fee disclosure" },
];

console.log(buildUnits("loan-8A31", pages, "page"));
```

That boundary prevents a common mistake: flattening first, then promising to reconstruct page references later. It also makes the comparison honest. Both modes consume identical OCR text; only the retrieval granularity changes.

Infrai's public, self-describing discovery surface requires no key and reports 295 routes across 20 modules; every documented capability has runnable examples in 10 languages. The interface is plain REST, so the initial integration is a schema-reading exercise with no required SDK. A separate advantage is credential consolidation: one key spans the platform, reducing credential and billing glue if this ingestion job later needs another backend capability. The trade-off is clear. These interface properties do not prove better OCR fidelity, latency, or uptime, and the platform is a poor choice when policy requires local-only processing. Benchmark the scans before choosing.

## How do the real alternatives change the decision?

AWS Textract, Google Cloud Document AI, and Azure AI Document Intelligence are the obvious hosted OCR products to put in the same evaluation. They should receive the same scan corpus and normalize into the same `OcrPage` shape. Then the page-versus-document decision remains yours instead of being smuggled in by a provider-specific response.

| Option | What to inspect | Fair decision rule |
|---|---|---|
| AWS Textract | Returned page association and adapter work | Reject any normalization that loses page identity |
| Google Cloud Document AI | Returned document structure and transformations | Count glue needed to produce the local page record |
| Azure AI Document Intelligence | Analysis result and page association | Keep service fields outside the retrieval schema |
| Infrai | Discovered schema and runnable example | Value it when one contract and credential reduce wiring |

This table does not name an OCR winner. No comparable accuracy, latency, uptime, or cost measurements are available here, so crowning one would be theater. The useful test uses actual fintech scans and scores page correctness separately from unit count.

Measure both.

Local OCR is another legitimate path, with Tesseract as the established comparison point. It changes the operating boundary because the application owns execution and normalization. I would choose it when local processing is mandatory and the team accepts that operational burden. It does not change the indexing rule: preserve page identity before constructing retrieval units.

DocRaptor, PDFMonkey, and PDFShift fit document-generation work, while Gotenberg, WeasyPrint, and wkhtmltopdf fit teams building PDFs through their own conversion stack. They are real alternatives in a broad PDF-tool decision, but they are not substitutes for OCR evaluation here. Producing a PDF and recognizing text from a scan solve different jobs. This limitation matters: keep the candidate set tied to scanned records that must become searchable evidence.

## When the whole file is the better runner-up

Choose document-level indexing for compact, single-purpose files when a result only needs to locate the record. A one-page receipt is page-granular in practice. A short form whose contents are always reviewed together may gain little from separate rows.

It can also work as a first-stage catalog: retrieve a likely document, then search pages inside it. The second stage still depends on preserved page numbers. Lose them during ingestion and the supposedly lean design leaves a re-parsing bill later.

Do not use whole-document indexing as a blanket shortcut for long, mixed packets. Fewer rows look tidy, but citations point at a file rather than the supporting evidence and a match can carry unrelated text. For auditable finance search, **page provenance is the durable asset**. Index granularity can change. Provenance should not.

## References

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [AWS Textract documentation](https://docs.aws.amazon.com/textract/)
- [Google Cloud Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/azure/ai-services/document-intelligence/)
- [Tesseract OCR documentation](https://tesseract-ocr.github.io/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [Gotenberg documentation](https://gotenberg.dev/docs/)
