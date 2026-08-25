# In-Cluster Regression Testing: Options

**Context.** We run a fleet of third-party applications (UI, web, and API surfaces) in a
Kubernetes cluster. Junior engineers apply upgrades on a normal cadence. The failure mode we
care about is silent: an upgrade lands, the pods go `Ready`, and nobody notices that a form
field disappeared or an endpoint started returning 500 on a boundary value. Cluster holds test
apps only — no production data. We want open source, self-hosted, running in the same cluster,
and cheap.

## The constraint that shapes everything

We do not own these applications. We cannot add test hooks, we don't control their release
notes, and their DOM and routes change without warning. That makes hand-written, selector-heavy
test scripts the most expensive thing we could build — every upstream release becomes test
maintenance.

So the strategy is to **weight the suite toward techniques with near-zero per-app authoring
cost**, and hand-write scripted flows only for the handful of journeys that genuinely matter:

| Technique | Per-app authoring cost | Catches |
| --- | --- | --- |
| Visual diffing (screenshot baselines) | Near zero — a URL list | Layout breaks, missing UI elements, CSS regressions |
| Console + network error assertions | Near zero — one shared fixture | JS exceptions, 4xx/5xx on page load, broken assets |
| Spec-driven API fuzzing | Near zero *if* an OpenAPI spec exists | Boundary/edge-case handling, schema violations, crashes |
| Spec diffing between versions | Near zero | Breaking API changes, before you even deploy |
| Accessibility/DOM structure scans | Near zero | Structural breakage, missing labels/roles |
| Scripted user journeys | High — and re-paid every upgrade | Real workflow breakage that nothing else sees |

Roughly: get 80% of the coverage from the top five rows, and spend the scripting budget on
login + the two or three flows per app that would actually page someone.

## Layer 1 — Test runners

What actually drives the browser or the HTTP client.

### Browser / UI

- **Playwright** (Apache-2.0, Microsoft) — the default recommendation. One tool covers UI *and*
  API assertions, ships built-in screenshot comparison (`toHaveScreenshot()`), a recorder
  (`codegen`) that turns clicking around into a test file, and a trace viewer that gives a
  DOM-level replay of any failure. Official container images. Auto-waiting removes most of the
  classic flake class.
- **Robot Framework** + **Browser library** (Apache-2.0) — Playwright underneath, keyword-driven
  plain-language syntax on top. Worth considering specifically because juniors will be reading
  and eventually writing these tests; the reports are human-readable by default.
- **Cypress** (MIT core) — good authoring experience, but parallelisation and the dashboard are
  paid (Cypress Cloud). `sorry-cypress` is a self-hosted OSS dashboard replacement; check its
  current maintenance status before betting on it.
- **Selenium Grid** (Apache-2.0) — only if we inherit existing Selenium suites. Heavier to run in
  cluster and more flake-prone than Playwright.

### API

- **Hurl** (Apache-2.0) — plain-text HTTP request/assert files, single static binary, tiny
  container. Best cost-to-value for smoke checks. Trivial for a junior to read and edit.
- **Bruno** (open-source core) — git-native API client with a CLI runner; a reasonable Postman
  replacement when someone wants a GUI to author in. Some features are behind a paid edition.
- **Schemathesis** (MIT) — property-based testing generated from an OpenAPI or GraphQL schema.
  This is the direct answer to "juniors don't know when edge cases are hit": it generates
  boundary values, type mismatches and constraint violations automatically and flags responses
  that violate the app's own spec. Zero per-endpoint maintenance. Requires the app to publish a
  spec.
- **k6** (AGPL-3.0, Grafana) — load testing, but its `check()` API also does functional
  assertions, and `k6-operator` runs it as a CRD in cluster with Prometheus output. Use it to
  catch performance regressions on upgrade, which a functional suite will never see.
- **oasdiff** (OSS) — diffs two OpenAPI specs and classifies breaking changes. Run it *before*
  the upgrade rolls out, against the new image's spec. Cheapest possible early warning.
- **Newman** — works, but Postman collections drift toward the hosted product. Prefer Hurl or
  Bruno for new work.

### Cross-cutting checks

- **axe-core** (MPL-2.0) via `@axe-core/playwright` — accessibility rules double as a structural
  integrity check. A disappeared label or a broken landmark shows up here.
- **BackstopJS** (MIT) — dedicated visual regression with its own HTML diff report, Playwright or
  Puppeteer backed. Use it if we want visual testing decoupled from the functional suite;
  otherwise Playwright's built-in comparison is one less moving part.
  (Note: **Lost Pixel**, the usual OSS Percy/Chromatic alternative, was archived in April 2026 —
  don't start there.)

## Layer 2 — In-cluster orchestration

What schedules the runners, holds their config, and collects results.

- **Plain `Job` / `CronJob`** — the floor, and genuinely viable. A container image with the suite,
  a `CronJob` for the nightly baseline, `kubectl create job --from=cronjob/...` for ad-hoc runs,
  `ttlSecondsAfterFinished` for cleanup. Zero added infrastructure and zero added cost. Its
  weakness is ergonomics: no UI, results live wherever we push them.
- **Testkube** (agent dual-licensed MIT + Testkube Community License) — Kubernetes-native test
  orchestration. Tests become CRDs; it runs Playwright, Cypress, k6, Postman, JMeter, curl and
  others as pods; it can trigger suites off Kubernetes events (e.g. a `Deployment` image change),
  which maps directly onto our upgrade workflow. The agent runs standalone and self-hosted. Note
  the split: the agent is the OSS piece, while the polished dashboard/multi-environment features
  belong to the commercial Control Plane — verify the current OSS feature boundary against their
  docs before committing.
- **Argo Workflows** (Apache-2.0, CNCF) — general DAG engine. If we want fan-out (30 apps × 3
  suites in parallel), retries, artifact passing and a decent UI without adopting a testing
  product, this is the flexible option. Pairs with **Argo Events** for triggering.
- **Tekton** (Apache-2.0) — Kubernetes-native pipelines. Reasonable if we already run Tekton;
  not worth adopting solely for this.
- **Kuberhealthy** (CNCF) — an operator that runs synthetic checks as pods on a schedule and
  exports pass/fail to Prometheus. Different shape from the others: it's for *continuous*
  lightweight probes ("can I log in, does the search endpoint answer"), not full regression
  suites. Complementary, and cheap.
- **k6-operator** (Apache-2.0) — CRD-driven distributed k6 runs, if we adopt k6.
- **Chainsaw** / **kuttl** — declarative YAML assertions about *Kubernetes resources* after an
  upgrade (did the ConfigMap keys survive, did the CRD version migrate). Not app testing, but a
  cheap complement that catches a real class of upgrade breakage.

## Layer 3 — Triggering on upgrade

This is the part that makes the difference for the actual problem. The junior engineer should
not have to remember to run tests.

- **Argo CD PostSync hook** — if we're on Argo CD, a `Job` annotated
  `argocd.argoproj.io/hook: PostSync` runs automatically after every successful sync. Test
  failure marks the sync failed. Effectively free, and it's the natural fit for "engineer bumps
  an image tag."
  Caveat: a failed PostSync hook does **not** roll back on its own. Pair with a `SyncFail` hook,
  or with an alert, or use Argo Rollouts for real reversion.
- **Flagger** (Apache-2.0, Flux) / **Argo Rollouts** — progressive delivery. The new version goes
  live to a slice of traffic, a webhook runs our test suite as an analysis gate, and the
  controller **automatically reverts** if the gate fails. This is the strongest version of the
  answer: the junior engineer's bad upgrade undoes itself.
- **Testkube event triggers** — trigger a suite on a Kubernetes event, without needing GitOps.
- **CronJob** — nightly full regression as a backstop regardless of what else fires. Catches
  things that only surface after data accumulates.
- **Manual** — a documented one-liner (or a Testkube/Argo UI button) so an engineer can run the
  suite for one app on demand, before and after.

## Layer 4 — Results, reporting, alerting

- **Playwright HTML report / trace** pushed to a self-hosted **MinIO** bucket, link posted to
  chat. Cheapest useful reporting. The trace viewer is the thing that lets a junior engineer see
  *what* broke without reproducing it.
- **Allure Report** (Apache-2.0) + `allure-docker-service` — cross-framework reports with history
  and flakiness trends. Moderate cost, good payoff once there are more than a handful of suites.
- **Prometheus + Grafana + Alertmanager** — pass/fail and duration as time series; alert to
  Slack/Mattermost/email. Kuberhealthy and k6 emit metrics natively; anything else can push to a
  Pushgateway. Likely already in the cluster.
- **ReportPortal** (Apache-2.0) — powerful aggregation and failure clustering, but it wants
  Postgres + RabbitMQ + OpenSearch. That's a real, ongoing infrastructure bill. Skip it unless
  the suite count justifies it.

## Layer 5 — Optional: reducing selector churn with AI

Because we don't own these apps, selectors break on upgrade — and a test that breaks because
the vendor renamed a CSS class is a false positive that erodes trust in the suite.

- **Playwright `codegen`** — the non-AI answer, and the first thing to reach for. Re-record a
  broken flow in a minute instead of debugging selectors.
- **Midscene.js** (MIT, ByteDance) — vision-driven UI automation: steps are natural language
  ("click the login button") and a multimodal model locates the element visually, so the test
  survives a redesign that would break a CSS selector. Supports self-hosted open-weight models
  (UI-TARS, Qwen-VL). **Cost warning:** self-hosting a vision model means GPU nodes, which
  dwarfs the rest of this budget. Using a hosted model API instead is cheap per call but is not
  self-hosted.
- **Pragmatic middle ground:** keep tests deterministic, and use an LLM only for *triage* — on
  failure, feed the diff/trace to a model to draft "likely a real regression" vs "likely a
  selector rename." A handful of calls per failed run costs cents and doesn't touch the hot path.

## Three assembled stacks

### Stack A — Lean (recommended starting point)

Playwright (UI + API + visual) → container image → Argo CD PostSync `Job` + nightly `CronJob`
→ HTML report to MinIO → Alertmanager to chat.

- **Added infrastructure:** none beyond a bucket.
- **Added cost:** burst compute only.
- **Time to first value:** days.
- **Weakness:** no self-service UI; results are links, not a dashboard.

### Stack B — Balanced

Stack A **+ Testkube** (or Argo Workflows) as orchestrator, **+ Schemathesis** on every app that
publishes an OpenAPI spec, **+ Kuberhealthy** for continuous probes, **+ Allure** for history and
flake trends.

- **Added infrastructure:** one operator, one report service.
- **Added cost:** small always-on footprint (~a few hundred millicores / ~1 GB).
- **Buys:** a place juniors can click "run the suite for app X," and trend data that shows
  whether a failure is new.

### Stack C — Full quality gate

Stack B **+ Flagger or Argo Rollouts**, with the test suite wired as an analysis webhook so a
failing upgrade **auto-reverts**, **+ k6** performance gates, **+ oasdiff** as a pre-sync check.

- **Added infrastructure:** a progressive-delivery controller; service mesh or ingress support
  for traffic splitting.
- **Buys:** the upgrade problem stops being a detection problem and becomes a non-event.
- **Weakness:** the most setup, and it changes how deployments work — a bigger ask of the same
  juniors we're trying to help.

## Cost engineering

- Browser pods are the expensive part: budget roughly **1 vCPU and 1.5–2 GB per concurrent
  Chromium instance**. Cap workers rather than letting Playwright autoscale to node core count.
- Put test workloads on a **dedicated spot/preemptible node pool that scales to zero** between
  runs. Test workloads are bursty and interruptible — exactly the spot-instance profile — and
  there's no production data at risk.
- Set `ttlSecondsAfterFinished` on every test `Job`, and prune reports/artifacts on a schedule.
  Screenshot baselines and traces grow quickly.
- Prefer pre-built runner images (the official Playwright image) over installing browsers at pod
  start; it saves minutes and bandwidth per run.
- Resist always-on platforms until the suite count justifies them. ReportPortal-class
  infrastructure can cost more than the tests it reports on.

## Known traps

- **Screenshot flake.** Font rendering, animations, timestamps, and live data all produce diffs
  that aren't regressions. Mitigate by generating baselines *in the same pinned container image*
  that CI runs, disabling animations, freezing the clock, and masking dynamic regions. Pin the
  browser version — an unpinned browser bump invalidates every baseline at once.
- **Third-party apps with no OpenAPI spec.** Schemathesis and oasdiff are then unavailable. Fall
  back to recorded-traffic replay (capture a HAR of a known-good session, replay and assert on
  status codes and response shape).
- **State accumulation.** These apps keep data between runs, so tests drift. Either reset state
  via the app's own API at suite start, or deploy a fresh ephemeral namespace per test run and
  tear it down — the second is cleaner and, with scale-to-zero nodes, not much more expensive.
- **Credentials.** Tests need real logins. Keep them in Kubernetes Secrets (External Secrets or
  Sealed Secrets if we're GitOps), never in the test repo, and never reuse a credential that
  exists anywhere outside this cluster.
- **Baselines as a review artifact.** When a visual diff is an *intended* change, someone must
  approve the new baseline. Make that a pull request, not a `--update-snapshots` run on a
  laptop, or the safety net quietly dissolves.
- **Trust.** A suite that cries wolf gets ignored, and then the upgrade problem is back. Quarantine
  flaky checks into a non-blocking lane rather than letting them fail the gate.

## Recommendation

Build **Stack A** first, against two or three representative apps — one UI-heavy, one API-only,
one that's both. Wire it to Argo CD PostSync so it fires without anyone remembering. That proves
the approach at essentially zero infrastructure cost.

Then add, in this order:

1. **Schemathesis** for every app with a spec — highest ratio of edge cases found to effort spent.
2. **Testkube or Argo Workflows** once the suite count makes bare `Job`s awkward, mainly for the
   self-service UI.
3. **Flagger/Argo Rollouts auto-revert** once the suite is trusted enough to be a gate. Do not
   gate on a suite that still flakes.
