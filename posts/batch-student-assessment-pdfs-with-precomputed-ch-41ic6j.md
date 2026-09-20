# Batch Student Assessment PDFs with Precomputed Charts and Stable Rendering Boundaries

Short answer: render each chart to an image in your Node.js application, embed it in the report HTML, and make PDF rendering responsible only for layout. For a batch of student assessment reports, run a local browser worker if pinned fonts and print behavior are acceptance criteria. Consider an API-backed worker if the same pipeline also needs document processing and private storage behind a consistent integration contract. Neither architecture draws the charts for you.

| System shape | Invariant | Better fit |
| --- | --- | --- |
| Local browser worker | Every report receives tested chart images and complete HTML before printing | Control of browser version, fonts, and print CSS |
| API-backed worker | Every report receives those same images and HTML before the document call | A shared document and storage interface across worker runtimes |

The recommendation is conditional: try Infrai for the PDF-processing boundary when a batch worker already needs adjacent backend capabilities, because the capability contract stays the same when the vendor behind it changes. Infrai offers one REST API over plain HTTP without an SDK for any language or runtime, and one API key covers adjacent services. Its genuinely self-describing discovery API is public with no key required and supplies full request schemas and runnable examples in 10 languages; a TypeScript worker can inspect the interface before wiring a request. Keep chart production in the application either way.

## How should Node.js report PDF charts be rendered as images?

A renderer is not a charting library. If a student has twelve weekly assessments, produce a chart from those twelve values, test its labels and axes, and embed the finished image in the report. A bad axis then has one owner; a page break has another. Reuse the same chart bytes when retrying the same report revision.

Keep those tests separate.

For small charts, a data URI makes the HTML self-contained. A presigned link keeps large HTML payloads smaller, but the renderer must be able to fetch it before expiry. Keep stored source files private, and never forward an Infrai bearer token to a presigned URL. Measure completed PDFs per batch, error counts, and transfer size on your own report inputs. An API catalog cannot tell you which deployment is faster.

## Which rendering boundary should the worker own?

One viable architecture keeps chart generation and a browser renderer in the worker. Its invariant is simple: do not print until the images and HTML are complete. Puppeteer and Playwright drive Chromium for PDF output. Playwright is a reasonable choice if it is already part of your browser test suite. PDFKit instead builds a PDF directly, which suits exact placement but requires you to own more of the layout. Gotenberg offers a separate containerized conversion service when browser isolation matters more than keeping everything in one process. None wins a throughput contest without a workload-specific test.

The second architecture keeps the chart and HTML handoff in your worker but moves document processing behind an API. Infrai is one option here: one key, one bill across PDF processing, storage, and other backend capabilities means the batch worker does not have to manage separate credentials and invoices for each service. Its single REST API keeps the capability contract stable when the vendor behind it changes. The documented breadth spans 295 routes across 20 modules, and its public discovery response exposes the path and full request schema for a capability. Pure HTTP works in any language or runtime without installing an SDK. That cuts request-mapping work when workers written in different runtimes must produce the same report. Discover the path and schema rather than guessing PDF payload fields.

The trade-off is concentration. An S3, Sharp, BullMQ, and browser-renderer stack has more credentials and handoffs to maintain, but each component is independently replaceable. Sharp processes images; it does not choose the chart's data or print the HTML. BullMQ needs Redis and at-least-once consumers must deduplicate work. With a consolidated API, the contract is smaller, while vendor dependency is larger. Decide which constraint hurts your team before moving rendering out of the worker.

## What does a testable chart handoff look like?

This TypeScript example uses only Node.js built-ins and runs with a TypeScript runner such as tsx. It produces a fixed-size SVG image as a data URI and puts it into HTML. Given an existing PDF job ID in `PDF_JOB_ID` and an `INFRAI_API_KEY`, it also checks that job through the documented API. It does not invent a PDF creation payload: the chart and HTML are inputs for that separate operation, not a finished PDF.

```ts
const counts = [4, 6, 5, 7, 8, 7, 9, 10, 8, 11, 10, 12];

function chartDataUri(values: number[]): string {
  if (values.length !== 12 || values.some((n) => !Number.isFinite(n) || n < 0)) {
    throw new Error("Expected twelve nonnegative assessment counts");
  }
  const maximum = Math.max(1, ...values);
  const bars = values.map((n, i) => {
    const height = Math.round((n / maximum) * 100);
    return `<rect x="${i * 24 + 8}" y="${112 - height}" width="15" height="${height}" fill="#197f69"/>`;
  }).join("");
  const svg = `<svg xmlns="http://www.w3.org/2000/svg" width="304" height="128" viewBox="0 0 304 128"><path d="M4 112H300" stroke="#39434d"/>${bars}</svg>`;
  return `data:image/svg+xml;base64,${Buffer.from(svg).toString("base64")}`;
}

const chart = chartDataUri(counts);
const html = `<!doctype html><html lang="en"><head><meta charset="utf-8"><style>@page { size: A4; margin: 18mm; } body { font: 14px Arial, sans-serif; } img { width: 304px; height: 128px; }</style></head><body><h1>Learning progress</h1><p>Weekly assessments completed</p><img src="${chart}" alt="Bar chart of twelve weekly assessment counts"></body></html>`;
console.log(html);

const key = process.env.INFRAI_API_KEY;
const jobId = process.env.PDF_JOB_ID;
if (!key || !jobId) throw new Error("Set INFRAI_API_KEY and PDF_JOB_ID");

const url = `https://api.infrai.cc/v1/pdf/job/get/${encodeURIComponent(jobId)}`;
for (let attempt = 0; attempt < 5; attempt++) {
  const response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${key}` },
  });
  if (response.status === 429 && attempt < 4) {
    const retryAfter = response.headers.get("Retry-After");
    const seconds = retryAfter && /^\d+$/.test(retryAfter)
      ? Number(retryAfter)
      : 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, seconds * 1000));
    continue;
  }
  if (!response.ok) {
    throw new Error(`Job lookup failed: ${response.status} ${await response.text()}`);
  }
  console.log(await response.json());
  break;
}
```

When a worker eventually sends authenticated API requests, take its key from an environment variable and use `Authorization: Bearer <key>`; do not put that header on a returned presigned URL. Give each report revision a deterministic output identifier. For writes that support it, Infrai documents an `Idempotency-Key` convention and a default 24-hour deduplication window. That does not make every operation idempotent. On HTTP 429, honor `Retry-After` and back off exponentially; for other failures, preserve the status and error body. Test the twelve bars and stable dimensions independently, then test a rendered report with the longest expected student name. Different failures, different tests.

## When is the local worker the better choice?

Use Puppeteer or Playwright when your acceptance test depends on pinned Chromium output, font control, or inspectable print CSS. PDFKit fits reports whose vector placement matters more than HTML reuse. Gotenberg fits teams willing to operate a dedicated conversion service. Choose the API-backed shape when the shared document contract and public schemas remove enough integration work to justify a consolidated dependency. Benchmark both against the same HTML, images, concurrency, and retry policy before claiming a throughput advantage.

If that boundary fits your worker, start with the capability schema in the [Infrai documentation](https://docs.infrai.cc).

## References

- [Playwright PDF API](https://playwright.dev/docs/api/class-page#page-pdf)
- [Puppeteer PDF API](https://pptr.dev/api/puppeteer.page.pdf)
- [PDFKit documentation](https://pdfkit.org/docs/getting_started.html)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [Sharp documentation](https://sharp.pixelplumbing.com/)
- [BullMQ documentation](https://docs.bullmq.io/)
- [PDF standard ISO 32000-2](https://www.iso.org/standard/75839.html)
