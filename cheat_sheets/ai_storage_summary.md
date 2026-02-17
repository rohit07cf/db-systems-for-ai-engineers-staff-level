# AI/ML Storage Architecture -- Quick Reference

## Component Selection Matrix

| AI Workload | Primary Storage | Why | Alternatives |
|---|---|---|---|
| **Embedding / Vector Search** | Vector DB (Pinecone, Milvus, Qdrant) | ANN index, sub-ms retrieval | pgvector (< 5M), Redis (< 1M) |
| **Feature Store (online)** | Redis, DynamoDB, Bigtable | Low-latency key-value lookup (< 10 ms) | Feast + Redis, Tecton |
| **Feature Store (offline)** | Parquet on S3, Delta Lake, BigQuery | Columnar, cheap, time-travel | Hive, Iceberg |
| **Training Data** | Object storage (S3, GCS) + data lake | Throughput-optimized, cheap | HDFS (legacy), Alluxio (caching) |
| **Model Registry** | S3 + metadata DB (Postgres, MLflow) | Versioned blobs + metadata queries | Weights & Biases, Neptune |
| **Model Serving Cache** | Redis, Memcached | Cache frequent inference results | CDN for static models |
| **Experiment Tracking** | Postgres + S3 (MLflow, W&B) | Structured metrics + artifact blobs | Specialized: Neptune, Comet |
| **RAG Document Store** | Vector DB + document store (Elasticsearch, MongoDB) | Hybrid: semantic + full-text | Single vector DB with payload |
| **Graph / Knowledge Base** | Neo4j, Neptune, TigerGraph | Relationship traversal for KG-RAG | Postgres with recursive CTEs |
| **Real-time Feature Pipeline** | Kafka + Flink -> Feature Store | Stream processing for fresh features | Spark Structured Streaming |
| **GPU Training I/O** | NVMe local + distributed (Lustre, GPFS, WekaFS) | High IOPS, sequential read throughput | FUSE-mount S3 (s3fs, Mountpoint) |

## Common Architecture Patterns

### RAG (Retrieval-Augmented Generation)

```
User Query
    |
    v
Embedding Model (e.g., text-embedding-3-large)
    |
    v
Vector DB (semantic search, top-K chunks)
    |
    +--> [Optional] Reranker (cross-encoder)
    |
    v
Context Assembly (top-K chunks + system prompt)
    |
    v
LLM (GPT-4, Claude, etc.)
    |
    v
Response (grounded in retrieved context)

Storage components:
  - Document Store: S3 (raw docs) + Postgres (metadata)
  - Chunk Store: Vector DB (embeddings + text payload)
  - Cache Layer: Redis (query -> response cache, TTL-based)
```

**Key sizing**: 1M documents at ~10 chunks each = 10M vectors.
At d=1536 with HNSW: ~120 GB RAM. Consider PQ or DiskANN beyond this.

### Feature Store Architecture

```
Offline Path (batch):
  Data Warehouse --> Spark/Dbt --> Feature Table (Parquet/Delta)
                                        |
                                        v
                                  Offline Store (S3/GCS)
                                        |
                                   [Materialization]
                                        |
                                        v
Online Path (serving):           Online Store (Redis/DynamoDB)
  Request --> Feature Server --> Low-latency lookup --> Model Server
                  |
                  +--> Compute on-demand features (real-time)

Key metrics:
  - Online latency: p99 < 10 ms
  - Feature freshness: minutes (streaming) to hours (batch)
  - Feature count: 1K-100K features typical at FAANG scale
```

### Model Registry

```
Training Job
    |
    v
Model Artifact --> S3/GCS (versioned, immutable)
    |
    v
Metadata DB (Postgres):
  - model_id, version, framework, metrics
  - training_dataset_ref, hyperparameters
  - lineage (parent model, fine-tune source)
  - deployment_status, A/B test results
    |
    v
Model Serving (Triton, TFServing, vLLM)
  - Pull model from S3 on deploy
  - Cache hot models on local NVMe
  - Canary deploy with traffic splitting
```

## Scale Thresholds -- When Architecture Changes

| Scale | Characteristic | Architecture Shift |
|---|---|---|
| **< 1M vectors** | Small RAG, single model | Single Postgres + pgvector |
| **1M -- 50M vectors** | Medium RAG, multiple models | Dedicated vector DB (single node HNSW) |
| **50M -- 500M vectors** | Large-scale search | Distributed vector DB, quantization |
| **> 1B vectors** | Web-scale semantic search | DiskANN, custom sharding, multi-tier |
| **< 100 features** | Early ML platform | Feature dict in application code |
| **100 -- 10K features** | Growing ML org | Feast/Tecton feature store |
| **> 10K features** | FAANG-scale platform | Custom feature platform, dedicated team |
| **< 10 models in prod** | Startup ML | MLflow + manual deployment |
| **10 -- 100 models** | Scaling ML org | Model registry + CI/CD pipeline |
| **> 100 models** | FAANG-scale | Custom ML platform (Michelangelo, FBLearner) |

## Cost Estimation Reference

### Vector Search Costs

```
Cloud vector DB (managed):
  Pinecone: ~$0.096/hr per pod (s1), 5M vectors per pod
  Qdrant Cloud: ~$0.078/hr (4 vCPU, 16GB)
  Weaviate Cloud: ~$0.125/hr (Standard)

Self-hosted estimate (AWS):
  10M vectors (d=1536, HNSW):
    Memory needed: ~120 GB
    Instance: r6g.4xlarge (128 GB) = ~$0.80/hr = ~$580/mo
    With replica: ~$1,160/mo

  100M vectors (d=1536, PQ):
    Memory needed: ~15 GB (PQ) + 20 GB (metadata)
    Instance: r6g.xlarge (32 GB) + fast SSD for DiskANN
    Cost: ~$300-600/mo
```

### Feature Store Costs

```
Online store (Redis):
  1M features * 1 KB avg = 1 GB -> t3.small = ~$25/mo
  100M features * 1 KB avg = 100 GB -> r6g.4xlarge cluster = ~$2,000/mo

Offline store (S3 + compute):
  1 TB Parquet on S3 = ~$23/mo (storage)
  Daily Spark job (4 nodes * 2hr) = ~$50/day = ~$1,500/mo
  Total offline: ~$1,500-2,000/mo
```

### Training Data I/O Costs

```
S3 storage:      $0.023/GB/mo (Standard), $0.0125/GB/mo (IA)
S3 GET requests: $0.0004 per 1,000 requests
S3 throughput:   ~100 MB/s per stream, parallelize for GPUs

Training bottleneck estimate:
  8x A100 GPUs need ~20 GB/s sustained read
  S3 can deliver this with ~200 parallel streams
  Or: local NVMe cache (3.5 GB/s per drive * 8 drives = 28 GB/s)
```

## Embedding Pipeline Design

```
Source Documents
    |
    v
Chunking Strategy:
  - Fixed size: 512 tokens, 50-token overlap (simple, robust)
  - Semantic: split by paragraph/section (better quality)
  - Recursive: try large chunks first, split if too big
    |
    v
Embedding Model:
  | Model                     | Dims  | Speed      | Quality |
  |---------------------------|-------|------------|---------|
  | OpenAI text-embedding-3-large | 3072  | 3K tok/s   | Best    |
  | OpenAI text-embedding-3-small | 1536  | 6K tok/s   | Good    |
  | Cohere embed-v3           | 1024  | Fast       | Great   |
  | BGE-large-en-v1.5 (OSS)  | 1024  | Self-host  | Good    |
  | all-MiniLM-L6 (OSS)      | 384   | Very fast  | Fair    |
    |
    v
Vector DB (with metadata payload)
```

**Throughput estimate**: Embedding 1M documents (~500 tokens each) with OpenAI API:
- 500M tokens / 3,000 tokens/s = ~46 hours (single thread)
- With batching + parallelism (10 threads): ~5 hours
- Cost: 500M tokens * $0.00013/1K = ~$65

## Storage Tiering for AI Workloads

| Tier | Technology | Latency | Cost/GB/mo | Use Case |
|---|---|---|---|---|
| **Hot** | GPU HBM / NVMe | < 1 us / < 0.1 ms | $$$$ | Active model weights, current batch |
| **Warm** | Redis / in-memory DB | < 1 ms | $$$ | Feature store online, embedding cache |
| **Cool** | SSD-backed DB (DiskANN) | 1-10 ms | $$ | Large vector indices, historical features |
| **Cold** | S3 / GCS (Standard) | 50-100 ms | $0.023 | Training datasets, model artifacts |
| **Archive** | S3 Glacier / GCS Archive | Minutes-hours | $0.004 | Old model versions, audit logs |

## Key Talking Points for AI Infrastructure Interviews

1. **"Storage is the bottleneck in ML, not compute."** GPU utilization drops below 50% when
   data loading is not optimized. Invest in data pipeline before buying more GPUs.

2. **"Feature stores separate feature engineering from model training."** This enables
   feature reuse across teams, consistent online/offline computation, and point-in-time
   correctness to prevent training/serving skew.

3. **"RAG is not just vector search."** A production RAG system needs: chunking strategy,
   embedding pipeline, hybrid search (keyword + semantic), reranking, caching, and
   freshness management. The vector DB is 20% of the system.

4. **"Embedding model choice determines your entire storage architecture."** Dimensions
   directly impact memory, cost, and latency. Going from 1536 to 384 dims can cut
   storage costs 4x while losing only ~5% relevance with good fine-tuning.

5. **"Online feature stores must be co-located with model serving."** Feature fetch at
   p99 < 10 ms means same-region, same-VPC deployment. Cross-region feature lookup
   adds 60-200 ms, which is unacceptable for real-time inference.

6. **"The model registry is your ML system's source of truth."** Every deployed model must
   be traceable to its training data, hyperparameters, and metrics. This is both an
   engineering best practice and a regulatory requirement (EU AI Act).

7. **"Plan for embedding model upgrades."** When you retrain or change embedding models,
   ALL vectors must be recomputed. Design for dual-index operation during migration.
   Budget: ~5 hours per 10M documents for recomputation + reindexing.

8. **"Cost optimization: quantize vectors before scaling hardware."** PQ or scalar
   quantization can reduce memory 4-32x with < 5% recall loss. Always try compression
   before adding nodes.

## Key Numbers to Remember

| Metric | Value |
|---|---|
| OpenAI embedding cost (3-small) | ~$0.02 / 1M tokens |
| OpenAI embedding cost (3-large) | ~$0.13 / 1M tokens |
| A100 GPU memory (80 GB HBM) | Fits ~13M float32 vectors at d=1536 |
| S3 throughput (single stream) | ~100 MB/s |
| Redis feature lookup (p99) | < 1 ms (same AZ) |
| DynamoDB feature lookup (p99) | < 10 ms |
| Acceptable feature freshness | Seconds (real-time) to hours (batch) |
| HNSW build speed | ~5K-20K vectors/sec/core |
| Target GPU utilization | > 70% (below = I/O bottleneck) |
| Vector reindex budget | ~5 hrs per 10M docs |
