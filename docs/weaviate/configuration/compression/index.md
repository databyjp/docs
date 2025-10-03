---
title: Compression
sidebar_position: 5
image: og/docs/configuration.jpg
# tags: ['configuration', 'compression', 'pq']
---

Uncompressed vectors can be large. Compressed vectors lose some information, but they use fewer resources and can be very cost effective.

## Vector quantization

To balance resource costs and system performance, consider one of these options:

- **[Rotational Quantization (RQ)](rq-compression.md)** (_recommended_)
- **[Product Quantization (PQ)](pq-compression.md)**
- **[Binary Quantization (BQ)](bq-compression.md)**
- **[Scalar Quantization (SQ)](sq-compression.md)**

## Rotational Quantization (RQ)

Rotational Quantization is a compression technique that reduces vector size while maintaining high recall. It comes in two variants:
- **8-bit RQ**: Provides 4x compression with 98-99% recall
- **1-bit RQ**: Offers up to 32x compression with improved performance over Binary Quantization

You can also [disable quantization](uncompressed.md) for a collection.

import CompressionByDefault from '/_includes/compression-by-default.mdx';

<CompressionByDefault/>

## Multi-vector encoding

Aside from quantization, Weaviate also offers encodings for multi-vector embeddings:

- **[MUVERA encoding](./multi-vectors.md)**