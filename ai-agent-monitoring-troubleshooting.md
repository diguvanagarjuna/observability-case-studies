# Troubleshooting AI Agent Monitoring in Splunk Observability Cloud

A practical reference for diagnosing common issues when AI/LLM application telemetry
doesn't show up correctly in Splunk Observability Cloud's AI Monitoring pages
(AI Overview, AI Agents, AI Trace Data). Based on OpenTelemetry GenAI semantic
conventions and general observability troubleshooting patterns — not tied to any
specific customer or case.

## Why AI Agent Monitoring behaves differently from normal APM

Standard APM assumes one instrumentation library per service, and one clear
"is it healthy" signal (latency, error rate). AI/LLM applications break both
assumptions:

- Multiple instrumentation libraries can participate in a single request — your
  agent framework (e.g., LangChain), your model client SDK (e.g., a Bedrock/OpenAI
  client), and sometimes auto-instrumentation for the underlying HTTP library
  (e.g., botocore, requests) can all generate spans for the same call.
- Different UI pages read different signals. AI Overview, AI Agents, and AI Trace
  Data don't necessarily consume the same telemetry — one can look healthy while
  another is empty.
- "The trace exists" does not mean "the required metrics and dimensions exist."
  Traces and metrics are separate pipelines, and a technically complete trace can
  still be missing the specific `gen_ai.*` dimensions a UI page depends on.

Keep this principle in mind throughout: **traces present ≠ AI Agents metrics present.**

---

## Step 1: Confirm the basics

Before diving into instrumentation details, verify:
- AI Agent Monitoring is enabled for your org/environment
- Your AI conversation data source is understood (Splunk Observability Cloud vs.
  Splunk Enterprise/Cloud Platform) — this changes which evaluation features are
  available
- The Collector is running with OTLP receiver enabled, and both traces and metrics
  pipelines are active

---

## Step 2: Fix `unknown_service`

If your application shows up as `unknown_service`, the runtime environment is
missing service identity:

```bash
export OTEL_SERVICE_NAME=my-ai-application
export OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production
```

Common mistake: setting these in a shell session that isn't actually the process
running the application (e.g., set in a parent shell but not inherited by a
containerized process, or set after the app already started). Restart the
application after setting these so new telemetry carries the correct identity.

---

## Step 3: Validate GenAI instrumentation configuration

For Python applications using OpenTelemetry GenAI instrumentation, key environment
variables to check:

```bash
# Ensures span-based metrics are emitted, not just traces
export OTEL_INSTRUMENTATION_GENAI_EMITTERS=span_metric

# Controls whether prompt/response content is captured, and where
export OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=SPAN_ONLY
```

If you plan to use platform-side evaluations (run inside Splunk Observability Cloud
after ingestion), message content generally needs to be captured at the span level.
If you're doing instrumentation-side evaluation instead, you may need content on
both spans and events — check current product documentation, since these behaviors
evolve.

---

## Step 4: Verify agent/workflow metadata is on spans — not just in your code

`agent_name` and `workflow_name` (or their `gen_ai.agent.name` /
`gen_ai.workflow.name` equivalents) need to actually propagate onto the emitted
spans. It's easy to assume that because your code references an agent name, it's
automatically present in telemetry — verify this directly by inspecting a live
trace, not by reading your application code.

---

## Step 5: Understand why AI Agents page can be empty while AI Overview works

This is one of the most common points of confusion. If:

- AI Overview shows data
- AI Trace Data shows traces
- AI Agents page is empty or incomplete

...the issue is almost always **missing metric dimensions**, not missing
telemetry. The AI Agents page depends on specific dimensions being present on
specific metrics — commonly:

```
gen_ai.agent.name
gen_ai.workflow.name
gen_ai.operation.name
gen_ai.request.model
gen_ai.provider.name
deployment.environment
```

A trace can be complete and valid while the corresponding metric is missing one
of these dimensions — which is enough to keep it off the AI Agents page even
though the raw trace data is fine.

**How to find the actual cause:** identify which instrumentation library is
generating the LLM/model-client spans specifically (as opposed to the
agent/workflow-level spans). It's common for an agent framework (e.g., LangChain)
and an underlying HTTP/SDK auto-instrumentation (e.g., for the cloud provider's
SDK) to both be active — and for the auto-instrumentation layer to not emit the
GenAI-specific dimensions the AI Agents page needs, even though it produces a
technically valid span.

**General resolution pattern:** where a purpose-built GenAI instrumentation
exists for your model provider/client, prefer it over generic auto-instrumentation
for that same library, and disable the overlapping generic instrumentation to
avoid duplicate/conflicting spans for the same call. The exact package and
environment variable will depend on your provider and current library versions —
check current instrumentation documentation for your specific stack rather than
assuming a fixed combination, since supported packages change over time.

---

## Step 6: Validate independently — don't rely on one tool alone

A useful pattern when troubleshooting:

1. Check Metric Finder — does the metric exist at all?
2. If yes, query it directly (e.g., via SignalFlow) with the exact dimension
   filters the UI page would use.

If the metric exists in Metric Finder but a direct query with the expected
dimension filter returns nothing, that's a strong signal you're missing a
required dimension — not that ingestion is broken.

---

## Step 7: Cost metrics — a separate concern from telemetry

If token/latency metrics work but cost estimates don't appear, check:

```
gen_ai.request.model
gen_ai.provider.name
gen_ai.client.token.usage
```

Cost calculation depends on the reported model name being recognized by the
platform's cost engine. A model reported as `unknown_model`, or a genuinely
unsupported/uncommon model, will prevent cost calculation — treat this as a
model-support/product limitation rather than an instrumentation bug, and confirm
against current supported-model documentation before spending time debugging
instrumentation further.

Also worth remembering: cost shown by AI Agent Monitoring is an estimate based
on token counts and published pricing — it is not the same as your actual
provider billing.

---

## Step 8: Historical data caveat

If you fix instrumentation and start seeing correct metrics going forward, don't
assume historical traces retroactively have the missing metrics too. Traces and
metrics are separate data, generated at the time telemetry was emitted — a trace
from before your fix will not gain the dimensions your fix now produces. When
comparing "before vs after," compare against the actual deployment timestamp of
your instrumentation change.

---

## Quick Validation Checklist

- [ ] AI Agent Monitoring enabled for the environment
- [ ] Correct AI conversation data source identified
- [ ] Collector running, OTLP receiver + traces/metrics pipelines active
- [ ] `OTEL_SERVICE_NAME` / `OTEL_RESOURCE_ATTRIBUTES` correctly set on the actual running process
- [ ] `OTEL_INSTRUMENTATION_GENAI_EMITTERS=span_metric` set where full telemetry is needed
- [ ] `agent_name` / `workflow_name` verified present on live spans (not just assumed from code)
- [ ] Identified which instrumentation library generates LLM/client spans specifically
- [ ] Checked for overlapping/duplicate instrumentation on the same call path
- [ ] Verified required `gen_ai.*` dimensions on the actual metrics (not just trace presence)
- [ ] Cross-checked Metric Finder existence vs. direct dimension-filtered query results
- [ ] Confirmed model name/provider are populated and supported, if cost metrics are expected
- [ ] Compared "before/after" data using actual deployment timestamps, not assumptions

---

## Key Principle to Remember

AI Overview, AI Agents, and AI Trace Data pages do not necessarily consume the
same telemetry. When one page works and another doesn't, don't assume ingestion
is broken — validate the specific metrics and dimensions that page depends on,
and identify which instrumentation source is actually producing your LLM/agent
telemetry.

---

*This guide reflects general OpenTelemetry GenAI semantic conventions and common
troubleshooting patterns. Exact package names, environment variables, and
supported models change over time — always cross-check against current official
documentation for your specific stack and provider before applying fixes in
production.*

*Maintained by D Nagarjuna — Full Stack Observability Consultant.*
