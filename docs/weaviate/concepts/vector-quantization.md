---
title: Compression (Vector Quantization)
sidebar_position: 19
description: "Vector compression techniques reducing memory footprint and costs while improving search speed performance."
image: og/docs/concepts.jpg
# tags: ['vector compression', 'quantization']
---

**Vector quantization** reduces the memory footprint of the [vector index](./indexing/vector-index.md) by compressing the vector embeddings, and thus reduces deployment costs and improves the speed of the vector similarity search process.

Weaviate currently offers four vector quantization techniques:

- [Binary quantization (BQ)](#binary-quantization)
- [Product quantization (PQ)](#product-quantization)
- [Scalar quantization (SQ)](#scalar-quantization)
- [Rotational quantization (RQ)](#rotational-quantization)

import CompressionByDefault from '/\_includes/compression-by-default.mdx';

<CompressionByDefault/>

## What is quantization?

In general, quantization techniques reduce the memory footprint by representing numbers with lower precision numbers, like rounding a number to the nearest integer. In neural networks, quantization reduces the values of the weights or activations of the model stored as a 32-bit floating-point number (4 bytes) to a lower precision number, such as an 8-bit integer (1 byte).

### What is vector quantization?

Vector quantization is a technique that reduces the memory footprint of vector embeddings. Vector embeddings have been typically represented as 32-bit floating-point numbers. Vector quantization techniques reduce the size of the vector embeddings by representing them as smaller numbers, such as 8-bit integers or binary numbers. Some quantization techniques also reduce the number of dimensions in the vector embeddings.

## Product quantization

[Product quantization](https://ieeexplore.ieee.org/document/5432202) is a multi-step quantization technique that is available for use with `hnsw` indexes in Weaviate.

PQ reduces the size of each vector embedding in two steps. First, it reduces the number of vector dimensions to a smaller number of "segments", and then each segment is quantized to a smaller number of bits from the original number of bits (typically a 32-bit float).

import PQTradeoffs from '/\_includes/configuration/pq-compression/tradeoffs.mdx' ;

<PQTradeoffs />

In PQ, the original vector embedding is represented as a product of smaller vectors that are called 'segments' or 'subspaces.' Then, each segment is quantized independently to create a compressed vector embedding.

![PQ illustrated](./img/pq-illustrated.png "PQ illustrated")

After the segments are created, there is a training step to calculate `centroids` for each segment. By default, Weaviate clusters each segment into 256 centroids. The centroids make up a codebook that Weaviate uses in later steps to compress the vector embeddings.

Once the codebook is ready, Weaviate uses the id of the closest centroid to compress each vector segment. The new vector embedding reduces memory consumption significantly. Imagine a collection where each vector embedding has 768 four byte elements. Before PQ compression, each vector embeddingrequires `768 x 4 = 3072` bytes of storage. After PQ compression, each vector requires `128 x 1 = 128` bytes of storage. The original representation is almost 24 times as large as the PQ compressed version. (It is not exactly 24x because there is a small amount of overhead for the codebook.)

To enable PQ compression, see [Enable PQ compression](/weaviate/configuration/compression/pq-compression#enable-pq-compression)

### Segments

The PQ `segments` controls the tradeoff between memory and recall. A larger `segments` parameter means higher memory usage and recall. An important thing to note is that the segments must divide evenly the original vector dimension.

Below is a list segment values for common vectorizer modules:

| Module      | Model                                   | Dimensions | Segments               |
| ----------- | --------------------------------------- | ---------- | ---------------------- |
| openai      | text-embedding-ada-002                  | 1536       | 512, 384, 256, 192, 96 |
| cohere      | multilingual-22-12                      | 768        | 384, 256, 192, 96      |
| huggingface | sentence-transformers/all-MiniLM-L12-v2 | 384        | 192, 128, 96           |

### PQ compression process

PQ has a training stage where it creates a codebook. We recommend using 10,000 to 100,000 records per shard to create the codebook. The training step can be triggered manually or automatically. See [Configuration: Product quantization](../configuration/compression/pq-compression.md) for more details.

When the training step is triggered, a background job converts the index to the compressed index. While the conversion is running, the index is read-only. Shard status returns to `READY` when the conversion finishes.

Weaviate uses a maximum of `trainingLimit` objects (per shard) for training, even if there are more objects available.

After the PQ conversion completes, query and write to the index as normal. Distances may be slightly different due to the effects of quantization.

:::info Which objects are used for training?

- (`v1.27` and later) If the collection has more objects than the training limit, Weaviate randomly selects objects from the collection to train the codebook.
- (`v1.26` and earlier) Weaviate uses the first `trainingLimit` objects in the collection to train the codebook.
- If the collection has fewer objects than the training limit, Weaviate uses all objects in the collection to train the codebook.

:::

### Encoders

In the configuration above you can see that you can set the `encoder` object to specify how the codebook centroids are generated. Weaviate's PQ supports using two different encoders. The default is `kmeans` which maps to the traditional approach used for creating centroid.

Alternatively, there is also the `tile` encoder. This encoder is currently experimental but does have faster import times and better recall on datasets like SIFT and GIST. The `tile` encoder has an additional `distribution` parameter that controls what distribution to use when generating centroids. You can configure the encoder by setting `type` to `tile` or `kmeans` the encoder creates the codebook for product quantization. For configuration details, see [Configuration: Vector index](../config-refs/indexing/vector-index.mdx).

### Distance calculation

With product quantization, distances are then calculated asymmetrically with a query vector with the goal being to keep all the original information in the query vector when calculating distances.

:::tip
Learn more about [how to configure product quantization in Weaviate](../configuration/compression/pq-compression.md).<br/><br/>
You might be also interested in our blog post [How to Reduce Memory Requirements by up to 90%+ using Product Quantization](https://weaviate.io/blog/pq-rescoring).
:::

## Binary quantization

**Binary quantization (BQ)** is a quantization technique that converts each vector embedding to a binary representation. The binary representation is much smaller than the original vector embedding. Usually each vector dimension requires 32 bits, but the binary representation only requires 1 bit, representing a 32x reduction in storage requirements. This works to speed up vector search by reducing the amount of data that needs to be read from disk, and simplifying the distance calculation.

The tradeoff is that BQ is lossy. The binary representation by nature omits a significant amount of information, and as a result the distance calculation is not as accurate as the original vector embedding.

Some vectorizers work better with BQ than others. Anecdotally, we have seen encouraging recall with Cohere's V3 models (e.g. `embed-multilingual-v3.0` or `embed-english-v3.0`), and OpenAI's `ada-002` model with BQ enabled. We advise you to test BQ with your own data and preferred vectorizer to determine if it is suitable for your use case.

Note that when BQ is enabled, a vector cache can be used to improve query performance. The vector cache is used to speed up queries by reducing the number of disk reads for the quantized vector embeddings. Note that it must be balanced with memory usage considerations, with each vector taking up `n_dimensions` bits.

## Scalar quantization

**Scalar quantization (SQ)** The dimensions in a vector embedding are usually represented as 32 bit floats. SQ transforms the float representation to an 8 bit integer. This is a 4x reduction in size.

SQ compression, like BQ, is a lossy compression technique. However, SQ has a much greater range. The SQ algorithm analyzes your data and distributes the dimension values into 256 buckets (8 bits).

SQ compressed vectors are more accurate than BQ compressed vectors. They are also significantly smaller than uncompressed vectors.

The bucket boundaries are derived by determining the minimum and maximum values in a training set, and uniformly distributing the values between the minimum and maximum into 256 buckets. The 8 bit integer is then used to represent the bucket number.

The size of the training set is configurable. The default is 100,000 objects per shard.

When SQ is enabled, Weaviate boosts recall by over-fetching compressed results. After Weaviate retrieves the compressed results, it compares the original, uncompressed vectors that correspond to the compressed result against the query. The second search is very fast because it only searches a small number of vectors rather than the whole database.

## Rotational quantization

:::info Added in `v1.32`

**8-bit Rotational quantization (RQ)** was added in **`v1.32`**.

:::

:::caution Preview

**1-bit Rotational quantization (RQ)** was added in **`v1.33`** as a **preview**.<br/>

This means that the feature is still under development and may change in future releases, including potential breaking changes.
**We do not recommend using this feature in production environments at this time.**

:::

**Rotational quantization (RQ)** is a quantization technique that provides significant compression while maintaining high recall in internal testing. Unlike SQ, RQ requires no training phase and can be enabled immediately at index creation. RQ is available in two variants: **8-bit RQ** and **1-bit RQ**.

### 8-bit RQ

8-bit RQ provides 4x compression while maintaining 98-99% recall in internal testing. The method works as follows:

1. **Fast pseudorandom rotation**: The input vector is transformed using a fast rotation based on the Walsh Hadamard Transform. This rotation takes approximately 7-10 microseconds for a 1536-dimensional vector. The output dimension is rounded up to the nearest multiple of 64.

2. **Scalar quantization**: Each entry of the rotated vector is quantized to an 8-bit integer. The minimum and maximum values of each individual rotated vector define the quantization interval.

### 1-bit RQ

1-bit RQ is an asymmetric quantization method that provides close to 32x compression as dimensionality increases. **1-bit RQ serves as a more robust and accurate alternative to BQ** with the following characteristics:

- Approximately 10% decrease in throughput compared to BQ in internal testing
- More performant than PQ in terms of encoding time and distance calculations
- Slightly lower recall than well-tuned PQ

The method works with a unique asymmetric approach:

1. **Fast pseudorandom rotation**: Similar to 8-bit RQ, with output dimension always padded to at least 256 bits to improve performance on low-dimensional data.

2. **Asymmetric quantization**:
   - **Data vectors**: Quantized using 1 bit per dimension by storing only the sign of each entry
   - **Query vectors**: Scalar quantized using 5 bits per dimension during search

This asymmetric approach improves recall by:
- Using more precision for query vectors during distance calculation
- Essentially matching BQ recall on datasets like OpenAI embeddings
- Performing well on datasets where BQ typically struggles (such as SIFT)

### Performance and Compression Characteristics

Key performance insights include:
- High native recall of 98-99% with RQ
- Potential to disable rescoring for maximum query performance
- Compression rates approaching 4x (8-bit) and 32x (1-bit) as dimensionality increases

:::caution Preview Status

**1-bit Rotational quantization (RQ)** is currently in **preview** status:
- Feature may change in future releases
- Potential for breaking changes
- **Not recommended for production environments**

:::

Learn more about how to [configure rotational quantization](../configuration/compression/rq-compression.md) in Weaviate or dive deer into the [implementation details and theoretical background](https://weaviate.io/blog/8-bit-rotational-quantization).

:::
## Over-fetching / Re-scoring

Weaviate employs an over-fetching and re-scoring strategy for SQ, RQ, and BQ to compensate for the potential loss in accuracy during compressed vector distance calculations.

### How Over-fetching Works

1. Retrieve compressed objects beyond the query limit
2. Fetch original, uncompressed vector embeddings
3. Recalculate query distance scores using uncompressed vectors

**Example Scenario:**
- Query limit: 10 results
- Rescore limit: 200 objects
- Weaviate fetches 200 objects and returns top 10 after rescoring

### RQ Optimization

With RQ's high native recall (98-99%), you can often:
- Disable rescoring by setting `rescoreLimit` to 0
- Achieve maximum query performance
- Minimize impact on search quality

## Vector Compression Use Cases

### Use Case Recommendations

#### Multi-tenant Data (<1M vectors)
- **Recommended Compression:** Binary Quantization (BQ)
- Advantages:
  - Drastically reduced memory usage
  - Minimal impact on recall due to overfetching
  - No training required

#### Medium Datasets (<100M vectors)
- **Recommended Compression:** Scalar Quantization (SQ)
- Advantages:
  - Good balance between compression and accuracy
  - Predictable performance
  - Suitable for single-tenant scenarios

#### Large Datasets (>100M vectors)
- **Recommended Compression:** Product Quantization (PQ)
- Advantages:
  - Significant memory and cost reduction
  - Maximum flexibility
  - Advanced compression for complex vector spaces

### Performance Comparison

| Index Type     | Indexing Time | Memory Usage | Compression Ratio | Recall Impact |
|---------------|--------------|--------------|------------------|--------------|
| HNSW          | 8m42s        | 6437.05MB    | No compression   | Baseline     |
| HNSW + PQ     | 21m25s       | 930.36MB     | ~88%             | Minor        |
| HNSW + BQ     | 3m43s        | 711.38MB     | ~97%             | Moderate     |
| FLAT + BQ     | 54s          | 260.16MB     | ~97%             | Moderate     |

## Vector Compression Configuration Examples

### Binary Quantization
```python
vector_index_config=wc.Configure.VectorIndex.hnsw(
    quantizer=wc.Configure.VectorIndex.Quantizer.bq(
        cache=True,
        rescore_limit=-1
    )
)
```

### Scalar Quantization
```python
vector_index_config=wc.Configure.VectorIndex.hnsw(
    quantizer=wc.Configure.VectorIndex.Quantizer.sq(
        rescore_limit=-1,
        training_limit=100_000,
        cache=True
    )
)
```

### Product Quantization
```python
vector_index_config=wc.Configure.VectorIndex.hnsw(
    quantizer=wc.Configure.VectorIndex.Quantizer.pq(
        segments=128,
        encoder_type='kmeans'
    )
)
```

## Further Resources

:::info Related Pages
- [Configuration: Compression Techniques](/configuration/compression/)
- [Weaviate Academy: Vector Compression Course](/academy/compression/)
- [Blog: Reducing Memory with Product Quantization](https://weaviate.io/blog/pq-rescoring)
:::
:::

## Questions and feedback

import DocsFeedback from '/\_includes/docs-feedback.mdx';

<DocsFeedback/>