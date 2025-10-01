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

### Kubernetes Cluster
* A recent Kubernetes Cluster (at least version 1.23). If you are in a development environment, consider using the kubernetes cluster that is built into Docker desktop. For more information, see the [Docker documentation](https://docs.docker.com/desktop/kubernetes/).
* The cluster needs to be able to provision `PersistentVolumes` using Kubernetes' `PersistentVolumeClaims`.
* A file system that can be mounted read-write by a single node to allow Kubernetes' `ReadWriteOnce` access mode.
* Helm version v3 or higher. The current Helm chart is version `||site.helm_version||`.

### Memory Requirements

The HNSW index **must** fit in memory. You can estimate memory usage using this formula:

**Memory = 2 × numDimensions × numVectors × 4B**

For example, 100M vectors with 512 dimensions would require:
512 × 100,000,000 × 4 × 2 ≈ 400GiB

Keep an additional 20% of memory for page cache and other requirements.

#### Strategies to Reduce Memory Usage
- Use **vector compression** (quantization)
- **Reduce vector dimensionality** using techniques like PCA
- **Reduce HNSW maxConnections** while increasing `ef` and `efConstruction`

### CPU Requirements

Estimate CPU requirements using:

**minimumNumberOfCpu = QPS × AverageCPULatencyPerQuery + 6**

Average CPU latency estimates:
- Pure vector search: ~3-4ms (1M objects)
- Filtered search: ~10ms (depends on filter complexity)
- Hybrid search: ~20ms (combines BM25 and vector search)
- Latency roughly doubles when vector count increases 10-fold

Example: For 100M vectors with hybrid search (~80ms latency) at 200 QPS peak:
200 × 0.080 + 6 ≈ 22 CPU cores

### Network Requirements

Weaviate requires these ports:
- **REST API**: 8080 (HTTP)
- **gRPC API**: 50051 (gRPC/Protocol Buffer)
- **Gossip Protocol**: 7102 (internal cluster communication)
- **Data Bind**: 7103 (internal data exchange)

### Storage Requirements

- **SAN-based storage preferred**: EBS, VMDK, etc.
- **⚠️ Avoid NFS** or NFS-like storage classes
- Storage class must support `ReadWriteOnce` access mode
- Persistent Volume provisioning capability required
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

### Modify values.yaml

To customize the Helm chart for your environment, edit the [`values.yaml`](https://github.com/weaviate/weaviate-helm/blob/master/weaviate/values.yaml)
file. The default `yaml` file is extensively documented to help you configure your system.

#### Replication

The default configuration defines one Weaviate replica cluster.

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

## High Availability Setup

### Multi-Node Cluster Configuration

For production environments, deploy a minimum 3-node cluster:

```yaml
replicas: 3

# Configure replication factor
env:
  REPLICATION_MINIMUM_FACTOR: "3"
```

### Pod Anti-Affinity

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

### Sharding Strategy

Configure proper sharding for your collections using the client libraries:

```python
# Example Python configuration
Configure.sharding(virtual_per_physical=128, desired_count=6)
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


In this example, the key `readonly-key` will authenticate a user as the `readonly@example.com` identity, and `secr3tk3y` will authenticate a user as `admin@example.com`.

OIDC authentication is also enabled, with WCD as the token issuer/identity provider. Thus, users with WCD accounts could be authenticated. This configuration sets `someuser@weaviate.io` as an admin user, so if `someuser@weaviate.io` were to authenticate, they will be given full (read and write) privileges.

import WCDOIDCWarning from '/_includes/wcd-oidc.mdx';

<WCDOIDCWarning/>

For further, general documentation on authentication and authorization configuration, see:
- [Authentication](../configuration/authentication.md)
- [Authorization](../configuration/authorization.md)

## Performance and Resource Configuration

### Memory Configuration

#### GOMEMLIMIT
Sets the memory limit for the Go runtime. Keep at least **10 to 20%** of available memory for the Go runtime and other processes.

```yaml
env:
  GOMEMLIMIT: "8GiB"  # For a 10GiB memory node
```

#### LIMIT_RESOURCES
This setting automatically configures memory usage to **80%** of the total available memory and uses **all but one CPU core**. It overrides any `GOMEMLIMIT` values but respects `GOMAXPROCS` settings.

```yaml
env:
  LIMIT_RESOURCES: "true"
```

#### PERSISTENCE_LSM_ACCESS_STRATEGY
By default, Weaviate accesses data using `mmap`. While mmap provides memory management benefits, if you experience stalling under heavy memory load, try `pread` instead.

```yaml
env:
  PERSISTENCE_LSM_ACCESS_STRATEGY: "mmap"  # or "pread"
```

#### GOGC
Controls the garbage collector's target percentage, determining how much memory can grow before triggering a collection cycle.

```yaml
env:
  GOGC: "100"  # Default value
```

### CPU Configuration

#### GOMAXPROCS
Sets the maximum number of threads for concurrent execution.

```yaml
env:
  GOMAXPROCS: "8"  # Match your CPU cores
```

### Performance Optimization

#### Async Indexing
For large-scale deployments, enable asynchronous indexing:

```yaml
env:
  ASYNC_INDEXING: "true"
```

#### Advanced Search Features
For large hybrid or keyword search workloads:

```yaml
env:
  USE_BLOCKMAX_WAND: "true"
  # ACORN HNSW index improves filtering vector search
```

#### Profiling
Enable Go profiler for performance monitoring:

```yaml
env:
  GO_PROFILING_DISABLE: "false"
```

### Schema and Query Configuration

#### Schema Management
For production environments, disable automatic schema creation:

```yaml
env:
  AUTOSCHEMA_ENABLED: "false"
```

#### Logging Configuration
```yaml
env:
  LOG_LEVEL: "info"  # debug, info, warning, error
  LOG_FORMAT: "json"  # json or text
```

#### Query Limits
```yaml
env:
  QUERY_DEFAULTS_LIMIT: "10"
  QUERY_MAXIMUM_RESULTS: "10000"
  QUERY_SLOW_LOG_ENABLED: "true"  # Enable for performance monitoring
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

## Backup Strategy

### Backup Module Configuration

Configure the appropriate backup module based on your environment:

#### Production Environments
```yaml
# For AWS S3
modules:
  backup-s3:
    enabled: true
    env:
      BACKUP_S3_BUCKET: "my-weaviate-backups"
      BACKUP_S3_REGION: "us-east-1"

# For Google Cloud Storage
modules:
  backup-gcs:
    enabled: true
    env:
      BACKUP_GCS_BUCKET: "my-weaviate-backups"

# For Azure Blob Storage
modules:
  backup-azure:
    enabled: true
    env:
      BACKUP_AZURE_CONTAINER: "my-weaviate-backups"
```

#### Development and Testing
```yaml
modules:
  backup-filesystem:
    enabled: true
    env:
      BACKUP_FILESYSTEM_PATH: "/tmp/backups"
```

### Backup Best Practices

#### Backup Schedule Strategy
Implement a comprehensive backup schedule:
- **Daily backups**: Retain for 7 days
- **Weekly backups**: Retain for 4 weeks
- **Monthly backups**: Retain for 6 months

#### Alternative: Disk Snapshots
You can also rely on disk-level snapshots:
- **LVM snapshots**
- **VMDK snapshots**
- **EBS snapshots**
- **Other cloud provider snapshot services**

:::note
Disk snapshots should be configured outside of Helm charts through your infrastructure management tools.
:::

#### Recovery Planning
- **Test recovery procedures** regularly
- **Document disaster recovery** plans
- **Validate backup integrity** periodically

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


## Understanding Weaviate's Data Structure

### The `data` Folder Structure

Weaviate organizes data on disk in a specific structure:

#### Cluster State
- **`raft/`** - Contains cluster state and schema definitions not yet included in snapshots

#### Per Collection and Shard
- **`proplength`** - Global statistics used for BM25 scoring
- **`indexcount`** - Number of documents in the shard
- **`version`** - Shard version (currently `2` since v1.10.1)
- **`main.hnsw.commitlog.d/`** - Write-Ahead Log (WAL) containing vectors for search
- **`lsm/`** - Contains all non-vector indexes

#### LSM Tree Structure
Within the `lsm/` directory:
- **`dimensions/`** - Tracks vector dimensions
- **`objects/`** - Contains complete object data
- **`property_id`** - Converts document UUID to internal uint64 ID
- **`property_*/`** - Stores property values for filtering
- **`property_*_searchable/`** - Contains posting lists for BM25 search
- **`property_*_rangeable/`** - Contains roaring bitmaps for range queries
- **`property_*___meta_count`** - Used for cross-reference joins
- **`vectors`** - Contains vectors for flat index (only if HNSW is disabled)

#### LSM Segment Files
Each LSM maintains several file types:
- **`segment-XXXX.bloom`** - Bloom filters for improved query performance
- **`segment-XXXX.cna`** - Count net additions
- **`segment-XXXX.wal`** - Write-ahead log for newly added documents
- **`segment-XXXX.db`** - Main LSM data file
- **`segment-XXXX.*.tmp`** - Temporary files during creation

:::note
`XXXX` represents the segment timestamp in the filename.
:::

### Log-Structured Merge Tree (LSM) Components

Weaviate uses its own LSM implementation optimized for write-heavy workloads:

- **Memtable**: In-memory buffer for new writes (200MB default)
- **WAL (Write-Ahead Log)**: Ensures durability
- **SSTable (String Sorted Table)**: Immutable disk structures
- **Bloom filters**: Optimize read operations
- **Native Roaring Bitmap** support for efficient filtering

#### Write Path
1. Write goes to Memtable & WAL on disk
2. When Memtable is full, flush to segments (*.db files)
3. Background compaction merges segments continuously

#### Read Path
1. Check Memtable first
2. Use Bloom filters to skip segments if key not present
3. Search through segments until found
4. Merge results if key exists in multiple levels
