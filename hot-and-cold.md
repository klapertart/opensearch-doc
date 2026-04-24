# OpenSearch: Hot-Cold Architecture & Index Naming Convention

> Rangkuman diskusi teknis mengenai mekanisme hot-cold di OpenSearch dan hubungannya dengan naming convention index.

---

## 1. Apa Itu Hot-Cold Architecture?

Hot-Cold adalah strategi **dua manfaat sekaligus**:

- **Cost Optimization** — menempatkan data lama di hardware murah (HDD/S3)
- **Query Performance Optimization** — memastikan data recent selalu berada di hardware terbaik (SSD + RAM besar)

> *"OpenSearch implements a Hot-Warm-Cold storage architecture to optimize query performance and costs by migrating indices across tiers based on data age and access frequency."*
> — docs.aws.amazon.com

### Tier Hardware

| Tier | Hardware | Karakteristik |
|---|---|---|
| **Hot** | SSD, RAM besar | Data recent, sering diakses, akses cepat |
| **Warm** | HDD | Data agak lama, jarang diakses |
| **Cold** | HDD / S3-backed | Data lama, sangat jarang diakses, akses lambat |

---

## 2. Mekanisme Teknis Hot-Cold

Yang menentukan sebuah index ada di tier mana adalah **dua komponen ini**, bukan nama index:

### Node Attributes (Infrastructure Level)

Node attributes adalah cara OpenSearch **menandai** setiap node dengan label tertentu, sehingga ISM tahu harus menaruh index di node mana.

#### Step 1 — Konfigurasi `opensearch.yml` di Setiap Node

Setiap node dalam cluster harus dikonfigurasi sesuai perannya. File `opensearch.yml` terletak di direktori config OpenSearch (biasanya `/etc/opensearch/opensearch.yml` atau `config/opensearch.yml`).

```yaml
# ============================================================
# HOT NODE — opensearch.yml
# Hardware: SSD, RAM besar (misal 32GB+)
# ============================================================
cluster.name: my-cluster
node.name: hot-node-1

# Tandai node ini sebagai "hot"
node.attr.temperature: hot

# Role node ini: menerima data baru (ingest) + menyimpan data hot
node.roles: [data, ingest, master]

# Path storage → arahkan ke SSD
path.data: /mnt/ssd/opensearch/data
path.logs: /mnt/ssd/opensearch/logs

network.host: 0.0.0.0
discovery.seed_hosts: ["hot-node-1", "warm-node-1", "cold-node-1"]
```

```yaml
# ============================================================
# WARM NODE — opensearch.yml
# Hardware: HDD biasa, RAM sedang (misal 16GB)
# ============================================================
cluster.name: my-cluster
node.name: warm-node-1

# Tandai node ini sebagai "warm"
node.attr.temperature: warm

# Role: hanya menyimpan data, tidak menerima ingest langsung
node.roles: [data]

# Path storage → arahkan ke HDD
path.data: /mnt/hdd/opensearch/data
path.logs: /mnt/hdd/opensearch/logs

network.host: 0.0.0.0
discovery.seed_hosts: ["hot-node-1", "warm-node-1", "cold-node-1"]
```

```yaml
# ============================================================
# COLD NODE — opensearch.yml
# Hardware: HDD besar tapi murah / S3-backed
# ============================================================
cluster.name: my-cluster
node.name: cold-node-1

# Tandai node ini sebagai "cold"
node.attr.temperature: cold

# Role: hanya menyimpan data read-only
node.roles: [data]

# Path storage → HDD besar atau mount point S3
path.data: /mnt/cold-storage/opensearch/data
path.logs: /mnt/cold-storage/opensearch/logs

network.host: 0.0.0.0
discovery.seed_hosts: ["hot-node-1", "warm-node-1", "cold-node-1"]
```

#### Step 2 — Verifikasi Node Berhasil Join Cluster

Setelah semua node distart, verifikasi via API:

```bash
GET /_cat/nodes?v&h=name,node.role,attr.temperature

# Output yang diharapkan:
# name          node.role  attr.temperature
# hot-node-1    dimr       hot
# warm-node-1   d          warm
# cold-node-1   d          cold
```

#### Step 3 — Buat Index Template agar Index Baru Otomatis ke Hot Node

```json
PUT _index_template/logs_template
{
  "index_patterns": ["log_*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.routing.allocation.require.temperature": "hot"
    },
    "mappings": {
      "properties": {
        "timestamp": { "type": "date" },
        "message":   { "type": "text" },
        "level":     { "type": "keyword" }
      }
    }
  }
}
```

Setiap kali index baru `log_*` dibuat, OpenSearch **otomatis** menaruhnya di hot node karena template ini.

#### Step 4 — Attach ISM Policy ke Index Template

Hubungkan ISM policy (yang sudah dibuat sebelumnya) ke template, agar lifecycle berjalan otomatis:

```json
PUT _index_template/logs_template
{
  "index_patterns": ["log_*"],
  "template": {
    "settings": {
      "index.routing.allocation.require.temperature": "hot",
      "plugins.index_state_management.policy_id": "hot_cold_policy"
    }
  }
}
```

#### Ringkasan Alur Konfigurasi

```
1. opensearch.yml tiap node  →  node.attr.temperature: hot/warm/cold
                                 menandai hardware tier

2. Index Template             →  index baru otomatis masuk hot node
                                 mapping field sudah benar dari awal

3. ISM Policy                 →  otomatis pindahkan index antar tier
                                 berdasarkan umur yang didefinisikan

4. Verifikasi via API         →  pastikan node join dengan attribute benar
```

### ISM Policy (Index State Management)

ISM adalah scheduler native OpenSearch yang **otomatis memindahkan index** antar tier berdasarkan umur atau ukuran.

```
Index baru dibuat → masuk Hot Node (SSD)
    ↓ umur > 7 hari (ISM)
Warm Node (HDD)
    ↓ umur > 30 hari (ISM)
Cold Node (HDD/S3)
    ↓ umur > 90 hari (ISM)
Deleted
```

ISM **tidak peduli nama index** — hanya peduli umur dan kondisi yang didefinisikan.

---

## 3. Naming Convention: `log_YYYYMMDD`

### Fungsi Utama: Query-Time Index Pruning

Naming convention dengan tanggal berfungsi di **layer aplikasi** — memangkas index yang tidak relevan **sebelum** query dieksekusi di OpenSearch.

### Perbandingan Teknis

#### ❌ Tanpa YYYYMMDD (`log_1`, `log_2`, `log_3`)

```
Java → buka semua index (log_*) 
     → OpenSearch scan semua dokumen dari semua index
     → filter by timestamp
     → buang dokumen yang tidak relevan

= Filter terjadi SETELAH data di-load ke memory
= Menyentuh cold node meskipun datanya tidak relevan
= Boros resource: CPU, memory, I/O
```

```java
// Java terpaksa query semua index karena tidak tahu mana yang relevan
new SearchRequest("log_*")
    .source(new SearchSourceBuilder()
        .query(QueryBuilders.rangeQuery("timestamp")
            .gte("now-7d")
            .lte("now")));
```

#### ✅ Dengan YYYYMMDD (`log_20260118`, `log_20260119`, ...)

```
Java → hitung nama index yang relevan (7 hari terakhir)
     → buka hanya index tersebut
     → OpenSearch query hanya di index yang dipilih

= Filter terjadi SEBELUM data di-load, di level nama index
= Cold node tidak tersentuh sama sekali
= Hemat resource secara signifikan
```

```java
// Java surgical — hanya buka index yang relevan
// Jangan hardcode 7 hari di business logic, gunakan index pattern saja
public SearchRequest buildQuery(int daysBack) {
    // OpenSearch akan otomatis query ke node yang tepat
    // berdasarkan di mana index itu berada (hot/cold node)

    String[] indices = buildIndexPattern(daysBack);
    // ["log_20260117", "log_20260118", ..., "log_20260124"]

    return new SearchRequest(indices)
        .source(new SearchSourceBuilder()
            .query(QueryBuilders.rangeQuery("@timestamp")
                .gte("now-" + daysBack + "d/d")
                .lte("now/d")));
}

private String[] buildIndexPattern(int daysBack) {
    return IntStream.range(0, daysBack)       // hasilkan stream: 0, 1, 2, ..., daysBack-1
        .mapToObj(i -> LocalDate.now().minusDays(i))  // tiap angka → tanggal mundur dari hari ini
        .map(date -> "log_" + date.format(DateTimeFormatter.BASIC_ISO_DATE)) // format jadi nama index
        .toArray(String[]::new);              // kumpulkan jadi array
}

// Contoh hasil untuk daysBack=3, hari ini 2026-01-24:
// buildIndexPattern(3) → ["log_20260124", "log_20260123", "log_20260122"]
// buildQuery(7)        → SearchRequest ke 7 index, hanya hot node yang disentuh
```

---

## 4. Hubungan Naming Convention dengan Hot-Cold

### Keduanya Independen, Tapi Saling Melengkapi

| Aspek | Naming Convention (YYYYMMDD) | Hot-Cold Tiering |
|---|---|---|
| **Fungsi** | Identifikasi & index pruning | Physical data placement |
| **Dikelola oleh** | Developer / Java layer | ISM Policy + node attributes |
| **Level** | Aplikasi | Infrastruktur |
| **Tujuan** | Tentukan index mana yang dibuka | Tentukan di hardware mana index tinggal |
| **Saling bergantung?** | ❌ Tidak | ❌ Tidak |

### Kombinasi Keduanya = Hasil Optimal

```
Query data 7 hari terakhir:

  Step 1 — Index Pruning (YYYYMMDD, layer Java):
    Hanya buka log_20260118 s/d log_20260124
    → Skip semua index lama, cold node tidak tersentuh

  Step 2 — Hot Tier (SSD, layer infrastruktur):
    7 index yang dipilih tadi ada di hot node
    → Akses hardware cepat

= Query sangat cepat dan efisien
```

---

## 5. Apa yang Mempercepat Query (Bukan Hot-Cold Saja)

Hot-cold berkontribusi pada performa, tapi bukan satu-satunya faktor. Root cause query lambat perlu didiagnosa lebih dulu:

| Teknik | Fungsi |
|---|---|
| **Naming `log_YYYYMMDD`** | Index pruning — skip index tidak relevan |
| **Proper mapping & field types** | Pastikan field `timestamp` bertipe `date`, bukan `text` |
| **Index template** | Konsistensi mapping dari awal |
| **Shard sizing yang tepat** | Distribusi beban optimal |
| **Filter sebelum aggregation** | Kurangi dokumen yang di-scan |
| **Force merge** | Kurangi segment count di index lama |
| **Hot-cold tiering** | Data recent di SSD, akses cepat |

---

## 6. Kesimpulan

```
log_YYYYMMDD  →  membantu MENEMUKAN index yang tepat (layer aplikasi)
Hot-Cold      →  menentukan SEBERAPA CEPAT index itu diakses (layer infrastruktur)
```

> **Naming convention `log_YYYYMMDD` adalah filter di level nama index yang terjadi sebelum OpenSearch bekerja. Jauh lebih ringan karena OpenSearch tidak perlu membuka, membaca, lalu membuang dokumen yang tidak relevan. Inilah kenapa naming convention ini menjadi standar di production system manapun yang berbasis time-series data.**