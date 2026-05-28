# Week 9 — Observability: Monitoring, Logging, and Tracing

## Bird's eye view

- **Observability** is the ability to understand the **internal states** of a system by analysing its **external outputs** — it gives insight into behaviour, performance, and health.
- The four key components are **Monitoring**, **Logging**, **Tracing**, and **Metrics**. Together they form a unified picture; no single pillar is sufficient alone.
- **Monitoring** is a sub-component of observability: it provides real-time data on performance and health. Observability *extends* monitoring by correlating it with logs and traces.
- **Logs** capture discrete events at a specific point in time. **Traces** capture the end-to-end flow of a request across services. These are complementary: logs explain *what* happened, traces explain *where* time was spent.
- The principal toolchain: **Prometheus + Grafana** (metrics/monitoring), **ELK Stack / Loki** (log aggregation), **OpenTelemetry + Jaeger/Zipkin** (distributed tracing).
- **Load testing** (Gatling, JMeter, k6) is a *complementary and upfront* activity — run it *before* production to validate performance, and use observability tools to analyse the results.
- **Alert automation** (Prometheus Alertmanager, Grafana Alerts) is essential for proactive issue detection; alert fatigue from too many/too few alerts is a key operational pitfall.

---

## Detailed notes

### 1. What is Observability?

**Definition:** Observability is the ability to understand the internal states of a system by analysing its external outputs.

**Key components (the pillars):**

| Pillar | What it captures | Example tools |
|---|---|---|
| **Monitoring** | Real-time performance and health metrics | Prometheus, Nagios, Zabbix, Datadog |
| **Logging** | Discrete events/messages at a point in time | ELK Stack, Splunk, Loki, Graylog |
| **Tracing** | Flow of a request across services | Jaeger, Zipkin, OpenTelemetry |
| **Metrics** | Numerical measurements over intervals | Prometheus, Datadog, New Relic |

**Other metadata types:**
- **Events** — significant system occurrences (user sign-up, system alert, order placed).
- **Audit Trails** — activity records for compliance/security (user access reviews, data modification logs).

**Benefits of observability:**
- **Comprehensive Visibility** — holistic view integrating all four pillars.
- **Proactive Issue Detection** — real-time alerting before users are impacted.
- **Enhanced Diagnostics** — root cause analysis and performance tuning support.

---

### 2. Monitoring vs. Observability — the distinction

| Monitoring | Observability |
|---|---|
| Asks: "Is the system up/down?" | Asks: "Why is it behaving this way?" |
| Provides real-time health metrics | Extends monitoring with logs and traces |
| Reactive: notifies when a threshold is breached | Proactive: gives context to understand *why* |
| Tools: Prometheus, Nagios | Platforms: Datadog, New Relic (integrated stacks) |

Observability **includes** monitoring — it does not replace it.

---

### 3. Enterprise App Monitoring

**What monitoring covers:**
- CPU usage, memory usage, response time, error rates.
- Identifies and resolves issues proactively.

**Good practices:**
- Set up alerts for critical metrics.
- Use dashboards for real-time visibility.
- Regularly review and update monitoring configurations.
- Integrate with incident management tools such as **PagerDuty**.

**Bad practices (pitfalls):**
- Ignoring alerts or setting too many alerts (alert fatigue).
- Not reviewing monitoring data regularly.
- Relying solely on manual checks.
- Failure to integrate monitoring with other DevOps tools.

**Monitoring tools overview:**

| Tool | Description |
|---|---|
| **Prometheus** | Open-source monitoring & alerting toolkit; pull-based; multi-dimensional data model; PromQL query language |
| **Grafana** | Open-source dashboard platform; integrates with Prometheus, Jaeger, and other data sources |
| **Nagios** | Open-source network monitoring system |
| **Zabbix** | Enterprise-grade monitoring with advanced alerting |
| **Datadog** | Cloud-based monitoring and analytics platform |
| **New Relic** | Application performance monitoring (APM) for apps and infrastructure |

---

### 4. Log Management

#### 4.1 What are Logs?

**Definition:** Logs are records of events that occur within a system.

**Purpose:** Diagnose issues and errors; monitor system behaviour and performance; audit and track user activities.

#### 4.2 Log Levels (severity hierarchy)

| Level | Meaning |
|---|---|
| **DEBUG** | Detailed diagnostic information; typically only useful during development/debugging |
| **INFO** | General operational information about application progress |
| **WARN** | Something unexpected happened, or a potential problem in the near future (e.g., "disk space low") — app still works |
| **ERROR** | A serious problem; the app failed to perform a specific function |
| **FATAL** | A very serious error that may prevent the app from continuing to run |

#### 4.3 Types of logs

- **Error Logs** — exceptions and errors. Example: `ERROR: NullPointerException at line 42`
- **Access Logs** — user access and actions. Example: `INFO: User 'john_doe' logged in at 2023-10-01 12:34:56`
- **System Logs** — system-level events. Example: `WARN: Disk space low on /dev/sda1`

#### 4.4 Managing and exploiting logs

| Concern | Approach | Tools |
|---|---|---|
| **Centralized Logging** | Aggregate logs from all sources into one system | ELK Stack, Splunk, Graylog |
| **Log Rotation** | Prevent disk exhaustion; rotate/archive old logs | Logrotate (Linux), built-in library rotation |
| **Log Analysis** | Search, filter, and visualize log data | Kibana, Splunk, Grafana |
| **Alerting** | Alert on specific log events (e.g., ERROR rate spike) | ElastAlert, Splunk Alerts, Grafana Alerts |

#### 4.5 Best practices for log management

- **Structured Logging:** Use JSON format so logs are machine-parseable.

```json
{
  "timestamp": "2023-10-01T12:34:56Z",
  "level": "ERROR",
  "service": "auth-service",
  "message": "NullPointerException at line 42",
  "trace_id": "abc123"
}
```

- **Consistent format and level** across the whole application.
- **Avoid logging sensitive data** (passwords, PII, secrets).
- **Define log retention policies** for storage and compliance.
- **Regularly review** logging configurations.

**Bad practices:**
- Storing logs indefinitely without rotation.
- Logging sensitive information (passwords, PII).
- No centralized logging system.
- Ignoring log data during incident investigations.

---

### 5. Trace Management

#### 5.1 What are Traces?

**Definition:** Traces capture the flow of requests through a system — specifically across multiple services in a distributed architecture.

**Purpose:** Identify performance bottlenecks; understand service interactions and dependencies; diagnose issues in distributed systems.

#### 5.2 Trace anatomy — key concepts

| Concept | Description |
|---|---|
| **Span** | The basic unit of work in a trace; represents a single operation (e.g., an HTTP call, a DB query) |
| **Trace** | A collection of spans that represent a complete transaction or request flow |
| **Tags / Attributes** | Key-value pairs providing additional context (e.g., HTTP method, status code, service name) |
| **Events** | Specific points in time within a span that capture significant occurrences |

**Distributed tracing example (waterfall view):**

```
Trace ID: 12345
├── Span: api-gateway          [0ms — 120ms]
│   ├── Span: auth-service     [5ms — 55ms]   (Span ID: 67890, Duration: 50ms)
│   ├── Span: product-service  [56ms — 90ms]
│   │   └── Span: db-query     [60ms — 88ms]  (DB insert, Status: Success)
│   └── Span: cart-service     [91ms — 115ms]
```

Each span carries a **Trace ID** (shared across the whole request) and its own **Span ID**. Context propagation headers (e.g., W3C `traceparent`) pass these IDs across service boundaries.

**Transaction trace example:**
```
Transaction ID: abcdef, Operation: DB insert, Status: Success
```

#### 5.3 Managing and exploiting traces

| Concern | Approach | Tools |
|---|---|---|
| **Centralized Tracing** | Collect all traces in one backend | Jaeger, Zipkin, OpenTelemetry Collector |
| **Trace Analysis** | Visualize trace waterfalls and latency distributions | Jaeger UI, Zipkin UI, Grafana + OpenTelemetry |
| **Trace Correlation** | Correlate traces with logs and metrics via Correlation IDs | OpenTelemetry, ELK Stack + APM |
| **Alerting** | Alert on trace anomalies (slow spans, error spans) | Prometheus Alertmanager, Grafana Alerts |

#### 5.4 Best practices for trace management

- **Consistent Instrumentation:** Instrument all services uniformly to avoid blind spots.
- **Context Propagation:** Propagate trace context headers across every service boundary.
- **Sampling Strategy:** Do not record every trace at full volume in production — implement head-based or tail-based sampling to manage overhead.
- **Trace Retention:** Define retention policies; traces are large and expensive to store indefinitely.
- **Regular Review:** Keep tracing configurations up to date as services evolve.

---

### 6. Monitoring vs. Log/Trace Management — comparison

| Dimension | Monitoring | Log/Trace Management |
|---|---|---|
| **Focus** | Real-time performance and health | Detailed application behaviour |
| **Key data** | CPU, memory, response time, error rates | Logs, traces, error messages, user actions |
| **Primary use** | Proactive issue detection and alerting | Debugging, performance tuning, auditing |
| **Tools** | Prometheus, Nagios, Zabbix, Datadog | ELK Stack, Splunk, Jaeger, OpenTelemetry |

**Integration pattern:** Use monitoring to *detect* that something is wrong, and log/trace management to *diagnose* the root cause. Grafana can visualise data from both simultaneously.

---

### 7. Observability Tools in depth

#### 7.1 OpenTelemetry

- Open-source, vendor-neutral observability framework (CNCF project).
- Supports all three pillars: metrics, logs, and traces through a single SDK and collector.
- Integrates with various backends: Jaeger, Prometheus, Zipkin, Datadog, etc.
- **SDK instrumentation example (Python):**

```python
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter

provider = TracerProvider()
jaeger_exporter = JaegerExporter(agent_host_name="localhost", agent_port=6831)
provider.add_span_processor(BatchSpanProcessor(jaeger_exporter))
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

with tracer.start_as_current_span("auth-service.login") as span:
    span.set_attribute("user.id", "john_doe")
    # ... perform login logic
```

#### 7.2 Jaeger

- Open-source distributed tracing system (originally from Uber, now CNCF).
- Helps monitor and troubleshoot microservices-based architectures.
- Provides a UI for visualising trace waterfalls, latency histograms, and service dependency graphs.

#### 7.3 Prometheus

- Open-source monitoring and alerting toolkit.
- **Pull-based model:** Prometheus scrapes metrics from configured endpoints (exporters) at regular intervals.
- Supports a **multi-dimensional data model** (metrics labelled with key-value pairs).
- **PromQL** is the query language for selecting and aggregating time-series data.

**Prometheus scraping topology (described):**
Prometheus Server polls multiple targets (application `/metrics` endpoints, Node Exporter, cAdvisor for containers). Each target exposes metrics in a text format. Prometheus stores them in its time-series database (TSDB). AlertManager receives firing rules from Prometheus and routes alerts to notification channels (email, Slack, PagerDuty).

**PromQL example queries:**

```promql
# HTTP request rate over the last 5 minutes
rate(http_requests_total[5m])

# 95th percentile response time
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# CPU usage percentage per instance
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# Error rate (5xx responses)
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
```

**Prometheus alert rule YAML example:**

```yaml
groups:
  - name: app_alerts
    rules:
      - alert: HighErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High HTTP error rate detected"
          description: "Error rate is above 5% for more than 2 minutes."

      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
```

#### 7.4 Grafana

- Open-source platform for monitoring and observability dashboards.
- Connects to multiple data sources: Prometheus, Jaeger, Elasticsearch, Loki, InfluxDB, and cloud services.
- Supports creating, exploring, and sharing dashboards.
- **Grafana Alerts** can be configured on any panel; support multiple notification channels (email, Slack, PagerDuty).

#### 7.5 ELK Stack

- **Elasticsearch** — distributed search and analytics engine; stores and indexes logs.
- **Logstash** — data pipeline for ingesting, parsing, and transforming logs before indexing.
- **Kibana** — web UI for searching, visualising, and dashboarding log data.
- Popular open-source log management solution.

#### 7.6 Loki

- Horizontally scalable, highly available, multi-tenant log aggregation system (by Grafana Labs).
- Does **not** index the contents of logs (unlike Elasticsearch), only the metadata/labels — making it cheaper.
- Designed to work natively with Grafana and Prometheus's label model.

#### 7.7 Splunk

- Powerful enterprise platform for searching, monitoring, and analysing machine-generated data.
- Supports log management, metrics, and security (SIEM) use cases.

#### 7.8 Fluentd / Fluentbit

- Open-source data collectors for a unified logging layer.
- Fluentd: feature-rich, runs as a daemon; Fluentbit: lightweight, embedded.
- Used in EFK (Elasticsearch + Fluentd/bit + Kibana) as an alternative to Logstash.

---

### 8. Automated Monitoring Alerts

#### 8.1 Why automate alerts?

- **Proactive Issue Detection:** Detect and alert on issues before users are impacted.
- **Reduced Response Time:** Notify the right team immediately.
- **Efficient Resource Management:** Focus engineering attention on critical alerts; reduce alert fatigue.

#### 8.2 Tools for alert automation

| Tool | Key features |
|---|---|
| **Prometheus Alertmanager** | Integrates with Prometheus; supports routing, grouping, silencing, and inhibition of alerts |
| **Grafana Alerts** | Panel-based alerts; multiple notification channels (email, Slack, PagerDuty) |
| **Nagios** | Built-in alerting via email, SMS |
| **Zabbix** | Enterprise alerting with custom rules and channels |
| **Datadog** | Real-time cloud-based alerting with service integrations |

#### 8.3 Setting up automated alerts — workflow

1. **Define Alert Conditions:** Set thresholds for key metrics (CPU, memory, response time). Use PromQL expressions or dashboard queries.
2. **Configure Notification Channels:** Email, Slack, PagerDuty, SMS — configure integration settings per channel.
3. **Create Alert Rules:** Define severity levels (critical, warning, info).
4. **Test Alerts:** Simulate conditions to verify alerts fire and notifications are delivered.

#### 8.4 Best practices for alerting

- **Set Meaningful Thresholds:** Base on historical data; avoid thresholds too low (noise) or too high (miss real issues).
- **Multi-Channel Notifications:** Escalation policies for critical alerts.
- **Regular Review:** Update alert rules as the system evolves.
- **Alert Fatigue Mitigation:** Use grouping and silencing; prioritise by severity and business impact.

---

### 9. Deployment Architecture for Observability Tools

#### 9.1 Core principle — run monitoring in a dedicated environment

- **Do not** run monitoring tools on production application servers.
- Security risk: monitoring data may expose sensitive system information.
- Performance risk: monitoring tools can consume significant CPU and memory, competing with application workloads.

#### 9.2 Architecture options

| Option | How it works |
|---|---|
| **On-Premises Monitoring Server** | Dedicated physical/virtual server; install Prometheus, Grafana, ELK on it; agents/exporters on prod servers ship data to it |
| **Containerized Monitoring Stack** | Deploy all monitoring tools as Docker containers; orchestrate with Kubernetes |
| **Cloud-Based Monitoring** | AWS CloudWatch, Azure Monitor, Google Cloud Operations — managed, scalable, no infra to maintain |

#### 9.3 Security and performance considerations

- Encrypt monitoring data in transit and at rest.
- Implement access controls on monitoring UIs (Grafana, Kibana are authentication-gated).
- Monitor the resource usage of monitoring tools themselves.
- Design for **high availability**: redundant Prometheus instances, Alertmanager clustering, replicated Elasticsearch.

---

### 10. Use Cases in Enterprise Computing

#### 10.1 Performance Optimization

- Monitoring identifies bottlenecks (high latency, high CPU).
- Log/Trace Management pinpoints inefficient DB queries and slow code paths.

#### 10.2 Incident Response

- Monitoring alerts trigger incident response processes.
- Log/Trace Management provides the data needed to understand and resolve the incident.
- Workflow: Alert fires → team notified → logs/traces consulted → root cause identified → fix deployed.

#### 10.3 Compliance and Auditing

- Monitoring ensures performance SLAs are being met.
- Log/Trace Management provides audit trails for regulatory compliance (user access logs, data modification logs).

#### 10.4 Real-world cases from the slides

**Case 1 — E-commerce Site Outage (traffic spike):**
- Problem: Site unresponsive during sale event.
- Detection: Prometheus alerts on high CPU + memory exhaustion.
- Diagnosis: Logs reveal surge in DB queries.
- Resolution: Scale DB and optimize queries.

**Case 2 — Financial Services Fraud Detection:**
- Problem: Fraudulent transactions not detected in real time.
- Solution: Real-time monitoring of transaction patterns + log-based tracking of suspicious user activity + fraud detection algorithm integration.

**Case 3 — Healthcare System Performance:**
- Problem: Slow response times for patient record access.
- Solution: Monitor performance metrics to find slow queries (high latency) → analyze logs to pinpoint inefficient DB queries → optimize code and queries.

---

### 11. Load Testing as a Complementary Upfront Activity

#### 11.1 What is load testing?

- Simulates user load on an application to assess performance *before* production.
- **Objectives:** Ensure system handles expected and peak loads; identify bottlenecks; validate scalability and stability.

#### 11.2 Role of observability during load tests

| Observability layer | What it provides during load tests |
|---|---|
| **Monitoring** | Real-time CPU, memory, response time, error rates as load increases; alerts on threshold breaches |
| **Log/Trace Management** | Detailed request flow; identifies which component degrades under load; error correlation with load level |
| **Integration** | Dashboards show both metrics and logs simultaneously for holistic analysis |

#### 11.3 Load testing tools

| Tool | Language/approach | Key features |
|---|---|---|
| **Apache JMeter** | Java GUI/CLI | Supports HTTP, FTP, SOAP, JDBC; highly extensible with plugins |
| **Gatling** | Scala DSL | High performance, scalability; supports HTTP, WebSockets, SSE |
| **Locust** | Python code | Define user behaviour in Python; web UI for real-time monitoring; supports distributed load |
| **k6** | JavaScript | Open-source + SaaS; real-time analytics and reporting |
| **LoadRunner** | Various | Enterprise-grade by Micro Focus; wide protocol support; comprehensive analytics |

#### 11.4 Gatling exercise (from slides)

The exercise demonstrates a complete load test workflow:
1. Install Gatling from the official website.
2. Create a simulation using Gatling's Scala DSL — define user scenarios (browse products, add to cart, check out).
3. Configure load parameters: number of virtual users, ramp-up time, test duration.
4. Run the simulation and monitor performance metrics in real time.
5. Analyse the generated HTML report: response times, error rates, resource utilisation.

**Good practice — gradual load increase example:**
- Baseline: 1,000 concurrent users.
- Increment: +500 users every 10 minutes.
- Peak: 10,000 concurrent users.
- Monitor CPU, memory, response time at each step.
- Benefit: bottlenecks appear at their actual load threshold, not masked by sudden shock.

**Bad practice — sudden load spike:**
- Jump directly to 10,000 concurrent users.
- Issues: inaccurate performance measurements; difficult to pinpoint which component broke and at what load level.

#### 11.5 Good vs. bad practices in load testing

| Good practices | Bad practices |
|---|---|
| Realistic scenarios mimicking real user behaviour (think times, varied actions) | Unrealistic scenarios ignoring user variability |
| Gradual load ramp from baseline to peak | Applying peak load suddenly |
| Monitor all key metrics (CPU, memory, response time, error rate) | Focusing only on response time |
| Regular testing (especially before major releases) | Infrequent or one-off testing |
| Thorough analysis of results | Superficial analysis ignoring bottleneck details |
| Integrate with CI/CD for automated performance validation | — |

---

## Key terms

- **Observability** — the ability to infer internal system state from external outputs (metrics, logs, traces).
- **Monitoring** — real-time collection and alerting on performance and health metrics; a sub-component of observability.
- **Log** — a timestamped record of a discrete event in the system.
- **Trace** — a collection of spans representing the complete journey of a single request across services.
- **Span** — the basic unit of work in a trace; represents one operation with a start time and duration.
- **Metric** — a numerical measurement collected at intervals (e.g., CPU %, request rate).
- **Structured Logging** — logs stored as key-value JSON rather than unstructured strings, enabling machine parsing.
- **Log Level** — severity label (DEBUG < INFO < WARN < ERROR < FATAL).
- **ELK Stack** — Elasticsearch + Logstash + Kibana; popular log aggregation and search stack.
- **EFK Stack** — Elasticsearch + Fluentd/Fluentbit + Kibana; lightweight alternative pipeline.
- **Loki** — label-indexed log aggregation system by Grafana Labs; cheaper than Elasticsearch at scale.
- **Prometheus** — pull-based open-source metrics and alerting system with PromQL.
- **PromQL** — Prometheus Query Language for selecting and aggregating time-series data.
- **Alertmanager** — Prometheus component that routes, groups, and silences alerts.
- **Grafana** — open-source dashboard and visualisation platform; integrates with Prometheus, Loki, Jaeger.
- **OpenTelemetry** — CNCF open-source framework providing a single SDK and collector for metrics, logs, and traces.
- **Jaeger** — open-source distributed tracing backend, originally from Uber; now CNCF.
- **Zipkin** — another open-source distributed tracing system.
- **Context Propagation** — passing trace/span IDs across service boundaries (via HTTP headers, message metadata) to maintain trace continuity.
- **Sampling** — recording only a fraction of traces to reduce overhead; head-based or tail-based.
- **Trace Correlation** — linking a trace ID to related log entries and metrics for unified root-cause analysis.
- **Alert Fatigue** — state where too many alerts cause engineers to ignore them; mitigated by grouping, silencing, and meaningful thresholds.
- **Load Testing** — simulating user load to assess system performance before production.
- **Centralized Logging** — aggregating logs from all services into one searchable system.
- **Log Rotation** — archiving/deleting old log files to prevent disk exhaustion.
- **Audit Trail** — records of activities for compliance and security review.

---

## Exam targets

1. **Define observability** and explain how it differs from monitoring — observability *includes* monitoring but adds logs and traces to enable *why* questions.
2. **Name and describe the four pillars** of observability: monitoring, logging, tracing, metrics. Give an example tool for each.
3. **Explain the five log levels** (DEBUG, INFO, WARN, ERROR, FATAL) in order and give a concrete scenario for each.
4. **Describe the span/trace model** — what is a span, how do spans form a trace, what does a trace waterfall look like, and what is context propagation?
5. **Compare logs vs. traces** — logs: discrete events at a point in time; traces: sequence of related spans across services. Give an example of each.
6. **Explain Prometheus's pull model** — Prometheus scrapes `/metrics` endpoints; contrast with push-based models. Write or read a simple PromQL rate query.
7. **Describe the ELK Stack components** — Elasticsearch (store/search), Logstash (ingest/transform), Kibana (visualise). Explain when to use Loki instead.
8. **Explain structured logging** — why JSON logs are preferable to plaintext for machine parsing.
9. **Describe the Alertmanager workflow** — alert rule fires in Prometheus → Alertmanager receives → routes/groups/silences → notifies channel.
10. **Explain monitoring vs. log/trace management integration** — monitoring detects, log/trace diagnoses; Grafana can surface both.
11. **Describe good practices for running monitoring infrastructure** — dedicated environment, containerized or cloud-based, encrypted data, access controls.
12. **Explain load testing's role in observability** — it is an *upfront* activity; observability tools (monitoring + logging) are used *during* load tests to identify bottlenecks; gradual ramp-up is the correct approach.
13. **Name three load testing tools** and differentiate them (JMeter = GUI/Java; Gatling = Scala DSL/high perf; Locust = Python/distributed; k6 = JS/SaaS).
14. **Walk through a use-case incident response** — e.g., e-commerce outage: monitoring alerts on CPU spike → logs reveal DB query surge → scale DB and optimise queries.

---

## Pitfalls

- **Observability is NOT just monitoring.** Monitoring is one pillar; without logs and traces, you can detect but not diagnose problems.
- **Logs ≠ Traces.** Logs are point-in-time events per service. Traces follow a request across service boundaries. A log entry in one service does not show what happened in the downstream services it called.
- **Never log sensitive data** (passwords, PII, tokens). This is both a security and compliance violation.
- **Log levels must be used correctly.** DEBUG is not the same as ERROR. Using ERROR for informational messages causes alert fatigue; using INFO for errors causes missed incidents.
- **Running monitoring tools on production servers** is a bad practice — they compete for resources and expose sensitive metrics data.
- **Alert fatigue is real.** Setting too many alerts (or thresholds too low) causes engineers to start ignoring them. Grouping, silencing, and severity levels are the mitigation.
- **Sudden load spikes in load testing give misleading results.** Always use gradual ramp-up so that the failure threshold is identified precisely.
- **No centralized logging** means incident response requires SSH-ing into individual servers — unacceptable at scale.
- **Storing logs indefinitely** exhausts disk space; implement log rotation and retention policies.
- **Context propagation must be consistent.** If one service in a chain does not forward the trace headers, the trace is broken and the waterfall is incomplete.
- **Sampling strategy matters.** Recording 100% of traces in high-traffic production systems is expensive; but sampling too aggressively misses rare failure traces. Tail-based sampling (record if the trace had an error) is often best practice.
- **Monitoring data must be secured.** Grafana dashboards and Kibana are not secure by default — access control and encryption in transit/at rest are required.
