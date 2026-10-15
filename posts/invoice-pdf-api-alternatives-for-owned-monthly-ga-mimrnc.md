# Invoice PDF API Alternatives for Owned Monthly Game Report Templates

**Short answer:** DocRaptor, PDFMonkey, API2PDF, and any other invoice PDF API alternative sit behind the decisive boundary: a reproducible monthly game report requires an owned, immutable template version in its archive record.

Let a rendering service own it only when non-engineers need to change presentation without a code release and the team accepts that operational boundary.

| Choice | Template owner | Best fit | Main trade-off |
|---|---|---|---|
| Template in the application repository | Engineering | Auditable monthly close | Changes require the normal release path |
| Template in a rendering system | Operations or design | Frequent copy and layout edits | Report output depends on separately managed state |
| Versioned template package | Shared | Several report producers | Package publication adds one controlled step |

**Recommendation:** for a monthly game report that is rendered once and archived, keep the template beside the data contract and record its immutable version with every render job. This is not a renderer recommendation. It is a custody decision.

## How should you compare an invoice PDF API alternative?

A PDF API can answer a narrow question: can this input become a PDF? The archive has a harder question: can the team explain which rules turned a given month's ledger into the document reviewers saw? DocRaptor, PDFMonkey, and API2PDF belong in the candidate set only after that ownership question has an answer. The same applies to a self-hosted renderer or another alternative. This ordering keeps the comparison honest without pretending that unlike template models are interchangeable.

They aren't.

Those rules live in more places than the HTML. They include currency formatting, timezone cutoffs, the ordering of game titles, how refunds are represented, what happens when a title has zero activity, and which legal footer appears for the reporting entity. Put those decisions in remotely editable template state and a successful rerender can still be the wrong artifact. The bytes arrive. The evidence does not.

Template custody therefore belongs in the same threat model as source data custody. For this workflow, I would make the render request carry two immutable identifiers: a report snapshot ID and a template version. The archive manifest stores both, plus a digest of the produced PDF. Short and inspectable beats a page of renderer configuration. The trade-off is explicit: repository ownership makes each report definition reviewable but puts presentation changes on the engineering release path; delegated ownership makes editing faster for another team but creates separately managed state that every queued and replayed job must identify.

PDF itself is standardized by ISO 32000-2. That standard defines the document format; it does not define the business provenance of a monthly revenue report. A conforming file can still be impossible to reproduce because the caller failed to retain its inputs and rendering rules.

## Two criteria decide the boundary

The first criterion is **change authority**. Ask who is allowed to alter totals, labels, grouping, and footnotes. Cosmetic edits are rarely purely cosmetic in financial reporting. A narrower column can wrap a title onto a new page. A renamed field can hide a distinction reviewers relied on. If those changes require review, the template should use the same reviewable release path as the report schema.

The second criterion is **replayability**. “We still have the HTML” is not enough if that HTML refers to mutable assets or if the job silently selects the latest template. A replayable job resolves every dependency to an immutable value before rendering. Fonts, styles, logos, locale, and template revision need stable identities. No mystery defaults.

I benchmark this boundary by time-to-first-trust, not time-to-first-PDF. A tiny proof should render the same fixture twice, preserve the template revision in the archive manifest, and fail closed when the requested revision is missing. If demonstrating those three properties needs a large config graph, the integration is already telling you what production will feel like.

Count the glue.

## A small rendering contract

Keep the renderer behind a boring interface. The application prepares normalized report data; the adapter renders bytes; the archive step records the evidence needed to verify what happened.

```ts
type MonthlyGameReport = {
  reportId: string;
  month: string;
  currency: string;
  games: Array<{
    title: string;
    grossMinor: number;
    refundsMinor: number;
  }>;
};

type RenderReceipt = {
  reportId: string;
  templateVersion: string;
  pdfSha256: string;
  archivedAt: string;
};

interface PdfRenderer {
  render(input: {
    templateVersion: string;
    report: MonthlyGameReport;
  }): Promise<Uint8Array>;
}

async function renderAndArchive(
  renderer: PdfRenderer,
  report: MonthlyGameReport,
  templateVersion: string,
  archive: (pdf: Uint8Array, receipt: RenderReceipt) => Promise<void>,
  sha256: (bytes: Uint8Array) => Promise<string>,
  now: () => string,
): Promise<RenderReceipt> {
  if (!/^\d{4}-\d{2}$/.test(report.month)) {
    throw new Error("month must use YYYY-MM");
  }

  const pdf = await renderer.render({ templateVersion, report });
  const receipt: RenderReceipt = {
    reportId: report.reportId,
    templateVersion,
    pdfSha256: await sha256(pdf),
    archivedAt: now(),
  };

  await archive(pdf, receipt);
  return receipt;
}
```

There is deliberately no vendor-shaped request object in the domain code. Adapters can translate this contract to a hosted invoice API, a browser process, or a separate rendering worker. The monthly-close path doesn't need to know.

Test with one compact fixture and one hostile fixture. The compact fixture catches accidental padding and pagination changes. The hostile fixture should include a long game title, a zero-activity row, a refund, and enough rows to cross a page boundary. Compare semantic expectations first: report ID, period, totals, row order, and page count. A byte-for-byte comparison may be too strict when document metadata changes, so store the digest as archive evidence rather than pretending it is always a universal regression oracle.

Treat rendering as a job, even if the first implementation is synchronous. Give the job an idempotency key derived from the report snapshot and template version. A retry must not create a second logical archive record. Log the two identifiers, duration, output size, and terminal status. Do not log the entire revenue payload.

## When the runner-up is the better choice

Renderer-owned templates are reasonable when copy changes happen much more often than application releases, and the people responsible for those changes cannot use the repository workflow. The boundary can work if the system can pin a template revision, export or otherwise retain that revision, and prevent a later edit from changing an already queued job.

A versioned template package is the better compromise when several services produce related reports. It gives them one reviewed layout contract without granting each service access to mutable remote state. The cost is release coordination. That cost is visible, which I prefer to hidden coupling.

Choose repository ownership when audit and replay dominate. Choose delegated ownership when editing autonomy dominates. Choose a package when reuse dominates. Do not choose by the shortest quickstart or a temporary unit price; neither tells you who controls the document definition during month-end close.

## Archive the decision, not just the document

The finished PDF is one record. Keep the normalized input snapshot, template version, render receipt, and content digest with it. Retention and access policy should apply to that bundle, because any one component alone gives an incomplete account of the report.

This makes renderer replacement much less dramatic. The adapter changes. The report contract and archive manifest do not. More important, a future reviewer can distinguish “the numbers changed” from “the presentation rules changed” without reconstructing state from an old deployment.

For a monthly game revenue report, template ownership is the durable selection axis. Pin it, test it with ugly data, and archive its identity beside the PDF.

## References

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
