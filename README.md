# stdlib-jsonlog

**Hardened, stdlib-first JSON logging for production Python services** — with bounded serialization, context propagation, non-blocking remote transport, backpressure shedding, optional TCP/TLS streaming, hot-reloadable log levels, and introspection health stats.

> **Mental model:** this is not “just a formatter.”
> It treats logging as a **loss-tolerant transport pipeline** embedded in-process.

---

## Why this exists

Most Python logging setups solve formatting.
Production systems need more than formatting:

* log events must be **structured**
* log emission must be **bounded** under pathological inputs
* remote collectors must be **optional** and **never stall app threads**
* failure modes must be **predictable** under backpressure / network flaps
* operators need **health visibility** into the logging subsystem itself

`stdlib-jsonlog` is a **correctness- and ops-biased** logging mini-agent that stays **stdlib-first** and is designed to **degrade gracefully** rather than block or crash your app.

<img width="929" height="940" alt="RLNlJoCt4Fs-VyM8VYYv8mgbFrITJkWWHu0MI9f4lIz8rSIUtVN5iTsnbv1M_TyxzYKekGC9rjXldj-yl9ryY0avTIjR9PCiGKpQcaN_S_JE-WQ2HKKhSWcC0uIq2KQX1Gm1tifMpGW5PW9PuwYgujAuW8mhY2rglpQkWHfjfBqJvLV1Eo6TfOqbO589NMFt8Ml6yCderCBPsUdU_WgVx" src="https://github.com/user-attachments/assets/33ec6b3c-e3ab-4fc0-98d4-ac0ac8f22cbb" />


## What it gives you

### Core capabilities

* **Strict JSON logs** (ISO-8601 timestamps w/ ms + timezone)
* **Context propagation** via `contextvars` (`trace_id`, `span_id`, `correlation_id`)
* **Structured extras** with deep sanitization + truncation limits
* **Secret redaction** (key-based, optional free-text redaction)
* **High-res emit-time timestamps** (`time_ns` + cached ISO formatter)
* **Idempotent initialization** (safe re-init, handler tagging/dedupe)
* **Config validation** (env-driven, fail-fast for invalid settings)

### Throughput & resilience

* **Non-blocking queueing for remote streaming path**
* **Explicit shedding** (`drop-oldest` / `drop-newest`) under queue pressure
* **Accurate queue depth accounting** (ingest + network)
* **Async sender thread** with:

  * connect/send timeouts
  * exponential backoff + jitter
  * DNS caching
  * dynamic batching
  * protocol fallback (`v1` ⇄ `len`)
* **Optional disk spool** for transient network outages
* **Graceful shutdown drain** (`flush_and_drain`, signal handlers)

### Operations & control plane

* **Hot reload** of:

  * root level (K8s file)
  * per-logger levels (JSON control file)
* **Watcher backoff** (avoids noisy errors when config files flap/misbehave)
* **Introspection APIs**:

  * `get_log_stats()`
  * `is_streaming_enabled()`
  * `is_streaming_healthy()`
* **Thread hygiene checks** (leak-ish background thread detection)

---

## Who should use this

This is built for teams running **real production services**, especially:

* **High-throughput APIs / event processors / realtime pipelines**
* **Kubernetes/container fleets** with centralized log shipping
* **Unreliable networks / flapping collectors**
* **PII/security-sensitive environments**
* **Multi-tenant services** with untrusted/adversarial input objects
* **Tracing-heavy systems** that need consistent correlation fields
* **Platform teams** standardizing JSON logs without forcing large deps
* **Batch jobs / CLIs** that want hardened JSON logging with minimal setup

### When it’s overkill

If you just need local console logs for scripts/small apps, this is probably too much.
Use `logging.basicConfig()` or `init_simple_logging()` and skip streaming/watcher features.

---

## Architecture

```text
Application Code
       ↓
QueueHandler (non-blocking, with pre-error ring buffer)
       ↓
Queue (with shedding: drop-oldest or drop-newest)
       ↓
QueueListener
       ↓
DepthProxyHandler (tracks queue depth accurately)
       ↓
Network Handler (TCP/TLS with framing)
       ↓
Async Sender Thread (with exponential backoff)
       ↓
Remote Log Aggregator
```

> Local stdout/stderr/file handlers remain normal logging handlers.
> The **remote streaming path** is where the non-blocking queue/async transport applies.

---

## Installation

This library is **stdlib-first** and works without third-party dependencies.

### Optional acceleration

* [`orjson`](https://pypi.org/project/orjson/) (if installed) is used automatically for faster JSON serialization unless disabled.

### Typical usage patterns

* **Vendored module** in a shared platform repo
* **Internal package** installed in multiple services

---

## Quick start

### 1) Minimal hardened JSON logging (no watcher / no network)

```python
import logging
from stdlib_jsonlog import init_simple_logging, shutdown_logging

init_simple_logging(level="INFO")

log = logging.getLogger("demo")
log.info("service started", extra={"extra_fields": {"port": 8080, "mode": "dev"}})

shutdown_logging()
```

Use this for CLIs, batch jobs, or local dev.

---

### 2) Full initialization (env-driven settings)

```python
import logging
from stdlib_jsonlog import init_logging, shutdown_logging

init_logging()  # reads env, validates config, installs handlers

log = logging.getLogger("myapp")
log.info("ready")

shutdown_logging()
```

---

### 3) Context propagation (trace/span/correlation)

```python
import logging
from stdlib_jsonlog import init_logging, trace_context

init_logging()
log = logging.getLogger("payments")

with trace_context(trace_id="t-123", span_id="s-abc", correlation_id="corr-42"):
    log.info("charge requested")
```

---

### 4) Structured event logging helper (`FastEventLogger`)

```python
from stdlib_jsonlog import init_logging, FastEventLogger

init_logging()

ev = FastEventLogger("orders").bind(service="checkout", component="pricing")
ev.info("quote_generated", order_id="o-123", amount=1999, currency="USD")
```

---

## Example JSON output

```json
{
  "version":"1.0.0",
  "service_id":"app",
  "instance_id":"d14c8e4c5a9f0c11",
  "container_name":"business-app",
  "host":"pod-7b9c4",
  "timestamp":"2026-02-25T14:21:33.417+00:00",
  "severity":"INFO",
  "logger":"orders",
  "module":"checkout",
  "function":"create_order",
  "file":"checkout.py",
  "line":118,
  "process":123,
  "thread":140120807184128,
  "message":"quote_generated",
  "trace_id":"t-123",
  "span_id":"s-abc",
  "correlation_id":"corr-42",
  "order_id":"o-123",
  "amount":"1999",
  "currency":"USD"
}
```

---

## Design goals

### 1) **Bounded under bad inputs**

Logging should not explode when someone logs a giant nested object, request body, or weird type.

This library bounds and sanitizes:

* extra field count
* per-value size
* structured nesting depth
* total structured nodes/items
* stack frame count
* stack string length

### 2) **Loss-tolerant under pressure**

When buffers fill or collectors fail, the system should:

* **shed predictably**
* **keep app threads moving**
* **surface health/degradation stats**

### 3) **Ops-first behavior**

* validated config
* hot-reload levels
* backoff for noisy watchers and network retries
* introspection for alerting/debugging
* graceful drain on shutdown

---

## Configuration (env-driven)

The library is configured via `LOG_*` environment variables (with a few compatibility/deprecated aliases supported).

Below are the **most important** knobs. (There are many more.)

### Basic output

* `LOG_STDOUT=true|false` — emit JSON to stdout
* `LOG_STDERR=true|false` — emit JSON to stderr
* `LOG_FILE=/path/to/file.log` — optional rotating file
* `LOG_LEVEL=INFO|DEBUG|...` — root level
* `TZ=UTC` — timestamp timezone

### Remote streaming

* `LOG_STREAM=true|false` — enable remote streaming
* `LOG_STREAM_HOST=log-transformer`
* `LOG_STREAM_TCP_PORT=5025`
* `LOG_STREAM_TLS_PORT=5024`
* `LOG_STREAM_PROTOCOLS=v1,len` — fallback preference order
* `LOG_STREAM_CONNECT_TIMEOUT_SEC=5`
* `LOG_STREAM_SEND_TIMEOUT_SEC=1.0`

### Queue / backpressure

* `LOG_STREAM_MAX_QUEUE_SIZE=5000`
* `LOG_STREAM_DROP_OLDEST=true|false` — ingress queue shedding policy
* `LOG_STREAM_NET_DROP_OLDEST=true|false` — sender queue shedding policy
* `LOG_PRE_ERROR_BUFFER_SIZE=64` — flush pre-error ring on first ERROR+
* `LOG_PRE_ERROR_BUFFER_LEVEL=DEBUG`

### TLS

* `LOG_STREAM_TLS_ENABLED=true|false`
* `LOG_STREAM_ENFORCE_TLS=true|false`
* `LOG_STREAM_CERT_FILE=...`
* `LOG_STREAM_KEY_FILE=...`
* `LOG_STREAM_CA_FILE=...`
* `LOG_STREAM_TLS_MIN_VERSION=1.2|1.3`
* `LOG_ALLOW_INSECURE=true|false` — explicit fallback permission
* `LOG_STREAM_TLS_FALLBACK_RETRY_SEC=300`

### Sampling / rate control

* `LOG_MAX_LOGS_PER_SEC=0` — 0 disables global rate limit
* `LOG_MAX_LOGS_BURST=0`
* `LOG_SAMPLE_DEBUG=1.0`
* `LOG_SAMPLE_INFO=1.0`
* `LOG_SAMPLE_WARNING=1.0`
* `LOG_SAMPLE_ERROR=1.0`
* `LOG_SAMPLE_CRITICAL=1.0`
* `LOG_ADAPTIVE_SAMPLING=true|false`

### Redaction / safety bounds

* `LOG_REDACT_SECRETS=true|false`
* `LOG_REDACT_KEYS=password,token,authorization,...`
* `LOG_REDACT_TEXT=true|false`
* `LOG_REDACT_TEXT_MIN_LEVEL=ERROR`
* `LOG_MAX_EXTRA_FIELDS=256`
* `LOG_MAX_EXTRA_VALUE_CHARS=2048`
* `LOG_MAX_STACK_CHARS=16384`
* `LOG_MAX_STACK_FRAMES=64`
* `LOG_MAX_STRUCTURED_EXTRA_ITEMS=256`
* `LOG_MAX_STRUCTURED_EXTRA_NODES=1000`

### Watcher / hot reload

* `WATCH_LOG_CONF=true|false`
* `LOG_REFRESH_TIME=5`
* `LOG_CONF_K8S_PATH=/etc/config`
* `LOG_CONTROL=/etc/adp/logcontrol.json`

---

## Hot-reload control plane

This library supports **runtime log level changes** without redeploying.

### A) Root level from K8s file

Reads a file like:

`/etc/config/LOG_LEVEL`

Example file contents:

```text
DEBUG
```

### B) Root + per-logger levels from JSON control file

Reads a JSON document (default path: `/etc/adp/logcontrol.json`).

Example:

```json
{
  "logcontrol": [
    {
      "container": "business-app",
      "severity": "INFO",
      "loggers": {
        "myapp.db": "WARNING",
        "myapp.http": "DEBUG",
        "urllib3": "ERROR"
      }
    }
  ]
}
```

Watcher reads are **bounded** and **backoff-protected** so broken/missing files don’t spam logs.

---

## Public API

### Initialization / shutdown

* `init_logging(settings: Optional[LoggingSettings] = None) -> None`
* `init_simple_logging(level=INFO, stdout=True, stderr=False) -> None`
* `flush_and_drain(timeout_sec=5.0) -> bool`
* `shutdown_logging() -> None`
* `install_shutdown_handlers(flush_timeout_sec=8.0, exit_on_term=True) -> None`

### Runtime control

* `set_log_level(level) -> None`

### Context propagation

* `set_trace_context(trace_id=None, span_id=None, correlation_id=None) -> None`
* `clear_trace_context() -> None`
* `trace_context(...)` (context manager)

### Introspection / health

* `get_log_stats() -> dict`
* `is_streaming_enabled() -> bool`
* `is_streaming_healthy() -> bool`

### Structured helper

* `FastEventLogger`

---

## Introspection & health

You can inspect the logging subsystem itself at runtime:

```python
from stdlib_jsonlog import get_log_stats, is_streaming_healthy

print(get_log_stats())
print("stream healthy:", is_streaming_healthy())
```

### Useful counters/fields (from `get_log_stats()`)

Examples include:

* `records_seen`
* `records_attempted`
* `formatter_calls`
* `dropped_lines`
* `dropped_rate_limited`
* `dropped_sampled`
* `dropped_shutdown`
* `dropped_by_severity`
* `dropped_by_logger_top`
* `queue_ingest_cur_depth`
* `queue_net_cur_depth`
* `queue_cur_depth_total`
* `stream_dns_failures`
* `stream_connect_failures`
* `stream_send_failures`
* `current_backoff_seconds`
* `last_stream_error`
* `last_stream_ok_at`
* `watcher_heartbeat_at`

This is ideal for:

* dashboards
* alerts
* “why are logs missing?” investigations

---

## Failure semantics (important)

This library is intentionally **loss-tolerant**, not a guaranteed delivery bus.

### What happens when things go wrong?

* **Collector unreachable / network flapping**

  * sender backs off with jitter
  * app threads continue
  * local handlers keep working
  * optional disk spool can buffer remote frames

* **Queues fill up**

  * records are shed based on configured policy
  * drop counters increment
  * depth remains measurable

* **TLS misconfigured**

  * behavior depends on `LOG_STREAM_ENFORCE_TLS` and `LOG_ALLOW_INSECURE`
  * can fail fast (strict) or degrade to TCP (explicitly allowed)

* **Bad config / invalid settings**

  * validation raises early instead of running in a half-broken state

> The system is designed to **degrade predictably**, not silently wedge your service.

---

## Security & safety posture

### Secret redaction

Key-based secret redaction masks values for keys like:

* `password`
* `token`
* `authorization`
* `api_key`
* `cookie`
* etc.

Configurable via `LOG_REDACT_KEYS`.

### Optional free-text redaction

Can redact bearer tokens and key/value patterns in message/stack/exception text:

* controlled by `LOG_REDACT_TEXT`
* gated by minimum level (`LOG_REDACT_TEXT_MIN_LEVEL`)

### Bounded structured extras

Structured extras are deeply sanitized with:

* cycle detection
* max depth
* node/item budgets
* string truncation
* safe coercion of odd types

This is especially important on untrusted input surfaces.

---

## Performance notes

This library is designed to be **fast on the hot path**, with optimizations such as:

* cached ISO timestamp formatting
* optional `orjson`
* bytes-first formatting for stdio/network handlers
* dynamic network batching
* per-thread RNG for sampling decisions
* bounded sanitizer budgets to avoid pathological recursion/work

That said, performance depends on:

* log volume
* log payload sizes
* structured extra complexity
* enabled features (redaction, sampling, streaming, spool, watcher)

Benchmark in your own workload before making hard throughput claims.

---

## Graceful shutdown (Kubernetes-friendly)

For services that care about draining logs on exit:

```python
from stdlib_jsonlog import init_logging, install_shutdown_handlers

init_logging()
install_shutdown_handlers(flush_timeout_sec=8.0)
```

On `SIGTERM` / `SIGINT`, the library will attempt a bounded drain + shutdown so in-flight logs are less likely to be lost during rollouts or job completion.

---

## Recommended deployment modes

### 1) **CLI / Batch / Local tool**

Use:

* `init_simple_logging()`
* stdout JSON only
* no watcher
* no remote streaming

### 2) **Typical service**

Use:

* stdout JSON
* optional remote streaming
* secret redaction on
* bounded extras defaults
* shutdown handlers

### 3) **High-throughput production**

Enable/tune:

* streaming queue sizes + shedding policy
* sampling / adaptive sampling
* disk spool
* dynamic batching
* watcher-based log level control
* dashboards on `get_log_stats()`

## What this is not

* Not a full observability SDK
* Not an OTLP exporter
* Not a guaranteed-delivery message queue
* Not a replacement for centralized collectors/agents in every deployment

It is a **hardened in-process logging subsystem** that improves behavior at the app boundary.

## Example: Full-service setup (practical)

```python
import logging
from stdlib_jsonlog import (
    init_logging,
    install_shutdown_handlers,
    trace_context,
    FastEventLogger,
    get_log_stats,
)

init_logging()
install_shutdown_handlers(flush_timeout_sec=8.0)

log = logging.getLogger("service")
ev = FastEventLogger("service.events").bind(service="checkout")

with trace_context(trace_id="trace-1", span_id="span-root", correlation_id="corr-99"):
    log.info("http request received", extra={"extra_fields": {"path": "/checkout", "method": "POST"}})
    ev.info("order_created", order_id="ord_123", amount=4999, currency="USD")

# e.g. expose this in a debug/admin endpoint
stats = get_log_stats()
log.info("logging health snapshot", extra={"extra_fields": {"logging_stats": stats}})
```

## Operational checklist

Before production rollout, verify:

* [ ] JSON logs parse correctly in your collector
* [ ] `trace_id` / `span_id` / `correlation_id` appear where expected
* [ ] secret redaction covers your org’s sensitive keys
* [ ] queue sizes and shedding policy match your traffic profile
* [ ] streaming TLS mode matches your security requirements
* [ ] `get_log_stats()` is scraped / visible somewhere
* [ ] `flush_and_drain()` works within your shutdown budget
* [ ] hot-reload watcher files/paths exist and are mounted correctly (if enabled)

