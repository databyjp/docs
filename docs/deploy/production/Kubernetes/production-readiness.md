---
sidebar_label: Production Readiness Self-Assessment
---

# Kubernetes Production Readiness Self-Assessment

Think you’re ready for production? Ensuring that your Weaviate cluster is production-ready requires careful planning, configuration, and ongoing maintenance. Ensuring that you have a stable, reliable deployment requires you to think of your *ending* at the *beginning.* This guide provides you with introspective questions to assess readiness and identify any potential gaps before moving your workloads into production.

:::tip
If you *do* identify gaps within your deployment, be sure to reach out to your SE (solutions engineer) who can help steer you on the path to production success!
:::


### High Availability and Resilience

- [ ]  Are your clusters deployed across multiple availability zones (AZs) or regions to prevent downtime?
- [ ]  Are you running Weaviate in a highly available setup with a 3-node minimum?
- [ ]  Have you configured your schema to use a replication factor of 3 to ensure copies of data are available during node outage?
- [ ]  Are replicas deployed across multiple nodes for redundancy?
- [ ]  Is your control plane highly available?
- [ ]  Is your application fault-tolerant *without* your control plane?
- [ ]  Are there automatic node repair or self-healing mechanisms in place?
- [ ]  Have failover scenarios been tested to validate resilience?
- [ ]  Are you utilizing Weaviate's backup capabilities for disaster recovery?
    - [ ]  Are production-appropriate backup modules configured?
      - [ ] `backup-s3`, `backup-gcs`, or `backup-azure` for production environments
      - [ ] `backup-filesystem` only for development & testing environments
    - [ ]  Is incremental backup scheduling implemented (e.g., 7 daily, 4 weekly, 6 monthly)?
    - [ ]  Are disk snapshots configured as an alternative backup method?
      - [ ] LVM or VMDK snapshots
      - [ ] EBS snapshots
    - [ ]  How often are these mechanisms tested?
    - [ ]  Has the ability to recover from different failure scenarios been tested?
      - [ ] Recovery from node failure
      - [ ] Recovery from database corruption
      - [ ] Recovery from complete cluster loss
- [ ]  Have you thought about the retention period of backups?
  - [ ]  How do you clean up any out-of-date backups?
  - [ ]  Is backup storage properly configured and secured?
- [ ]  Are rolling updates performed to avoid downtime?
- [ ]  Are canary deployments implemented to safely test new releases?
- [ ]  Do you have development or test environments to safely test changes?

### Data Ingestion and Query Performance

- [ ] Is there a strategy for handling heavy ingestion loads?
- [ ] Has the percentage of resources for indexing vs querying applications been specified?
- [ ] Is there a defined strategy for data deduplication and cleanup before ingestion?
- [ ] How frequently is data added, updated, or deleted?
  - [ ] Is data updated in place or mostly append-only
  - [ ] How often do deletion operations trigger garbage collection?
- [ ] Have you implemented a scheduling strategy for large ingestion jobs?
- [ ] Have you tested query performance under load?
  - [ ] Is query performance monitored using Prometheus or Grafana?
- [ ] Have replica shards been deployed for load balancing and failover support?


### Resource Management

- [ ]  Have you considered your data's consumption pattern(s)?
    - [ ]  Has your memory allocation been right-sized to match workload demand?
    - [ ]  Has your storage/compute allocation also been right-sized to match workload demand?
    - [ ]  Is there a process to delete old or unused objects?
- [ ] Have multiple replicas been configured to balance read-heavy workloads?
- [ ] Has the proper storage class been selected for your needs?
    - [ ] Does your storage class support volume expansion so that you can support growth over time?
    - [ ] Have you avoided NFS or NFS-like storage classes?
    - [ ] Is SAN-based storage (EBS, VMDK, etc.) configured?
- [ ] Is the data within your cluster properly backed up, including the persistent storage?
- [ ] Is the sharding strategy aligned with the size and access patterns of the dataset?
- [ ] Is `GOMEMLIMIT` properly configured for memory management?
  - [ ] Is `GOMEMLIMIT` set with 10-20% headroom for Go runtime and other processes?
- [ ] Are `GOMAXPROCS` settings appropriate for your CPU allocation?
- [ ] Is `LIMIT_RESOURCES` enabled if you prefer automatic resource management (sets memory to 80% and uses all but one CPU core)?
- [ ] Have you selected the appropriate `PERSISTENCE_LSM_ACCESS_STRATEGY`?
  - [ ] Is `mmap` used for general cases or `pread` for heavy memory load situations?
- [ ] Is `GOGC` configured appropriately for your garbage collection patterns?
- [ ] Have you calculated memory requirements using the HNSW formula: `2 * numDimensions * numVectors * 4B`?
- [ ] Have you calculated CPU requirements using: `QPS * AverageCPULatencyPerQuery + 6`?
- [ ] Have you considered vector quantization techniques to reduce memory requirements?
- [ ] Do you have a strategy for handling LSM compaction and segment management?
### Performance Optimization

- [ ] Is `ASYNC_INDEXING` enabled for large-scale deployments?
- [ ] Is `USE_BLOCKMAX_WAND` enabled for large hybrid/keyword searches?
- [ ] Is ACORN HNSW index configured for improved vector filtering performance?
- [ ] Is `GO_PROFILING_DISABLE` configured appropriately for your monitoring needs?
- [ ] Are query performance settings optimized?
  - [ ] Is `QUERY_DEFAULTS_LIMIT` set appropriately for your use case?
  - [ ] Is `QUERY_MAXIMUM_RESULTS` configured based on your query patterns?
- [ ] Is slow query logging enabled for performance monitoring?
  - [ ] Is `QUERY_SLOW_LOG_ENABLED` configured for troubleshooting?
- [ ] Have you tested query performance under realistic load conditions?
  - [ ] Pure vector search performance (typically 3-4ms for 1M objects)
  - [ ] Filtered search performance (typically ~10ms)
  - [ ] Hybrid search performance (typically ~20ms, doubles with 10x vector growth)

### Memory Optimization

- [ ] Have you implemented vector compression/quantization to reduce memory usage?
- [ ] Is dimensionality reduction (PCA) considered where appropriate?
- [ ] Are HNSW parameters optimized for your use case?
  - [ ] Is `maxConnections` reduced if memory is constrained?
  - [ ] Are `ef` and `efConstruction` increased to maintain recall/precision when reducing connections?
- [ ] Have you estimated memory requirements accurately?
  - [ ] Does your memory allocation include 20% extra for page cache and additional requirements?
### Tenant State Management

- [ ] Are you implementing multi-tenancy?
  - [ ] Are there limits or quotas per tenant to avoid noisy neighbor issues?
- [ ] Is there a strategy for offloading inactive tenant data?

### Schema Management

- [ ] Is `AUTOSCHEMA_ENABLED` disabled for production environments?
- [ ] Are schema definitions properly versioned and managed?
- [ ] Is there a process for schema migrations and updates?
- [ ] Have you planned your sharding strategy for collections?
  - [ ] Is `virtual_per_physical` configured appropriately?
  - [ ] Is `desired_count` set based on your data distribution needs?
### Security and Network

- [ ]  Are the components of your cluster communicating via SSL/TLS and trusted certificates?
- [ ]  Is the *"principle of least privilege"* being followed?
- [ ]  Are your container security defaults set properly?
- [ ]  Is access to your cluster strictly limited?
- [ ]  Has [RBAC](/weaviate/configuration/rbac/index.mdx) been implemented to restrict access?
- [ ]  Have network policies been implemented to limit pod-to-pod communication?
- [ ]  Are secrets secured with K8s Secrets or a vault solution?
- [ ]  Do you have a process for when secrets are exposed, when access is lost to a key or certificate, and when secrets need to be rotated?
- [ ]  Are all required ports properly configured and secured?
  - [ ] Port 8080 for REST API (HTTP)
  - [ ] Port 50051 for gRPC (Protocol Buffer)
  - [ ] Port 7102 for Gossip protocol
  - [ ] Port 7103 for internal data exchange
- [ ]  Is network communication between nodes properly secured?

### Monitoring and Observability

- [ ]  Is logging implemented and properly configured?
    - [ ]  Are the collected logs stored centrally?
    - [ ]  Is `LOG_LEVEL` set appropriately (default: info)?
    - [ ]  Is `LOG_FORMAT` configured for your logging infrastructure (default: json)?
- [ ]  Is metric collection enabled using Prometheus (or Alloy, DataDog, or another monitoring platform)?
- [ ]  Are health and performance metrics being visualized in Grafana?
- [ ]  Are alerts configured for critical events?
  - [ ] Memory usage approaching limits
  - [ ] Query performance degradation
  - [ ] Backup failures
  - [ ] Node failures or connectivity issues
- [ ]  Is query performance actively monitored?
  - [ ] Are slow queries being logged and analyzed?
  - [ ] Are query patterns and load trends being tracked?

Evaluate these key areas to build a highly available, resilient, and efficient deployment that will scale to meet your business needs. By ensuring that these self-assessment questions have been addressed, you can proactively identify potential risks and maximize the reliability of your deployment.

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>