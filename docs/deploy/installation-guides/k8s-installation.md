---
title: Kubernetes
sidebar_position: 3
image: og/docs/installation.jpg
# tags: ['installation', 'Kubernetes']
---

:::tip End-to-end guide
For a tutorial on how to use [minikube](https://minikube.sigs.k8s.io/docs/) to deploy Weaviate on Kubernetes, see the Weaviate Academy course, [Weaviate on Kubernetes](../../academy/deployment/k8s/index.md).
:::
## Prerequisites

### Kubernetes Cluster Requirements
* **Kubernetes Version**: A recent Kubernetes Cluster running version 1.23 or higher.
  - For development environments, consider using:
    - Kubernetes built into Docker Desktop
    - `kind` (Kubernetes in Docker)
    - Minikube

### Storage Requirements
* **Persistent Volume Provisioning**:
  - Cluster must be able to create `PersistentVolumes` using Kubernetes `PersistentVolumeClaims`
  - **Recommended Storage Types**:
    - SAN-based storage (preferred)
      - Amazon EBS
      - VMware VMDK
      - Cloud provider block storage volumes
  - **Avoid**:
    - NFS or NFS-like storage classes
    - Shared file systems that don't support `ReadWriteOnce` access mode

* **Storage Class Considerations**:
  - Require a Storage Class with an associated Provisioner
  - Must support single-node read-write access
  - Recommended: Dynamic volume provisioning

### Additional Tools
* Helm version v3 or higher (current chart version: `||site.helm_version||`)
* `kubectl` configured to access the Kubernetes cluster

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

To customize the Helm chart for your environment, edit the [`values.yaml`](https://github.com/weaviate/weaviate-helm/blob/master/weaviate/values.yaml) file.

#### Performance and Resource Optimization

Configure resources and performance-related settings:

```yaml
resources:
  requests:
    cpu: '500m'     # Minimum CPU resources
    memory: '4Gi'   # Minimum memory allocation
  limits:
    cpu: '2'        # Maximum CPU usage
    memory: '8Gi'   # Maximum memory allocation

env:
  - ASYNC_INDEXING: 'true'     # Enable async indexing for improved performance
  - GOMEMLIMIT: '7168MB'       # Go runtime memory limit (80% of total memory)
  - GOGC: '100'                # Garbage collection tuning
  - GO_PROFILING_DISABLE: 'false'  # Enable Go profiling for performance analysis
```

#### Replication and High Availability

```yaml
replication:
  enabled: true
  count: 3  # Recommended minimum for high availability
```

#### gRPC Service Configuration

```yaml
grpcService:
  enabled: true
  name: weaviate-grpc
  ports:
    - name: grpc
      protocol: TCP
      port: 50051
  type: LoadBalancer  # Allows external access to gRPC API
```

#### Performance Tuning Parameters

```yaml
env:
  - USE_BLOCKMAX_WAND: 'true'     # Recommended for large hybrid searches
  - QUERY_DEFAULTS_LIMIT: '100'   # Adjust default query result limit
  - QUERY_MAXIMUM_RESULTS: '10000'  # Maximum allowed query results
```

:::note Performance Considerations
- Adjust `resources` based on your expected workload
- Monitor and tune `GOMEMLIMIT` and `GOGC` for optimal memory management
- Enable `ASYNC_INDEXING` for write-heavy workloads
:::

#### Authentication and Authorization

:::tip Security Best Practices
Weaviate Helm charts automatically generate secure, random credentials for inter-node communication.
:::

##### API Key Authentication

```yaml
authentication:
  apikey:
    enabled: true
    allowed_keys:
      - readonly-key     # Read-only access
      - admin-key        # Full access key
    users:
      - readonly@example.com
      - admin@example.com
```

##### Anonymous Access

```yaml
authentication:
  anonymous_access:
    enabled: false  # Recommended to disable in production
```

##### OIDC Configuration

```yaml
authentication:
  oidc:
    enabled: true
    issuer: https://your-identity-provider.com
    username_claim: email
    groups_claim: groups
    client_id: weaviate-client
```

##### Role-Based Access Control (RBAC)

```yaml
authorization:
  admin_list:
    enabled: true
    users:
      - admin@example.com
      - cluster-admin@example.com
    readonly_users:
      - readonly@example.com
```

:::note Authentication Workflow
1. API keys are mapped to specific user identities
2. OIDC provides centralized identity management
3. RBAC controls user permissions and access levels
:::

##### Inter-node Communication
- Credentials are automatically generated and rotated
- Secure communication between Weaviate nodes is ensured

###### Additional Security Resources
- [Authentication Configuration](../configuration/authentication.md)
- [Authorization Configuration](../configuration/authorization.md)

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

### Networking Issues

- If you encounter `No private IP address found, and explicit IP not provided`, ensure your pod subnet is in a valid IP address range:

    ```
    10.0.0.0/8      # Common private network range
    100.64.0.0/10   # Carrier-grade NAT range
    172.16.0.0/12   # Another private network range
    192.168.0.0/16  # Local network range
    198.19.0.0/16   # Another private range
    ```

### Common Deployment Issues

- **Pod Scheduling Failures**:
  - Check node resources and capacity
  - Verify storage class and persistent volume configuration
  - Ensure sufficient CPU and memory are available

- **Authentication Problems**:
  - Verify OIDC configuration
  - Check API key and user mappings
  - Validate identity provider settings

- **Performance Bottlenecks**:
  - Monitor resource utilization
  - Adjust `GOMEMLIMIT` and `GOGC` settings
  - Enable Go profiling for detailed performance analysis

### Set `CLUSTER_HOSTNAME` if it may change over time

In some systems, the cluster hostname may change over time. This is known to create issues with a single-node Weaviate deployment. To avoid this, set the `CLUSTER_HOSTNAME` environment variable in the `values.yaml` file to the cluster hostname.

```yaml
env:
  - CLUSTER_HOSTNAME: "node-1"
```

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>