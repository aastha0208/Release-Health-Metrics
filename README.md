# Release Health Metrics

**One consistent, trustworthy view of how each release is tracking, and how releases compare over time.**

It has two parts:

- **A Python pipeline (this repo)** pulls test-plan data from Xray Cloud through its GraphQL API, applies one set of metric definitions, and produces release-level metrics.
- **A Power BI dashboard** combines those metrics with Jira issue data, so leaders can go from the release summary down to the specific failures and blockers behind it. The Power BI file isn’t included in this repo: a .pbix file stores a copy of the data it was built on, and that data belongs to my former employer.

Post-release outcomes, meaning escaped defects and incidents, are the next layer on the roadmap below and are not built yet.

I built it at BeyondTrust while leading a 21-person cross-functional engineering team on an enterprise identity-security SaaS product line with monthly major and maintenance releases, to give release reviews one agreed set of numbers.

---

## The decision this supports

The go/no-go conversation in each release's certification week, where it supplies the test-execution evidence: how much of the planned scope has been validated, and how it's performing. It also answers a slower question: is release quality trending better or worse across releases?

It does **not** make the go/no-go call on its own. Sign-offs and decisions about known issues still happen in the room.

**Who used it:** Product Managers and Owners, Engineering Managers, the Director of Engineering, the VP of Engineering, and the Release Review Committee.

**When:** mainly during certification week for each monthly release.

## The problem

Xray's built-in reports answer questions about one test plan at a time, and the raw numbers mislead in three ways:

1. **Re-runs distort the picture.** A test that failed twice and then passed shows up as three results. Depending on how you count, the release looks worse or better than it is.
2. **Tests that never ran disappear.** A planned test with no execution doesn't appear in run statistics, so a release can look 98% passing while a fifth of its scope hasn't been touched.
3. **There's no cross-release view.** Comparing this release with the last ten meant pulling each plan's report separately.

The underlying problem wasn't a missing chart. It was two connected gaps:

- **No shared metric definition.** Teams and leadership didn't agree on what "pass rate" or "complete" meant, so the same release could be described with different numbers.
- **No alignment on go/no-go.** Without shared numbers, teams and leadership had no common basis for deciding whether a release should ship.

Agreeing on the definitions came first. Until then, any dashboard would just have moved the argument somewhere else.

## Product decisions: the metric definitions

These are the definitions the tool enforces. Each one is a choice, and each choice has a reason.

| Metric | Definition | Why this definition |
|---|---|---|
| **Planned tests** | Unique tests in the release's test-plan scope | Scope is the denominator. Runs of tests outside the plan are ignored. |
| **Status per test** | The **latest** run per planned test, across every execution linked to the plan | Health is about current state, not history. One test has one vote. |
| **To Do** | Planned tests with no run at all, plus tests whose latest status is To Do | Unrun scope stays visible instead of silently dropping out. |
| **Pass %** | Passed ÷ (Passed + Failed) | Measures quality of what was actually exercised. Blocked tests are a progress problem, not a quality signal. |
| **Execution coverage %** | (Passed + Failed + Blocked + Executing) ÷ Planned | How much of the scope has been touched. |
| **Completion % / Remaining %** | Share of planned scope with a terminal or in-flight status, including Descoped | How close the cycle is to done, with descoping counted as an explicit decision. |
| **Raw execution** | All Passed/Failed runs, including re-runs | Kept separately as a measure of **effort**, so it never contaminates the quality numbers. |

**Health thresholds:** Pass % and Completion % show green at ≥ 95%, amber at ≥ 90% and red below that, against a 90% pass-rate target line. Remaining % shows green at ≤ 5% and amber at ≤ 10%.

## Trade-offs I chose

| Decision | Alternative I rejected | Why |
|---|---|---|
| Latest run per test | Count every run | Every-run counting rewards re-running a flaky test until it passes, and penalises fixing. |
| Separate quality metrics from effort | One blended number | Teams needed to see both "how good" and "how much work", and one number hides one of them. |
| Cache closed plans; always refetch active ones | Refetch everything / cache everything | Closed releases don't change, so refetching them is wasted API time. Active releases must be live. Freshness only where it matters. |
| A release spreadsheet as the release registry | Infer releases from Jira | The team already maintained that sheet. Reusing it avoided a second source of truth. The cost: the dashboard is only as current as the sheet. |
| Stop and ask a person when Jira needs re-indexing | Fail silently or skip affected plans | A partial result that looks complete is worse than a delay. |
| Notebook + Power BI | Build a web dashboard | Power BI was where stakeholders already looked. The cheapest path to adoption was meeting them there. |

## How it evolved

The twelve notebooks are kept as a record of the iterations, not as twelve products. This grouping is reconstructed from what each notebook does; commit history doesn't preserve the order.

| Stage | What changed | Notebooks |
|---|---|---|
| 1 · Count one plan | Pull runs for one test plan, roll statuses into buckets, print a table to paste into Jira, Confluence or Slack | `Release Test Report`, `Report with pandas` |
| 2 · Track over time | Daily snapshots appended to CSV, first charts | `Execution Test Report`, `Release Test Report with matpotlib` |
| 3 · Reconcile with Xray | Check my numbers against Xray's own dashboard counts before anyone relies on them | `Xray Reports API Dashboard Extractor`, `CSV_Release Test Report using Solution Test Plan` |
| 4 · Many releases | Multi-release view, Parquet caching, expanded status mapping | `v1_…`, `v2_…` |
| 5 · Fix the metric | Latest run per test; unrun scope counted as To Do | `Protect_v3_…` |
| 6 · Make it repeatable | Releases driven by the registry sheet (this year and last), charts, Jira re-index handling | `Protect v4_…`, `Protect_v5_…` |
| 7 · Hand off to stakeholders | Publish to the Power BI dashboard | `v6_Release Test Report_Power BI` |

Stage 3 is the one I'd point to: a metric people will make ship decisions on has to agree with the system of record, or have a documented reason why it doesn't.

## Outcomes

1. **A one-click view of an entire release for senior leadership.** Leaders could judge where a whole release stood from one screen, using one agreed set of numbers.

2. **Detail on demand.** When a leader wanted to go deeper, the same view drilled into Jira data to show:

    - how many tests passed and how many failed
    - which failures were blocking the release
    - which failures had to be resolved before ship
    - whether those fixes had been prioritised

The summary answered "where are we?" and the drill-down answered "what has to happen before we ship?" Leaders could move from the first question to the second themselves.

## Roadmap: from test evidence to release health

Prioritised as if this were a product roadmap. Items 1 and 2 complete the move from test-execution metrics to full release health.

1. **Move the defect view into the pipeline.** Today the Jira drill-down lives only in Power BI. Bringing defect metrics into the pipeline would make them consistent across releases, just as the test metrics are: open defects at release cut by severity with a zero-blocker gate, and defects found vs. fixed across the cycle.
2. **Add post-release outcomes (planned).** Escaped defects (bugs reported against a version after it ships), escape rate (escaped ÷ all defects found for that release), and Sev 1/2 incidents in the 30 days after release. This closes the loop: **do the pre-release signals predict what customers experience?** High pass rate plus high escapes means the validation scope, not the code, is what needs fixing.
3. **Fix one definition.** Completion % counts Blocked and Executing tests as complete, which overstates progress late in a cycle. Completion should mean a terminal status only: Passed, Failed or Descoped.
4. **Add a counter-metric.** Pass rate can rise because scope was descoped. Show descoped volume next to it.
5. **Make the thresholds release criteria.** Agree them with Product and Engineering leadership as explicit ship conditions, rather than colours on a chart.
6. **Schedule and harden it.** A scheduled job instead of a person running a notebook; proper certificate handling instead of disabled SSL verification; credentials from a secrets store; one parameterised script instead of notebooks.

## Running it

Requires Python 3.10+, `pandas`, `requests`, `pyarrow`, `matplotlib` and `openpyxl`, plus Xray Cloud API credentials (client ID and secret, not an Atlassian token). The latest version, `v6_Release Test Report_Power BI.ipynb`, prompts for credentials, finds the release registry file (`Release` and `Key` columns), and writes `release_metrics_dashboard.csv` for Power BI to read.

## Limitations

- The pipeline in this repo covers release test execution metrics. The Jira drill-down lived in the Power BI file, which isn't published because it embeds employer data. Post-release incidents aren't built.
- SSL verification is disabled for a corporate proxy. That's acceptable on a work laptop, not in production.
