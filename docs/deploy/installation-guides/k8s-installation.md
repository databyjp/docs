---
title: Kubernetes
sidebar_position: 3
image: og/docs/installation.jpg
# tags: ['installation', 'Kubernetes']
---

:::tip End-to-end guide
For a tutorial on how to use [minikube](https://minikube.sigs.k8s.io/docs/) to deploy Weaviate on Kubernetes, see the Weaviate Academy course, [Weaviate on Kubernetes](../../academy/deployment/k8s/index.md).
:::

## Requirements

:::caution Production Deployments
For **production environments**, Kubernetes is the **only supported deployment method**. Other deployment options (Docker, docker-compose) are suitable only for testing and development.
:::

* A recent Kubernetes Cluster (at least version 1.23). If you are in a development environment, consider using the kubernetes cluster that is built into Docker desktop. For more information, see the [Docker documentation](https://docs.docker.com/desktop/kubernetes/).
* The cluster needs to be able to provision `PersistentVolumes` using Kubernetes' `PersistentVolumeClaims`.
* A file system that can be mounted read-write by a single node to allow Kubernetes' `ReadWriteOnce` access mode.
* **SAN-based storage preferred** (EBS, VMDK, etc.). ⚠️ **NFS or NFS-like storage classes are not supported**.
* Helm version v3 or higher. The current Helm chart is version `||site.helm_version||`.

## Weaviate Helm chart

:::note Important: Set the correct Weaviate version
As a best practice, explicitly set the Weaviate version in the Helm chart.<br/><br/>

Set the version in your `values.yaml` file or [overwrite the default value](#deploy-install-the-helm-chart) during deployment.
:::

To install the Weaviate chart on your Kubernetes cluster, follow these steps:

### Verify tool setup and cluster access

```bash
# Check if helm is installed
helm version
# Make sure `kubectl` is configured correctly and you can access the cluster.
# For example, try listing the pods in the currently configured namespace.
kubectl get pods
```

### Get the Helm Chart

Add the Weaviate helm repo that contains the Weaviate helm chart.

```bash
helm repo add weaviate https://weaviate.github.io/weaviate-helm
helm repo update
```

Get the default `values.yaml` configuration file from the Weaviate helm chart:
```bash
helm show values weaviate/weaviate > values.yaml
```

## Key Configuration Parameters

Proper configuration of Weaviate requires understanding several critical parameters that affect performance, memory usage, and stability.

### Memory Configuration

#### GOMEMLIMIT
Sets the memory limit for the Go runtime. Keep at least **10 to 20%** for the Go runtime and other processes.

```yaml
env:
  GOMEMLIMIT: "16GiB"  # Example: 80% of 20GB total memory
```

#### GOMAXPROCS
Sets the maximum number of threads for concurrent execution.

```yaml
env:
  GOMAXPROCS: "8"  # Number of CPU cores to use
```

#### LIMIT_RESOURCES
Sets memory usage to **80%** of the total memory and uses **all but one CPU core**. It overrides any `GOMEMLIMIT` values but respects `GOMAXPROCS` settings.

```yaml
env:
  LIMIT_RESOURCES: "true"
```

#### PERSISTENCE_LSM_ACCESS_STRATEGY
By default, Weaviate accesses data using `mmap`. While mmap may be preferred for memory management benefits, if you experience stalling under heavy memory load, try `pread` instead.

```yaml
env:
  PERSISTENCE_LSM_ACCESS_STRATEGY: "mmap"  # or "pread"
```

#### GOGC
Environment variable controlling the garbage collector's target percentage. It determines how much memory can grow before triggering a collection cycle.

```yaml
env:
  GOGC: "100"  # Default value
```

### Memory Requirements Calculation

The HNSW index **must** fit in memory. You can estimate memory usage with this formula:

**Memory = 2 × numDimensions × numVectors × 4B**

For example, 100M vectors with 512 dimensions:
`512 × 10^8 × 4 × 2 ≈ 400GiB`

:::tip Memory Reduction Strategies
- Use vector compression/quantization
- Reduce vector dimensionality (e.g., PCA)
- Reduce HNSW `maxConnections` (increase `ef` and `efConstruction` to maintain recall)
:::

### CPU Requirements

Use this formula to estimate CPU requirements:

**minimumCPU = QPS × AverageCPULatencyPerQuery + 6**

Average CPU latency estimates:
- Pure Vector search: ~3-4ms (1M objects)
- Filtered search: ~10ms
- Hybrid search: ~20ms
- Latency roughly doubles every 10x increase in vectors

Example: For 100M vectors with hybrid search (~80ms latency) at 200 QPS peak:
`200 × 0.080 + 6 ≈ 22 CPU cores`

### Network Configuration

Weaviate requires the following ports:
- **REST API**: Port 8080 (HTTP)
- **gRPC API**: Port 50051 (gRPC/Protocol Buffer)
- **Gossip Protocol**: Port 7102 (internal cluster communication)
- **Data Bind**: Port 7103 (internal data exchange)

For external access, configure LoadBalancer or Ingress appropriately.


To customize the Helm chart for your environment, edit the [`values.yaml`](https://github.com/weaviate/weaviate-helm/blob/master/weaviate/values.yaml)
file. The default `yaml` file is extensively documented to help you configure your system.

#### High Availability and Replication

For production environments, deploy a minimum **3-node cluster** with proper replication:

```yaml
replicas: 3

env:
  REPLICATION_MINIMUM_FACTOR: 3
```

##### Anti-affinity Configuration
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

##### Sharding Strategy
Configure proper sharding for your collections:

```python
# Example Python client configuration
Configure.sharding(virtual_per_physical=128, desired_count=6)
```


#### Local models

Local models, such as `text2vec-transformers`, `qna-transformers`, and  `img2vec-neural` are disabled by default. To enable a model, set the model's
`enabled` flag to `true`.

#### Resource limits

Starting in Helm chart version 17.0.1, constraints on module resources are commented out to improve performance. To constrain resources for specific modules, add the constraints in your `values.yaml` file.

#### gRPC service configuration

Starting in Helm chart version 17.0.0, the gRPC service is enabled by default. If you use an older Helm chart, edit your `values.yaml` file to enable gRPC.

Check that the `enabled` field is set to `true` and the `type` field to `LoadBalancer`. These settings allow you to access the [gRPC API](https://weaviate.io/blog/grpc-performance-improvements) from outside the Kubernetes cluster.

```yaml
grpcService:
  enabled: true  # ⬅️ Make sure this is set to true
  name: weaviate-grpc
  ports:
    - name: grpc
      protocol: TCP
      port: 50051
  type: LoadBalancer  # ⬅️ Set this to LoadBalancer (from NodePort)
```

#### Authentication and authorization

:::tip

Weaviate Helm charts automatically generate random username/password values each time Weaviate is deployed to Kubernetes, this means that when Weaviate is deployed with Helm charts internode communication is always secured.

:::

An example configuration for authentication:

```yaml
authentication:
  apikey:
    enabled: true
    allowed_keys:
      - readonly-key
      - secr3tk3y
    users:
      - readonly@example.com
      - admin@example.com
  anonymous_access:
    enabled: false
  oidc:
    enabled: true
    issuer: https://auth.wcs.api.weaviate.io/auth/realms/SeMI
    username_claim: email
    groups_claim: groups
    client_id: wcs
authorization:
  admin_list:
    enabled: true
    users:
      - someuser@weaviate.io
      - admin@example.com
    readonly_users:
      - readonly@example.com
```

## Backup Strategy

Establish a proper backup strategy using appropriate backup modules:

### Backup Modules

**For production environments:**
- `backup-s3` - Amazon S3 backup
- `backup-gcs` - Google Cloud Storage backup
- `backup-azure` - Azure Storage backup

**For development/testing:**
- `backup-filesystem` - Local filesystem backup

```yaml
modules:
  backup-s3:
    enabled: true
    envconfig:
      BACKUP_S3_BUCKET: "my-weaviate-backups"
      BACKUP_S3_ENDPOINT: "s3.amazonaws.com"
      BACKUP_S3_REGION: "us-east-1"
```

### Alternative: Disk Snapshots

You can also rely on disk snapshots for backup:
- LVM or VMDK snapshots
- EBS snapshots
- Cloud provider snapshot services

:::note
Disk snapshots should be configured outside of Helm charts and require coordination with your infrastructure team.
:::

### Backup Best Practices

- **Schedule**: Implement incremental and regular backup schedules
  - Example: 7 daily, 4 weekly, 6 monthly backups
- **Testing**: Regularly test your recovery procedures
- **Disaster Recovery**: Plan and test disaster recovery scenarios
- **Retention**: Define appropriate backup retention policies


In this example, the key `readonly-key` will authenticate a user as the `readonly@example.com` identity, and `secr3tk3y` will authenticate a user as `admin@example.com`.

OIDC authentication is also enabled, with WCD as the token issuer/identity provider. Thus, users with WCD accounts could be authenticated. This configuration sets `someuser@weaviate.io` as an admin user, so if `someuser@weaviate.io` were to authenticate, they will be given full (read and write) privileges.

import WCDOIDCWarning from '/_includes/wcd-oidc.mdx';

<WCDOIDCWarning/>

For further, general documentation on authentication and authorization configuration, see:
- [Authentication](../configuration/authentication.md)
- [Authorization](../configuration/authorization.md)

## Performance Optimization

### General Performance Settings

```yaml
env:
  # Enable asynchronous indexing for better write performance
  ASYNC_INDEXING: "true"

  # Disable profiling in production for better performance
  GO_PROFILING_DISABLE: "true"

  # Query performance settings
  QUERY_DEFAULTS_LIMIT: "10"
  QUERY_MAXIMUM_RESULTS: "10000"
  QUERY_SLOW_LOG_ENABLED: "false"
```

### Large Scale Optimizations

```yaml
env:
  # Required for large hybrid or keyword search workloads
  USE_BLOCKMAX_WAND: "true"

  # ACORN HNSW index improves filtered vector search performance
  # Configure per collection via client libraries
```

### Schema Management

```yaml
env:
  # Disable auto-schema in production for better control
  AUTOSCHEMA_ENABLED: "false"
```

### Logging Configuration

```yaml
env:
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"
```

#### Run as non-root user

By default, weaviate runs as the root user. To run as a non-privileged user, edit the settings in the `containerSecurityContext` section.

The `init` container always runs as root to configure the node. Once the system is started, it run a non-privileged user if you have one configured.

### Deploy (install the Helm chart)

You can deploy the helm charts as follows:

```bash
# Create a Weaviate namespace
kubectl create namespace weaviate

# Deploy
helm upgrade --install \
  "weaviate" \
  weaviate/weaviate \
  --namespace "weaviate" \
  --values ./values.yaml
```

The above assumes that you have permissions to create a new namespace. If you
have only namespace-level permissions, you can skip creating a new
namespace and adjust the namespace argument on `helm upgrade` according to the
name of your pre-configured namespace.

Optionally, you can provide the `--create-namespace` parameter which will create the namespace if not present.

### Updating the installation after the initial deployment

The above command (`helm upgrade...`) is idempotent. In other words, you can run it multiple times after adjusting your desired configuration without causing any unintended changes or side effects.

### Upgrading to `1.25` or higher from pre-`1.25`

:::caution Important
:::

To upgrade to `1.25` or higher from a pre-`1.25` version, you must delete the deployed `StatefulSet`, update the helm chart to version `17.0.0` or higher, and re-deploy Weaviate.

See the [1.25 migration guide for Kubernetes](../migration/weaviate-1-25.md) for more details.

## Additional Configuration Help

- [Cannot list resource "configmaps" in API group when deploying Weaviate k8s setup on GCP](https://stackoverflow.com/questions/58501558/cannot-list-resource-configmaps-in-api-group-when-deploying-weaviate-k8s-setup)
- [Error: UPGRADE FAILED: configmaps is forbidden](https://stackoverflow.com/questions/58501558/cannot-list-resource-configmaps-in-api-group-when-deploying-weaviate-k8s-setup)

### Using EFS with Weaviate

In some circumstances, you may wish, or need, to use EFS (Amazon Elastic File System) with Weaviate. And we note in the case of AWS Fargate, you must create the [PV (persistent volume)](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) manually, as the PVC will NOT create a PV for you.

To use EFS with Weaviate, you need to:

- Create an EFS file system.
- Create an EFS access point for every Weaviate replica.
    - All of the Access Points must have a different root-directory so that Pods do not share the data, otherwise it will fail.
- Create EFS mount targets for each subnet of the VPC where Weaviate is deployed.
- Create StorageClass in Kubernetes using EFS.
- Create Weaviate Volumes, where each volume has a different AccessPoint for VolumeHandle(as mentioned above).
- Deploy Weaviate.

This code is an example of a PV for `weaviate-0` Pod:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: weaviate-0
spec:
  capacity:
    storage: 8Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: "efs-sc"
  csi:
    driver: efs.csi.aws.com
    volumeHandle: <FileSystemId>::<AccessPointId-for-weaviate-0-Pod>
  claimRef:
    namespace: <namespace where Weaviate is/going to be deployed>
    name: weaviate-data-weaviate-0
```

For more, general information on running EFS with Fargate, we recommend reading [this AWS blog](https://aws.amazon.com/blogs/containers/running-stateful-workloads-with-amazon-eks-on-aws-fargate-using-amazon-efs/).

### Using Azure file CSI with Weaviate
The provisioner `file.csi.azure.com` is **not supported** and will lead to file corruptions. Instead, make sure the storage class defined in values.yaml is from provisioner `disk.csi.azure.com`, for example:

```yaml
storage:
  size: 32Gi
  storageClassName: managed
```

you can get the list of available storage classes in your cluster with:

```
kubectl get storageclasses
```

## Troubleshooting

- If you see `No private IP address found, and explicit IP not provided`, set the pod subnet to be in an valid ip address range of the following:

    ```
    10.0.0.0/8
    100.64.0.0/10
    172.16.0.0/12
    192.168.0.0/16
    198.19.0.0/16
    ```

### Set `CLUSTER_HOSTNAME` if it may change over time

In some systems, the cluster hostname may change over time. This is known to create issues with a single-node Weaviate deployment. To avoid this, set the `CLUSTER_HOSTNAME` environment variable in the `values.yaml` file to the cluster hostname.

```yaml
env:
  - CLUSTER_HOSTNAME: "node-1"
```

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>