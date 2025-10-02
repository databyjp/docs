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

## Step 3: Scaling

- Implement horizontal scaling to ensure high availability:

```yaml
replicaCount: 3
```

- Define CPU/memory limits and requests to optimize pod efficiency.

<details>
<summary> An example of defining CPU and memory limits and cores </summary>

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    cpu: "2"
    memory: "4Gi"
```
</details>

## Step 4: Monitoring and Logging

- Use Prometheus and Grafana to collect and analyze performance metrics.
- Implement alerting for issue resolution.

<details>
<summary> An example of enabling service monitoring </summary>

```yaml
serviceMonitor:
  enabled: true
  interval: 30s
  scrapeTimeout: 10s
```
</details>

## Step 5: Memory and Performance Optimization

### Memory Management Strategies

Weaviate's performance heavily depends on proper memory allocation. Consider the following optimization techniques:

#### Vector Memory Estimation

Estimate your vector memory footprint using the formula:
`2 * numDimensions * numVectors * 4B`

For example, for 100M vectors with 512 dimensions:
- Estimated memory footprint: `512 * 10^8 * 4 * 2 ~= 400GiB`
- Keep 20% extra memory for page cache and additional requirements

#### Memory Reduction Techniques

1. **Vector Compression**
   - Utilize vector quantization to reduce memory footprint
   - Reduce dimensionality using Principal Component Analysis (PCA)

2. **HNSW Index Optimization**
   - Reduce `maxConnections`
   - Increase `ef` and `efConstruction` to maintain recall and precision

#### Memory Configuration

Configure memory-related environment variables:

```yaml
env:
  - name: GOMEMLIMIT
    value: '80%'  # Keep 10-20% for Go runtime
  - name: GOMAXPROCS
    value: '7'   # Use all but one CPU core
  - name: PERSISTENCE_LSM_ACCESS_STRATEGY
    value: 'mmap'  # Default, switch to 'pread' if experiencing memory stalls
  - name: GOGC
    value: '100'  # Garbage collection target percentage
```

### Performance Optimization

#### CPU Estimation

Estimate CPU requirements using the formula:
`minimumNumberOfCpu = QPS * AverageCPULatencyPerQuery + 6`

Latency estimates:
- Pure Vector search: ~3-4ms (1M objects)
- Filtered search: ~10ms
- Hybrid search: ~20ms

Example for 100M vectors at peak 200 QPS:
- Estimated hybrid search latency: 80ms
- Required CPUs: `200 * 0.080 + 6 ~= 22` CPUs

#### Performance Configuration

```yaml
env:
  - name: ASYNC_INDEXING
    value: 'true'
  - name: USE_BLOCKMAX_WAND
    value: 'true'  # For large hybrid keyword searches
  - name: QUERY_DEFAULTS_LIMIT
    value: '10'
  - name: QUERY_MAXIMUM_RESULTS
    value: '10000'
```

### Logging and Monitoring

```yaml
env:
  - name: LOG_LEVEL
    value: 'info'
  - name: LOG_FORMAT
    value: 'json'
  - name: QUERY_SLOW_LOG_ENABLED
    value: 'true'
```

:::tip
Always test and benchmark your specific use case to fine-tune these parameters for optimal performance.
:::

## Step 6: Upgrades and Backups
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