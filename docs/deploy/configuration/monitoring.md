---
title: Monitoring
image: og/docs/configuration.jpg
# tags: ['configuration', 'operations', 'monitoring', 'observability']
---

Weaviate can expose Prometheus-compatible metrics for monitoring. A standard
Prometheus/Grafana setup can be used to visualize metrics on various
dashboards.

Metrics can be used to measure request latencies, import
speed, time spent on vector vs object storage, memory usage, application usage,
and more.
## How to Monitor Weaviate

### Monitoring Methods

#### Prometheus/OpenMetrics Endpoint
Enable Prometheus monitoring by setting:

```sh
PROMETHEUS_MONITORING_ENABLED=true
```

By default, metrics are exposed at `<hostname>:2112/metrics`. You can customize the port:

```sh
PROMETHEUS_MONITORING_PORT=3456
```

#### Weaviate API Logs
Use API logs to gain insights into:
- Query performance
- Cluster stability
- Detailed request and response information

#### Third-party APM Integration
Integrate with Application Performance Monitoring (APM) tools such as:
- Datadog
- New Relic
- Dynatrace

These tools provide comprehensive monitoring and visualization of Weaviate's performance and health.
### Scrape metrics from Weaviate

Metrics are typically scraped into a time-series database, such as Prometheus.
How you consume metrics depends on your setup and environment.

The [Weaviate examples repo contains a fully pre-configured setup using
Prometheus, Grafana and some example
dashboards](https://github.com/weaviate/weaviate-examples/tree/main/monitoring-prometheus-grafana).
You can start up a full-setup including monitoring and dashboards with a single
command. In this setup the following components are used:

* Docker Compose is used to provide a fully-configured setup that can be
  started with a single command.
* Weaviate is configured to expose Prometheus metrics as outlined in the
  section above.
* A Prometheus instance is started with the setup and configured to scrape
  metrics from Weaviate every 15s.
* A Grafana instance is started with the setup and configured to use the
  Prometheus instance as a metrics provider. Additionally, it runs a dashboard
  provider that contains a few sample dashboards.

### Multi-tenancy

When using multi-tenancy, we suggest setting the `PROMETHEUS_MONITORING_GROUP` [environment variable](/deploy/configuration/env-vars/index.md) as `true` so that data across all tenants are grouped together for monitoring.

## Key Metrics to Monitor

### Performance Metrics

#### CPU Usage
- Metric: `container_cpu_usage_seconds_total`
- Purpose: Ensure sufficient resources for queries
- Monitor: Overall CPU utilization and per-container performance

#### Memory Usage
- Metrics:
  - `go_memstats_heap_inuse_bytes`
  - `container_memory_working_set_bytes`
- Purpose: Prevent memory leaks and optimize cache
- Monitor: Heap usage, working set memory

#### Disk IO
- Metrics:
  - `container_fs_writes_bytes_total`
  - `container_fs_read_bytes_total`
- Purpose: Track read/write performance
- Monitor: Disk throughput and potential bottlenecks

### Query Performance Metrics

#### Query Latency
- Metric: `queries_durations_ms_bucket`
- Purpose: Track read response times
- Monitor: Query performance across different operations

#### Batch Operations
- Metric: `batch_durations_ms`
- Purpose: Measure write operation performance
- Monitor: Batch import and update speeds

### System State Metrics

#### Object Count
- Metric: `object_count`
- Purpose: Track number of vectors in collections
- Monitor: Data growth and collection sizes

#### Tenant Management
- Metric: `weaviate_schema_shards`
- Purpose: Track tenant states
- Monitor: `Active`, `Inactive`, and `Offloaded` tenant statuses

## Query Performance Debugging

### Slow Query Logging

Configure slow query logging to automatically trace and log performance issues:

```sh
QUERY_SLOW_LOG_ENABLED=true
QUERY_SLOW_LOG_THRESHOLD=50ms  # Log queries taking over 50ms
```

Example slow query log output:
```
level=warning msg="Slow query detected (0s)" ...
keyword_ranking="<nil>" limit=100
query=ObjectSearch shard=multitenant_tenantA
sort="[]" tenant=tenantA
took="61.917µs"
```

## Weaviate Troubleshooting

### Go Profiling

Enable profiling to diagnose performance issues:

```sh
GO_PROFILING_DISABLE=false
GO_PROFILING_PORT=6060
```

#### CPU Profiling
```bash
# Local CPU Profiling
go tool pprof --http=:6061 http://localhost:6060/debug/pprof/profile?seconds=30

# Remote CPU Profiling
go tool pprof -proto http://localhost:6060/debug/pprof/profile?seconds=10
```

#### Memory Profiling
```bash
# Local Memory Profiling
go tool pprof -http=:6061 -lines http://localhost:6060/debug/pprof/heap

# Remote Memory Profiling
go tool pprof --http=:6061 profile002.pb.gz
```

## Recommended Alerts

### Critical Monitoring Alerts

1. **Out of Memory (OOM)**
   ```yaml
   - alert: OutOfMemory
     expr: kube_pod_container_status_last_terminated_reason{container="weaviate", reason="OOMKilled"}
     labels:
       severity: critical
   ```

2. **Weaviate Restart Incidents**
   ```yaml
   - alert: WeaviateRestarting
     expr: kube_pod_container_status_waiting_reason{reason=~"CrashLoopBackOff|ErrImagePull|ImagePullBackOff"}
     labels:
       severity: high
   ```

3. **Disk Space Utilization**
   ```yaml
   - alert: LowDiskSpace
     expr: sum by (namespace, persistentvolumeclaim) (kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) > 0.9
     labels:
       severity: warning
   ```

4. **Memory Resource Limits**
   ```yaml
   - alert: HighMemoryUsage
     expr: sum by (pod) go_memstats_heap_inuse_bytes / kube_pod_container_resource_limits > 0.8
     labels:
       severity: warning
   ```

## Best Practices for Monitoring

1. **Implement Comprehensive Monitoring**
   - Create detailed dashboards
   - Configure intelligent alerts

2. **Continuous Improvement**
   - Regularly review alert thresholds
   - Update dashboards based on evolving use cases

3. **Smart Alerting**
   - Minimize false positives
   - Set appropriate sensitivity levels

## Resources

- [Weaviate Monitoring Guide](https://weaviate.io/developers/weaviate/configuration/monitoring)
- [Weaviate Grafana Dashboards](https://weaviate.io/developers/weaviate/configuration/monitoring#sample-dashboards)
- [Available Metrics Documentation](https://weaviate.io/developers/weaviate/configuration/monitoring#obtainable-metrics)
Extending Weaviate with new metrics is very easy. To suggest a new metric, see the [contributor guide](/contributor-guide).

### Versioning

Be aware that metrics do not follow the semantic versioning guidelines of other Weaviate features. Weaviate's main APIs are stable and breaking changes are extremely rare. Metrics, however, have shorter feature lifecycles. It can sometimes be necessary to introduce an incompatible change or entirely remove a metric, for example, because the cost of observing a specific metric in production has grown too high. As a result, it is possible that a Weaviate minor release contains a breaking change for the Monitoring system. If so, it will be clearly highlighted in the release notes.

## Sample Dashboards

Weaviate does not ship with any dashboards by default, but here is a list of
dashboards being used by the various Weaviate teams, both during development,
and when helping users. These do not come with any support, but may still be
helpful. Treat them as inspiration to design your own dashboards which fit
your uses perfectly:

| Dashboard                                                                                                                     | Purpose                                                                                                                 | Preview                                                                                                            |
| ----------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| [Cluster Workload in Kubernetes](https://github.com/weaviate/weaviate/blob/main/tools/dev/grafana/dashboards/kubernetes.json) | Visualize cluster workload, usage and activity in Kubernetes                                                            | ![Cluster Workload in Kubernetes](./img/weaviate-sample-dashboard-kubernetes.png 'Cluster Workload in Kubernetes') |
| [Importing Data Into Weaviate](https://github.com/weaviate/weaviate/blob/master/tools/dev/grafana/dashboards/importing.json)  | Visualize speed of import operations (including its components, such as object store, inverted index, and vector index) | ![Importing Data into Weaviate](./img/weaviate-sample-dashboard-importing.png 'Importing Data Into Weaviate')      |
| [Object Operations](https://github.com/weaviate/weaviate/blob/master/tools/dev/grafana/dashboards/objects.json)               | Visualize speed of whole object operations, such as GET, PUT, etc.                                                      | ![Objects](./img/weaviate-sample-dashboard-objects.png 'Objects')                                                  |
| [Vector Index](https://github.com/weaviate/weaviate/blob/master/tools/dev/grafana/dashboards/vectorindex.json)                | Visualize the current state, as well as operations on the HNSW vector index                                             | ![Vector Index](./img/weaviate-sample-dashboard-vector.png 'Vector Index')                                         |
| [LSM Stores](https://github.com/weaviate/weaviate/blob/master/tools/dev/grafana/dashboards/lsm.json)                          | Get insights into the internals (including segments) of the various LSM stores within Weaviate                          | ![LSM Store](./img/weaviate-sample-dashboard-lsm.png 'LSM Store')                                                  |
| [Startup](https://github.com/weaviate/weaviate/blob/master/tools/dev/grafana/dashboards/startup.json)                         | Visualize the startup process, including recovery operations                                                            | ![Startup](./img/weaviate-sample-dashboard-startup.png 'Vector Index')                                             |
| [Usage](https://github.com/weaviate/weaviate/blob/master/tools/dev/grafana/dashboards/usage.json)                             | Obtain usage metrics, such as number of objects imported, etc.                                                          | ![Usage](./img/weaviate-sample-dashboard-usage.png 'Usage')                                                        |
| [Aysnc index queue](https://github.com/weaviate/weaviate/blob/main/tools/dev/grafana/dashboards/index_queue.json)             | Observe index queue activity                                                                                            | ![Async index queue](./img/weaviate-sample-dashboard-async-queue.png 'Async index queue')                          |

## `nodes` API Endpoint

To get collection details programmatically, use the [`nodes`](/deploy/configuration/nodes.md) REST endpoint.

import APIOutputs from '/_includes/rest/node-endpoint-info.mdx';

<APIOutputs />

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>