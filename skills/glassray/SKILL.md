---
name: glassray
description: "Send an AI agent's traces to Glassray (app.glassray.ai) over OpenTelemetry and confirm they land. Use when asked to connect or integrate Glassray, send traces to Glassray, point an existing OpenTelemetry / OTLP setup (Python, Node, Ruby, Go, Java, .NET) at Glassray, connect the Vercel AI SDK, Strands Agents, Google ADK, Pydantic AI, OpenLLMetry, OpenInference, OpenLIT or an OpenTelemetry Collector to Glassray, tag traces with glassray.customer / glassray.agent / glassray.flow, debug traces that aren't arriving (401, 403, 415, nothing in the Traces view), or query traces, flows, deviations and LLM cost over the Glassray MCP server."
license: MIT
metadata:
  author: glassray
  version: "0.1.0"
  homepage: "https://glassray.ai/docs/otlp-ingestion"
---

# Glassray over OpenTelemetry

Glassray reads standard OpenTelemetry traces. If an app already emits OTel spans - through
the OTel SDK, an agent framework, a GenAI instrumentation library, or a Collector - it can
send them to Glassray by pointing its exporter at one endpoint. No Glassray library is
required. This skill gets traces flowing from whatever is already there, tags them so the
product can slice them, and proves they arrived.

## 1 · Ground rules

- **Never `git commit` or `git push`.** Leave changes uncommitted for the human to review.
- **Never hardcode the key.** It lives in the environment as `GLASSRAY_API_KEY` (or the
  OTel `OTEL_EXPORTER_OTLP_HEADERS` var); never in source or a committed file.
- **Change the exporter, not the instrumentation.** Reuse the tracer provider, processors
  and instrumentations the repo already has. Add or repoint one exporter; don't rewire.
- **Don't invent.** Every endpoint, attribute and option below is checked against the
  shipped ingest code. Anything else: read the docs first (§10).

## 2 · The endpoint

```
POST https://app.glassray.ai/api/public/otel/v1/traces
Authorization: Bearer <key with traces:write>
Content-Type: application/x-protobuf   or   application/json     (gzip accepted)
```

- OTLP over **HTTP** only. **gRPC is not supported** - an exporter pointed at a `:4317`-style
  gRPC target never connects. Both encodings are first-class; most SDKs default to protobuf,
  the Node `exporter-trace-otlp-http` package sends JSON. Neither needs configuring.
- **Getting the key:** the user adds an **OpenTelemetry** source under **Settings → Sources**
  in Glassray; the key is shown once. If you're connected to the Glassray MCP server (§9)
  with an `mcp:write` key, `connect_otlp_source` creates the source and returns the key
  once (a retry returns `existing: true` without it). One source per environment - the
  key decides which project a trace lands in; no attribute does.

## 3 · Find the starting point

Read the repo before writing anything. What you find decides the section:

| In the repo                                                                              | Do                                                                               | Section |
| ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- | ------- |
| An OTel SDK already set up (`TracerProvider`, `NodeSDK`, `OpenTelemetry::SDK.configure`) | Add or repoint one OTLP/HTTP exporter                                            | §4      |
| An OTel-native agent framework - Vercel AI SDK, Strands Agents, Google ADK, Pydantic AI  | Set the exporter env vars, plus the framework's one-liner                        | §5      |
| A GenAI instrumentation library - OpenLLMetry, OpenInference, OpenLIT                    | Point its exporter at the endpoint                                               | §5      |
| An OpenTelemetry Collector                                                               | Add an `otlphttp/glassray` exporter to its pipeline                              | §5      |
| Deployed on Vercel                                                                       | A Trace Drain, or the env vars                                                   | §5      |
| Langfuse / LangSmith / PostHog already tracing the agent                                 | Don't add OTel - connect it as a pull source (`connect_pull_source` / dashboard) | -       |
| No tracing at all, TypeScript / JavaScript                                               | `@glassray/tracing` is faster: https://glassray.ai/docs/sdk-quickstart           | -       |
| No tracing at all, any other language                                                    | The stock OTel SDK, configured as in §4                                          | §4      |

Whatever the path, §6 (tags) and §8 (verify) are not optional.

## 4 · Point the exporter at Glassray

**Environment variables - the universal route.** Every OTel SDK and most frameworks read
these; the exporter needs no code changes:

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT="https://app.glassray.ai/api/public/otel"   # base URL - the SDK appends /v1/traces
export OTEL_EXPORTER_OTLP_HEADERS="Authorization=Bearer $GLASSRAY_API_KEY"
```

Or the signal-specific form, which is used as-is (no path appended):

```bash
export OTEL_EXPORTER_OTLP_TRACES_ENDPOINT="https://app.glassray.ai/api/public/otel/v1/traces"
```

Don't set both to different values. `OTEL_EXPORTER_OTLP_PROTOCOL` may be `http/protobuf`
(the default) or `http/json`; never `grpc`. Python and Ruby ignore the JSON setting and
send protobuf regardless - that's fine.

**Explicit exporter - when the repo configures its exporter in code.** Add to the existing
provider rather than creating a second one:

```python
# pip install opentelemetry-sdk opentelemetry-exporter-otlp-proto-http
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor

provider = TracerProvider(resource=Resource.create({
    "service.name": "support-bot",
    "glassray.agent": "support-bot",
}))
provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter(
    endpoint="https://app.glassray.ai/api/public/otel/v1/traces",     # full traces URL in code
    headers={"Authorization": f"Bearer {os.environ['GLASSRAY_API_KEY']}"},
)))
trace.set_tracer_provider(provider)   # load-bearing: without it the global tracer is a no-op
```

```ts
// npm i @opentelemetry/api @opentelemetry/sdk-trace-node @opentelemetry/exporter-trace-otlp-proto @opentelemetry/resources
import { BatchSpanProcessor, NodeTracerProvider } from "@opentelemetry/sdk-trace-node";
import { OTLPTraceExporter } from "@opentelemetry/exporter-trace-otlp-proto";
import { resourceFromAttributes } from "@opentelemetry/resources";

export const provider = new NodeTracerProvider({
  resource: resourceFromAttributes({
    "service.name": "support-bot",
    "glassray.agent": "support-bot",
  }),
  spanProcessors: [
    new BatchSpanProcessor(
      new OTLPTraceExporter({
        url: "https://app.glassray.ai/api/public/otel/v1/traces",
        headers: { Authorization: `Bearer ${process.env.GLASSRAY_API_KEY}` },
      }),
    ),
  ],
});
provider.register();
```

Ruby is the same shape with the stock `opentelemetry-exporter-otlp` gem
(`OpenTelemetry::Exporter::OTLP::Exporter.new(endpoint:, headers:)` inside
`OpenTelemetry::SDK.configure`). Go, Java and .NET: the stock OTLP/HTTP exporter with the env
vars above. Full per-language setup: https://glassray.ai/docs/otlp-ingestion.

## 5 · Frameworks, instrumentation libraries, Collector

All of these ride on an OTel provider. Configure the exporter as in §4; then apply the one
framework-specific detail below. **None of them set Glassray's tags** - §6 still applies.

- **Vercel AI SDK.** Two routes. _On Vercel (Pro / Enterprise):_ the user creates a **Vercel**
  source under Settings → Sources, which shows the endpoint URL and one
  `Authorization: Bearer …` header line; in Vercel, **Team Settings → Drains → Add Drain →
  Traces**, Custom Endpoint, **Encoding: JSON**, paste both, Test, Create. Register OTel once
  in `instrumentation.ts` with `registerOTel({ serviceName })` from `@vercel/otel`. _Anywhere
  else (or Hobby plan):_ the §4 env vars in the project with
  `OTEL_EXPORTER_OTLP_PROTOCOL="http/json"`. Either way, enable telemetry on each call:
  `experimental_telemetry: { isEnabled: true, functionId: "handle-ticket", metadata: { customer, agent, flow, userId } }` -
  Glassray reads those `metadata` keys (as `ai.telemetry.metadata.*`) as its tags. On the newer
  `@ai-sdk/otel` + `telemetry:` API spans are standard `gen_ai.*`; tag via resource
  attributes (`OTEL_RESOURCE_ATTRIBUTES=glassray.agent=support-bot`) and root-span attributes.
  **AI routes must run on the Node runtime** - spans from the Edge runtime never reach a
  drain. Details and troubleshooting: https://glassray.ai/docs/vercel.
- **Strands Agents (Python).** Set the §4 env vars, then before running the agent:
  `from strands.telemetry import StrandsTelemetry; StrandsTelemetry().setup_otlp_exporter()`
  (it builds a `BatchSpanProcessor` + `OTLPSpanExporter` from the env vars). Spans carry
  `gen_ai.*`. Per-run tags go on the agent span via `Agent(trace_attributes={...})`.
- **Google ADK (Python).** ADK does not register a provider itself. Set
  `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` (the full traces URL), `OTEL_EXPORTER_OTLP_HEADERS`,
  `OTEL_SERVICE_NAME` and `OTEL_RESOURCE_ATTRIBUTES=glassray.agent=<name>`, then call
  `maybe_set_otel_providers()` from `google.adk.telemetry.setup` before creating the agent.
- **Pydantic AI.** `Agent.instrument_all()` (or `InstrumentationSettings(tracer_provider=…)`
  on one agent) emits `gen_ai.*` spans into whichever provider is registered - configure that
  provider per §4 (env vars or code).
- **OpenLLMetry (Traceloop).** Python: `Traceloop.init(app_name="…",
api_endpoint="https://app.glassray.ai/api/public/otel", headers={"Authorization": f"Bearer {key}"})`;
  Node: `Traceloop.initialize({ appName, baseUrl, headers })`. Give it the **base** URL - it
  appends `/v1/traces`. Env alternative: `TRACELOOP_BASE_URL`, `TRACELOOP_HEADERS=Authorization=Bearer%20<key>`.
- **OpenLIT.** `openlit.init(otlp_endpoint="https://app.glassray.ai/api/public/otel",
otlp_headers={"Authorization": f"Bearer {key}"})`, or just `openlit.init()` with the §4 env vars.
- **OpenInference (Arize instrumentors).** They add spans to the provider you give them:
  build the provider per §4, then `<X>Instrumentor().instrument(tracer_provider=provider)`.
- **OpenTelemetry Collector.** Add an exporter and fan out; the app sends to the Collector
  with no key:

  ```yaml
  exporters:
    otlphttp/glassray:
      traces_endpoint: https://app.glassray.ai/api/public/otel/v1/traces
      headers:
        Authorization: "Bearer ${env:GLASSRAY_API_KEY}"
  service:
    pipelines:
      traces:
        exporters: [otlphttp/glassray, otlphttp/other-tool]
  ```

  `otlphttp`, not the gRPC `otlp` exporter. Keep the `batch` processor.

## 6 · Tag and shape the spans - what Glassray reads

Tags drive every breakdown, filter and cost view; a missing one fails **silently** (the trace
lands and looks fine, the dashboards quietly stop slicing). Treat this as part of wiring.

| Attribute                                                 | Set on        | What it's for                                                                |
| --------------------------------------------------------- | ------------- | ---------------------------------------------------------------------------- |
| `glassray.agent` (and `service.name`)                     | Resource      | Which agent sent the trace - fixed per process                               |
| `glassray.customer`                                       | Root span     | Whose run it was - your customer / tenant id, varies per run                 |
| `glassray.flow`                                           | Root span     | Optional hint for which flow the run belongs to                              |
| `user.id`, `session.id`                                   | Root span     | The end user; the conversation the turn belongs to                           |
| `input.value`, `output.value`                             | Root span     | What the run was asked and what it answered                                  |
| `gen_ai.operation.name`                                   | Each LLM span | Marks an LLM call (`chat`); `execute_tool` + `gen_ai.tool.name` for tools    |
| `gen_ai.provider.name`, `gen_ai.request.model`            | Each LLM span | Provider and the model to price at (`gen_ai.response.model` is the fallback) |
| `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens` | Each LLM span | Token counts → cost                                                          |
| `gen_ai.input.messages`, `gen_ai.output.messages`         | Each LLM span | Prompt and response (JSON strings)                                           |

Rules that matter:

- **Root-span attributes override resource attributes** of the same name. Put what's fixed
  per process on the resource and what varies per run on the root span. Env vars and a
  Collector can only set resource-level values - per-run `glassray.customer` needs code.
- Any other **scalar attribute on the root span or resource** (`ticket_id`, `region`, …)
  becomes a filter in the Traces view - up to 48 per trace, values ≤ 256 chars. Child-span
  attributes and OTel namespaces (`http.*`, `service.*`) don't.
- Use **stable ids or slugs**, never display names: values are matched exactly.
- Prompt-cache tokens: Anthropic-style `gen_ai.usage.cache_read_input_tokens` /
  `cache_creation_input_tokens` are counted _beside_ `input_tokens`; OpenAI-style
  `gen_ai.usage.cached_input_tokens` is counted _inside_ it. Emit one convention per span.
  A known USD cost goes in `glassray.usage.cost` and wins over the estimate.
- Also recognised, no work needed: `gen_ai.system` (older provider spelling), the AI SDK's
  `ai.usage.*`, and `customer_id` / `tenantId` / `merchantId` on traces migrating from another
  tool. `environment` / `deployment.environment.name` are ignored - the key selects the project.

A root span and one LLM span, in Python (the same names in every language):

```python
with tracer.start_as_current_span("handle-ticket", attributes={
    "glassray.customer": ticket.customer_id,
    "glassray.flow": "refunds",
    "user.id": ticket.user_id,
    "session.id": ticket.conversation_id,
    "input.value": ticket.message,
}) as root:
    with tracer.start_as_current_span("draft-reply", attributes={
        "gen_ai.operation.name": "chat",
        "gen_ai.provider.name": "anthropic",
        "gen_ai.request.model": "claude-sonnet-4-6",
        "gen_ai.input.messages": json.dumps(messages),
    }) as llm:
        response = anthropic.messages.create(model="claude-sonnet-4-6", max_tokens=1024, messages=messages)
        llm.set_attributes({
            "gen_ai.usage.input_tokens": response.usage.input_tokens,
            "gen_ai.usage.output_tokens": response.usage.output_tokens,
            "gen_ai.output.messages": json.dumps([{"role": "assistant", "content": response.content[0].text}]),
        })
    root.set_attribute("output.value", response.content[0].text)
```

Nested spans need no parent passed in - a span started while another is active becomes its
child. Full attribute list incl. cost: https://glassray.ai/docs/sdk-byo-otel.

## 7 · Delivery - make sure the spans actually leave

- **Flush before exit.** `BatchSpanProcessor` holds spans until size/time triggers. Node:
  `await provider.shutdown()` at exit, `await provider.forceFlush()` at the end of a serverless
  invocation. Python: `provider.shutdown()` (the Python SDK also shuts down at interpreter
  exit by default). Ruby: `OpenTelemetry.tracer_provider.shutdown`. Frameworks that build the
  provider for you (Strands, ADK) still need this on short-lived processes.
- **Spans of one trace split across requests are merged** on ingest, and re-sending the same
  spans is idempotent - the default batch processor is safe, and retries are safe.
- **Source filters are evaluated per request.** A request that carries only child spans is
  matched on a child's name, so a filter on the trace name can drop those spans before the
  root arrives. Add filters after traces flow, or use the descendant ("any span") scope.
- Limits: a request over **32 MB** is rejected (`413`); a single trace over **16 MiB** is
  skipped while the rest of the batch is stored. Vercel truncates span attributes over ~1 MB.

## 8 · Verify before you call it done

1. **Smoke-test the endpoint and key with a raw request** - it isolates the SDK from the
   network path and tells you which problem you have:

   ```bash
   TRACE_ID=$(openssl rand -hex 16); SPAN_ID=$(openssl rand -hex 8); NOW=$(date +%s)000000000
   curl -sS -o /dev/null -w '%{http_code}\n' -X POST "https://app.glassray.ai/api/public/otel/v1/traces" \
     -H "Authorization: Bearer $GLASSRAY_API_KEY" -H "Content-Type: application/json" \
     -d "{\"resourceSpans\":[{\"resource\":{\"attributes\":[{\"key\":\"service.name\",\"value\":{\"stringValue\":\"smoke-test\"}},{\"key\":\"glassray.agent\",\"value\":{\"stringValue\":\"smoke-test\"}}]},\"scopeSpans\":[{\"scope\":{\"name\":\"smoke-test\"},\"spans\":[{\"traceId\":\"$TRACE_ID\",\"spanId\":\"$SPAN_ID\",\"name\":\"smoke-test\",\"kind\":1,\"startTimeUnixNano\":\"$NOW\",\"endTimeUnixNano\":\"$NOW\",\"attributes\":[{\"key\":\"glassray.customer\",\"value\":{\"stringValue\":\"smoke\"}}],\"status\":{\"code\":1}}]}]}]}"
   ```

   `200` = key and endpoint are right; go look for the `smoke-test` trace. Anything else, see
   the error table below.

2. **Run the app once** so it produces a real trace, then confirm it arrived **with its tags**:
   the **Traces** view in Glassray (the source flips to _Connected_), or over MCP (§9)
   `get_setup_status` → `list_traces` → `get_trace` and read the tags on the trace.
3. **Nothing arrived?** Turn on the OTel SDK's own diagnostics (`OTEL_LOG_LEVEL=debug`, or a
   console span exporter alongside the OTLP one) and read the exporter's status code:

   | Code              | Meaning                                                                                      |
   | ----------------- | -------------------------------------------------------------------------------------------- |
   | `400`             | Body isn't valid OTLP for its `Content-Type`, or bad gzip                                    |
   | `401`             | Key missing or invalid - check the env var reaches the process, and the header spelling      |
   | `403`             | Key lacks `traces:write`, the source is **disabled**, or ingest is suspended for the org     |
   | `404`             | Almost always a doubled path: `OTEL_EXPORTER_OTLP_ENDPOINT` set to the full `/v1/traces` URL |
   | `413`             | Request over 32 MB - batch smaller                                                           |
   | `415`             | Wrong `Content-Type` / `Content-Encoding` - gRPC exporter, or a non-gzip encoding            |
   | `503`             | Temporary; retry is safe                                                                     |
   | no request at all | Process exited before flush (§7); a gRPC exporter that never connected; Vercel Edge runtime  |

   Full checklist: https://glassray.ai/docs/sdk-troubleshooting.

## 9 · Query and act on the data - the Glassray MCP server

Don't guess at REST endpoints:

```bash
claude mcp add --transport http glassray https://app.glassray.ai/api/public/mcp
```

OAuth by default; headless: `--header "Authorization: Bearer <org-key>"` with a key from
**Settings → API keys → MCP access** (`mcp:read` for reads, `mcp:write` to act). That org key is
deliberately different from the `traces:write` ingest key.

- **Read:** `list_traces` / `get_trace`, `list_flows` / `get_flow` / `list_flow_traces`,
  `list_deviations` / `get_deviation`, `list_trace_sources` / `list_sync_jobs`,
  `get_setup_status`, `get_run_status`, `cost_rollup` (spend by model / customer / user / flow).
- **Ask:** `find_traces` → `analyze_traces` → `refine_traces`; carry the set of trace ids
  between calls.
- **Act:** `connect_otlp_source` / `connect_pull_source`, `trigger_trace_sync`,
  `run_deviation_discovery` / `run_deep_search` (metered; poll `get_run_status`),
  `create_deviations`, `update_deviation_status`, flow management. `delete_traces` is
  admins-over-OAuth only and two-step - always show its impact report and get an explicit yes.

Other clients and the full tool reference: https://glassray.ai/docs/mcp-server.

## 10 · Reading the docs

- https://glassray.ai/docs/otlp-ingestion - the endpoint, per-language setup, what Glassray reads.
- https://glassray.ai/docs/sdk-byo-otel - the full attribute list, usage and cost conventions.
- https://glassray.ai/docs/vercel - Trace Drains and the AI SDK.
- https://glassray.ai/docs/traces - sources, filters, and how pulled and pushed traces meet.
- https://glassray.ai/docs/llms.txt (index) · https://glassray.ai/docs/llms-full.txt (everything).
- Docs-search MCP, unauthenticated: `claude mcp add --transport http glassray-docs https://glassray.ai/docs/mcp`
