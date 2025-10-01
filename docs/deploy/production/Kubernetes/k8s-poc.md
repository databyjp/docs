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

:::tip Authorization Methods
Weaviate supports two primary authorization methods:

1. **Admin List**: Simple user/API-key pair management across all Weaviate resources
2. **RBAC (Role-Based Access Control)**: More granular permissions via roles, assignable through API keys or OIDC

Choose the method that best fits your security requirements and operational complexity.
:::

## Step 3: Scaling

- Implement horizontal scaling to ensure high availability:

```yaml
replicaCount: 3
```

## Resource Management and Performance

### Memory Estimation and Allocation

When planning your Weaviate deployment, use this memory estimation formula:

**Memory Footprint = 2 * numDimensions * numVectors * 4 Bytes**

For example:
- 100M vectors with 512 dimensions would require ~400 GiB of memory
- Always keep 20% extra memory for page cache and additional requirements

#### Memory Configuration Strategies

1. **GOMEMLIMIT Configuration**
   - Set memory limit for Go runtime
   - Keep 10-20% of total memory for runtime and other processes

2. **Vector Memory Optimization**
   - Use vector compression techniques
   - Reduce dimensionality via Principal Component Analysis (PCA)
   - Adjust HNSW index parameters like `maxConnections`

```yaml
resources:
  requests:
    cpu: '500m'
    memory: '1Gi'
  limits:
    cpu: '2'
    memory: '4Gi'
  environment:
    GOMEMLIMIT: '3Gi'
    PERSISTENCE_LSM_ACCESS_STRATEGY: 'mmap'
    GOGC: '100'
```

### Performance Optimization

Configure performance-critical settings to enhance Weaviate's efficiency:

```yaml
environment:
  ASYNC_INDEXING: true
  USE_BLOCKMAX_WAND: true
  ACORN_HNSW_INDEX: true
```

### Scaling and Replication

Implement robust scaling strategies:

```yaml
replicaCount: 3
replication:
  factor: 3
  sharding:
    virtual_per_physical: 128
    desired_count: 6
```

### Backup and Disaster Recovery

Select appropriate backup modules based on your environment:

- Production: `backup-s3`, `backup-gcs`, `backup-azure`
- Development: `backup-filesystem`

Implement a comprehensive backup strategy:
- Regular incremental backups
- Retention policy (e.g., 7 daily, 4 weekly, 6 monthly)
- Test recovery procedures

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