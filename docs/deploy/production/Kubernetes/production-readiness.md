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
  - [ ]  Verify replication factor configuration in your schema definition
  - [ ]  Test failover scenarios using canary deployments to validate data availability
- [ ]  Are replicas deployed across multiple nodes for redundancy?
- [ ]  Is your control plane highly available?
- [ ]  Is your application fault-tolerant *without* your control plane?
- [ ]  Are there automatic node repair or self-healing mechanisms in place?
- [ ]  Have failover scenarios been tested to validate resilience?
  - [ ]  Conduct controlled failover tests with minimal service disruption
  - [ ]  Simulate node failures and verify data consistency and recovery
- [ ]  Are you utilizing Weaviate's backup capabilities for disaster recovery?
    - [ ]  How often are these mechanisms tested?
    - [ ]  Has the ability to recover from a node failure or database corruption been tested?
- [ ]  Have you thought about the retention period of backups?
    - [ ]  How do you clean up any out-of-date backups?
- [ ]  Are rolling updates performed to avoid downtime?
- [ ]  Are canary deployments implemented to safely test new releases?
    - [ ]  Configure gradual rollout strategies
    - [ ]  Implement traffic splitting for controlled testing
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
- [ ] Is the data within your cluster properly backed up, including the persistent storage?
- [ ] Is the sharding strategy aligned with the size and access patterns of the dataset?
    - [ ] Define sharding strategy based on:
        - Expected data volume
        - Query access patterns
        - Performance requirements
- [ ] Is `GOMEMLIMIT` properly configured for memory management?
    - [ ] Is `GOMEMLIMIT` set based on available system memory to prevent excessive garbage collection pauses?
    - [ ] Configure memory limits for different workload types:
        - Indexing workloads
        - Query-heavy workloads
        - Mixed workload scenarios
- [ ] Have you considered vector quantization techniques to reduce memory requirements?
    - [ ] Evaluate quantization impact on search recall and precision
    - [ ] Implement selective quantization strategies
    - [ ] Monitor memory usage before and after quantization
### Tenant State Management

- [ ] Are you implementing multi-tenancy?
  - [ ] Are there limits or quotas per tenant to avoid noisy neighbor issues?
- [ ] Is there a strategy for offloading inactive tenant data?

### Security

- [ ]  Are the components of your cluster communicating via SSL/TLS and trusted certificates?
- [ ]  Is the *"principle of least privilege"* being followed?
- [ ]  Are your container security defaults set properly?
- [ ]  Is access to your cluster strictly limited?
- [ ]  Has [RBAC](/weaviate/configuration/rbac/index.mdx) been implemented to restrict access?
- [ ]  Have network policies been implemented to limit pod-to-pod communication?
    - [ ]  Define granular network policy rules
    - [ ]  Restrict communication between namespaces
    - [ ]  Implement egress and ingress controls
- [ ]  Are gRPC service communications secured?
    - [ ]  Enable TLS for gRPC services
    - [ ]  Configure mutual TLS authentication
    - [ ]  Implement certificate rotation mechanisms
- [ ]  Are secrets secured with K8s Secrets or a vault solution?
- [ ]  Do you have a process for when secrets are exposed, when access is lost to a key or certificate, and when secrets need to be rotated?
    - [ ]  Establish secret rotation schedule
    - [ ]  Implement automated secret management

### Monitoring and Observability

- [ ]  Is logging implemented?
    - [ ]  Are the collected logs stored centrally?
    - [ ]  Configure log rotation and retention policies
- [ ]  Is metric collection enabled using Prometheus (or Alloy, DataDog, or another monitoring platform)?
    - [ ]  Key Prometheus metrics to monitor:
        - Vector index performance
        - Query latency
        - Async indexing queue metrics
        - Memory usage and garbage collection
        - Node resource utilization
- [ ]  Are health and performance metrics being visualized in Grafana?
    - [ ]  Create dashboards for:
        - Vector search performance
        - Hybrid search latency
        - Cluster health
        - Resource utilization
- [ ]  Are alerts configured for events?
    - [ ]  Set up alerting for critical performance thresholds
    - [ ]  Configure notifications for anomalous behavior
### Performance Optimization

- [ ]  Evaluate vector search performance with different index configurations
    - [ ]  Test varying HNSW parameters (ef, efConstruction)
    - [ ]  Compare search recall and precision
- [ ]  Test hybrid search latency under various load conditions
    - [ ]  Simulate mixed vector and keyword search scenarios
    - [ ]  Measure p95 and p99 latencies
- [ ]  Implement async indexing for improved throughput
    - [ ]  Configure async indexing settings
    - [ ]  Monitor indexing queue performance
- [ ]  Optimize filtering and vector search performance
    - [ ]  Use `USE_BLOCKMAX_WAND` for large-scale hybrid searches
    - [ ]  Implement efficient filtering strategies

Evaluate these key areas to build a highly available, resilient, and efficient deployment that will scale to meet your business needs. By ensuring that these self-assessment questions have been addressed, you can proactively identify potential risks and maximize the reliability of your deployment.

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>