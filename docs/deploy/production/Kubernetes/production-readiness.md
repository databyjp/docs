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
    - [ ]  Configured incremental backup strategy
    - [ ]  Defined backup retention policies (e.g., 7 daily, 4 weekly, 6 monthly)
    - [ ]  Tested backup and restore procedures
    - [ ]  Validated disaster recovery process
    - [ ]  Implemented offsite or multi-region backup storage
- [ ]  How often are these mechanisms tested?
    - [ ]  Periodic disaster recovery drills
    - [ ]  Simulated node failure and data recovery scenarios
- [ ]  Have you thought about the retention period of backups?
  - [ ]  Automated backup cleanup process
  - [ ]  Compliance with data retention requirements
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
- [ ] Have vector quantization techniques been evaluated to reduce memory and improve performance?
- [ ] Is there a strategy for managing large vector indexes?
  - [ ] Estimated vector index memory requirements
  - [ ] Monitoring vector search latency and recall
  - [ ] Configured with appropriate HNSW index parameters (maxConnections, ef, efConstruction)


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
- [ ] Vector Index Memory Management
    - [ ] `GOMEMLIMIT` properly configured for memory management
    - [ ] Estimated total memory requirements for vector indexes
    - [ ] Considered vector compression or quantization techniques
- [ ] Garbage Collection Strategies
    - [ ] Configured `GOGC` to optimize memory usage
    - [ ] Monitored garbage collection pauses
    - [ ] Implemented strategies to minimize GC impact on performance
  - [ ] Are there limits or quotas per tenant to avoid noisy neighbor issues?
  - [ ] Tenant-specific vector index management
- [ ] Is there a strategy for offloading inactive tenant data?
- [ ] Vector Search Optimization Checklist
  - [ ] Estimated vector index memory requirements
  - [ ] HNSW index configured with appropriate maxConnections
  - [ ] Considered vector quantization techniques
  - [ ] Monitoring vector search latency and recall
  - [ ] Implemented tenant-specific vector search performance monitoring
### Security

- [ ]  Are the components of your cluster communicating via SSL/TLS and trusted certificates?
- [ ]  Is the *"principle of least privilege"* being followed?
- [ ]  Are your container security defaults set properly?
- [ ]  Is access to your cluster strictly limited?
- [ ]  RBAC Configuration
    - [ ]  Implemented granular role-based access control
    - [ ]  Defined least-privilege roles for different user types
    - [ ]  Regularly audited and updated RBAC policies
- [ ]  Network Policy Recommendations
    - [ ]  Implemented strict network policies to limit pod-to-pod communication
    - [ ]  Segmented network access based on service requirements
    - [ ]  Configured ingress and egress rules
- [ ]  Secret Management
    - [ ]  Secrets secured with K8s Secrets or a vault solution
    - [ ]  Implemented secret rotation strategy
    - [ ]  Defined process for secret exposure or loss
    - [ ]  Regular key and certificate rotation
### Monitoring and Observability

- [ ]  Is logging implemented?
    - [ ]  Are the collected logs stored centrally?
- [ ]  Is metric collection enabled using Prometheus (or Alloy, DataDog, or another monitoring platform)?
- [ ]  Are health and performance metrics being visualized in Grafana?
- [ ]  Are alerts configured for events?

Evaluate these key areas to build a highly available, resilient, and efficient deployment that will scale to meet your business needs. By ensuring that these self-assessment questions have been addressed, you can proactively identify potential risks and maximize the reliability of your deployment.

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>