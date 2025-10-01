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

## Step 2: Network Security and Service Exposure

### Network Configuration

- Configure services with appropriate network policies:

```yaml
services:
  rest:
    port: 8080
    type: LoadBalancer
  grpc:
    port: 50051
    type: LoadBalancer

networkPolicy:
  enabled: true
  ingress:
    - ports:
      - port: 8080
      - port: 50051
```

### Service Exposure Options

1. **LoadBalancer**:
```yaml
service:
  type: LoadBalancer
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
```

2. **Ingress Controller**:
```yaml
ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - host: weaviate.example.com
      paths:
        - path: /
```

### Authentication and Authorization

Configure robust access controls:

```yaml
authorization:
  rbac:
    enabled: true
    root_users:
      - admin_user1
      - admin_user2

  admin_list:
    enabled: true
    users:
      - admin_user1
      - api-key-user-admin
    read_only_users:
      - readonly_user1
```

:::tip Security Best Practices
- Use TLS encryption
- Implement role-based access control
- Rotate API keys regularly
:::

:::tip
Using an admin list will allow you to define your admin or read-only user/API-key pairs across all Weaviate resources. Whereas RBAC allows you more granular permissions by defining roles and assigning them to users either via API keys or OIDC.
:::

## Step 3: High Availability and Resilience
            matchExpressions:
              - key: "app"
                operator: In
                values:
                  - weaviate
```

### Sharding and Replication

Configure an optimal sharding strategy for your collections:

```yaml
sharding:
  virtualPerPhysical: 128
  desiredCount: 6
```

### Backup and Disaster Recovery

Configure appropriate backup modules for your environment:

```yaml
backup:
  module: backup-s3  # For production
  schedule:
    dailyBackups: 7
    weeklyBackups: 4
    monthlyBackups: 6
  storageConfiguration:
    bucket: my-weaviate-backups
    region: us-east-1
```

### Resource Allocation

Define robust CPU and memory limits to support high availability:

```yaml
resources:
  requests:
    cpu: "2"
    memory: "8Gi"
  limits:
    cpu: "4"
    memory: "16Gi"
```

## Step 4: Memory and Performance Optimization

### Memory Management

- Set appropriate `GOMEMLIMIT` to control Go runtime memory usage:

```yaml
env:
  - name: GOMEMLIMIT
    value: "7GiB"  # Leave room for runtime and other processes
  - name: GOGC
    value: "100"  # Adjust garbage collection threshold
```

### Vector Index Memory Considerations

Estimate vector memory footprint using the formula: $2 * numDimensions * numVectors * 4B$

```yaml
# Example for 100M vectors with 512 dimensions
# Estimated memory: ~400GiB
memory:
  vectorIndex:
    maxConnections: 64
    efConstruction: 128
    ef: 256
```

### Performance Tuning

Optimize Weaviate performance with key configuration parameters:

```yaml
env:
  - name: ASYNC_INDEXING
    value: "true"
  - name: USE_BLOCKMAX_WAND
    value: "true"
  - name: PERSISTENCE_LSM_ACCESS_STRATEGY
    value: "mmap"
```

:::tip Performance Optimization
Reduce memory footprint by:
- Using vector compression
- Reducing vector dimensionality
- Adjusting HNSW index parameters
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