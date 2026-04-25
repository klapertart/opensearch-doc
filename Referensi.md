# Referensi & Sumber

> Daftar link referensi untuk semua topik yang dibahas:
> OpenSearch Hot-Cold Architecture, Sharding & Replica, dan Data 
Architecture Stack.

---

## 1. OpenSearch: Hot-Cold Architecture & ISM

| Sumber | Topik | Link |
|---|---|---|
| OpenSearch Docs (official) | ISM Policies | 
https://docs.opensearch.org/latest/im-plugin/ism/policies/ |
| OpenSearch Docs (official) | Creating a Cluster & Hot-Warm Config | 
https://docs.opensearch.org/latest/tuning-your-cluster/ |
| AWS Docs | UltraWarm Storage | 
https://docs.aws.amazon.com/opensearch-service/latest/developerguide/ultrawarm.html 
|
| AWS Docs | Cold Storage | 
https://docs.aws.amazon.com/opensearch-service/latest/developerguide/cold-storage.html 
|
| Opster | Setting Up Hot-Warm Architecture for ISM | 
https://opster.com/guides/opensearch/opensearch-data-architecture/setting-up-hot-warm-architecture-for-ism/ 
|
| Opster | Cold Storage in OpenSearch | 
https://opster.com/guides/opensearch/opensearch-data-architecture/cold-storage-in-opensearch/ 
|
| Elastic Blog | Implementing Hot-Warm-Cold with ILM | 
https://www.elastic.co/blog/implementing-hot-warm-cold-in-elasticsearch-with-index-lifecycle-management 
|
| OpenSearch Forum | Hot Warm Cold Architecture Discussion | 
https://forum.opensearch.org/t/hot-warm-cold-architecture/12552 |
| OpenSearch Forum | Configure Hot-Warm-Cold on Existing Cluster | 
https://forum.opensearch.org/t/configure-hot-warm-and-cold-architecture-in-an-existing-openserch-cluster/12284 
|
| GitHub OpenSearch | About Hot-Warm-Cold Architecture Issue | 
https://github.com/opensearch-project/OpenSearch/issues/1990 |

---

## 2. OpenSearch: Sharding & Replica

| Sumber | Topik | Link |
|---|---|---|
| OpenSearch Docs (official) | Intro to Shards & Replica | 
https://docs.opensearch.org/latest/getting-started/intro/ |
| OpenSearch Blog | Optimize Index Shard Size | 
https://opensearch.org/blog/optimize-opensearch-index-shard-size/ |
| AWS Docs | Choosing the Number of Shards | 
https://docs.aws.amazon.com/opensearch-service/latest/developerguide/bp-sharding.html 
|
| AWS Docs | Operational Best Practices (Shard per Node) | 
https://docs.aws.amazon.com/opensearch-service/latest/developerguide/bp.html 
|
| AWS Docs | Sizing OpenSearch Domains | 
https://docs.aws.amazon.com/opensearch-service/latest/developerguide/sizing-domains.html 
|
| AWS Blog | How Many Shards Do I Need? | 
https://aws.amazon.com/blogs/big-data/amazon-opensearch-service-101-how-many-shards-do-i-need/ 
|
| DEV.to | Seven Rules for OpenSearch Sizing | 
https://dev.to/aws-builders/seven-rules-for-opensearch-sizing-jo3 |
| Aiven Docs | Optimal Number of Shards | 
https://aiven.io/docs/products/opensearch/concepts/shards-number |
| Oracle Docs | OpenSearch Cluster Performance Sizing | 
https://docs.oracle.com/en-us/iaas/Content/search-opensearch/Concepts/performance-sizing.htm 
|
| GitHub | OpenSearch Index Shard Size Examples | 
https://github.com/ev2900/OpenSearch_Index_Shard_Size |

---

## 3. Data Architecture: Primary Storage & Stack Selection

### PostgreSQL vs ClickHouse

| Sumber | Topik | Link |
|---|---|---|
| Tinybird | ClickHouse vs PostgreSQL (with extensions) | 
https://www.tinybird.co/blog/clickhouse-vs-postgresql-with-extensions |
| DB Pro Blog | ClickHouse vs PostgreSQL — OLAP vs OLTP | 
https://www.dbpro.app/blog/clickhouse-vs-postgres |
| OctaByte Blog | ClickHouse vs PostgreSQL for Analytics | 
https://blog.octabyte.io/topics/open-source-databases/clickhouse-vs-postgresql-analytics/ 
|
| Instaclustr | ClickHouse vs PostgreSQL — 5 Key Differences | 
https://www.instaclustr.com/education/clickhouse/clickhouse-vs-postgres-5-key-differences-and-how-to-choose/ 
|
| Tasrie IT | ClickHouse vs PostgreSQL 2026 Complete Comparison | 
https://tasrieit.com/blog/clickhouse-vs-postgres-2026 |
| DoubleCloud | ClickHouse vs PostgreSQL Key Differences & Use Cases | 
https://double.cloud/blog/posts/2023/05/clickhouse-vs-postgresql/ |

### Pemilihan Database & Stack Analytics

| Sumber | Topik | Link |
|---|---|---|
| ClickHouse | How to Choose a Database for Real-Time Analytics 2026 | 
https://clickhouse.com/resources/engineering/how-to-choose-a-database-for-real-time-analytics-in-2026 
|
| DB Pro Blog | ClickHouse vs Druid — Real-Time Analytics | 
https://www.dbpro.app/blog/clickhouse-vs-druid |
| Parseable | S3 Parquet vs ClickHouse untuk Log Analytics | 
https://www.parseable.com/blog/clickhouse-vs-parseable |
| Medium | Benchmark: ClickHouse vs InfluxDB vs PostgreSQL vs Parquet(S3) 
vs MongoDB vs DuckDB | 
https://medium.com/@ev_kozloski/timeseries-databases-performance-testing-7-alternatives-56a3415e6e9e 
|

---

## Ringkasan Dokumen yang Sudah Dibuat

| File | Topik |
|---|---|
| `opensearch-hot-cold.md` | Hot-Cold Architecture, ISM Policy, Node 
Config, Java Query |
| `opensearch-sharding-replica.md` | Sharding, Replica, Shard Sizing, 
Capacity Management |
| `data-architecture-stack.md` | Jenis Data, Primary Storage, Search & 
Analytics Stack |
| `referensi.md` | Dokumen ini — daftar semua sumber referensi |
