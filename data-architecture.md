# Data Architecture: Jenis Data, Primary Storage & Search Analytics

> Panduan memilih stack yang tepat berdasarkan karakteristik data.

---

## 1. Data Lake vs Data Warehouse

Dua konsep ini sering tertukar. Penting dipahami dulu sebelum memilih stack.

### Definisi

| | Data Lake | Data Warehouse |
|---|---|---|
| **Definisi** | Tempat menyimpan semua data dalam bentuk **raw/mentah** | Tempat menyimpan data yang sudah **terstruktur & siap dianalisis** |
| **Format data** | Raw: JSON, CSV, Parquet, image, video, log mentah | Terstruktur: tabel dengan schema yang ketat |
| **Schema** | Schema-on-read (schema ditentukan saat dibaca) | Schema-on-write (schema ditentukan saat disimpan) |
| **Pengguna** | Data engineer, data scientist | Data analyst, business user |
| **Query** | Fleksibel, eksplorasi, ML | SQL terstruktur, laporan, dashboard |
| **Biaya storage** | Sangat murah (S3, GCS) | Lebih mahal (compute + storage) |
| **Contoh tools** | S3 + Glue, Delta Lake, Apache Iceberg | BigQuery, Snowflake, Redshift, ClickHouse |

---

### Karakteristik Detail

#### Data Lake
```
Prinsip: "Simpan dulu, pikirkan struktur belakangan"

Cocok untuk:
  - Volume data sangat besar dan terus bertambah
  - Format data beragam (log, JSON, CSV, binary, image)
  - Kebutuhan analisis belum jelas saat data masuk
  - Tim data science butuh akses ke raw data untuk ML/AI
  - Retensi jangka sangat panjang dengan biaya minimal

Contoh use case:
  - Semua log aplikasi disimpan ke S3 dalam format JSON mentah
  - Data clickstream dari jutaan user disimpan apa adanya
  - Data IoT dari ribuan sensor disimpan untuk analisis mendatang
```

#### Data Warehouse
```
Prinsip: "Data sudah bersih, terstruktur, dan siap dipakai"

Cocok untuk:
  - Analisis bisnis yang sudah jelas kebutuhannya
  - Query SQL oleh tim non-teknis (analyst, finance, marketing)
  - Dashboard dan laporan yang butuh konsistensi data
  - Agregasi cepat atas data yang sudah di-transform

Contoh use case:
  - Laporan revenue per produk per bulan
  - Cohort analysis user berdasarkan tanggal registrasi
  - Dashboard KPI bisnis yang diakses tiap hari
```

---

### Hubungan Keduanya: Bukan Pilihan, Tapi Lapisan

Di arsitektur modern, Data Lake dan Data Warehouse **tidak saling menggantikan** — keduanya bekerja berlapis:

```
[Source Systems]
  App, Server, IoT, API
        ↓
   Data Lake (S3 / GCS)        ← simpan semua data raw, murah, jangka panjang
   (raw zone)
        ↓
   ETL / ELT Pipeline           ← bersihkan, transform, strukturkan
   (dbt, Spark, Glue)
        ↓
   Data Warehouse               ← data siap pakai untuk analisis bisnis
   (BigQuery / Snowflake)
        ↓
   BI Tool / Dashboard          ← Metabase, Superset, Looker
   (business user)
```

Data Lake adalah **sumber kebenaran mentah**. Data Warehouse adalah **sumber kebenaran terstruktur**.

---

### Konsep Terkait: Data Lakehouse

Perkembangan terbaru menggabungkan keduanya dalam satu sistem — **Data Lakehouse**:

```
Data Lakehouse = Data Lake (storage murah) + Data Warehouse (query terstruktur)

Tools: Delta Lake, Apache Iceberg, Apache Hudi
       Databricks, BigQuery (bisa query langsung dari GCS)

Manfaat:
  - Simpan data raw di S3/GCS (murah)
  - Query dengan SQL seperti data warehouse (tanpa ETL berat)
  - Tidak perlu duplikasi data ke dua sistem terpisah
```

---

### Posisi OpenSearch di Antara Keduanya

```
Data Lake      → bukan, OpenSearch tidak dirancang untuk raw storage
Data Warehouse → bukan, OpenSearch tidak support JOIN & agregasi kompleks

OpenSearch adalah:
  Search & Analytics Layer  ← di atas Data Lake atau Data Warehouse
                               untuk kebutuhan full-text search
                               dan observability / log analytics
```

---

## Ringkasan 

| Jenis Data | Primary Storage | Search & Analytics |
|---|---|---|
| Transaksional (OLTP) | PostgreSQL / MySQL | Elasticsearch / OpenSearch |
| Log & Event | S3 + Parquet | OpenSearch / ClickHouse |
| Dokumen / Semi-structured | MongoDB / DynamoDB | OpenSearch / Typesense |
| Analitik / Warehouse (OLAP) | BigQuery / Snowflake / ClickHouse | Metabase / Grafana / Superset |
| Time-Series | TimescaleDB / InfluxDB | Grafana |
| File & Blob | S3 / GCS | - |

---

## 2. Data Transaksional (OLTP)

### Karakteristik
```
- Relasional, butuh JOIN antar tabel
- ACID (Atomicity, Consistency, Isolation, Durability)
- Banyak operasi INSERT, UPDATE, DELETE
- Volume per transaksi kecil tapi frekuensi tinggi
- Schema ketat dan terstruktur
```

### Contoh Data
```
- Data user, profil, akun
- Order, payment, invoice
- Inventory, stok barang
- Booking, reservasi
```

### Stack

```
Primary Storage:
  PostgreSQL   ← pilihan utama, fitur lengkap, support JSONB
  MySQL        ← alternatif, lebih sederhana
  Aurora       ← managed PostgreSQL/MySQL di AWS

Search & Analytics:
  OpenSearch / Elasticsearch  ← sync dari PostgreSQL via CDC (Change Data Capture)
                                 untuk kebutuhan full-text search
  Metabase / Grafana          ← dashboard & reporting bisnis
```

### Arsitektur
```
[App] → PostgreSQL (primary, source of truth)
              ↓ CDC (Debezium)
          OpenSearch  ← full-text search, filter kompleks
          Metabase    ← laporan bisnis, dashboard
```

---

## 3. Data Log & Event (Time-Series, Append-Only)

### Karakteristik
```
- Volume sangat besar, terus bertambah
- Tidak pernah diupdate setelah ditulis
- Query dominan by rentang waktu
- Tidak butuh JOIN
- Butuh retention policy (data lama dihapus otomatis)
```

### Contoh Data
```
- Application log (error, info, debug)
- Server & infrastructure log
- Audit trail, security event
- Clickstream, user activity
- IoT sensor event
```

### Stack

```
Primary Storage:
  S3 + Parquet  ← murah, unlimited, compressed, tahan lama
                   query via Athena/Glue kalau dibutuhkan sewaktu-waktu
  ClickHouse    ← self-hosted, columnar, sangat efisien untuk log
                   query agregasi jutaan baris tetap cepat

Search & Analytics (jangka pendek, 7-90 hari):
  OpenSearch    ← full-text search, dashboard real-time
  ClickHouse    ← bisa jadi primary sekaligus analytics layer

Transport / Buffer (opsional):
  Kafka         ← hanya jika volume sangat tinggi atau butuh multi-consumer
                   retention Kafka: 1-3 hari saja, bukan long-term storage
  Logstash / Fluentd / Vector  ← pipeline langsung tanpa broker
                                  cocok untuk skala sedang
```

### Arsitektur (Skala Besar)
```
[App / Server]
      ↓
   Kafka (buffer 1-3 hari)
      ↓
  Logstash / Flink
    ↓           ↓
OpenSearch      S3 + Parquet
(7-30 hari,     (long-term archive,
 fast search,    murah, query via Athena)
 dashboard)
```

### Arsitektur (Skala Sedang — Lebih Simple)
```
[App / Server]
      ↓
  Fluentd / Vector / Logstash
    ↓              ↓
OpenSearch       S3 + Parquet
(search &        (archive)
 dashboard)
```

---

## 4. Data Dokumen / Semi-Structured

### Karakteristik
```
- Schema tidak fix, sering berubah
- Nested object, array dalam satu dokumen
- Tidak relasional
- Read lebih banyak dari write
```

### Contoh Data
```
- Profil user yang fleksibel (tiap user beda field)
- Konfigurasi aplikasi per tenant
- Konten CMS (artikel, produk dengan atribut beda-beda)
- Hasil response API third-party
```

### Stack

```
Primary Storage:
  MongoDB      ← dokumen fleksibel, nested object, horizontal scale
  DynamoDB     ← managed di AWS, serverless, auto-scale
  PostgreSQL   ← untuk skala sedang, pakai tipe JSONB

Search & Analytics:
  OpenSearch / Elasticsearch  ← full-text search, filter nested field
  Typesense                   ← alternatif ringan, lebih mudah di-setup
                                 cocok untuk search produk / katalog
```

### Arsitektur
```
[App] → MongoDB (primary)
              ↓ sync / change stream
          OpenSearch  ← search, filter, faceted search
```

---

## 5. Data Analitik / Warehouse (OLAP)

### Karakteristik
```
- Query agregasi besar (SUM, AVG, GROUP BY jutaan baris)
- JOIN banyak tabel / dataset
- Schema relatif stabil
- Write batch (tidak real-time)
- Untuk kebutuhan laporan dan business intelligence
```

### Contoh Data
```
- Sales report, revenue breakdown
- Cohort analysis, retention analysis
- Marketing attribution
- Financial reporting
```

### Stack

```
Primary Storage (Data Warehouse):
  BigQuery     ← serverless, managed, scale otomatis, bayar per query
  Snowflake    ← multi-cloud, sharing data antar tim mudah
  Redshift     ← managed di AWS, cocok kalau sudah di ekosistem AWS
  ClickHouse   ← self-hosted, sangat cepat untuk agregasi, open source

Search & Analytics / BI Tool:
  Metabase     ← self-hosted BI, mudah dipakai non-technical user
  Superset     ← open source, fleksibel, bisa custom chart
  Grafana      ← lebih ke metrics & time-series, bisa connect ke warehouse
  Looker       ← enterprise, mahal tapi powerful
```

### Arsitektur
```
[Source Systems]
  PostgreSQL, MongoDB, Log, API
        ↓
   ETL / ELT Pipeline
   (dbt, Airbyte, Fivetran)
        ↓
   BigQuery / Snowflake    ← data warehouse, source of truth analitik
        ↓
   Metabase / Superset     ← dashboard bisnis, laporan
```

---

## 6. Data Time-Series (Metrics)

### Karakteristik
```
- Data berupa angka dengan timestamp
- Insert sangat sering (per detik / per menit)
- Query: trend, agregasi per interval waktu
- Retention policy: data lama di-downsample atau dihapus
```

### Contoh Data
```
- CPU usage, memory, disk per server
- Response time, error rate aplikasi
- Harga saham, kurs mata uang
- Sensor suhu, kelembaban IoT
```

### Stack

```
Primary Storage:
  InfluxDB      ← purpose-built time-series, retention & downsampling bawaan
  TimescaleDB   ← PostgreSQL extension, familiar bagi yang sudah pakai Postgres
  Prometheus    ← untuk metrics infrastruktur & aplikasi (pull-based)
  VictoriaMetrics ← alternatif Prometheus, lebih efisien di skala besar

Search & Analytics:
  Grafana       ← standar industri untuk visualisasi metrics & time-series
```

### Arsitektur
```
[Server / App]
      ↓
  Prometheus (scrape metrics)
      ↓
  VictoriaMetrics / InfluxDB   ← long-term storage
      ↓
  Grafana                      ← dashboard, alerting
```

---

## 7. Data File & Blob

### Karakteristik
```
- Binary file: gambar, video, PDF, dokumen
- Tidak di-query isinya, hanya disimpan dan diambil
- Ukuran besar per file
```

### Contoh Data
```
- Upload foto profil user
- Attachment dokumen
- Video konten
- Backup file
```

### Stack

```
Primary Storage:
  S3 (AWS)         ← standar industri, murah, reliable
  GCS (Google)     ← alternatif di ekosistem Google
  MinIO            ← self-hosted, S3-compatible, untuk on-premise

Search & Analytics:
  Tidak perlu search engine khusus
  Metadata file disimpan di PostgreSQL/MongoDB → bisa di-search dari sana
```

---

## Kesimpulan: Cara Memilih Stack

```
Tanya dulu:

1. Apakah data butuh ACID & relasional?
   Ya  → PostgreSQL / MySQL

2. Apakah data adalah log / event append-only volume besar?
   Ya  → S3 + Parquet (archive) + OpenSearch (search jangka pendek)

3. Apakah schema tidak fix / nested / dokumen?
   Ya  → MongoDB / DynamoDB

4. Apakah butuh agregasi besar untuk laporan bisnis?
   Ya  → BigQuery / Snowflake / ClickHouse

5. Apakah data berupa angka dengan timestamp (metrics)?
   Ya  → InfluxDB / TimescaleDB + Grafana

6. Apakah butuh full-text search di atas data yang sudah ada?
   Ya  → OpenSearch / Elasticsearch (sebagai secondary, bukan primary)
```

> **OpenSearch selalu posisinya sebagai search & analytics layer,
> bukan primary storage. Data di OpenSearch idealnya adalah salinan
> atau turunan dari primary storage yang sesungguhnya.**

---

