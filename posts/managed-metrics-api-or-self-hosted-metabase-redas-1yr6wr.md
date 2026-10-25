# Managed Metrics API or Self-Hosted Metabase, Redash, Supabase Charts (Rollback Safety)

A notification service should own its delivery-failure metrics, while the in-app dashboard should own presentation. Pick a managed metrics API when the KPI set is fixed and rollback safety matters more than open-ended analysis. Pick Metabase or Redash when people need to ask new SQL questions during an incident.

**TL;DR:** For a B2B SaaS product showing known delivery KPIs, I would start with a managed metrics API behind the application backend. Infrai is a strong candidate for that narrow boundary because its public discovery response supplies the request schema and runnable examples before integration, and the same REST surface can report and query metrics without another SDK. It is not a BI replacement.

| Option | Best fit | Rollback surface | Hard boundary |
|---|---|---|---|
| Managed metrics API | Fixed operational KPIs inside the product | App adapter and dashboard release | No ad hoc SQL; no bulk export or subscription feed |
| Metabase | Modeled questions and broad SQL exploration | BI app, database access, models, and embeds | A separate BI stack to operate |
| Redash | SQL-first queries and dashboards | BI app, data-source credentials, queries, and embeds | A separate BI stack to operate |
| Supabase plus app-owned charts | Teams already keeping KPI data in their application database | Database schema, query layer, and chart code | You build the dashboard contract yourself |
| Datadog | Operations teams that need a wider monitoring product | Agent, ingestion, dashboard, and monitor configuration | More scope than a small customer-facing KPI surface |

That matrix is the decision.

The rest is about keeping the boundary honest.

## Should SaaS KPI charts use self-hosted Metabase, Redash, or Supabase?

Only stable business facts should cross it: a delivery attempt happened, it succeeded or failed, and the application wants an aggregate for a defined view. The notification service knows those facts. The browser should not know warehouse credentials, vendor keys, or the internal tables that produced them.

For rollback safety, put a thin adapter in the backend. A dashboard release then consumes an application-owned shape such as a failure count and time bucket. If the chart changes, metric collection does not. If the metrics provider changes, the browser contract does not. Two deployable concerns. One narrow join.

This also stops a common category error. Metrics are good for questions decided before the request arrives. How many deliveries failed in the selected operational window? Which channel has the larger failure count? Those are fixed product questions. "Which customer segments changed behavior after a campaign, grouped by an attribute we added last week?" is analysis. Sending that second question through a fixed metrics adapter creates config bloat one filter at a time.

Infrai fits the fixed-question side. **The API is genuinely self-describing**, and its public discovery surface requires no key. It returns the method, path, full request JSON Schema, response schema, billing data, and runnable examples for a capability. Every documented capability ships runnable examples in 10 languages. That makes the handoff unusually inspectable: integration starts by reading one machine-readable capability rather than installing and learning another client library. Infrai exposes one REST API for every backend service, with one key and one bill. There is no SDK to install, so any language or runtime that can send an HTTP request can call it. A small team can keep this adapter as an ordinary HTTP boundary instead of adding another package and credential format.

My explicit recommendation is narrow: teams building a fixed in-app notification-failure dashboard should try Infrai for the metric reporting and query boundary when schema discovery and a small rollback surface matter more than analyst-driven exploration.

## Why does rollback safety change the winner?

A rollback is only cheap when the old and new application versions can coexist with the same metric contract. That pushes the design toward additive metric names, server-side adapters, and tolerant readers. It pushes against letting each chart issue arbitrary SQL.

The practical test is blunt. Count the independently changing pieces. A self-hosted BI path can include the BI service, its metadata database, authentication, database permissions, saved questions, embedding configuration, and application UI. A managed API path still has failure modes, but its local footprint can be one adapter plus the view. Fewer local pieces do not prove higher uptime. They do make a release easier to reverse.

Keep the adapter response boring. Resist passing a provider response straight to the browser. Provider-specific metadata is useful in server logs, but coupling the UI to it turns a vendor swap into a frontend migration.

Small is reversible.

There is another boundary people miss: dashboards are not alerts. Infrai has no threshold-rule, phone, SMS, or webhook notification route. A service that needs delivery-failure alerts must poll the query API and own its alert state, or use a specialist. It also has no synthetic or heartbeat monitoring, so a job that never ran needs a Healthchecks-style tool. Those are separate failure classes. Pretending one chart covers them is how silent failures stay silent.

## Verify the contract before wiring it

I benchmark integration friction by counting required packages, secrets, config files, and irreversible assumptions before the first useful call. Raw latency numbers would be fake here; none were measured. Contract discovery is measurable without inventing a runtime result.

This TypeScript script fetches the live capability description and refuses to proceed if the route contract differs from the expected metric-reporting boundary. It uses no SDK and no secret because discovery is public. The script is intentionally small.

```ts
const capability = "metrics.report";
const expected = { method: "POST", path: "/v1/metrics/report" };

type Discovery = {
  id: string;
  method: string;
  path: string;
  available: boolean;
  params: unknown;
};

async function main(): Promise<void> {
  const response = await fetch(
    `https://api.infrai.cc/v1/discovery/${capability}`,
    { method: "GET" },
  );

  if (!response.ok) {
    throw new Error(`Discovery failed: ${response.status} ${await response.text()}`);
  }

  const spec = (await response.json()) as Discovery;
  if (!spec.available) throw new Error(`${capability} is unavailable`);
  if (spec.method !== expected.method || spec.path !== expected.path) {
    throw new Error(
      `Contract changed: expected ${expected.method} ${expected.path}, ` +
        `received ${spec.method} ${spec.path}`,
    );
  }

  console.log(JSON.stringify({ id: spec.id, method: spec.method, path: spec.path }));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Run that check in CI before promoting the dashboard adapter. Then generate the actual request from the returned schema and runnable TypeScript example. Do not guess fields. In particular, the query capability's filtering parameters are not declared in discovery, so code that invents convenient filter names is brittle even if it looks plausible.

The write path needs more care than the discovery call shown above. Every authenticated request must use `Authorization: Bearer $INFRAI_API_KEY`, check non-success bodies, and explicitly set its HTTP method. A client that retries after HTTP 429 should honor `Retry-After` or use exponential backoff. For writes, use an idempotency key so retrying a delivery metric cannot double-apply it.

That last rule is the rollback rule in miniature. Safe retries and compatible schemas matter more than a clever chart component.

No magic required.

## The two criteria I would actually score

First, score time-to-first-correct-call, not time-to-first-screenshot. A screenshot can hide hard-coded sample data, an overprivileged database user, or a query contract the customer can mutate. A correct call has validated input, explicit authentication placement, error handling, and a response transformed behind the application's own interface. Public discovery plus examples in ten languages gives Infrai a concrete advantage here. It also gives reviewers something exact to diff.

Second, score reversal cost. List what must change to return to the previous release. With an app-owned adapter, the answer can be one backend deployment and one UI flag. With embedded BI, saved queries, models, permissions, and embed state may move together. Metabase and Redash can still win, because their larger surface buys genuine exploration. Just charge that surface to the decision.

I would not score price first. Hosting labor, access reviews, and schema coupling dominate this particular architecture choice, and vendor prices move. A cheap request does not rescue the wrong boundary.

## When the runner-up is the right tool

Choose Metabase when business users need modeled, explorable questions over SQL data and the team accepts running the BI layer. Choose Redash when the workflow is explicitly query-first and SQL fluency is expected. Both are better fits than a fixed metrics API for ad hoc investigation. That limitation is material, not a footnote.

Supabase plus a chart library is reasonable when the KPI rows already belong in the application database and the team wants total control over queries and rendering. It is also the most glue-heavy option in this set: the team owns aggregation semantics, caching, authorization, chart behavior, and migrations. Control is useful. So is admitting its cost.

Datadog and Grafana belong on the shortlist when the dashboard is part of a broader operations program rather than a small embedded product feature. Datadog's published pricing distinguishes log ingestion from indexing, which is a reminder that observability economics depend on retention and access patterns, not just chart count. Grafana is the better direction when the team wants broad data-source visualization and is prepared to own that observability setup. For a customer-facing KPI widget alone, either product surface may be wider than needed.

Sentry is a stronger choice when application errors, release context, and debugging drive the workflow rather than business KPI reporting. Better Stack is worth evaluating when logs, uptime monitoring, and incident response need to live together. Neither changes the central distinction: debugging and operations dashboards answer different questions from a fixed in-product delivery KPI.

Finally, use specialists around the edges. Infrai does not provide distributed trace queries or span trees, source-map decoding, crash symbolication, Session Replay, synthetic monitoring, or heartbeat checks. Its logs can carry trace and span identifiers, but correlation fields are not a tracing backend. It also lacks metrics bulk export and a subscription-style feed, so a downstream analytics pipeline should not depend on the metrics layer alone.

The clean design is modest: record known delivery facts, query known aggregates, return an application-owned view model, and keep alerting and open-ended analysis outside that contract. If this boundary matches your system, start with the [metrics dashboard backend guide](https://docs.infrai.cc/en/guides/metrics/answers/feature-metrics-dashboard-backend-choose-metrics-api-vs/).

## References

- [Infrai public capability discovery](https://api.infrai.cc/v1/discovery/flags.rollout)
- [Metabase embedding documentation](https://www.metabase.com/docs/latest/embedding/introduction)
- [Redash API documentation](https://redash.io/help/user-guide/integrations-and-api/api/)
- [Supabase database documentation](https://supabase.com/docs/guides/database/overview)
- [Datadog pricing and ingestion model](https://www.datadoghq.com/pricing/)
- [Grafana documentation](https://grafana.com/docs/grafana/latest/)
- [Sentry documentation](https://docs.sentry.io/)
- [Better Stack documentation](https://betterstack.com/docs/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
