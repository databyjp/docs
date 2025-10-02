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

Backup modules are a critical component of Weaviate's data protection strategy, providing robust mechanisms to safeguard your vector database against potential data loss. These modules enable:

- **Comprehensive data protection**: Capture complete database state, including objects, schemas, and vector embeddings
- **Disaster recovery**: Quickly restore data in case of logical failures, accidental deletions, or system errors
- **Business continuity**: Minimize downtime and potential data loss
- **Flexible storage options**: Support multiple backend storage systems

### Backup Types
- **Full Backup**: Captures entire database state
- **Incremental Backup**: (Planned future feature) Saves only changed data

### External Provider

External backup providers are the recommended solution for production environments, offering several key advantages:

- **High Availability**: Store backups separately from the Weaviate instance
- **Multi-Node Cluster Support**: Essential for distributed system architectures
- **Enhanced Data Durability**: Ensures backup accessibility even if a Weaviate node becomes unreachable

#### Supported External Backup Providers

| Provider | Use Case | Best Suited For |
|----------|----------|-----------------|
| **AWS S3** | Cloud-native storage | AWS-based infrastructures, scalable applications |
| **Google Cloud Storage (GCS)** | Google Cloud environments | Google Cloud Platform deployments, enterprise solutions |
| **Azure Blob Storage** | Microsoft Cloud storage | Azure-based systems, hybrid cloud architectures |

Multi-node Weaviate clusters *require* external providers due to the complexity of capturing and maintaining consistent backup states across distributed systems.
The supported external backup providers are:
- [S3](/deploy/configuration/backups.md#s3-aws-or-s3-compatible)
- [GCS](/deploy/configuration/backups.md#gcs-google-cloud-storage)
- [Azure](/deploy/configuration/backups.md#azure-storage)

Thanks to the extensibility of the module system, new providers can be readily added. If you are interested in an external provider other than the ones listed above, feel free to reach out via our [forum](https://forum.weaviate.io/), or open an issue on [GitHub](https://github.com/weaviate/weaviate).

### Internal Provider

Internal providers are **strictly intended for development and testing purposes**:

- **Limited Functionality**: Only supports filesystem-based backups
- **Not Recommended for Production**:
  - Lacks multi-node support
  - Reduces backup reliability
  - Does not provide enterprise-grade data protection

**Warning**: Internal providers should never be used in production environments due to significant limitations in data durability and recoverability.

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