# Performance Observability Tool — Industry Execution Plan

## What You're Building (One Paragraph)

A **complete performance observability pipeline** for a microservice running on Linux. You instrument an Nginx + PostgreSQL stack, collect system-level and application-level metrics using Linux kernel interfaces (`/proc`, `perf`, eBPF), store them in a time-series database, visualize them on dashboards, set up alerting, and generate FlameGraphs to diagnose performance bottlenecks. By the end, you can point at a running system and say: *"I built this. I can instrument any Linux system, diagnose any bottleneck, and I understand what happens from the hardware counter to the HTTP response."*

---

## Why Each Component Matters for Interviews

When you interview for kernel/performance engineering roles (at companies like Meta, Google, Netflix, Intel, AMD, Cloudflare), interviewers ask questions like these. Your project gives you concrete answers.

| Interview Question | What In Your Project Answers It |
|:-------------------|:-------------------------------|
| *"How would you debug a latency spike in production?"* | Your dashboards showing p99 latency correlated with CPU/disk/network metrics |
| *"Explain how /proc/stat works"* | You wrote a collector that parses it — you know every field |
| *"What's the difference between perf and eBPF?"* | You used both and can explain the tradeoffs from experience |
| *"How do you read a FlameGraph?"* | You generated them from `perf record` on your own microservice |
| *"What are hardware performance counters?"* | You used `perf stat` to measure IPC, cache misses, branch misses |
| *"Walk me through a request from NIC to response"* | Your instrumentation covers every layer of this path |
| *"How does Prometheus scraping work?"* | You wrote custom exporters that Prometheus scrapes |
| *"What causes high tail latency?"* | You load-tested your system and diagnosed real p99 issues |

---

## The Stack

```
┌─────────────────────────────────────────────────────────┐
│                     LOAD TESTING                        │
│              wrk / k6 (generate traffic)                │
└───────────────────────┬─────────────────────────────────┘
                        │ HTTP
                        ▼
┌─────────────────────────────────────────────────────────┐
│                    MICROSERVICE                         │
│                                                         │
│   Nginx (:80) ──→ Go REST API (:8080) ──→ PostgreSQL   │
│   reverse proxy    3 endpoints             sample data  │
│                    /healthz                              │
│                    /users (DB read)                      │
│                    /compute (CPU work)                   │
└───────────────────────┬─────────────────────────────────┘
                        │ instrumented by
                        ▼
┌─────────────────────────────────────────────────────────┐
│                  INSTRUMENTATION LAYER                  │
│                                                         │
│   Layer 1: System metrics                               │
│     → Custom /proc parser (CPU, mem, disk, net)         │
│     → Exposed as Prometheus metrics                     │
│                                                         │
│   Layer 2: Application metrics                          │
│     → Request count, latency histogram, error rate      │
│     → DB query duration, connection pool stats          │
│     → Exposed as Prometheus metrics                     │
│                                                         │
│   Layer 3: Deep profiling (stretch)                     │
│     → perf record → FlameGraphs                         │
│     → perf stat → hardware counters (IPC, cache-miss)   │
│     → eBPF tracing (per-process I/O, syscall latency)   │
└───────────────────────┬─────────────────────────────────┘
                        │ scraped by
                        ▼
┌─────────────────────────────────────────────────────────┐
│                  STORAGE + VISUALIZATION                │
│                                                         │
│   Prometheus (time-series DB, scrapes every 15s)        │
│        ↓                                                │
│   Grafana (dashboards + alerting)                       │
│     Dashboard 1: System Overview (CPU/mem/disk/net)     │
│     Dashboard 2: Application Performance (latency/RPS)  │
│     Dashboard 3: Deep Dive (FlameGraphs, counters)      │
│        ↓                                                │
│   Alerting Rules                                        │
│     → CPU > 80% for 5 min                               │
│     → p99 latency > 100ms                               │
│     → Error rate > 1%                                   │
└─────────────────────────────────────────────────────────┘
```

---

## Why Go (Not Python) for the Microservice

| Reason | Detail |
|:-------|:-------|
| **Industry standard** | Most microservices at Google, Uber, Cloudflare, etc. are in Go |
| **Built-in Prometheus library** | `prometheus/client_golang` makes metric exposition trivial |
| **Compiled binary** | `perf record` and eBPF uprobes work cleanly on Go binaries |
| **Low-level visibility** | Go's runtime exposes goroutine scheduling, GC pauses — great for profiling |
| **Interview relevance** | Performance engineering roles often use Go or C |
| **pprof built-in** | Go has native CPU/memory/goroutine profiling at `/debug/pprof` |

If nobody on the team knows Go, use **Python (FastAPI)** instead. It's fine — the project value is in the instrumentation, not the app.

---

## Detailed Component Breakdown

### Component 1: The Microservice (10 hours)

A simple REST API that does real work:

```go
// Three endpoints that exercise different system resources:

GET /healthz        → returns {"status": "ok"} (trivial, for monitoring)

GET /users?limit=10 → queries PostgreSQL, returns JSON
                      (exercises: network, disk I/O, DB connections)

POST /compute       → computes SHA-256 hashes in a loop
                      (exercises: CPU, memory, branch prediction)
```

PostgreSQL has a `users` table with 100K rows (seeded with `pgbench` or a script). Nginx sits in front as a reverse proxy (standard production pattern).

**What you learn:** How a real microservice stack works. Request flow from client → proxy → app → database.

### Component 2: System Metrics Collector (20 hours)

You write a **custom Prometheus exporter** that reads Linux kernel interfaces:

```
WHAT YOU READ              WHAT IT TELLS YOU              INTERVIEW RELEVANCE
────────────────           ──────────────────             ───────────────────
/proc/stat                 Per-CPU utilization            "Explain usr/sys/idle/iowait"
                           (user, system, idle,           
                           iowait, irq, softirq)         

/proc/meminfo              Memory breakdown               "What's the difference between
                           (total, free, available,        free and available memory?"
                           buffers, cached, swap)         

/proc/diskstats            Per-disk I/O                   "What do read_ticks and
                           (reads, writes, bytes,          write_ticks mean?"
                           time in queue, inflight)       

/proc/net/dev              Per-interface network           "How do you detect packet drops?"
                           (bytes, packets, errors,       
                           drops, fifo, colls)            

/proc/loadavg              Load averages                  "What does load average actually
                           (1min, 5min, 15min)             measure?"

/proc/<pid>/stat           Per-process CPU                "How do you find which process
/proc/<pid>/status         Per-process memory              is using the most CPU?"
/proc/<pid>/io             Per-process I/O                
```

You parse each file, compute rates (bytes/sec, ops/sec), and expose them as Prometheus metrics:

```go
// Example: expose CPU utilization as a Prometheus gauge
cpuUsage := prometheus.NewGaugeVec(
    prometheus.GaugeOpts{
        Name: "node_cpu_usage_percent",
        Help: "CPU utilization percentage by mode",
    },
    []string{"cpu", "mode"},  // labels: cpu0/cpu1/.., user/system/idle/..
)
```

**What you learn:** How the Linux kernel exposes system state. This is the #1 thing performance engineers need to know.

> [!TIP]
> **Interview gold:** Writing this collector forces you to understand every field in `/proc/stat` and `/proc/meminfo`. Interviewers at Meta/Google regularly ask "how does the kernel track CPU utilization?" or "what's the difference between `buffers` and `cached` in meminfo?" You'll know because you parsed them yourself.

### Component 3: Application Metrics (10 hours)

Instrument your Go API to expose request-level metrics:

```go
// RED metrics (Rate, Errors, Duration) — the industry standard

// 1. Request rate (counter)
httpRequestsTotal := prometheus.NewCounterVec(
    prometheus.CounterOpts{Name: "http_requests_total"},
    []string{"method", "endpoint", "status_code"},
)

// 2. Request duration (histogram — gives you p50/p95/p99)
httpRequestDuration := prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Buckets: []float64{.001, .005, .01, .025, .05, .1, .25, .5, 1},
    },
    []string{"method", "endpoint"},
)

// 3. DB query duration
dbQueryDuration := prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "db_query_duration_seconds",
        Buckets: []float64{.0005, .001, .005, .01, .05, .1},
    },
    []string{"query_type"},  // "select", "insert"
)
```

**Also expose from PostgreSQL** using `pg_stat_statements`:
- Query execution counts and times
- Buffer cache hit ratio
- Connection pool usage
- Locks and deadlocks

**Also expose from Nginx** using the `stub_status` module:
- Active connections
- Requests per second
- Connection states (reading/writing/waiting)

**What you learn:** The RED method (Rate, Errors, Duration) — the standard framework used at Google, Grafana Labs, and every SRE team.

### Component 4: Prometheus + Grafana (15 hours)

#### Prometheus Setup

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'system-metrics'
    static_configs:
      - targets: ['localhost:9100']  # your custom exporter

  - job_name: 'go-api'
    static_configs:
      - targets: ['localhost:8080']  # Go app's /metrics endpoint

  - job_name: 'nginx'
    static_configs:
      - targets: ['localhost:9113']  # nginx-prometheus-exporter

  - job_name: 'postgres'
    static_configs:
      - targets: ['localhost:9187']  # postgres_exporter
```

#### Grafana Dashboards (3 dashboards)

**Dashboard 1: System Overview**
```
Row 1: CPU utilization (stacked area: user/system/iowait/idle) | Load average
Row 2: Memory (used/cached/buffers/free) | Swap usage
Row 3: Disk I/O (read/write MB/s) | Disk latency | Queue depth
Row 4: Network (bytes in/out) | Packets | Errors/drops
```

**Dashboard 2: Application Performance**
```
Row 1: Request rate (req/s by endpoint) | Error rate (%)
Row 2: Latency heatmap | p50/p95/p99 latency timeseries
Row 3: DB query duration | Cache hit ratio | Active connections
Row 4: Nginx connections (active/reading/writing/waiting)
```

**Dashboard 3: Deep Dive (populated in stretch goal)**
```
Row 1: Hardware counters (IPC, cache-miss-rate, branch-miss-rate)
Row 2: Embedded FlameGraph SVG link
Row 3: Per-process CPU/memory/I/O breakdown
```

#### Alerting Rules

```yaml
# alert_rules.yml
groups:
  - name: performance-alerts
    rules:
      - alert: HighCPU
        expr: node_cpu_usage_percent{mode="idle"} < 20
        for: 5m
        labels: { severity: warning }
        annotations: { summary: "CPU utilization above 80% for 5 minutes" }

      - alert: HighP99Latency
        expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 0.1
        for: 2m
        labels: { severity: critical }
        annotations: { summary: "p99 latency exceeds 100ms" }

      - alert: HighErrorRate
        expr: rate(http_requests_total{status_code=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.01
        for: 1m
        labels: { severity: critical }
```

**What you learn:** PromQL queries, dashboard design, alerting — core SRE/performance engineering skills.

### Component 5: Load Testing + Stress Analysis (10 hours)

Use **wrk** or **k6** to generate realistic traffic and stress-test:

```bash
# Baseline: steady 1000 req/s for 5 minutes
wrk -t4 -c100 -d300s --latency http://localhost/users?limit=10

# Stress test: ramp up until the system breaks
k6 run --vus 500 --duration 5m stress_test.js

# Spike test: sudden burst
k6 run --stages "1m:50,10s:500,2m:500,10s:50,1m:50" spike_test.js
```

While load testing, **watch your dashboards.** You should see:
- CPU ramping up
- Latency increasing
- Maybe disk I/O spiking as PostgreSQL reads from disk
- Maybe connection pool exhaustion

**Then diagnose:** WHY did latency spike? Was it CPU saturation? Disk I/O wait? Lock contention in PostgreSQL? Connection pool exhaustion?

**What you learn:** How to systematically stress a system and diagnose bottlenecks. This is what you DO every day as a performance engineer.

### Component 6: FlameGraphs + Hardware Counters (Stretch — 15 hours)

#### FlameGraph Generation

```bash
# Record CPU profile of your Go API for 30 seconds
sudo perf record -F 99 -g -p $(pgrep go-api) -- sleep 30

# Generate FlameGraph
sudo perf script | \
  ./FlameGraph/stackcollapse-perf.pl | \
  ./FlameGraph/flamegraph.pl > cpu_flame.svg
```

Also use **Go's built-in pprof** (easier, works out of the box):

```bash
# CPU profile
go tool pprof -http=:6060 http://localhost:8080/debug/pprof/profile?seconds=30

# Memory profile
go tool pprof -http=:6060 http://localhost:8080/debug/pprof/heap

# Goroutine profile (find concurrency issues)
go tool pprof -http=:6060 http://localhost:8080/debug/pprof/goroutine
```

#### Hardware Counters

```bash
# Measure IPC, cache misses, branch misses for your Go API
sudo perf stat -e cycles,instructions,cache-misses,branch-misses \
  -p $(pgrep go-api) -- sleep 30

# Output:
#   1,234,567,890  cycles
#     987,654,321  instructions    # IPC = 0.80
#      12,345,678  cache-misses   
#       1,234,567  branch-misses  
```

#### eBPF Tracing (if time allows)

```bash
# Trace all syscalls by your Go API — what is it doing?
sudo bpftrace -e '
  tracepoint:raw_syscalls:sys_enter /comm == "go-api"/ {
    @syscalls[ksym(*(kaddr("sys_call_table") + args.id * 8))] = count();
  }
'
# Shows: read: 45000, write: 32000, futex: 12000, epoll_wait: 8000...

# Trace disk I/O latency
sudo bpftrace -e '
  tracepoint:block:block_rq_issue { @start[args.sector] = nsecs; }
  tracepoint:block:block_rq_complete /@start[args.sector]/ {
    @usecs = hist((nsecs - @start[args.sector]) / 1000);
    delete(@start[args.sector]);
  }
'
```

**What you learn:** Reading FlameGraphs, using `perf`, understanding hardware counters, basic eBPF — the advanced performance engineering toolkit.

---

## 100-Hour Budget

```
Total: 100 hours across 4 people = 25 hours per person

Week 1 (20 hrs total):  FOUNDATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Person A (5 hrs): Set up Linux VM, install Nginx + PostgreSQL
Person B (5 hrs): Write the Go REST API (3 endpoints + /metrics)
Person C (5 hrs): Install Prometheus + Grafana, basic configuration
Person D (5 hrs): Seed PostgreSQL with 100K rows, configure Nginx proxy

Deliverable: Microservice running, accessible at http://localhost
             Prometheus scraping the Go app's built-in metrics
             Grafana showing basic Go runtime metrics

Week 2 (25 hrs total):  SYSTEM INSTRUMENTATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Person A (8 hrs): Write /proc collector — CPU + memory metrics
Person B (7 hrs): Write /proc collector — disk I/O + network metrics  
Person C (5 hrs): Add per-process metrics (/proc/<pid>/stat, /proc/<pid>/io)
Person D (5 hrs): Expose all metrics as Prometheus exporter (HTTP /metrics)

Deliverable: Custom exporter running, Prometheus scraping it
             Raw system data flowing into Prometheus

Week 3 (25 hrs total):  APP METRICS + DASHBOARDS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Person A (5 hrs): Add RED metrics to Go API (request rate, errors, duration)
Person B (5 hrs): Add PostgreSQL metrics (pg_stat_statements, cache hit ratio)
Person C (8 hrs): Build Grafana Dashboard 1 (System Overview) + Dashboard 2 (App)
Person D (7 hrs): Set up alerting rules + write load test scripts (wrk/k6)

Deliverable: Two polished Grafana dashboards
             Alerting rules firing on test conditions
             Load test scripts ready

Week 4 (20 hrs total):  LOAD TESTING + DIAGNOSIS + STRETCH
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Person A (5 hrs): Run load tests, capture baseline, stress until breaking
Person B (5 hrs): Diagnose bottlenecks using dashboards (where does it break?)
Person C (5 hrs): perf record + FlameGraph generation during load test
Person D (5 hrs): perf stat hardware counters + bpftrace one-liners

Deliverable: Load test report with findings ("system breaks at X req/s because Y")
             CPU FlameGraph SVG
             Hardware counter measurements
             Dashboard 3 (Deep Dive) with FlameGraph embed

Week 5 (10 hrs total):  POLISH + DOCUMENTATION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Person A (3 hrs): Write README with architecture diagram
Person B (3 hrs): Record a demo video / screenshot walkthrough
Person C (2 hrs): Clean up code, add comments
Person D (2 hrs): Write "Lessons Learned" + "How to reproduce" docs

Deliverable: GitHub repo ready for resume
             Demo-quality screenshots/video
             Clean documentation
```

---

## What Goes On Your Resume

```
PERFORMANCE OBSERVABILITY TOOL FOR MICROSERVICES
─────────────────────────────────────────────────
• Built end-to-end observability pipeline for Nginx + Go + PostgreSQL
  microservice stack on Linux, instrumenting system and application metrics

• Wrote custom Prometheus exporter parsing /proc filesystem (CPU, memory,
  disk I/O, network) with per-process granularity

• Designed Grafana dashboards tracking RED metrics (request rate, errors,
  p50/p95/p99 latency) with automated alerting on SLA breaches

• Conducted systematic load testing with wrk/k6, diagnosing bottlenecks
  using perf FlameGraphs, hardware performance counters (IPC, cache-miss
  rate), and eBPF syscall tracing

• Technologies: Linux /proc, perf, eBPF/bpftrace, Prometheus, Grafana,
  Go, PostgreSQL, Nginx, wrk/k6
```

That resume bullet hits every keyword a performance engineering hiring manager searches for.

---

## What You Should Be Able to Explain in an Interview After This Project

### System Level
- [ ] How does `/proc/stat` report CPU time? What are the fields? How do you compute utilization from them?
- [ ] What's the difference between `free` and `available` memory in `/proc/meminfo`?
- [ ] What does `iowait` mean? Is high iowait always a problem?
- [ ] How do you detect a disk I/O bottleneck from `/proc/diskstats`?
- [ ] What does load average measure and why can it be misleading?

### Application Level
- [ ] What are the RED metrics? Why are they the standard?
- [ ] Why is p99 latency more important than average latency?
- [ ] What is a Prometheus histogram and how does `histogram_quantile` work?
- [ ] How does Prometheus scraping work? Pull vs push model?
- [ ] What causes connection pool exhaustion?

### Deep Performance
- [ ] How do you read a FlameGraph? What does width mean? What does depth mean?
- [ ] What is IPC (Instructions Per Cycle)? What does low IPC indicate?
- [ ] What are cache misses and why do they cause stalls?
- [ ] What's the difference between `perf record` (sampling) and `perf stat` (counting)?
- [ ] What is eBPF and how is it different from a kernel module?

### Diagnosis
- [ ] Walk through how you'd diagnose a latency spike using your tool
- [ ] How did your system break under load? What was the bottleneck?
- [ ] How would you determine if a problem is CPU-bound vs I/O-bound vs memory-bound?

---

## Minimum Viable Demo (What to Show in an Interview)

When someone asks "tell me about a project," you show:

**Screen 1:** The Grafana System Overview dashboard during a load test. CPU climbing, disk I/O spiking, network throughput increasing.

**Screen 2:** The Application Performance dashboard. Request rate at 5000 req/s, p99 latency rising from 10ms to 200ms, one endpoint's error rate hitting 2%.

**Screen 3:** A FlameGraph showing where the CPU time goes. "See this wide bar? That's `json.Marshal` in our response serialization. It's 40% of CPU. We optimized it by using a faster JSON library and p99 dropped from 200ms to 50ms."

**Screen 4:** `perf stat` output showing IPC of 0.5 for the compute endpoint. "This means the CPU is stalling half the time. We used a FlameGraph sampled on `cache-misses` and found `matrix_multiply()` thrashing the L2 cache."

That's a story. That's what gets you hired.

---

## Tools to Install on Day 1

```bash
# System
sudo apt update && sudo apt install -y \
  build-essential git curl wget \
  linux-tools-$(uname -r) \
  bpfcc-tools python3-bpfcc bpftrace \
  nginx postgresql postgresql-contrib \
  sysstat iotop htop

# Go (if using Go for the API)
wget https://go.dev/dl/go1.22.4.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.22.4.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin

# Prometheus
wget https://github.com/prometheus/prometheus/releases/download/v2.52.0/prometheus-2.52.0.linux-amd64.tar.gz
tar xvf prometheus-*.tar.gz

# Grafana
sudo apt install -y grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server

# FlameGraph tools
git clone https://github.com/brendangregg/FlameGraph.git

# Load testing
sudo apt install -y wrk
# or: go install go.k6.io/k6@latest
```
