# Hardware-Counter-Annotated Per-Request FlameGraphs: Analysis

## Quick Verdict

| Dimension | Assessment |
|:----------|:-----------|
| **Novelty** | ⭐⭐⭐⭐⭐ Genuinely novel. Nobody has built this cleanly. |
| **Technical Difficulty** | 🔴🔴🔴🔴 Very hard. The core problem (attributing global hardware counters to individual requests) is an unsolved systems research challenge. |
| **Feasibility in 6 weeks** | ⚠️ Risky. A working prototype is possible. A polished, publishable system is tight. |
| **Publication potential** | ⭐⭐⭐⭐⭐ If you pull it off, this is a full-conference paper, not just a workshop. |

---

## 1. What Exists Today (The Building Blocks)

### Standard FlameGraphs (Brendan Gregg, 2011)

What they show: **which functions** are using CPU time. Width = % of samples.

```
How it's made:
  perf record -F 99 -g -p <pid> -- sleep 30    ← sample stack traces
  perf script | stackcollapse-perf.pl | flamegraph.pl > flame.svg
```

Limitation: **Only shows TIME.** A wide bar means "this function was running a lot" but doesn't tell you WHY it was running a lot.

### CPI FlameGraphs (Brendan Gregg)

An existing extension that uses **color** to show Cycles-Per-Instruction:
- **Red** bars = low CPI (fast, compute-efficient code)
- **Blue** bars = high CPI (slow, stalling on memory)

```
How it's made:
  1. Collect cycles profile:      perf record -e cycles -g ...
  2. Collect stall-cycles profile: perf record -e stalled-cycles-backend -g ...
  3. Use difffolded.pl to color-code the difference
```

Limitation: **Global, not per-request.** Shows the whole program's behavior, not "what happened during request #4827."

### Hardware Counter FlameGraphs (perf -e cache-misses)

You can make a FlameGraph where width = cache misses instead of time:

```bash
perf record -e cache-misses -c 10000 -g -p <pid> -- sleep 30
perf script | stackcollapse-perf.pl | flamegraph.pl > cache_miss_flame.svg
```

Limitation: **Single counter, global.** Shows which functions cause cache misses for the entire program, not per-request. And only one counter at a time.

### Per-Request Distributed Tracing (Jaeger, Zipkin, OpenTelemetry)

Shows the **timeline** of a single request across microservices:

```
Request #4827:
  ├─ Nginx (2ms)
  │   └─ Flask handler (15ms)
  │       ├─ compute_result() (8ms)
  │       └─ db_query() (7ms)
  │           └─ PostgreSQL (5ms)
  └─ Total: 17ms
```

Limitation: **Only shows time.** You know `compute_result()` took 8ms, but NOT why — was it CPU-bound? Cache-miss-bound? TLB-miss-bound?

---

## 2. What Your Idea Adds (The Novel Part)

You want to **combine** the last two things:

```
Request #4827:
  ├─ Nginx (2ms)
  │   └─ Flask handler (15ms)
  │       ├─ compute_result() (8ms)
  │       │   ├── 45,000 cache-misses ← THIS IS NEW
  │       │   ├── 12,000 TLB-misses   ← THIS IS NEW
  │       │   └── IPC: 0.3 (stalling) ← THIS IS NEW
  │       └─ db_query() (7ms)
  │           ├── 2,000 cache-misses
  │           ├── 500 TLB-misses
  │           └── IPC: 2.1 (compute-efficient)
```

Now you know: `compute_result()` is slow because it has terrible cache behavior (45K misses, IPC of 0.3). The database query is fine — it's just waiting for I/O.

**AND** you show this as a FlameGraph where each frame is annotated with multiple hardware counters, and filtered to a specific request.

---

## 3. Why This Is Hard (The Core Technical Problem)

### The PMU-to-Request Attribution Problem

Hardware performance counters live in a **completely different world** than HTTP requests:

```
HARDWARE WORLD:                     APPLICATION WORLD:
─────────────                       ──────────────────
CPU Core 0                          Request #4827
  └── counter: cache-misses = 847     └── Thread 1542
CPU Core 1                            └── handle_request()
  └── counter: cache-misses = 231       └── compute_result()
CPU Core 2                          Request #4828
  └── counter: cache-misses = 1203    └── Thread 1543
CPU Core 3                            └── handle_request()
  └── counter: cache-misses = 92        └── db_query()
```

The CPU has no concept of "requests." It only knows about:
- Processes (PIDs)
- Threads (TIDs)
- CPU cores
- Memory addresses

So when the PMU says "cache-miss happened at instruction 0x7fff4a2b on core 2" — **which request caused it?** You have to figure that out yourself.

### Why This Is NOT a Simple Lookup

Several things make this genuinely hard:

**Problem 1: Thread Reuse**
Most web servers use thread pools. Thread 1542 handles request #4827, then immediately handles request #4828. If you see a cache-miss on thread 1542, which request was it for?

```
Thread 1542:
  [─── request #4827 ───][─── request #4828 ───][─── request #4829 ───]
                      ^
              cache-miss here — which request?
```

You need to know the **exact time boundary** of each request on each thread.

**Problem 2: CPU Migration**
A thread can move between CPU cores during execution. The hardware counter on core 0 counted some misses, then the thread moved to core 2 and that counter counted more misses. You need to track this.

**Problem 3: Sampling vs. Counting**
Hardware counters fire samples **probabilistically** (every N-th cache miss, not every cache miss). For a short request (say 2ms), you might get **zero samples** because no counter overflow happened during that brief window. For long requests you get many. This creates statistical bias.

**Problem 4: Multiple Counters Simultaneously**
Most CPUs can only monitor 4-8 hardware events simultaneously (limited by the number of PMU counter registers). If you want cache-misses, TLB-misses, branch-misses, and IPC all at once, you may need **multiplexing** (rotating between counters), which reduces accuracy.

---

## 4. How You Would Actually Build It

Despite the challenges, here's a concrete architecture:

### The Architecture

```
LAYER 1: Request Boundary Tracking (eBPF uprobes)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Attach uprobes to your Flask/FastAPI request handler:
  - On ENTRY:  record {thread_id → request_id, start_time}
  - On EXIT:   record {thread_id → end_time}
  - Store in BPF_MAP: active_requests[tid] = {req_id, start_ns}

LAYER 2: Hardware Counter Sampling (eBPF perf_event)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Attach BPF_PROG_TYPE_PERF_EVENT to hardware counters:
  - perf_event_open(type=HARDWARE, config=CACHE_MISSES)
  - perf_event_open(type=HARDWARE, config=HW_INSTRUCTIONS)
  - On each sample trigger:
      1. Get current tid via bpf_get_current_pid_tgid()
      2. Look up active_requests[tid] → get request_id
      3. Capture stack trace via bpf_get_stackid()
      4. Store: samples[{req_id, stack_id}] += count

LAYER 3: Stack Trace Storage (BPF Maps)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
BPF_MAP_TYPE_STACK_TRACE stores unique stack traces
Each stack gets a stack_id
samples map: {request_id, stack_id, event_type} → count

LAYER 4: Userspace Aggregation + FlameGraph Generation
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Python script reads BPF maps:
  - For each request_id, collect all stack traces
  - For each stack frame, aggregate counter values
  - Generate annotated FlameGraph SVG
```

### Pseudocode for the eBPF Programs

```c
// ─── PROGRAM 1: Track request boundaries ───
// Attached as uprobe to your_app::handle_request()

struct request_info {
    u64 request_id;
    u64 start_ns;
};

BPF_HASH(active_requests, u32, struct request_info);  // tid → info
static u64 req_counter = 0;

// On request START
SEC("uprobe/handle_request")
int trace_request_entry(struct pt_regs *ctx) {
    u32 tid = (u32)bpf_get_current_pid_tgid();
    struct request_info info = {};
    info.request_id = __sync_fetch_and_add(&req_counter, 1);
    info.start_ns = bpf_ktime_get_ns();
    active_requests.update(&tid, &info);
    return 0;
}

// On request END
SEC("uretprobe/handle_request")
int trace_request_exit(struct pt_regs *ctx) {
    u32 tid = (u32)bpf_get_current_pid_tgid();
    active_requests.delete(&tid);
    return 0;
}


// ─── PROGRAM 2: Correlate hardware events with requests ───
// Attached to perf_event (cache-misses)

struct sample_key {
    u64 request_id;
    int stack_id;
    u32 event_type;  // 0=cache-miss, 1=TLB-miss, etc.
};

BPF_HASH(samples, struct sample_key, u64);
BPF_STACK_TRACE(stack_traces, 16384);

SEC("perf_event")
int on_cache_miss(struct bpf_perf_event_data *ctx) {
    u32 tid = (u32)bpf_get_current_pid_tgid();
    
    // Is this thread handling a request right now?
    struct request_info *info = active_requests.lookup(&tid);
    if (!info) return 0;  // Not in a request — ignore
    
    // Capture the stack trace
    int stack_id = stack_traces.get_stackid(ctx, BPF_F_USER_STACK);
    if (stack_id < 0) return 0;
    
    // Record: this request, at this stack, had a cache miss
    struct sample_key key = {};
    key.request_id = info->request_id;
    key.stack_id = stack_id;
    key.event_type = 0;  // cache-miss
    
    u64 *count = samples.lookup(&key);
    if (count) {
        (*count)++;
    } else {
        u64 one = 1;
        samples.update(&key, &one);
    }
    return 0;
}
```

### What The Output Looks Like

For a single request, you'd generate something like:

```
REQUEST #4827  (total: 15.2ms)
════════════════════════════════════════════════════════════════

                        TIME    CACHE-MISS   TLB-MISS    IPC
handle_request()        15.2ms     47,230      12,450    0.4
├─ parse_json()          1.1ms        120          30    2.8
├─ compute_result()      8.3ms     45,000      12,000    0.3  ← BOTTLENECK
│  ├─ matrix_mult()      6.1ms     42,000      11,500    0.2  ← WORST
│  └─ normalize()        2.2ms      3,000         500    1.1
├─ db_query()            5.0ms      2,100         400    2.1
│  ├─ pg_send()          0.5ms         10          20    3.0
│  └─ pg_wait()          4.5ms          0           0    N/A  ← I/O wait
└─ serialize_response()  0.8ms         10          0    3.2

DIAGNOSIS: compute_result()→matrix_mult() is memory-bound.
           IPC=0.2 with 42K cache misses indicates data doesn't
           fit in cache. Consider tiling or prefetching.
```

---

## 5. What Exists vs. What's Novel

| Component | Exists? | Who Did It? | What's Missing? |
|:----------|:--------|:------------|:----------------|
| FlameGraphs showing time | ✅ Yes | Brendan Gregg (2011) | — |
| FlameGraphs showing one hardware counter | ✅ Yes | `perf -e cache-misses` + flamegraph.pl | Only global, one counter |
| CPI FlameGraphs (color = IPC) | ✅ Yes | Brendan Gregg | Global only, not per-request |
| Per-request distributed tracing | ✅ Yes | Jaeger, Zipkin, OpenTelemetry | No hardware counters |
| eBPF request boundary tracking | ✅ Yes | eBeeMetrics, MiSeRTrace | No hardware counter integration |
| eBPF hardware counter sampling | ✅ Yes | BPF_PROG_TYPE_PERF_EVENT | No request correlation |
| **Per-request FlameGraph with multiple hardware counters annotated per frame** | ❌ **NO** | **Nobody** | **This is your contribution** |

---

## 6. Relevant Papers (Bibliography)

### Directly Related

| # | Paper | Year | Venue | Relevance |
|:--|:------|:-----|:------|:----------|
| 1 | Brendan Gregg, *"The Flame Graph"* | 2016 | CACM / USENIX ATC '17 (talk) | Original FlameGraph. Your baseline. |
| 2 | Brendan Gregg, *"CPI Flame Graphs"* | 2014 | Blog post / ACM Queue | Closest prior art: color-codes CPI onto FlameGraphs. But global, not per-request. |
| 3 | Daniel Wong, *"Characterizing In-Kernel Observability of Latency-Sensitive Request-level Metrics with eBPF"* | 2024 | IEEE ISPASS | Per-request kernel metrics via eBPF. Finds hardware counters correlate poorly with request latency — **you'd need to address this finding.** |
| 4 | *eBeeMetrics* | 2024 | UC Riverside (eScholarship) | Uses kprobes + uprobes to correlate kernel events with request IDs. No hardware counters. |
| 5 | *MiSeRTrace: Kernel-level Request Tracing for Microservices* | 2024 | SPEC/ICPE | Traces request flow through kernel without app modification. No hardware counter integration. |
| 6 | *CRISP: Critical Path Analysis of Microservice Traces* | 2022 | USENIX ATC | Request-level critical path analysis + FlameGraph visualization. No hardware counters. |

### Hardware Profiling Foundations

| # | Paper | Year | Venue | Relevance |
|:--|:------|:-----|:------|:----------|
| 7 | V. Weaver, *"Self-monitoring Overhead of the Linux perf_event Performance Counter Interface"* | 2015 | IEEE ISPASS | Overhead of hardware counter access. Relevant to your overhead concerns. |
| 8 | *"PEBS: Precise Event-Based Sampling"* | Various | Intel docs | Hardware mechanism for accurate counter attribution. You'll likely need PEBS. |
| 9 | Netflix Tech Blog, *"Java in Flames"* | 2015 | Blog | Practical FlameGraph usage in production microservices. Good framing reference. |

### Context & Background

| # | Paper | Year | Venue | Relevance |
|:--|:------|:-----|:------|:----------|
| 10 | *TraceWeaver* | 2023 | UIUC / NSDI | Request trace reconstruction without app modification. Complementary approach. |
| 11 | Landau et al., *"eBPF-Based Instrumentation for Generalisable Diagnosis of Performance Degradation"* | 2025 | arXiv | 16 eBPF kernel metrics for diagnosis. Related but no per-request hardware counters. |
| 12 | *Parca / Pyroscope* (open-source continuous profilers) | 2023+ | CNCF | Industry tools doing eBPF-based profiling. No per-request hardware counter correlation. |

---

## 7. Feasibility Assessment: Can You Do This in 6 Weeks?

### Honest Breakdown

| Component | Difficulty | Time Estimate | Risk |
|:----------|:-----------|:-------------|:-----|
| Set up microservice (Nginx + Flask + PG) | Easy | 2–3 days | Low |
| eBPF request boundary tracking (uprobes) | Medium | 3–5 days | Medium — uprobes on Python are tricky |
| eBPF hardware counter sampling (BPF_PROG_TYPE_PERF_EVENT) | **Hard** | 5–7 days | High — less documented, kernel version sensitive |
| Correlating counters with requests (the core problem) | **Hard** | 5–7 days | High — timing precision, thread pool handling |
| Userspace aggregation + FlameGraph rendering | Medium | 3–5 days | Medium |
| Paper writing | Medium | 7–10 days | Low |
| **Total** | | **~30–40 days** | |

With 4 people × 7 hrs × 42 days = **1,176 person-hours available**, this is technically possible but **leaves little margin for error**.

### The Biggest Risks

> [!WARNING]
> **Risk 1: Python uprobes are fragile.** Python is interpreted, so you can't easily attach uprobes to Python function entry points like you can with C/Go. You'd need to either:
> - Use a **Go or C** application instead of Flask (much easier for uprobes)
> - Use **USDT probes** that Python/CPython exposes
> - Instrument at the **WSGI/ASGI layer** (C level) rather than Python function level

> [!WARNING]
> **Risk 2: Sample sparsity for short requests.** If a request takes 2ms and you sample cache-misses every 10,000th miss, you might get 0–2 samples per request. You'd need to either:
> - Aggregate across many similar requests (loses the "per-request" claim)
> - Increase sampling frequency (increases overhead)
> - Use counting mode with delta calculation (complex)

> [!CAUTION]
> **Risk 3: This might need a Go/C microservice.** eBPF uprobes work best with compiled languages where function boundaries map cleanly to instruction addresses. If you use Flask (Python), the uprobe needs to attach to CPython internals, which is fragile and kernel-version-dependent. **Strongly consider writing your microservice in Go.**

---

## 8. Recommendation: How to Fit This Into Your Project

### Option A: This as the MAIN project (high risk, high reward)

Drop the overhead analysis. Focus entirely on building the per-request hardware-annotated FlameGraph tool. If it works, you have a **strong conference paper** (not just a workshop).

**Risk:** If you hit a wall with uprobes or sample sparsity, you have nothing to publish.

### Option B: Overhead analysis PLUS this as stretch goal (safe + ambitious)

Keep the overhead analysis (3-way comparison of /proc vs perf vs eBPF) as your **core contribution**. Add the hardware-annotated FlameGraph as a **stretch goal / Section 6 of the paper**.

**This is the recommended path.** You get:
- A guaranteed publishable result (overhead analysis)
- A "wow factor" stretch goal that elevates the paper
- Even a partial prototype of the FlameGraph tool adds novelty

### Week-by-Week for Option B

| Week | Core Work (overhead analysis) | Stretch (annotated FlameGraphs) |
|:-----|:------------------------------|:-------------------------------|
| 1 | Set up microservice + baseline | Person A reads CPI FlameGraph + PEBS docs |
| 2 | Build /proc, perf, eBPF collectors | Person A: basic eBPF request tracking |
| 3 | Run overhead experiments | Person A: eBPF perf_event counter sampling |
| 4 | Analysis + advanced experiments | Person A: correlate counters with requests |
| 5 | Write paper (core sections) | Person A: generate annotated FlameGraph prototype |
| 6 | Polish + submit | Include FlameGraph as Section 6 / demo |

---

## 9. Key Technical Decisions to Make Now

| Decision | Recommended Choice | Why |
|:---------|:-------------------|:----|
| **Microservice language** | **Go** (not Python) | uprobes work cleanly with compiled binaries |
| **eBPF framework** | **libbpf + CO-RE** (not BCC) | BPF_PROG_TYPE_PERF_EVENT is better supported in libbpf |
| **Hardware counters to target** | cache-misses, instructions, cycles (start with 3) | These are universally available on all x86 CPUs |
| **PEBS support** | Required | Without PEBS, counter attribution has "skid" (wrong instruction) |
| **Request tracking** | uprobe on Go HTTP handler | Clean function boundary, no interpreter overhead |
| **Sample aggregation** | Per-request-type, not per-individual-request | Avoids sparsity problem — aggregate all `/cpu-heavy` requests |
