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

Check out the Academy course [“Run Weaviate on Kubernetes”](https://docs.weaviate.io/academy/deployment/k8s) if you need assistance.

:::

#### Required Tools

- A running Kubernetes cluster with Weaviate installed
- `kubectl` installed
- Helm installed

## Step 1: Configure your Helm Chart

- Use the official [Weaviate Helm chart](https://github.com/weaviate/weaviate-helm) for your installation:

```
  helm repo add weaviate https://weaviate.github.io/weaviate-helm
  helm install my-weaviate weaviate/weaviate
```

- Customize the values to fit your enterprise requirements (e.g., resource allocation, storage settings).
- Deploy the chart and verify pod health.

## Step 2: Network Security and Authentication

### Authentication Strategies

Weaviate supports multiple authentication and authorization methods to secure your deployment:

#### API Key Authentication
Configure API key-based authentication to control access to your Weaviate instance:

```yaml
authentication:
  apikey:
    enabled: true
    allowed_keys:
      - readonly-key
      - admin-key
  anonymous_access:
    enabled: false
```

#### Authorization Methods

Weaviate provides two primary authorization approaches:

1. **RBAC (Role-Based Access Control)**
```yaml
authorization:
  rbac:
    enabled: true
    root_users:
      - admin_user1
      - admin_user2
    roles:
      - name: admin
        permissions: [read, write, delete]
      - name: reader
        permissions: [read]
```

2. **Admin List Configuration**
```yaml
authorization:
  admin_list:
    enabled: true
    users:
      - admin@example.com
    readonly_users:
      - readonly@example.com
```

:::tip
- **RBAC** offers granular permissions by defining roles and assigning them to users
- **Admin List** provides a simpler way to define admin and read-only user/API-key pairs across all Weaviate resources
:::

### Network Security Best Practices
- Configure an ingress controller to securely expose Weaviate
- Enable TLS with a certificate manager
- Enforce TLS encryption for all client-server communication
- Assign a domain name for external access
- Implement strict network policies to restrict access

## Step 3: Scaling and Resource Management

### Replica and High Availability Configuration
Implement horizontal scaling to ensure high availability:

```yaml
replicaCount: 3
```

### Advanced Resource Allocation
Define comprehensive CPU, memory limits, and performance tuning parameters:

```yaml
resources:
  requests:
    cpu: '500m'
    memory: '4Gi'
  limits:
    cpu: '2'
    memory: '8Gi'

env:
  - name: GOMEMLIMIT
    value: '6144MiB'
  - name: GOMAXPROCS
    value: '2'
  - name: LIMIT_RESOURCES
    value: 'true'
  - name: ASYNC_INDEXING
    value: 'true'
```

### Performance Tuning Recommendations
- Keep at least 10-20% memory for Go runtime and other processes
- Set `GOMAXPROCS` to control concurrent execution threads
- Use `LIMIT_RESOURCES` to automatically manage CPU and memory usage
- Enable `ASYNC_INDEXING` for improved write performance

### Resource Estimation Guidelines
- Estimate vector search memory: $2 * numDimensions * numVectors * 4B$
- Keep 20% extra memory for page cache and additional requirements
- For large-scale deployments, consider vector compression techniques
## Step 4: Performance Optimization

### Vector Index Memory Management
- Ensure HNSW index fits entirely in memory
- Use the estimation formula: $2 * numDimensions * numVectors * 4B$
- Example: 100M vectors with 512 dimensions ≈ 400GiB memory

### Memory Reduction Strategies
1. **Vector Compression**
   - Apply quantization techniques
   - Reduce vector dimensionality using Principal Component Analysis (PCA)
   - Adjust `maxConnections` parameter
   - Increase `ef` and `efConstruction` to maintain recall

2. **Search Performance Optimization**
   - Pure vector search: ~3-4ms for 1M objects
   - Filtered search: ~10ms
   - Hybrid search: ~20ms
   - Latency typically doubles with 10x vector increase

### Search Performance Configurations
```yaml
env:
  - name: USE_BLOCKMAX_WAND
    value: 'true'  # Required for large hybrid keyword searches
  - name: ACORN_HNSW_INDEX
    value: 'true'  # Improves filtering in vector search
```

### Workload Estimation
- Calculate CPU requirements: $QPS * AverageCPULatencyPerQuery + 6$
- Example: 200 QPS with 80ms hybrid search ≈ 22 CPU cores

### Recommended Tuning Parameters
- `PERSISTENCE_LSM_ACCESS_STRATEGY`: Default is `mmap`
- `GOGC`: Controls garbage collector's target percentage
- Consider switching to `pread` if experiencing memory stalls
## Step 5: Monitoring and Observability

### Prometheus and Grafana Integration
Configure comprehensive monitoring for your Weaviate deployment:

```yaml
serviceMonitor:
  enabled: true
  interval: 30s
  scrapeTimeout: 10s
  metrics:
    - vector_index_memory_usage
    - query_latency
    - indexing_throughput
    - resource_utilization
```

### Key Metrics to Monitor
1. **Vector Index Metrics**
   - Memory usage
   - Index size
   - Vector compression ratio

2. **Query Performance**
   - Latency
   - Throughput
   - Search type distribution (vector, filtered, hybrid)

3. **System Resources**
   - CPU utilization
   - Memory consumption
   - Disk I/O
   - Network traffic

### Alerting Configuration
Set up alerts for critical performance and availability indicators:
- High memory usage
- Increased query latency
- Indexing bottlenecks
- Node failures

### Logging Best Practices
```yaml
logging:
  level: info
  format: json
  slow_query_log:
    enabled: true
    threshold: 100ms
```

:::tip
Regularly review and adjust monitoring configurations to maintain optimal performance and catch potential issues early.
:::


## Step 5: Upgrades and Backups

- Use the rolling update strategy used by Helm to minimize downtime.

<details>
<summary> An example of configuring the rolling update strategy.</summary>

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 0
```
</details>

- Test new Weaviate versions before deploying into production.
- Implement disaster recovery procedures to ensure that data is restored quickly.

### Conclusion

Voila! You now have a deployment that is *somewhat* ready for production. Your next step will be to complete the self-assessment and identify any gaps.

### Next Steps: [Production Readiness Self-Assessment](./production-readiness.md)

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>