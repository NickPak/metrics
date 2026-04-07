## ## ⚡ High-Performance Fork — `optimize/zero-alloc` branch

This is a performance-optimized fork of [VictoriaMetrics/metrics](https://github.com/VictoriaMetrics/metrics), maintained on the **`optimize/zero-alloc`** branch.

### What's different?

This branch includes two key optimizations ([PR #115](https://github.com/VictoriaMetrics/metrics/pull/115), [PR #116](https://github.com/VictoriaMetrics/metrics/pull/116)) that were merged and then reverted upstream:

1. **`sync.Pool` for `bytes.Buffer`** in `WritePrometheus` — eliminates repeated buffer growth allocations
2. **`strconv.Append*` replacing `fmt.Fprintf`** in all `marshalTo` methods — achieves near-zero allocation serialization

### Benchmark Results

#### Micro-benchmarks ([benchstat](https://github.com/VictoriaMetrics/metrics/pull/116#issuecomment-4046793942), Apple M1 Pro)

| Metric Type                    | allocs/op (before → after) | B/op (before → after)       | Speed            |
| ------------------------------ | -------------------------- | --------------------------- | ---------------- |
| Counter                        | 4 → 2 (-50%)               | 80 B → 56 B (-30%)          | 2.1× faster      |
| Float Counter                  | 4 → 2 (-50%)               | 80 B → 56 B (-30%)          | 1.7× faster      |
| Gauge                          | 3 → 2 (-33%)               | 72 B → 56 B (-22%)          | 1.9× faster      |
| Histogram (vmrange)            | 328 → 2 (**-99.4%**)       | 7,465 B → 56 B (**-99.3%**) | **12.4× faster** |
| PromHistogram (11 buckets)     | 95 → 2 (**-97.9%**)        | 1,537 B → 56 B (**-96.4%**) | **5.3× faster**  |
| PromHistogram ext (20 buckets) | 165 → 2 (**-98.8%**)       | 3,060 B → 56 B (**-98.2%**) | **5.8× faster**  |
| Summary                        | 16 → 2 (-87.5%)            | 264 B → 96 B (-63.6%)       | 1.4× faster      |
| Summary ext                    | 6 → 2 (-66.7%)             | 150 B → 96 B (-36.2%)       | 1.6× faster      |

> The remaining 2 allocs/op come from `sync.Pool` buffer management in `Set.WritePrometheus`, not from `marshalTo`. The `marshalTo` methods achieve **true zero allocations**.

#### vs prometheus/client_golang ([benchmark](https://github.com/VictoriaMetrics/metrics/pull/116#issuecomment-4046793942), Windows amd64, i7-13700K, Go 1.26.1)

| Metric Type        | This fork                    | prometheus/client_golang v1.23.2 | Comparison                          |
| ------------------ | ---------------------------- | -------------------------------- | ----------------------------------- |
| Counter            | 87 ns, 56 B, 2 allocs        | 16,000 ns, 34,405 B, 32 allocs   | **184× faster, 99.8% less memory**  |
| Gauge              | 147 ns, 56 B, 2 allocs       | 20,400 ns, 34,402 B, 32 allocs   | **139× faster, 99.8% less memory**  |
| PromHistogram (11) | 3,690 ns, 1,242 B, 61 allocs | 28,100 ns, 36,278 B, 81 allocs   | **7.6× faster, 96.6% less memory**  |
| Summary            | 1,690 ns, 96 B, 2 allocs     | 24,200 ns, 34,844 B, 48 allocs   | **14.3× faster, 99.7% less memory** |

#### Production pprof ([data](https://github.com/VictoriaMetrics/metrics/issues/114#issuecomment-4021701809))

| Metric                | Before         | After (PR #115 + #116) | Change     |
| --------------------- | -------------- | ---------------------- | ---------- |
| alloc_space (total)   | 12,380 MB      | 64 MB                  | **-99.5%** |
| alloc_objects (total) | 71.6M          | 699K                   | **-99.0%** |
| `bytes.growSlice`     | 8,418 MB (68%) | eliminated             | **-100%**  |
| `fmt.Fprintf` (cum)   | 8,426 MB (68%) | eliminated             | **-100%**  |
| inuse_space (RSS)     | ~10.5 MB       | ~11.4 MB               | unchanged  |

> GC pressure is drastically reduced with no increase in steady-state memory usage.

### Who should use this?

If your application has **a large number of metrics** (e.g., high-connection ServiceMesh, API gateways with per-endpoint metrics), the default implementation's allocation overhead and GC pressure can become significant. This branch eliminates that bottleneck.

###### Usage

Since the `go.mod` module path remains `github.com/VictoriaMetrics/metrics`, use `replace` to point to this fork:

```bash
go mod edit -replace github.com/VictoriaMetrics/metrics=github.com/NickPak/metrics@v1.43.1-perf
go mod tidy
```

Your import statements remain unchanged:

```go
import "github.com/VictoriaMetrics/metrics"
```

> Tags follow the upstream version with a `-perf` suffix (e.g., `v1.43.1-perf`). Check [releases](https://github.com/NickPak/metrics/tags) for the latest version.###

### Upstream Sync

This branch is kept in sync with upstream `master`. Only the performance optimizations are added on top.

---

[![Build Status](https://github.com/VictoriaMetrics/metrics/workflows/main/badge.svg)](https://github.com/VictoriaMetrics/metrics/actions)
[![GoDoc](https://godoc.org/github.com/VictoriaMetrics/metrics?status.svg)](http://godoc.org/github.com/VictoriaMetrics/metrics)
[![Go Report](https://goreportcard.com/badge/github.com/VictoriaMetrics/metrics)](https://goreportcard.com/report/github.com/VictoriaMetrics/metrics)
[![codecov](https://codecov.io/gh/VictoriaMetrics/metrics/branch/master/graph/badge.svg)](https://codecov.io/gh/VictoriaMetrics/metrics)

# metrics - lightweight package for exporting metrics in Prometheus format

### Features

* Lightweight. Has minimal number of third-party dependencies and all these deps are small.
  See [this article](https://medium.com/@valyala/stripping-dependency-bloat-in-victoriametrics-docker-image-983fb5912b0d) for details.
* Easy to use. See the [API docs](http://godoc.org/github.com/VictoriaMetrics/metrics).
* Fast.
* Allows exporting distinct metric sets via distinct endpoints. See [Set](http://godoc.org/github.com/VictoriaMetrics/metrics#Set).
* Supports [easy-to-use histograms](http://godoc.org/github.com/VictoriaMetrics/metrics#Histogram), which just work without any tuning.
  Read more about VictoriaMetrics histograms at [this article](https://medium.com/@valyala/improving-histogram-usability-for-prometheus-and-grafana-bc7e5df0e350).
* Can push metrics to VictoriaMetrics or to any other remote storage, which accepts metrics
  in [Prometheus text exposition format](https://github.com/prometheus/docs/blob/main/content/docs/instrumenting/exposition_formats.md#text-based-format).
  See [these docs](http://godoc.org/github.com/VictoriaMetrics/metrics#InitPush).

### Limitations

* It doesn't implement advanced functionality from [github.com/prometheus/client_golang](https://godoc.org/github.com/prometheus/client_golang).

### Usage

```go
import "github.com/VictoriaMetrics/metrics"

// Register various metrics.
// Metric name may contain labels in Prometheus format - see below.
var (
    // Register counter without labels.
    requestsTotal = metrics.NewCounter("requests_total")

    // Register summary with a single label.
    requestDuration = metrics.NewSummary(`requests_duration_seconds{path="/foobar/baz"}`)

    // Register gauge with two labels.
    queueSize = metrics.NewGauge(`queue_size{queue="foobar",topic="baz"}`, func() float64 {
        return float64(foobarQueue.Len())
    })

    // Register histogram with a single label.
    responseSize = metrics.NewHistogram(`response_size{path="/foo/bar"}`)
)

// ...
func requestHandler() {
    // Increment requestTotal counter.
    requestsTotal.Inc()

    startTime := time.Now()
    processRequest()
    // Update requestDuration summary.
    requestDuration.UpdateDuration(startTime)

    // Update responseSize histogram.
    responseSize.Update(responseSize)
}

// Expose the registered metrics at `/metrics` path.
http.HandleFunc("/metrics", func(w http.ResponseWriter, req *http.Request) {
    metrics.WritePrometheus(w, true)
})

// ... or push registered metrics every 10 seconds to http://victoria-metrics:8428/api/v1/import/prometheus
// with the added `instance="foobar"` label to all the pushed metrics.
metrics.InitPush("http://victoria-metrics:8428/api/v1/import/prometheus", 10*time.Second, `instance="foobar"`, true)
```

By default, exposed metrics [do not have](https://github.com/VictoriaMetrics/metrics/issues/48#issuecomment-1620765811)
`TYPE` or `HELP` meta information. Call [`ExposeMetadata(true)`](https://pkg.go.dev/github.com/VictoriaMetrics/metrics#ExposeMetadata)
in order to generate `TYPE` and `HELP` meta information per each metric.

See [docs](https://pkg.go.dev/github.com/VictoriaMetrics/metrics) for more info.

### Users

* `Metrics` has been extracted from [VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics) sources.
  See [this article](https://medium.com/devopslinks/victoriametrics-creating-the-best-remote-storage-for-prometheus-5d92d66787ac)
  for more info about `VictoriaMetrics`.

### FAQ

#### Why the `metrics` API isn't compatible with `github.com/prometheus/client_golang`?

Because the `github.com/prometheus/client_golang` is too complex and is hard to use.

#### Why the `metrics.WritePrometheus` doesn't expose documentation for each metric?

Because this documentation is ignored by Prometheus. The documentation is for users.
Just give [meaningful names to the exported metrics](https://prometheus.io/docs/practices/naming/#metric-names)
or add comments in the source code or in other suitable place explaining each metric exposed from your application.

#### How to implement [CounterVec](https://godoc.org/github.com/prometheus/client_golang/prometheus#CounterVec) in `metrics`?

Just use [GetOrCreateCounter](http://godoc.org/github.com/VictoriaMetrics/metrics#GetOrCreateCounter)
instead of `CounterVec.With`. See [this example](https://pkg.go.dev/github.com/VictoriaMetrics/metrics#example-Counter-Vec) for details.

#### Why [Histogram](http://godoc.org/github.com/VictoriaMetrics/metrics#Histogram) buckets contain `vmrange` labels instead of `le` labels like in Prometheus histograms?

Buckets with `vmrange` labels occupy less disk space compared to Prometheus-style buckets with `le` labels,
because `vmrange` buckets don't include counters for the previous ranges. [VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics) provides `prometheus_buckets`
function, which converts `vmrange` buckets to Prometheus-style buckets with `le` labels. This is useful for building heatmaps in Grafana.
Additionally, its `histogram_quantile` function transparently handles histogram buckets with `vmrange` labels.

However, for compatibility purposes package provides classic [Prometheus Histograms](http://godoc.org/github.com/VictoriaMetrics/metrics#PrometheusHistogram) with `le` labels.
