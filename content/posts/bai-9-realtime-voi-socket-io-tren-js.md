---
title: "Realtime Với Socket Io Trên Js"
date: 2025-10-20T14:30:06+07:00
draft: false
tags: []
categories: []
---
# Mục tiêu bài học
- Hiểu cơ bản về observability: logging, metrics và distributed tracing.  
- Biết cách instrument ứng dụng Java: structured logs (MDC), Micrometer cho metrics, OpenTelemetry cho tracing.  
- Thiết kế logs/metrics/traces để debug, monitor và alert hợp lý.  
- Các lưu ý vận hành: chi phí lưu trữ, sampling, retention, và performance impact.

---

## 1) Observability là gì (tóm tắt)
- Observability = khả năng hiểu hệ thống dựa trên dữ liệu xuất ra: logs, metrics, traces.  
- Mỗi loại dữ liệu có điểm mạnh:
  - Logs: chi tiết sự kiện, tốt cho forensic và debugging.
  - Metrics: số liệu thời gian thực, tốt cho dashboards & alerting.
  - Traces: theo dõi luồng request qua nhiều service (latency, bottleneck).

---

## 2) Logging — best practices
- Levels: TRACE/DEBUG/INFO/WARN/ERROR; dùng đúng mục đích.
- Structured logging: ghi JSON hoặc key=value để dễ query (ELK/Opensearch).
- Correlation ID: propagate traceId / requestId qua MDC để liên kết logs với traces.
- Avoid sensitive data: mask token/password/PII trước khi log.
- Async appender & batching: giảm ảnh hưởng I/O, dùng logback/log4j2 async appenders.
- Rotation & retention: cấu hình logrotate / index lifecycle policy để quản lý chi phí.

Ví dụ: set MDC in servlet/filter:
```java
// Java - set MDC (correlation id)
import org.slf4j.MDC;
import java.util.UUID;

String id = UUID.randomUUID().toString();
MDC.put("traceId", id);
try {
    // xử lý request
} finally {
    MDC.remove("traceId");
}
```

---

## 3) Metrics — kiểu và thu thập
- Metric types:
  - Counter: tăng dần (requests, errors).
  - Gauge: trạng thái hiện tại (queue length, heap usage).
  - Timer / Histogram: latency distribution (P50/P95/P99).
- Prometheus + Micrometer (Java):
  - Micrometer cung cấp abstraction, xuất sang Prometheus, Datadog, NewRelic...
  - Exposition: ứng dụng expose /metrics endpoint; Prometheus scrape.

Ví dụ Micrometer (Spring Boot):
```java
// Java - Micrometer counter
import io.micrometer.core.instrument.MeterRegistry;

public class MyService {
  private final Counter requests;
  public MyService(MeterRegistry registry) {
    requests = registry.counter("app.requests");
  }
  public void handle() {
    requests.increment();
  }
}
```

Naming convention: <service>.<resource>.<metric> và luôn include unit labels (status, method).

---

## 4) Distributed Tracing — cơ chế & công cụ
- Trace = tập hợp spans; mỗi span = một operation với start/end, attributes, status.  
- Context propagation: traceId/spanId phải theo request xuyên các service (HTTP headers như W3C Trace Context).  
- Công cụ: OpenTelemetry (instrumentation + exporters), Jaeger, Zipkin, Tempo.  
- Sampling: giảm lưu lượng traces (head-based, tail-based). Cân bằng detail vs cost.

OpenTelemetry Java minimal example:
```java
// Java - OpenTelemetry tracer (simplified)
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.trace.Tracer;

Tracer tracer = GlobalOpenTelemetry.getTracer("com.example");
Span span = tracer.spanBuilder("handleRequest").startSpan();
try {
  // work
} finally {
  span.end();
}
```

---

## 5) Correlate logs, metrics và traces
- Include traceId/requestId in logs and metrics labels where possible.
- Dashboards: use metrics for trends, traces for root-cause, logs for details.
- Example workflow: alert on metric spike → jump to relevant traces (P95) → inspect logs for affected traceId.

---

## 6) Sampling, retention và cost control
- Sampling: set sampling rate to control trace volume; use adaptive/tail-based sampling for accuracy on errors.
- Aggregation: pre-aggregate high-cardinality labels to avoid cardinality explosion.
- Retention: lưu traces ngắn hơn metrics; logs retention tùy quy định bảo mật/chi phí.
- Monitor cost: estimated ingestion rate (events/sec), storage size, egress.

---

## 7) Alerts & Dashboards
- Alerts on metrics (error rate, latency, saturation) với clear thresholds and runbooks.
- Use SLOs: define SLI (latency/availability), SLO targets, alert on burn rate.
- Dashboards: top-level service health, latency percentiles, error budget, traffic.

---

## 8) Performance & operational tips
- Logging:
  - Avoid expensive string concat in hot path — use parameterized logging or guard with isDebugEnabled().
  - Use async appenders to decouple IO.
- Metrics:
  - Avoid high-cardinality labels (userId, requestId) on metrics.
  - Use histogram buckets wisely; rely on percentiles computed by backend.
- Tracing:
  - Instrument only important spans; avoid creating spans for trivial ops at high rate.
  - Prefer semantic attributes (http.method/http.status_code/db.system).

---

## 9) Troubleshooting common issues
- Missing traceId in logs: check propagation (MDC or logging integration) and header mapping.
- Cardinality explosion in metrics: audit labels and remove per-request unique ids.
- High logging I/O causing latency: enable async logging and reduce verbosity.
- No traces for downstream calls: ensure instrumentation libraries are installed and headers forwarded.

---

## 10) Bài tập & hướng đi tiếp
- Instrument một ứng dụng Spring Boot:
  - Add Micrometer + Prometheus endpoint, tạo vài metrics (requests, errors, latency).
  - Add OpenTelemetry auto-instrumentation; export traces to Jaeger.
  - Push logs structured JSON with traceId in MDC to a local ELK/Opensearch.
- Build dashboard: Grafana panel with P95 latency, error rate, and traces link.
- Implement basic alert: error rate > 1% for 5min → alert.
- Explore advanced: tail-based sampling, distributed context propagation for messaging systems, correlating logs from batch jobs.

---