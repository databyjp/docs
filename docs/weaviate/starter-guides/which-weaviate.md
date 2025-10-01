---
title: Weaviate configurations
sidebar_position: 1
image: og/docs/tutorials.jpg
# tags: ['getting started']
---

Weaviate can be configured and deployed in many different ways. Important configuration decisions include:

- The [deployment setup](/deploy/index.mdx)
- The [model integration](../model-providers/index.md) to enable

This page helps you to find the right combination for your project.

## Deploy Weaviate

Weaviate can be deployed in different ways depending on your use case:

**For testing & development:**
- [Embedded Weaviate](/deploy/installation-guides/embedded.md) *(experimental)*
- [Docker-Compose](/deploy/installation-guides/docker-installation.md)
- `brew install weaviate` on Mac
- Local Kubernetes clusters (e.g., `kind`)

**For production environments:**
- [Self-managed Kubernetes](/deploy/installation-guides/k8s-installation.md) *(only supported production method)*
- [Weaviate Cloud (WCD)](/deploy/installation-guides/weaviate-cloud.md) *(managed Kubernetes)*
- [Hybrid SaaS](https://weaviate.io/pricing) *(managed with configuration flexibility)*

:::warning Production Deployment
**Kubernetes is the only supported deployment method for production environments.** Docker-Compose and embedded Weaviate are suitable only for testing and development.
:::
## Vectorization options

When adding data objects to Weaviate, you have two choices:
- Specify the object vector directly.
- Use a Weaviate vectorizer module to generate the object vector.

If you are using a vectorizer module, your choices depend on your input medium/modality, as well as whether you would prefer a local or API-based vectorizer.

Generally speaking, an API-based vectorizer is more convenient to use, but it incurs additional costs. A local vectorizer can cost less, but may require specialized hardware (such as a GPU) to run at comparable speeds.

For text, [this open-source benchmark](https://huggingface.co/blog/mteb) provides a good overview of the performance of different vectorizers. Remember, domain-specific and real-world performance may vary.

## Use cases

Here are some recommendations for different use cases.

### Quick evaluation

If you are evaluating Weaviate, we recommend using one of these instance types to get started quickly:

- [Weaviate Cloud (WCD)](/cloud) sandbox
- [Embedded Weaviate](/deploy/installation-guides/embedded)

Use an inference-API based text vectorizer with your instance, for example, `text2vec-cohere`, `text2vec-huggingface`, `text2vec-openai`, or  `text2vec-google`.

The [Quickstart guide](/weaviate/quickstart) uses a WCD sandbox and an API based vectorizer to run the examples.

### Development
For development, we recommend using

- [Weaviate Cloud (WCD)](https://console.weaviate.cloud/) or [Docker Compose](/deploy/installation-guides/docker-installation.md).
- A vectorization strategy that matches your production vectorization strategy.

#### Docker-Compose vs. Weaviate Cloud (WCD)

Of the two, Docker-Compose is more flexible as it exposes all configuration options, and can be used in a local development environment. Additionally, it can use local vectorizer modules such as `text2vec-transformers` or `multi2vec-clip` for example.

On the other hand, WCD instances are easier to spin up, and takes away the need to manage the deployment yourself. However, WCD has some limitations in terms of available vectorizer modules and configuration options.

Note that Embedded Weaviate is currently not recommended for serious development use as it is at an experimental phase.

#### Vectorization strategy

For development, we recommend using a vectorizer module that closely matches your production needs.

As a first point, you must choose:
- Whether to vectorize data yourself and import it into Weaviate, or
- To use a Weaviate vectorizer module.

Then, we recommend choosing a vectorizer module that is as close as possible to your production needs. For example, if search quality is of paramount importance, we suggest using your preferred vectorizer module in development as well.

Keep in mind these important factors:
- **Cost**: API-based vectorization can be expensive, especially with large datasets
- **Memory footprint**: Vector lengths can vary by a factor of ~5, significantly impacting storage and memory requirements
- **Performance**: Local models require proper resource allocation but avoid API costs
- **Scalability**: Consider how your vectorization strategy will scale to production volumes
### Production

**For production deployments, Kubernetes is the only supported deployment method.** Consider one of these hosting models:

- [Self-managed Kubernetes](/deploy/installation-guides/k8s-installation.md)
- [Weaviate Cloud (WCD)](/cloud) *(managed Kubernetes)*
- [Hybrid SaaS](/cloud) *(managed with configuration flexibility)*

All of these options are scalable and run on Kubernetes infrastructure. Self-managed Kubernetes offers the most configuration flexibility, while WCD provides the easiest setup and maintenance. Hybrid SaaS offers a balance between control and convenience.

#### Key Production Considerations

**Memory Requirements:**
HNSW indexes must fit entirely in memory. Calculate memory needs using:
`Memory ≈ 2 × dimensions × vector_count × 4 bytes`

For example, 100M vectors with 512 dimensions requires approximately 400GiB of memory. Always allocate 20% additional memory for page cache and overhead.

**CPU Sizing:**
Estimate CPU requirements using:
`Minimum CPUs = (QPS × Average_Latency_Per_Query) + 6`

Typical latencies:
- Pure vector search: ~4ms (1M objects)
- Filtered search: ~10ms
- Hybrid search: ~20ms
- Latency approximately doubles as data size increases 10x

**High Availability:**
- Deploy minimum 3-node cluster
- Configure replication factor of 3
- Implement proper sharding strategy
- Use anti-affinity rules to distribute pods across nodes

**Storage Requirements:**
- Kubernetes cluster must support Persistent Volumes
- Use Storage Class with associated Provisioner
- **Avoid NFS** or NFS-like storage classes
- Prefer SAN-based storage (EBS, VMDK, etc.)

**Backup Strategy:**
- Configure appropriate backup module (`backup-s3`, `backup-gcs`, `backup-azure`)
- Implement incremental and regular backup schedules
- Test recovery procedures regularly
- Plan for disaster recovery scenarios
### Resource Planning

Proper resource planning is crucial for production Weaviate deployments:

**Memory Calculation Examples:**
- 1M vectors (384 dimensions): ~3GB
- 10M vectors (768 dimensions): ~60GB
- 100M vectors (1536 dimensions): ~1.2TB

**Storage Considerations:**
- Choose storage classes with high IOPS capability
- Ensure backup storage is properly configured
- Monitor disk usage for LSM compaction processes

**Network Planning:**
- REST API: Port 8080 (HTTP)
- gRPC: Port 50051 (Protocol Buffers)
- Gossip: Port 7102 (cluster communication)
- Data exchange: Port 7103 (internal)

**Monitoring Requirements:**
- Set up Prometheus ServiceMonitor
- Configure log aggregation (structured JSON logs recommended)
- Enable slow query logging for performance optimization
- Monitor memory usage patterns and GC behavior
## By Vectorizer & Reranker

Weaviate makes various vectorizer & reranker modules available for different media types, also called modalities.

Some model types such as Ollama or Transformers models are locally hosted, while others such as Cohere or OpenAI are API-based. This affects their availability in different Weaviate setups.

**Production Considerations:**
- **Local models**: Require proper CPU/GPU resource allocation and may need specialized hardware
- **API-based models**: Have cost implications at scale but are easier to deploy
- **Vector dimensionality**: Significantly impacts memory requirements (varies by 5x between models)
- **Throughput requirements**: Local models may require multiple replicas for high-throughput scenarios

We recommend reviewing from the available [model integrations](../model-providers/index.md) and their availability in different Weaviate setups. Then, choose the one that best fits your needs and resource constraints.
:::tip Key Production Factors
**Memory**: Requirements grow exponentially with vector dimensions and count. Plan for 2 × dimensions × vectors × 4 bytes minimum.

**Storage**: Type significantly impacts performance. Avoid NFS; prefer SAN-based storage with high IOPS.

**Backup**: Strategy must be planned from day one. Test recovery procedures regularly.

**Monitoring**: Essential for production operations. Set up metrics, logging, and alerting from the start.
:::

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>