# Handover: OpenTelemetry tracing for the Rails 8 blueprint

> Working note for the next Claude Code session. **Delete this file before opening the PR.**

## Context

- Upstream PR: palladius/rails8-app-on-gcp#240 (`feat(logging): correlate request logs with Cloud Trace`, refs #230).
  Its head is `b56a6ae` on fork branch `20261119_richard-kramer_cloud-trace-correlation`.
- This branch (`claude/user-request-tracing-nfvqaq`) starts at `b56a6ae`, so #240's changes are the baseline.
- Decision taken in the cloud session: the blueprint should show **best practice for Rails developers**,
  so go the whole way with **OpenTelemetry (W3C Trace Context)** instead of growing the hand-rolled middleware.
  #240 gets superseded by PR 1 below.

## What #240 does (baseline)

- `blog/lib/cloud_trace_context.rb`: Rack middleware, parses `X-Cloud-Trace-Context` or `traceparent` into `Current.trace_id`.
- `blog/config/initializers/cloud_trace.rb`: inserts it after `ActionDispatch::RequestId`.
- `blog/lib/google_json_formatter.rb` (pre-existing): writes `logging.googleapis.com/trace = projects/$GOOGLE_CLOUD_PROJECT/traces/$ID`.
  `GOOGLE_CLOUD_PROJECT` is set by `cloudbuild.yaml` and `iac/cloudrun.tf`.
- Removes `CloudErrorReportingMiddleware` (multi-line STDERR output = one log entry per line; duplicated Error Reporting groups).

Gaps found in review:
1. No `user.id` / `session.id`: one request is correlated, a user's journey across requests is not.
2. No `logging.googleapis.com/spanId` / `trace_sampled`.
3. Solid Queue jobs (`GenerateCoverImageJob`, `PodcastifierJob`, `worker` container) lose all context.
4. No DB visibility (SQL not logged at `info`; no spans; no sqlcommenter).
5. Browser errors are not correlated at all.
6. `config.log_tags = [:request_id]` is only used with `LOG_FORMAT=plain` (`blog/config/environments/production.rb`).

## Target architecture

- **App:** `opentelemetry-sdk` + instrumentations (Rack, Rails/ActionPack, ActiveRecord, PG, ActiveJob, Net::HTTP/Faraday)
  exporting **OTLP to `localhost:4318`**. No Google-specific gem in the app, no keys.
- **Cloud Run:** OTel Collector **sidecar** (Google's recommended pattern), auth via ADC (runtime SA needs
  `roles/cloudtrace.agent`), exports to Cloud Trace. Fits the existing multi-container story (`web`, `worker`, `cloudsql-proxy`).
- **Logs:** stay structured JSON on STDOUT (Google's recommendation; Ruby OTel logs SDK not stable).
  `GoogleJsonFormatter` reads `trace_id`, `span_id`, `trace_sampled` from `OpenTelemetry::Trace.current_span`
  and writes `logging.googleapis.com/{trace,spanId,trace_sampled}` plus `logging.googleapis.com/labels`
  (`user_id`, `session_id`, and `job_class`/`job_id` in jobs).
- **User/session:** span attributes `user.id`, `session.id` (OTel semconv) + log labels. IDs only; never e-mail,
  never the cookie value. `Current.session` is set in `Authentication#resume_session` (`blog/app/controllers/concerns/authentication.rb`),
  so early lines ("Started GET") have no user label but share the trace.
- **Jobs:** `opentelemetry-instrumentation-active_job` propagates context; decide `propagation_style` (`:link` default vs `:child`).
  The `worker` process must run the same SDK initializer.
- **DB:** AR/PG spans. Query Log Tags with `query_log_tags_format = :sqlcommenter` for Cloud SQL Query Insights.
  Start with `controller`, `action`, `job` only; per-request `traceparent` tag makes every SQL string unique
  (measure prepared-statement / Query Insights impact before adding).
- **Error Reporting:** unchanged mechanism (Rails' JSON exception line), now also with `spanId`.
- **500 page:** show the trace id as "error reference".
- **Browser:** *not* the OTel Web SDK (experimental, heavy, needs a public collector = security-review finding).
  Instead: small JS that adds `traceparent` on `turbo:before-fetch-request`, and posts `window.onerror` /
  `unhandledrejection` to a Rails endpoint `/client_errors`, which logs it under the page's trace with user/session labels
  (`@type: type.googleapis.com/google.devtools.clouderrorreporting.v1beta1.ReportedErrorEvent` for Error Reporting).
  Rate-limit the endpoint and check CSP (`blog/config/initializers/content_security_policy.rb`).

## Constraints from AGENTS.md / CONSTITUTION

- **Localhost invariant:** works offline without GCP. Default `OTEL_TRACES_EXPORTER=none` locally and in tests.
  Optional `docker compose --profile tracing` with Jaeger for a local waterfall (optional, so no "forking path").
- **Tests < 5 s**, TDD (failing test first). Exporter off in tests; use the in-memory span exporter for assertions.
- **No API keys**, ADC only. English for all repo content. SemVer bump in `VERSION` (currently `0.3.22`) + `CHANGELOG.md`.
- Never edit generated files under `workshop/build/`; workshop prose goes into `workshop/CODELAB.md` (see `workshop/AGENTS.md`).
- Ask: would a senior Rails dev cringe / a GCP SRE fail a security review?

## Open questions to verify first

1. Does Cloud Run (GFE) always send `traceparent`, or only `X-Cloud-Trace-Context`? If the latter, a small custom
   propagator is needed (or keep `CloudTraceContext` as fallback extractor).
2. Sampling: `parentbased_always_on` honouring Cloud Run's decision vs. ratio sampler; Cloud Trace cost.
3. Collector config: `googlecloud` exporter vs OTLP to `telemetry.googleapis.com`; image pinning; health check/startup order.
4. Deploy path: `cloudbuild.yaml` uses `gcloud run deploy` (single container?), `iac/cloudrun.tf` has one container,
   `blog/compose.prod.yaml` has `web`/`worker`/`cloudsql-proxy`. Find out which is canonical for sidecars before adding the collector.
5. Boot-time / test-time impact of the gems.

## Plan (three PRs)

1. **OTel core** (supersedes #240): gems, `config/initializers/opentelemetry.rb`, instrumentations, formatter reads OTel
   context + `spanId`/`trace_sampled`/labels, `user.id`/`session.id`, error reference on 500 page,
   remove `CloudTraceContext` unless needed as fallback. Tests for formatter, attributes, job propagation.
2. **Collector sidecar**: `iac/` + `compose.prod.yaml` + IAM role, local Jaeger profile, workshop chapter in `workshop/CODELAB.md`.
3. **Browser correlation + sqlcommenter / Query Insights**.

## Useful commands

```bash
git fetch origin 20261119_richard-kramer_cloud-trace-correlation   # #240 head
cd blog && bin/rails test test/lib test/controllers/booms_controller_test.rb
```
