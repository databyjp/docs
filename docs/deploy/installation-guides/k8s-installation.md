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

* A recent Kubernetes Cluster (at least version 1.23). If you are in a development environment, consider using the kubernetes cluster that is built into Docker desktop. For more information, see the [Docker documentation](https://docs.docker.com/desktop/kubernetes/).
* The cluster needs to be able to provision `PersistentVolumes` using Kubernetes' `PersistentVolumeClaims`.
* A file system that can be mounted read-write by a single node to allow Kubernetes' `ReadWriteOnce` access mode.
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

### Modify values.yaml

To customize the Helm chart for your environment, edit the [`values.yaml`](https://github.com/weaviate/weaviate-helm/blob/master/weaviate/values.yaml)
file. The default `yaml` file is extensively documented to help you configure your system.

#### Replication

The default configuration defines one Weaviate replica cluster.

#### Local models

Local models, such as `text2vec-transformers`, `qna-transformers`, and  `img2vec-neural` are disabled by default. To enable a model, set the model's
`enabled` flag to `true`.

#### Resource Limits and Performance Tuning

Starting in Helm chart version 17.0.1, constraints on module resources are commented out to improve performance. We recommend explicitly defining resource limits to optimize vector search performance:

```yaml
resources:
  requests:
    cpu: '500m'
    memory: '4Gi'
  limits:
    cpu: '2'
    memory: '16Gi'
  modules:
    text2vec-transformers:
      resources:
        limits:
          memory: '4Gi'
```

##### Vector Search Optimization

When deploying Weaviate for vector search, consider the following optimization strategies:

- **HNSW Index Configuration**:
  - Tune `maxConnections` to balance memory usage and search performance
  - Adjust `efConstruction` and `ef` parameters for optimal recall
  - Example configuration:
    ```yaml
    env:
      - VECTOR_INDEX_DEFAULT_MAX_CONNECTIONS: 64
      - VECTOR_INDEX_DEFAULT_EF_CONSTRUCTION: 200
      - VECTOR_INDEX_DEFAULT_EF: 100
    ```

- **Memory Allocation**:
  - Estimate vector memory footprint: $2 * numDimensions * numVectors * 4B$
  - Keep 20% extra memory for page cache and system requirements
  - Consider vector quantization to reduce memory usage
  - Use compression techniques to minimize vector storage overhead

- **Performance Tuning**:
  - Enable async indexing for write-heavy workloads
  - Use `LIMIT_RESOURCES` to optimize CPU and memory utilization
  - Consider enabling `USE_BLOCKMAX_WAND` for large hybrid searches
  ```yaml
  env:
    - ASYNC_INDEXING: 'true'
    - USE_BLOCKMAX_WAND: 'true'
  ```

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

#### Authentication and Authorization

:::tip Enhanced Authentication Management
Weaviate Helm charts provide flexible authentication options with granular key and user management.
:::

Example advanced authentication configuration:

```yaml
authentication:
  apikey:
    enabled: true
    allowed_keys:
      - key: readonly-key
        user: readonly@example.com
        permissions: read
      - key: admin-key
        user: admin@example.com
        permissions: write
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
  role_based_access_control:
    enabled: true
    roles:
      - name: admin
        permissions: [read, write]
      - name: reader
        permissions: [read]
```

Key improvements:
- Granular key-based permissions
- Role-based access control
- Detailed user and group management
OIDC authentication is also enabled, with WCD as the token issuer/identity provider. Thus, users with WCD accounts could be authenticated. This configuration sets `someuser@weaviate.io` as an admin user, so if `someuser@weaviate.io` were to authenticate, they will be given full (read and write) privileges.

import WCDOIDCWarning from '/_includes/wcd-oidc.mdx';

<WCDOIDCWarning/>

For further, general documentation on authentication and authorization configuration, see:
- [Authentication](../configuration/authentication.md)
- [Authorization](../configuration/authorization.md)

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

### Using Persistent Volumes for Vector Search

#### Storage Considerations for Vector Workloads

When deploying Weaviate with vector search, selecting the right storage solution is crucial:

- **Persistent Volume Requirements**:
  - High I/O performance
  - Low latency access
  - Sufficient capacity for vector indexes
  - Support for `ReadWriteOnce` access mode

##### EFS Configuration with Vector Search

For AWS deployments, use EFS with careful configuration:

- Create an EFS file system optimized for vector data
- Create separate EFS access points for each Weaviate replica
- Ensure unique root directories to prevent data sharing
- Create EFS mount targets for each subnet

Example PersistentVolume for vector search:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: weaviate-vector-storage
spec:
  capacity:
    storage: 100Gi  # Adjust based on vector index size
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: 'efs-vector-sc'
  csi:
    driver: efs.csi.aws.com
    volumeHandle: <FileSystemId>::<AccessPointId>
```

#### Azure File CSI Considerations

:::caution Azure Storage Provisioner
The provisioner `file.csi.azure.com` is **not supported** and may cause file corruptions.
:::

Use the `disk.csi.azure.com` provisioner:

```yaml
storage:
  size: 100Gi  # Increased for vector index
  storageClassName: managed-premium  # High-performance tier
```

Verify available storage classes:
```bash
kubectl get storageclasses
```

#### Storage Performance Recommendations

- Prefer SSD or NVMe-based storage classes
- Choose storage with high IOPS
- Consider separate storage for:
  - Vector indexes
  - Object metadata
  - Write-ahead logs

List available storage classes:
```bash
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