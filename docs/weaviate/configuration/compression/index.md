---
title: Compression
sidebar_position: 5
image: og/docs/configuration.jpg
# tags: ['configuration', 'compression', 'pq']
---

Uncompressed vectors can be large. Compressed vectors lose some information, but they use fewer resources and can be very cost effective.

## Vector quantization

To balance resource costs and system performance, consider these options (recommended order):

1. **[Rotational Quantization (RQ)](rq-compression.md)**
   - 8-bit RQ (recommended in v1.33)
   - 1-bit RQ (preview feature)
2. **[Scalar Quantization (SQ)](sq-compression.md)**
3. **[Product Quantization (PQ)](pq-compression.md)**
4. **[Binary Quantization (BQ)](bq-compression.md)**

:::info Preview Feature
1-bit Rotational Quantization (RQ) is available as a preview feature in v1.33. Not recommended for production use.
:::

You can also [disable quantization](uncompressed.md) for a collection.

import CompressionByDefault from '/_includes/compression-by-default.mdx';

<CompressionByDefault/>

## Multi-vector encoding

Aside from quantization, Weaviate also offers encodings for multi-vector embeddings:

- **[MUVERA encoding](./multi-vectors.md)**