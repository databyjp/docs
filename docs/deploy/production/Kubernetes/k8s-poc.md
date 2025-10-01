# Building with Weaviate: Getting to Production

## Introduction

Are you ready to deploy and test Weaviate on a self-managed K8s (Kubernetes) cluster? This guide shows how to validate Weaviate’s capabilities in your enterprise environment.

At the end of this guide, expect to have:

- A configured Helm-based deployment and networking setup
- Basic scaling, persistent storage, and resource management
- TLS, RBAC, and security best practices implements
- Monitoring, logging, and backup strategies enabled

### Prerequisites

Before beginning, ensure that you have the following:

#### Technical Knowledge

- Basic Kubernetes and containerization conceptual knowledge
- Basic experience with Helm and `kubectl`

:::note

Check out the Academy course ["Run Weaviate on Kubernetes"](https://docs.weaviate.io/academy/deployment/k8s) if you need assistance.

:::

#### Required Infrastructure

- **Kubernetes cluster** (version 1.23 or higher) with access
- **Persistent Volume** support with Storage Class and Provisioner
- **Helm** installed locally
- **kubectl** configured

#### Storage Requirements

:::warning Storage Compatibility

- ⚠️ **No NFS** or NFS-like storage class
- ✅ **SAN-based storage preferred** (EBS, VMDK, etc.)

:::

Kubernetes cluster must be able to create **Persistent Volumes**. If possible, with a **Storage Class** and a **Provisioner** associated.

## Step 1: Configure your Helm Chart

### Basic Installation

1. Add Weaviate Helm repository:

```bash
helm repo add weaviate https://weaviate.github.io/weaviate-helm
helm repo update
```

2. Create dedicated namespace:

```bash
kubectl create namespace weaviate
```

3. Get and customize values.yaml:

```bash
helm show values weaviate/weaviate > values.yaml
```

4. Deploy using Helm:

```bash
helm upgrade --install \
  "weaviate" \
  weaviate/weaviate \
  --namespace "weaviate" \
  --values ./values.yaml
```

### Critical Configuration Parameters

Customize the values.yaml file with these essential settings:

<details>
<summary>Memory Configuration</summary>

```yaml
env:
  # Sets memory limit for Go runtime (keep 10-20% for runtime)
  GOMEMLIMIT: '3200MiB'  # For 4GB total memory

  # Maximum number of threads for concurrent execution
  GOMAXPROCS: '2'

  # Alternative: Uses 80% of total memory and all but one CPU core
  LIMIT_RESOURCES: 'true'  # Overrides GOMEMLIMIT but respects GOMAXPROCS
```
</details>

<details>
<summary>Storage and Performance Settings</summary>

```yaml
env:
  # LSM access strategy - use 'pread' if experiencing memory stalls
  PERSISTENCE_LSM_ACCESS_STRATEGY: 'mmap'  # or 'pread'

  # Enable async indexing for better performance
  ASYNC_INDEXING: 'true'

  # Garbage collector target percentage
  GOGC: '100'
```
</details>

<details>
<summary>Large Scale Optimization</summary>

```yaml
env:
  # Required for large hybrid/keyword search workloads
  USE_BLOCKMAX_WAND: 'true'

  # Improved filtering for vector search
  ACORN: 'true'
```
</details>
## Step 2: Network Security

- Configure an ingress controller to securely expose Weaviate.
- Enable TLS with a certificate manager and enforce TLS encryption for all client-server communication.
- Assign a domain name for external access.
- Implement RBAC or admin lists to restrict user access.

<details>
  <summary> An example of RBAC enabled on your Helm chart </summary>

```yaml
  authorization:
  rbac:
    enabled: true
     root_users:
    - admin_user1
    - admin_user2
```
</details>

<details>
<summary> An example of admin lists implemented on your Helm chart (if not using RBAC)</summary>

```yaml
  admin_list:
    enabled: true
    users:
    - admin_user1
    - admin_user2
    - api-key-user-admin
    read_only_users:
    - readonly_user1
    - readonly_user2
    - api-key-user-readOnly
```
[Admin List Configuration](/deploy/configuration/authorization.md#admin-list-kubernetes)

</details>

:::tip
Using an admin list will allow you to define your admin or read-only user/API-key pairs across all Weaviate resources. Whereas RBAC allows you more granular permissions by defining roles and assigning them to users either via API keys or OIDC.
:::

## Step 3: High Availability and Scaling

### Minimum 3-Node Cluster Setup

For production environments, deploy at least 3 nodes for high availability:

```yaml
replicas: 3

# Configure replication factor
env:
  REPLICATION_MINIMUM_FACTOR: '3'
```

### Anti-Affinity Configuration

Ensure pods are distributed across different nodes:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        podAffinityTerm:
          topologyKey: "kubernetes.io/hostname"
          labelSelector:
            matchExpressions:
              - key: "app"
                operator: In
                values:
                  - weaviate
```

### Resource Requirements

#### Memory Calculation

HNSW index must fit in memory. Estimate using:
```
Memory = 2 × numDimensions × numVectors × 4B
```

For 100M vectors with 512 dimensions:
```
512 × 10^8 × 4 × 2 ≈ 400 GiB
```

Keep 20% extra memory for page cache and additional requirements.

#### CPU Calculation

```
Minimum CPU = QPS × Average_CPU_Latency_Per_Query + 6
```

Estimated latencies:
- Pure vector search: 3-4ms (1M objects)
- Filtered search: ~10ms
- Hybrid search: ~20ms
- Latency doubles every 10x increase in vectors

Example: For 100M vectors with hybrid search (~80ms) at 200 QPS peak:
```
200 × 0.080 + 6 ≈ 22 CPU cores
```

<details>
<summary>Resource Configuration Example</summary>

```yaml
resources:
  requests:
    cpu: "2000m"
    memory: "8Gi"
  limits:
    cpu: "4000m"
    memory: "16Gi"
```
</details>

### Memory Optimization Strategies

- **Vector compression** - Reduce memory footprint significantly
- **Reduce dimensionality** - Use PCA to reduce vector dimensions
- **Tune maxConnections** - Reduce connections, increase `ef` and `efConstruction` to maintain recall

## Step 4: Monitoring, Logging, and Observability

### Prometheus and Grafana Setup

Enable comprehensive monitoring with Prometheus integration:

```yaml
serviceMonitor:
  enabled: true
  interval: 30s
  scrapeTimeout: 10s
  labels:
    app: weaviate
  path: /metrics
```

### Logging Configuration

Configure appropriate logging levels and formats:

```yaml
env:
  # Set log level (debug, info, warn, error)
  LOG_LEVEL: 'info'

  # Set log format (text, json)
  LOG_FORMAT: 'json'

  # Enable slow query logging for performance monitoring
  QUERY_SLOW_LOG_ENABLED: 'true'
```

### Performance Monitoring

Enable Go profiling for performance analysis:

```yaml
env:
  # Disable to enable Go profiler
  GO_PROFILING_DISABLE: 'false'
```

### Alerting Rules

Implement alerting for critical metrics:

<details>
<summary>Example Prometheus Alerting Rules</summary>

```yaml
# Example alerting rules for Weaviate
groups:
  - name: weaviate.rules
    rules:
      - alert: WeaviateHighMemoryUsage
        expr: weaviate_memory_usage_ratio > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Weaviate memory usage is high"

      - alert: WeaviateSlowQueries
        expr: weaviate_query_duration_seconds{quantile="0.95"} > 1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "Weaviate queries are slow"
```
</details>


## Step 5: Backups and Disaster Recovery

### Rolling Update Strategy

Use Helm's rolling update strategy to minimize downtime:

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```

### Backup Module Configuration

Configure appropriate backup modules based on your environment:

<details>
<summary>Production Backup Modules</summary>

```yaml
modules:
  backup-s3:
    enabled: true
    env:
      AWS_REGION: 'us-west-2'
      BACKUP_S3_BUCKET: 'weaviate-backups'

  # Alternative cloud providers
  backup-gcs:
    enabled: false  # Enable for Google Cloud

  backup-azure:
    enabled: false  # Enable for Azure
```
</details>

<details>
<summary>Development/Testing Backup</summary>

```yaml
modules:
  backup-filesystem:
    enabled: true
    env:
      BACKUP_FILESYSTEM_PATH: '/var/lib/weaviate/backups'
```
</details>

### Backup Schedule Strategy

Implement a comprehensive backup schedule:

- **Daily backups**: Retain 7 days
- **Weekly backups**: Retain 4 weeks
- **Monthly backups**: Retain 6 months

### Alternative: Disk Snapshots

You can also rely on infrastructure-level snapshots:

- **LVM** or **VMDK** snapshots
- **EBS** snapshots (AWS)
- **Persistent Disk** snapshots (GCP)
- **Managed Disk** snapshots (Azure)

:::note
Disk snapshots should be configured outside of Helm charts using your cloud provider's tools.
:::

### Disaster Recovery Planning

Essential disaster recovery requirements:

1. **Test recovery procedures** regularly
2. **Document recovery steps** and RTO/RPO targets
3. **Automate backup verification** processes
4. **Plan for cross-region recovery** scenarios
5. **Test failover procedures** in staging environment
## Kubernetes Resources Created

The Helm chart creates the following Kubernetes resources:

| Resource Type | Name | Description |
|---------------|------|-------------|
| **StatefulSet** | weaviate | StatefulSet deploying Weaviate |
| **ServiceMonitor** | weaviate | ServiceMonitor for Prometheus integration |
| **Secret** | weaviate-cluster-api | Contains API keys |
| **ConfigMap** | weaviate-config | Contains Weaviate configuration |
| **Service** (ClusterIP) | weaviate-headless | Used for bootstrapping RAFT |
| **Service** (LoadBalancer) | weaviate | REST and gRPC services for Weaviate |

### Network Ports

Weaviate requires the following ports:

- **REST API**: Port 8080 (HTTP)
- **gRPC API**: Port 50051 (gRPC/Protocol Buffer)
- **Gossip Protocol**: Port 7102 (Internal cluster communication)
- **Data Bind**: Port 7103 (Internal data exchange)

### Network Exposure Options

For external access, configure one of the following:

- **LoadBalancer** (recommended) - Exposes service with external IP
- **Ingress** - HTTP/HTTPS routing with domain names
- **ExternalName** - DNS-based service exposure

:::note
External IP and OpenShift routes can be configured but are not covered by Weaviate Helm charts.
:::
## Step 6: Performance Optimization

### Schema Management

Disable auto-schema for production environments:

```yaml
env:
  # Disable auto-schema creation in production
  AUTOSCHEMA_ENABLED: 'false'
```

### Query Configuration

Optimize query defaults and limits:

```yaml
env:
  # Default query limit
  QUERY_DEFAULTS_LIMIT: '10'

  # Maximum results per query
  QUERY_MAXIMUM_RESULTS: '10000'

  # Enable slow query logging
  QUERY_SLOW_LOG_ENABLED: 'true'
```

### Advanced Performance Settings

```yaml
env:
  # For large-scale keyword/hybrid search
  USE_BLOCKMAX_WAND: 'true'

  # Improved HNSW index for filtered vector search
  ACORN: 'true'

  # Async indexing for better write performance
  ASYNC_INDEXING: 'true'
```

### Collection Configuration

Configure proper sharding strategy:

```python
# Example Python configuration
Configure.sharding(
    virtual_per_physical=128,
    desired_count=6
),
```

### Conclusion

Congratulations! You now have a Weaviate deployment that covers the essential production requirements. However, this is just the foundation. Your deployment still needs:

- **Enhanced Security**: Comprehensive authentication, authorization, and network policies
- **Advanced Monitoring**: Custom dashboards, alerting rules, and performance optimization
- **Operational Procedures**: Upgrade processes, incident response, and maintenance windows

:::warning Production Readiness
While this guide covers the core deployment aspects, achieving full production readiness requires additional security hardening, monitoring setup, and operational procedures.
:::

### Next Steps: [Production Readiness Self-Assessment](./production-readiness.md)

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>