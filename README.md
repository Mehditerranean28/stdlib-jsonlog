# stdlib-jsonlog
MEDIT8 is a Hardened stdlib-first JSON logging with non-blocking queues + optional TCP/TLS streaming.

It is a stdlib-first, “production-hard” JSON logging subsystem that treats logging as a loss-tolerant transport pipeline rather than a formatting step: bytes-first JSON serialization with bounded/typed extras (deep sanitization + node/item budgets + per-field caps + secret redaction), high-res emit-time timestamps (cached ISO8601 w/ TZ), context propagation via contextvars, and a non-blocking ingress queue with explicit shedding + accurate depth accounting. Downstream is an async network sender with framed protocols (len or v1 w/ crc32), connect/send timeouts, DNS caching, dynamic batching, protocol fallback, and monotonic-gated exponential backoff that never stalls the app thread; optional TLS with enforce/fallback semantics. Configuration is env-driven, validated, and hot-reloadable (root + per-logger levels) with its own backoff, plus introspection counters/health snapshot and leak-ish thread hygiene checks. Overall: a correctness- and ops-biased logging “mini-agent” embedded in-process—fast on the hot path, aggressively bounded under pathological inputs, and designed to degrade predictably under backpressure or misconfig rather than blocking or crashing.

<img width="929" height="940" alt="RLNlJoCt4Fs-VyM8VYYv8mgbFrITJkWWHu0MI9f4lIz8rSIUtVN5iTsnbv1M_TyxzYKekGC9rjXldj-yl9ryY0avTIjR9PCiGKpQcaN_S_JE-WQ2HKKhSWcC0uIq2KQX1Gm1tifMpGW5PW9PuwYgujAuW8mhY2rglpQkWHfjfBqJvLV1Eo6TfOqbO589NMFt8Ml6yCderCBPsUdU_WgVx" src="https://github.com/user-attachments/assets/33ec6b3c-e3ab-4fc0-98d4-ac0ac8f22cbb" />

**High-throughput services where logging must never block**
APIs, event processors, realtime pipelines. Your non-blocking queue + shedding + async sender keeps tail latency stable when stdout/network/collector is slow.

**Kubernetes / container fleets with centralized log shipping**
Built-in TCP/TLS streaming with framing + backoff behaves like a lightweight in-process forwarder, and your watcher supports “ops flips log level without redeploy”.

**Unreliable networks / intermittent collectors**
Backoff-gated reconnect + DNS caching + protocol fallback means you keep emitting locally while the remote path flaps, instead of wedging app threads.

**Security/PII-sensitive environments**
Key-based secret redaction + bounded extras/stack traces reduces “oops we logged tokens / huge payloads / deep objects” incidents.

**Multi-tenant / untrusted input surfaces**
The structured-extra sanitizer (depth/node/item budgets + caps) is exactly what you want when request bodies/objects can be adversarial.

**Tracing/correlation heavy systems**
Contextvars trace/span/correlation injection makes it easy to keep logs joinable with traces across async boundaries.

**Teams standardizing JSON logs without a big dependency**
Stdlib-first, idempotent init, handler tagging/dedupe: good for shared platform libs where you can’t dictate loguru/structlog everywhere.

**Systems that need “operational introspection” of logging itself**
Health snapshot (dropped, queue depth, backoff, last errors) is useful for dashboards/alerts and for diagnosing “why are logs missing”.

**Batch jobs / CLIs that still want hardened JSON**
init_simple_logging() gives a safe minimal mode (no watcher/network) while keeping the same schema + truncation/redaction behavior.

**Controlled graceful shutdown / drain semantics**
Services that care about draining logs on exit (deploy rollouts, job completion) benefit from flush_and_drain() + bounded shutdown.


