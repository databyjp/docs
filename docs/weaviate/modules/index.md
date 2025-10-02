---
title: Reference - Modules
description: Learn about Weaviate modules to extend its functionality.
sidebar_position: 0
image: og/docs/modules/_title.jpg
# tags: ['modules']
---

This section describes Weaviate's individual modules, including their capabilities and how to use them.

:::tip Looking for vectorizer, generative AI, or reranker integration docs?
They have moved to our [model provider integrations](../model-providers/index.md) section, for a more focussed, user-centric look at these integrations.
:::

## General

Weaviate's modules are built into the codebase, and [enabled through environment variables](../configuration/modules.md) to provide additional functionalities.

### Module types

Weaviate modules can be divided into the following categories:

- [Vectorizers](#vectorizer-reranker-and-generative-ai-integrations): Convert data into vector embeddings for import and vector search.
- [Rerankers](#vectorizer-reranker-and-generative-ai-integrations): Improve search results by reordering initial search results.
- [Generative AI](#vectorizer-reranker-and-generative-ai-integrations): Integrate generative AI models for retrieval augmented generation (RAG).
- [Backup](#backup-modules): Facilitate backup and restore operations in Weaviate.
- [Offloading](#offloading-modules): Facilitate offloading of tenant data to external storage.
- [Others]: Modules that provide additional functionalities.

#### Vectorizer, reranker, and generative AI integrations

For these modules, see the [model provider integrations](../model-providers/index.md) documentation. These pages are organized by the model provider (e.g. Hugging Face, OpenAI) and then the model type (e.g. vectorizer, reranker, generative AI).

For example:

- [The OpenAI embedding integration page](../model-providers/openai/embeddings.md) shows how to use OpenAI's embedding models in Weaviate.

<img
    src={require('../model-providers/_includes/integration_openai_embedding.png').default}
    alt="Embedding integration illustration"
    style={{ maxWidth: "50%", display: "block", marginLeft: "auto", marginRight: "auto"}}
/>
<br/>

- [The Cohere reranker integration page](../model-providers/cohere/reranker.md) shows how to use Cohere's reranker models in Weaviate.

<img
    src={require('../model-providers/_includes/integration_cohere_reranker.png').default}
    alt="Reranker integration illustration"
    style={{ maxWidth: "50%", display: "block", marginLeft: "auto", marginRight: "auto"}}
/>
<br/>

- [The Anthropic generative AI integration page](../model-providers/anthropic/generative.md) shows how to use Anthropic's generative AI models in Weaviate.

<img
    src={require('../model-providers/_includes/integration_anthropic_rag.png').default}
    alt="Generative integration illustration"
    style={{ maxWidth: "50%", display: "block", marginLeft: "auto", marginRight: "auto"}}
/>
<br/>

### Module characteristics

- Naming convention:
  - Vectorizer (Retriever module): `<media>2vec-<name>-<optional>`, for example `text2vec-contextionary`, `img2vec-neural` or `text2vec-transformers`.
  - Other modules: `<functionality>-<name>-<optional>`, for example `qna-transformers`.
  - A module name must be url-safe, meaning it must not contain any characters which would require url-encoding.
  - A module name is not case-sensitive. `text2vec-bert` would be the same module as `text2vec-BERT`.
- Module information is accessible through the `v1/modules/<module-name>/<module-specific-endpoint>` RESTful endpoint.
- General module information (which modules are attached, version, etc.) is accessible through Weaviate's [`v1/meta` endpoint](/deploy/configuration/meta.md).
- Modules can add `additional` properties in the RESTful API and [`_additional` properties in the GraphQL API](../api/graphql/additional-properties.md).
- A module can add [filters](../api/graphql/filters.md) in GraphQL queries.
- Which vectorizer and other modules are applied to which data collection is configured in the [schema](../manage-collections/vector-config.mdx#specify-a-vectorizer).

## Backup Modules

Backup and restore operations in Weaviate are facilitated by the use of backup provider modules.

These are interchangeable storage backends which exist either internally or externally.

### External Provider

External backup providers are critical for robust data protection and disaster recovery in production environments. They offer several key advantages:

- **Data Durability**: By storing backup data outside the Weaviate instance, you ensure that your data remains safe even if the primary system fails.
- **High Availability**: Decoupling backups from the main instance means you can recover data regardless of the primary system's state.
- **Scalability**: Essential for multi-node Weaviate clusters, which require external storage to maintain backup integrity.

#### Supported External Providers

Weaviate currently supports the following external backup storage backends:

- **Amazon S3**: Provides scalable object storage with broad compatibility
  - Supports AWS S3 and S3-compatible storage services
  - Ideal for organizations using AWS infrastructure

- **Google Cloud Storage (GCS)**: Google's cloud storage solution
  - Offers seamless integration with Google Cloud Platform
  - Provides robust data durability and global accessibility

- **Azure Blob Storage**: Microsoft's object storage solution
  - Designed for enterprises using Azure cloud services
  - Supports various redundancy and access tiers

#### Multi-Node Cluster Requirements

For multi-node Weaviate clusters, external providers are not just recommended—they are **mandatory**. Internal or single-node backup methods are incompatible with distributed cluster architectures.

Thanks to the extensibility of the module system, new providers can be readily added. If you are interested in an external provider other than the ones listed above, feel free to reach out via our [forum](https://forum.weaviate.io/), or open an issue on [GitHub](https://github.com/weaviate/weaviate).
### Backup Module Characteristics

#### Storage Backend Compatibility
- Supports multiple storage types: object storage, cloud storage, and local filesystem
- Each provider implements a standardized backup interface for consistent operations

#### Configuration Requirements
- Backup modules are configured through environment variables
- Requires proper credentials and access permissions for external storage
- Configuration includes backup ID, storage backend, and optional metadata

#### Backup Compatibility
- **Single-Node**: Supports both internal (filesystem) and external providers
- **Multi-Node**: **Requires external provider** (S3, GCS, Azure)
- Backups are tied to specific node configurations and cannot be restored across different cluster setups

#### Backup Limitations
- Backups include only active tenants
- LSM Compactions are temporarily paused during backup
- Incomplete backups cannot be resumed and must be restarted

### Internal Provider

Internal backup providers store and retrieve backed-up Weaviate data within the same Weaviate instance. They are designed strictly for development, testing, and experimental purposes.

#### Filesystem Provider
- As of Weaviate `v1.16`, the filesystem provider is the only supported internal backup method
- Stores backup data on the local disk of the Weaviate instance

#### Key Limitations
- **Not Recommended for Production**: Lacks the reliability and durability of external providers
- **Single-Node Only**: Incompatible with multi-node cluster configurations
- **High Risk**: Backup data is stored on the same system, increasing potential for data loss

**Best Practice**: Always use external providers for production environments to ensure data safety and recoverability.
### Upcoming Features

#### Incremental Backups
- **Coming Soon**: Incremental backup capability
- Will allow backing up only changed data since the last backup
- Reduces backup time and storage requirements
- Stay tuned to Weaviate release notes for availability

## Offloading Modules

:::info Added in `v1.26`
:::

Offloading modules facilitate the offloading of tenant data to external storage. This is useful for managing resources and costs.

See [how to configure: offloading](/deploy/configuration/tenant-offloading.md) for more information on how to configure and use offloading modules.

## Other modules

In addition to the above, there are other modules such as:

- [qna-transformers](./qna-transformers.md): Question-answering (answer extraction) capability using transformers models.
- [qna-openai](./qna-openai.md): Question-answering (answer extraction) capability using OpenAI models.
- [ner-transformers](./ner-transformers.md): Named entity recognition capability using transformers models.
- [text-spellcheck](./ner-transformers.md): Spell checking capability for GraphQL queries.
- [sum-transformers](./sum-transformers.md): Summarize text using transformer models.
- [usage-modules](./usage-modules.md): Collect and upload usage analytics to GCS or S3 for the purposes of billing.

## Related pages

- [Configuration: Modules](../configuration/modules.md)
- [Concepts: Modules](../concepts/modules.md)

## Other third party integrations

import IntegrationLinkBack from '/_includes/integrations/link-back.mdx';

<IntegrationLinkBack/>

## Questions and feedback

import DocsFeedback from '/_includes/docs-feedback.mdx';

<DocsFeedback/>