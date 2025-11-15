+++
author = "Smaine Kahlouch"
title = "`PostgreSQL`: From Metrics to Query Plan Analysis"
date = "2025-11-15"
summary = "**CloudNativePG** provides comprehensive PostgreSQL monitoring out of the box—let's see how to enhance it with SQL query analysis and **query plan history**."
featured = true
codeMaxLines = 25
usePageBundles = true
toc = true
series = [
  "observability"
]
tags = [
    "observability",
    "data"
]
thumbnail= "thumbnail.png"
+++

Having basic observability when running PostgreSQL in production isn't optional. Whether it's tracking the [Golden Signals](https://blog.ogenki.io/post/series/observability/alerts/#-the-golden-signals), monitoring slow queries, or analyzing connection patterns, this often requires juggling multiple tools, configuring various exporters, and manually extracting queries for analysis.

❓ What if we could achieve **complete production-grade monitoring with minimal configuration**?

In this article, we'll explore how to enhance CloudNativePG's already robust monitoring with simple, effective, and easy-to-use query performance analysis capabilities—leveraging the power of tools at our disposal: Vector, VictoriaMetrics, and VictoriaLogs.

## 📊 Included with CloudNativePG

When you deploy a PostgreSQL cluster with CNPG, you get **a wealth of metrics and comprehensive Grafana dashboards** out of the box. The operator exposes metrics via a dedicated endpoint on each PostgreSQL instance with the following information:

* **Database Operations**: Transaction rates, queries per second, tuple statistics
* **Replication Status**: Lag, streaming state, synchronization metrics
* **Resource Utilization**: Connections, cache hit ratio, buffer statistics
* **System Health**: Instance status, failover events, backup states

{{% notice tip "GitOps and Kubernetes Operators" %}}
<table>
  <tr>
        <td>
          <img src="repo_gift.png" style="width:80%;">
        </td>
        <td style="vertical-align:middle; padding-left:10px;" width="70%">

The examples in this article come from configurations available in the <strong><a href="https://github.com/Smana/cloud-native-ref">Cloud Native Ref</a></strong> repository.</br>
It leverages several operators, including [CloudNativePG](https://cloudnative-pg.io/) for PostgreSQL management, [VictoriaMetrics](https://victoriametrics.com/) for metrics collection, and [VictoriaLogs](https://github.com/VictoriaMetrics/VictoriaLogs) for log collection.


This project aims to <strong>quickly bootstrap a complete platform</strong> that follows best practices in automation, monitoring, security, and more. </br>
Comments and contributions are welcome 🙏
        </td>
  </tr>
</table>
{{% /notice %}}

### Collecting Metrics with VictoriaMetrics

CloudNativePG automatically exposes metrics on each PostgreSQL pod. To enable their collection, simply activate monitoring in the Helm chart:

```yaml
# CloudNativePG Helm chart
monitoring:
  podMonitorEnabled: true
```

This simple configuration creates a `PodMonitor` (Prometheus Operator resource) that is automatically converted by the VictoriaMetrics operator into a compatible native resource. Metrics from all PostgreSQL pods (primary and replicas) are thus collected and available in VictoriaMetrics.

{{% notice tip "Prometheus Compatibility" %}}
The VictoriaMetrics operator automatically converts Prometheus Operator resources (`PodMonitor`, `ServiceMonitor`, etc.) into their VictoriaMetrics equivalents. This transparent conversion allows using CloudNativePG without modification, while benefiting from VictoriaMetrics as the storage backend.
{{% /notice %}}

### Essential Metrics to Monitor

CloudNativePG exposes metrics that align perfectly with the [Golden Signals](https://blog.ogenki.io/en/post/series/observability/alerts/#-the-golden-signals) methodology discussed in previous articles:

**Latency** ⏳
```promql
# Average query duration
rate(cnpg_backends_total_seconds_sum[5m]) / rate(cnpg_backends_total_seconds_count[5m])
```

**Traffic** 📶
```promql
# Transactions per second
rate(pg_stat_database_xact_commit[5m]) + rate(pg_stat_database_xact_rollback[5m])
```

**Errors** ❌
```promql
# Connection failures and deadlocks
rate(pg_stat_database_deadlocks[5m])
rate(cnpg_pg_postmaster_start_time_seconds[5m])
```

**Saturation** 📈
```promql
# Connection pool usage
cnpg_backends_total / cnpg_pg_settings_max_connections

# Cache hit ratio (should be > 95%)
sum(rate(pg_stat_database_blks_hit[5m])) /
  (sum(rate(pg_stat_database_blks_hit[5m])) + sum(rate(pg_stat_database_blks_read[5m])))
```

### Visualization with Grafana

Using the Grafana Operator explored in [previous articles](https://blog.ogenki.io/en/post/series/observability/metrics/#-visualizing-metrics-with-the-grafana-operator), we can deploy CNPG dashboards declaratively:

```yaml
apiVersion: grafana.integreatly.org/v1beta1
kind: GrafanaDashboard
metadata:
  name: databases-cloudnative-pg
spec:
  allowCrossNamespaceImport: true
  folderRef: "databases"
  datasources:
    - inputName: "DS_PROMETHEUS"
      datasourceName: "VictoriaMetrics"
  instanceSelector:
    matchLabels:
      dashboards: "grafana"
  url: "https://grafana.com/api/dashboards/20417/revisions/4/download"
```

The dashboard provides comprehensive views of our PostgreSQL clusters, including replication lag, query performance, and resource utilization.

<center><img src="grafana_cnpg.png" width=1000 alt="CloudNativePG Grafana"></center>


## 🔍 Understanding Query Performance

While metrics give us the "_what_" and "_when_," they don't always tell us the "_why_." This is where query performance analysis becomes essential. Knowing that queries are slow is useful; understanding **why** they're slow often leads us to optimization opportunities.

Traditional PostgreSQL query analysis requires manually running `EXPLAIN` and `EXPLAIN ANALYZE` commands. While powerful, this approach has limitations:

* **Reactive**: You only analyze queries you suspect are problematic
* **Manual**: Requires active investigation by DBAs
* **Point-in-time**: Captures current execution plan, not historical trends
* **Incomplete**: Difficult to correlate with production load patterns

Ideally, we need **automatic and continuous query plan capture** that allows us to:

1. Automatically identify slow queries
2. Track execution plan changes over time
3. Correlate query performance with system metrics
4. Debug performance issues without manually reproducing them

This is exactly what some managed solutions offer. But can we achieve the same on Kubernetes?

## ✨ Query Plan History: Implementation with Open Source Tools

The good news is that we can build a sophisticated query performance monitoring system that rivals commercial offerings with minimal configuration.

### The Architecture

<center><img src="architecture.png" width=700></center>

Our solution leverages two PostgreSQL extensions and a configuration parameter, integrating them with the VictoriaMetrics ecosystem:

**PostgreSQL Extensions**:
* **pg_stat_statements**: Aggregates query execution statistics
* **auto_explain**: Automatically captures execution plans for slow queries

**PostgreSQL Configuration**:
* **compute_query_id**: Parameter that generates unique identifiers for query correlation

**Observability Stack**:
* **VictoriaMetrics**: Stores query metrics from pg_stat_statements
* **VictoriaLogs**: Stores execution plans with query correlation
* **Vector**: Parses PostgreSQL logs and extracts execution plans
* **Grafana**: Visualizes performance data and enables plan history exploration

The essential link between all these elements is the **correlation between metrics and logs** using the query identifier. This allows us to:
1. See that a query is slow (from metrics)
2. Click to view its execution plan history (from logs)
3. Identify plan changes that caused performance regressions

I've called this feature "**Performance Insights**". Any resemblance to an existing solution would be purely coincidental 😆.

### Enabling Performance Insights

Thanks to CloudNativePG's "[Managed Extensions](https://cloudnative-pg.io/documentation/1.27/postgresql_conf/#managed-extensions)" feature (available since v1.23), enabling comprehensive query monitoring is remarkably simple.

### 🏗️ Platform Engineering: The Right Level of Abstraction

One of the key principles of platform engineering is providing the right level of abstraction to application developers. They shouldn't need to understand PostgreSQL internals or memorize 15+ PostgreSQL-specific configuration parameters.

This is where **Crossplane compositions** excel. In the Cloud Native Ref project, we use Crossplane with KCL (Kubernetes Configuration Language) to create a higher-level abstraction called `SQLInstance`.

**Without Composition** (Raw CNPG Cluster):
```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: myapp-postgres
spec:
  instances: 3
  postgresql:
    shared_preload_libraries:
      - pg_stat_statements
      - auto_explain
    parameters:
      pg_stat_statements.max: "10000"
      pg_stat_statements.track: all
      pg_stat_statements.track_utility: "on"
      pg_stat_statements.track_planning: "on"
      pg_stat_statements.save: "on"
      auto_explain.log_format: json
      auto_explain.log_min_duration: "1000"
      auto_explain.log_analyze: "on"
      auto_explain.log_buffers: "on"
      auto_explain.log_timing: "off"
      auto_explain.log_triggers: "on"
      auto_explain.log_verbose: "on"
      auto_explain.log_nested_statements: "on"
      auto_explain.sample_rate: "0.2"
      compute_query_id: on
      track_activity_query_size: 2048
      track_io_timing: "on"
      log_min_duration_statement: 1000
      # ... and more
```

**With Composition** (Platform Engineering Approach):
```yaml
apiVersion: cloud.ogenki.io/v1alpha1
kind: App
metadata:
  name: myapp
spec:
  sqlInstance:
    enabled: true
    size: small
    storageSize: 20Gi
    instances: 3
    performanceInsights:
      enabled: true
      explain:
        sampleRate: 0.2       # 20% sampling (default: safe for production)
        minDuration: 1000     # Log queries > 1 second (default)
      logStatement: none      # Optional: none (default) / ddl / mod / all
```

The [Crossplane composition](https://github.com/Smana/cloud-native-ref/tree/main/infrastructure/base/crossplane/configuration/kcl/cloudnativepg) **SQLInstance** handles all the complexity.

This composition approach provides several benefits:

1. **Developer Experience**: Application developers don't need PostgreSQL expertise
2. **Consistency**: Performance monitoring is configured uniformly across all databases
3. **Maintainability**: Platform team controls monitoring configuration centrally
4. **Scalability**: Easy to update monitoring parameters for all instances
5. **Discoverability**: Developers can browse available options (`performanceInsights: true`) rather than memorizing parameter names

{{% notice tip "Platform Engineering Principle" %}}
The best abstractions **hide complexity without limiting power**. Developers get performance insights with simple parameters, while the platform team retains the ability to fine-tune the underlying PostgreSQL configuration for advanced use cases.
{{% /notice %}}

### Understanding the Configuration

Let's break down what each component does:

**pg_stat_statements**: This extension tracks execution statistics for all SQL statements executed by a server. It records:
* Total execution time and number of calls
* Rows processed and returned
* Buffer hits and reads
* Query planning time

**auto_explain**: Automatically logs execution plans for queries exceeding a duration threshold. Key parameters include:
* `log_format: json`: Structured output for parsing
* `log_min_duration: 1000`: Capture queries taking more than 1 second (default)
* `log_analyze: on`: Include actual row counts (includes ANALYZE data from the actual execution)
* `sample_rate: 0.2`: Sample 20% of slow queries to reduce overhead (default)

**compute_query_id**: The correlation key that ties everything together. This generates a unique identifier for each query that appears in both pg_stat_statements metrics and auto_explain logs.

{{% notice info "Default Values" %}}
By default, the composition uses production-safe values:
- `sampleRate: 0.2` → 20% sampling of slow queries
- `minDuration: 1000ms` → Capture queries taking more than 1 second

For **debugging**, increase these values:
- `sampleRate: 1.0` → 100% of slow queries
- `minDuration: 0` → All queries, even the fastest ones
{{% /notice %}}

### Vector Log Pipeline Configuration

Here's what Vector does concretely - **transforming a PostgreSQL auto_explain log into an indexable event**:

| **Raw Log (CloudNativePG)** | **After Vector Parsing** |
|------------------------------|--------------------------|
| <pre lang="json" style="font-size: 0.75em;">{<br>  "timestamp": "2025-01-15T14:32:18.456Z",<br>  "message": "{\\"level\\":\\"info\\",<br>    \\"record\\":{\\"query_id\\":\\"8765432109876543210\\",<br>      \\"database_name\\":\\"production\\",<br>      \\"message\\":\\"duration: 245.678 ms plan:\\\\n{...}\\"<br>    }<br>  }",<br>  "kubernetes": {<br>    "pod_labels": {<br>      "cnpg.io/cluster": "myapp-postgres"<br>    }<br>  }<br>}</pre> | <pre lang="json" style="font-size: 0.75em;">{<br>  "_time": "2025-01-15T14:32:18.456Z",<br>  "cluster_name": "myapp-postgres",<br>  "namespace": "apps",<br>  "database": "production",<br>  "query_id": "8765432109876543210",<br>  "duration_ms": 245.678,<br>  "query_text": "SELECT users.email...",<br>  "plan_json": {<br>    "Node Type": "Hash Join",<br>    "..."<br>  }<br>}</pre> |

#### 3-Step Pipeline

The Vector pipeline consists of **3 transforms** and **2 sinks** ([complete configuration](https://github.com/Smana/cloud-native-ref/blob/main/observability/base/victoria-logs/helmrelease-vlsingle.yaml#L62-L338)):

**1. Parse CloudNativePG JSON logs**
```vrl
if .kubernetes.container_name == "postgres" && exists(.kubernetes.pod_labels."cnpg.io/cluster") {
  .log = parse_json(.message)
}
```

**2. Filter for execution plans**
```vrl
exists(.log.record.message) && contains(.log.record.message, "plan:")
```

**3. Extract metadata and plan**
```vrl
.query_id = to_string!(.log.record.query_id)  # Correlation key
.cluster_name = .kubernetes.pod_labels."cnpg.io/cluster"
.database = .log.record.database_name
# Parse plan JSON from "duration: X ms plan: {...}"
.plan_json = parse_json(split(.log.record.message, "plan:")[1])
```

Parsed events are sent to two sinks:
* **Successful plans** → VictoriaLogs with indexing on `cluster_name,namespace,database,query_id`
* **Parse failures** → Separate stream for debugging

#### The Key: Correlation via query_id

The critical element is **query_id** which appears in both systems:
* **VictoriaMetrics**: `pg_stat_statements{queryid="8765432109876543210"}` (metrics)
* **VictoriaLogs**: `{query_id="8765432109876543210"}` (plans)

This correlation allows instant jumping from a performance metric to execution plan history.

## 🔬 Analyzing Query Performance in Action

Once we've identified a problematic query, we can view its execution plan history in VictoriaLogs. Using the query_id from pg_stat_statements, we can query `VictoriaLogs`:

```logsql
# Find all execution plans for a specific query
{cluster_name="myapp-postgres", query_id="1234567890"} | limit 50
```

This shows us:
* All captured execution plans for this query over time
* Plan variations (e.g., index scans vs. sequential scans)
* Actual row counts and execution times
* Buffer usage and I/O statistics

### Understanding EXPLAIN Output

When auto_explain captures a plan, it provides detailed information:

```json
{
  "Query Text": "SELECT * FROM users WHERE email = ?",
  "Query Identifier": 1234567890,
  "Duration": 1567.234,
  "Plan": {
    "Node Type": "Seq Scan",
    "Relation Name": "users",
    "Actual Rows": 1,
    "Actual Loops": 1,
    "Actual Total Time": 1567.123,
    "Shared Hit Blocks": 0,
    "Shared Read Blocks": 54321
  }
}
```

Key insights from this plan:
* **Sequential Scan**: Scanning the entire table instead of using an index
* **High block reads**: 54,321 blocks read from disk (poor cache usage)
* **Single row returned**: Despite scanning the entire table

This immediately suggests the need for an index on the `email` column.

### 🎨 Visualizing Execution Plans with pev2

Understanding complex execution plans from logs can be challenging. This is where **[pev2](https://github.com/dalibo/pev2)** (PostgreSQL Explain Visualizer 2) becomes very useful. It's a web tool that transforms JSON execution plans into interactive, visual diagrams.

```yaml
apiVersion: cloud.ogenki.io/v1alpha1
kind: App
metadata:
  name: xplane-pev2
  namespace: tooling
spec:
  image:
    repository: ghcr.io/smana/pev2
    tag: "v1.17.0"

  resources:
    requests:
      cpu: "10m" # Minimal CPU for static content
      memory: "32Mi" # Small memory footprint
    limits:
      cpu: "300m" # Cap to prevent runaway
      memory: "128Mi" # Limit memory usage

  # Accessible only via Tailscale VPN at: https://pev2.priv.cloud.ogenki.io
  route:
    enabled: true
    hostname: "pev2" # Results in: pev2.priv.cloud.ogenki.io
```

To ensure sensitive query data never leaves the network, pev2 is self-hosted in the cluster via the [App](https://github.com/Smana/cloud-native-ref/tree/main/infrastructure/base/crossplane/configuration/kcl/app) composition. This once again demonstrates the **platform abstraction level**: deploying a static web tool uses the same declarative API as a complete application with a database.

### Analyzing with Grafana

Grafana integration allows quick identification of problematic queries and navigation to their execution plans.

**Performance Analysis Dashboard**

<center><img src="grafana_perf_analysis.png" width=1000 alt="Grafana Dashboard - Performance Analysis"></center>

This dashboard displays key metrics from `pg_stat_statements`: top queries by total duration, average latency, number of calls. Each `query_id` is clickable to explore details.

**Correlation Dashboard**

<center><img src="grafana_plan_correlation.png" width=1000 alt="Grafana Dashboard - Metrics/Logs Correlation"></center>

This dashboard correlates metrics (VictoriaMetrics) with execution plans (VictoriaLogs) for a specific query. It shows performance evolution and plan changes over time.

### Workflow: From Grafana to pev2

The video below shows the complete investigation workflow: from identifying a slow query in Grafana to visual plan analysis with pev2.

<center>
  <video id="QueryPlan" controls width="1000" autoplay loop muted>
    <source src="workflow.mp4" type="video/mp4">
    Your browser does not support the video tag.
  </video>
  <script>
    document.addEventListener('DOMContentLoaded', function() {
      const video = document.getElementById('QueryPlan');
      video.playbackRate = 1.5;
    });
  </script>
</center>

**Workflow steps**:

1. **Identify the slow query** in the Grafana dashboard (`pg_stat_statements` metrics)
2. **Click on the query_id** to view plan history in VictoriaLogs
3. **Copy the JSON plan** and open pev2 (`https://pev2.priv.cloud.ogenki.io`)
4. **Paste the plan** (Ctrl+V) to visualize execution

### Leveraging pev2

<center><img src="pev2_query_plan.png" width=900 alt="pev2 Execution Plan Visualization"></center>

**pev2** transforms JSON plans into interactive diagrams that instantly reveal:

* **Bottlenecks**: Larger nodes = higher execution time (orange/yellow badges = warnings)
* **Planner estimates**: Discrepancies between estimated and actual rows (e.g., "under estimated by 443×" visible on the main Hash Join)
* **Inefficient sequential scans**: Indicators on large tables suggesting missing indexes
* **Join strategies**: Hash Join, Nested Loop, Merge Join with their respective costs
* **I/O patterns**: Ratios of blocks read from disk vs. cache (buffer hits)

The interactive interface allows **clicking on each node** to view details (cost, timing, row count, buffers). **Warning badges** immediately signal potential issues (wrong estimates, inefficient scans).

For performance regressions, VictoriaLogs allows comparing plans before/after by filtering by time period (`_time:[...]`), revealing changes in PostgreSQL planner strategy.

## 💭 Final Thoughts

We've built in this article a complete PostgreSQL performance analysis system that combines **metrics** (`pg_stat_statements`), **execution plans** (`auto_explain`), and **visualization** (pev2). The key to this approach lies in **correlation via query_id**: from a Grafana dashboard showing a slow query, a few clicks are enough to navigate to its execution plan visualized in pev2, enabling performance analysis and optimization.

This is, once again, a demonstration of the **power of available open source tools**. CloudNativePG with added extensions, VictoriaMetrics and VictoriaLogs efficiently store metrics and logs, Vector parses and structures data, and Grafana offers unified visualization. This Kubernetes-native approach is portable and gives complete control.

The abstraction provided by **Crossplane** further amplifies this ease. Thanks to the `App` and `SQLInstance` compositions, enabling Performance Insights boils down to `performanceInsights.enabled: true` with a few tuning parameters (`sampleRate`, `minDuration`). Developers don't need to understand PostgreSQL internals or Vector—the platform masks the complexity. This same declarative API deploys both a complete database and a static web tool like pev2, demonstrating the consistency of the abstraction level.

The [cloud-native-ref](https://github.com/Smana/cloud-native-ref) project brings all these pieces together and shows how Gateway API, Tailscale, Crossplane/KCL, and the VictoriaMetrics ecosystem assemble to create a complete observability platform.

{{% notice note "Performance Consideration" %}}
Enabling Performance Insights involves a measured overhead of **3-4% CPU** and **~200-250MB memory** with default values:
- `pg_stat_statements`: ~1% CPU, 50-100MB RAM
- `auto_explain` (sample_rate=0.2, log_timing=off): ~1% CPU, 50-100MB RAM
- `Vector` parsing: <1% CPU, ~128MB RAM

The default values (`sampleRate: 0.2`, `minDuration: 1000`) are suitable for production. Adjust according to your needs:

* **High-traffic production**: `sampleRate: 0.1` (10%) + `minDuration: 3000` (>3s) — reduce overhead to ~2-3% CPU
* **Debugging/Development**: `sampleRate: 1.0` (100%) + `minDuration: 0` + `log_timing: on` — ~5-7% CPU overhead for maximum capture
* **Standard production**: default values (`sampleRate: 0.2`, `minDuration: 1000`) — excellent balance at 3-4% CPU

The `sampleRate`, `log_timing`, and `logStatement` parameters allow fine-tuning performance impact.
{{% /notice %}}

## 🔖 References

**CloudNativePG Documentation**
- [CloudNativePG Official Documentation](https://cloudnative-pg.io/documentation/)
- [Auto-Managed Extensions](https://cloudnative-pg.io/documentation/current/managed_extensions/)
- [Monitoring and Observability](https://cloudnative-pg.io/documentation/current/monitoring/)

**PostgreSQL Extensions and Features**
- [pg_stat_statements](https://www.postgresql.org/docs/current/pgstatstatements.html) - Query statistics extension
- [auto_explain](https://www.postgresql.org/docs/current/auto-explain.html) - Automatic execution plan logging
- [compute_query_id](https://www.postgresql.org/docs/14/runtime-config-statistics.html#GUC-COMPUTE-QUERY-ID) - Query identifier generation
- [EXPLAIN Documentation](https://www.postgresql.org/docs/current/sql-explain.html) - Understanding query plans

**VictoriaMetrics Ecosystem**
- [VictoriaMetrics Documentation](https://docs.victoriametrics.com/)
- [VictoriaLogs LogsQL](https://docs.victoriametrics.com/victorialogs/logsql/)
- [Vector VRL Language](https://vector.dev/docs/reference/vrl/)

**Query Plan Visualization**
- [pev2 (PostgreSQL Explain Visualizer 2)](https://github.com/dalibo/pev2) - Interactive execution plan visualization
- [pev2 Docker Image](https://hub.docker.com/r/dalibo/pev2) - Self-hosted deployment

**Configuration and Implementation**
- [Vector VRL Configuration](https://github.com/Smana/cloud-native-ref/blob/main/observability/base/victoria-logs/README.md) - PostgreSQL log parsing pipeline
- [CloudNativePG Composition](https://github.com/Smana/cloud-native-ref/blob/main/infrastructure/base/crossplane/configuration/kcl/cloudnativepg/README.md) - SQLInstance abstraction with KCL
- [PostgreSQL Monitoring Architecture](https://github.com/Smana/cloud-native-ref/blob/main/docs/postgresql-monitoring-architecture.md) - Complete architecture documentation
