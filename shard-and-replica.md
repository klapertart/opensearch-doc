# OpenSearch: Sharding & Replica

> Resume lengkap konsep, cara kerja, sizing, dan konfigurasi shard & replica di OpenSearch.

---

## 1. Konsep Dasar

### Index, Shard, dan Dokumen

```
Index (abstraksi logis)
  └── Primary Shard 0  →  menyimpan sebagian dokumen
  └── Primary Shard 1  →  menyimpan sebagian dokumen
  └── Primary Shard 2  →  menyimpan sebagian dokumen
       └── Replica Shard 0  →  salinan identik Primary 0
       └── Replica Shard 1  →  salinan identik Primary 1
       └── Replica Shard 2  →  salinan identik Primary 2
```

- **Index** — nama logis yang kamu kenal (`log_20260124`). Tidak menyimpan data secara langsung.
- **Primary Shard** — potongan fisik index yang benar-benar menyimpan dokumen di disk.
- **Replica Shard** — salinan identik dari primary shard. Disimpan di node berbeda.

### Formula Shard Total

```
Total shard = number_of_shards × (1 + number_of_replicas)

Contoh:
  number_of_shards:   3
  number_of_replicas: 1
  Total shard = 3 × (1 + 1) = 6 shard per index
```

### Akumulasi Shard Seiring Waktu

Shard tetap ada selama index belum dihapus dan terus mengonsumsi resource:

```
1 index  =   6 shard
10 index =  60 shard
100 index = 600 shard  ← semua tetap aktif di memory
```

Inilah alasan ISM delete policy wajib ada di production.

---

## 2. Cara Kerja Sharding

### Routing Dokumen ke Shard

Saat dokumen masuk, OpenSearch menentukan shard tujuan via formula:

```
shard = hash(document_id) % number_of_primary_shards
```

```
Contoh: 9 juta dokumen, 3 primary shard

  Shard-0 → ~3 juta dokumen  (hash % 3 == 0)
  Shard-1 → ~3 juta dokumen  (hash % 3 == 1)
  Shard-2 → ~3 juta dokumen  (hash % 3 == 2)
```

> ⚠️ Karena routing bergantung pada `number_of_primary_shards`, jumlah primary shard
> **tidak bisa diubah setelah index dibuat**. Kalau diubah, semua dokumen salah routing.

### Distribusi Shard ke Node

OpenSearch tidak pernah menaruh primary shard dan replica-nya di node yang sama:

```
Cluster 2 node, index dengan 3 primary + 3 replica:

  Node 1 (hot-node-1): [P0] [P1] [R2]
  Node 2 (hot-node-2): [P2] [R0] [R1]

  P = Primary, R = Replica
  R0 = replica dari P0, dst.
```

Kalau Node 1 mati → Node 2 masih punya R0 dan R1, data tidak hilang.

---

## 3. Cara Kerja Replica

### Fungsi Replica

```
Primary Shard  →  menerima write (ingest dokumen baru)
Replica Shard  →  salinan read-only, bisa melayani query
```

### Replica Mempercepat Read

Query di-fan-out ke semua shard yang relevan secara paralel:

```
Client request → Coordinating Node
                      ↓
         ┌────────────┼────────────┐
         ↓            ↓            ↓
      Shard-0      Shard-1      Shard-2
   (P atau R)   (P atau R)   (P atau R)
         ↓            ↓            ↓
         └────────────┼────────────┘
                      ↓
              Merge + Sort hasil
                      ↓
                  Response
```

OpenSearch round-robin antara primary dan replica → semakin banyak replica,
semakin banyak node bisa melayani query secara bersamaan.

### Replica Bisa Diubah Kapan Saja

Berbeda dengan primary shard, replica boleh di-scale up/down tanpa reindex:

```
Awalnya:  number_of_replicas: 1  (production, butuh availability)
Cold tier: number_of_replicas: 0  (hemat storage, data lama jarang diquery)
```

---

## 4. Shard Sizing

### Aturan Maksimal Shard per Node

```
Rekomendasi OpenSearch: maksimal 20 shard per 1GB heap JVM

Heap JVM = setengah dari total RAM node (rekomendasi)

Contoh:
  RAM node  = 16GB
  Heap JVM  = 8GB  (16 / 2)
  Maks shard = 20 × 8 = 160 shard per node
```

> Dengan 160 shard maks per node dan 6 shard per index,
> satu node idealnya tidak menampung lebih dari ~26 index aktif sekaligus.

### Ukuran Ideal per Shard

| Kondisi | Ukuran Shard | Dampak |
|---|---|---|
| **Ideal** | 10GB – 50GB | Query cepat, recovery cepat |
| **Terlalu kecil** | < 1GB | Overhead metadata tinggi, banyak shard idle |
| **Terlalu besar** | > 100GB | Query lambat, recovery lama saat node mati |

### Cara Hitung Jumlah Shard yang Tepat

```
Data per index = 30GB
Target ukuran shard = 10GB
Jumlah primary shard = 30 / 10 = 3

Config:
  number_of_shards:   3
  number_of_replicas: 1
  Total shard = 6, tersebar minimal di 2 node
```

### Kapan Tambah Replica vs Tambah Primary Shard?

```
Read lambat, write sudah cepat   →  tambah REPLICA
                                    lebih banyak node melayani query

Write lambat, data makin besar   →  tambah PRIMARY SHARD
                                    tapi harus reindex dari awal

Node mati dan data hilang        →  pastikan number_of_replicas >= 1
```

---

## 5. Konfigurasi

### Cara 1 — Saat Buat Index Langsung

Berlaku hanya untuk index tersebut:

```json
PUT log_20260124
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "@timestamp": { "type": "date" },
      "message":    { "type": "text" },
      "level":      { "type": "keyword" }
    }
  }
}
```

---

### Cara 2 — Via Index Template (Rekomendasi Production)

Semua index baru yang match pattern `log_*` otomatis pakai config ini:

```json
PUT _index_template/logs_template
{
  "index_patterns": ["log_*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "index.routing.allocation.require.temperature": "hot",
      "plugins.index_state_management.policy_id": "hot_cold_policy"
    },
    "mappings": {
      "properties": {
        "@timestamp": { "type": "date" },
        "message":    { "type": "text" },
        "level":      { "type": "keyword" },
        "service":    { "type": "keyword" },
        "host":       { "type": "keyword" }
      }
    }
  }
}
```

---

### Cara 3 — Ubah Replica Setelah Index Dibuat

Primary shard tidak bisa diubah, tapi replica bisa kapan saja:

```json
// Turunkan replica ke 0 untuk index cold (hemat storage)
PUT log_20260101/_settings
{
  "index": {
    "number_of_replicas": 0
  }
}

// Naikkan replica kembali jika dibutuhkan
PUT log_20260101/_settings
{
  "index": {
    "number_of_replicas": 1
  }
}
```

---

### Cara 4 — Otomatis via ISM Policy

ISM mengubah replica secara otomatis saat index berpindah tier:

```json
PUT _plugins/_ism/policies/hot_cold_policy
{
  "policy": {
    "description": "Lifecycle: hot → warm → cold → delete",
    "default_state": "hot",
    "states": [
      {
        "name": "hot",
        "actions": [],
        "transitions": [
          {
            "state_name": "warm",
            "conditions": { "min_index_age": "7d" }
          }
        ]
      },
      {
        "name": "warm",
        "actions": [
          {
            "replica_count": { "number_of_replicas": 1 }
          },
          {
            "allocation": {
              "require": { "temperature": "warm" }
            }
          },
          {
            "force_merge": { "max_num_segments": 1 }
          }
        ],
        "transitions": [
          {
            "state_name": "cold",
            "conditions": { "min_index_age": "30d" }
          }
        ]
      },
      {
        "name": "cold",
        "actions": [
          {
            "replica_count": { "number_of_replicas": 0 }
          },
          {
            "allocation": {
              "require": { "temperature": "cold" }
            }
          },
          {
            "read_only": {}
          }
        ],
        "transitions": [
          {
            "state_name": "delete",
            "conditions": { "min_index_age": "90d" }
          }
        ]
      },
      {
        "name": "delete",
        "actions": [{ "delete": {} }],
        "transitions": []
      }
    ]
  }
}
```

---

### Cara 5 — Verifikasi Shard via API

```bash
# Lihat semua shard dan statusnya
GET /_cat/shards?v&h=index,shard,prirep,state,node,store

# Output contoh:
# index          shard prirep state   node         store
# log_20260124   0     p      STARTED hot-node-1   8.5gb
# log_20260124   0     r      STARTED hot-node-2   8.5gb
# log_20260124   1     p      STARTED hot-node-2   8.3gb
# log_20260124   1     r      STARTED hot-node-1   8.3gb
# log_20260124   2     p      STARTED hot-node-1   8.7gb
# log_20260124   2     r      STARTED hot-node-2   8.7gb

# Lihat total shard per node
GET /_cat/nodes?v&h=name,shards,heap.percent,ram.percent,cpu

# Cek setting shard sebuah index
GET /log_20260124/_settings
```

---

## 6. Hubungan Shard dengan Hot-Cold

```
Hot tier  →  number_of_replicas: 1
             Data recent, sering diquery, butuh availability tinggi
             Primary + Replica aktif di SSD

Cold tier →  number_of_replicas: 0  (di-set otomatis oleh ISM)
             Data lama, jarang diquery, availability tidak kritis
             Hanya Primary, di HDD/S3
             Hemat storage hingga 50%
```

ISM otomatis mengurangi replica saat index masuk cold tier — ini salah satu
cara hot-cold menghemat storage secara signifikan tanpa intervensi manual.

---

## 7. Manajemen Kapasitas Shard di Production

### Tidak Ada Auto-Eviction di OpenSearch

Ini poin kritis yang sering terlewat. OpenSearch **tidak otomatis** menghapus atau memindahkan shard lama saat node mendekati batas kapasitas.

```
Node sudah 240 shard → index baru masuk → shard ke-241 dibuat
OpenSearch TIDAK otomatis hapus/pindah shard lama

Yang terjadi:
  - Cluster status → YELLOW (shard tidak bisa dialokasikan)
  - Atau shard dipaksakan masuk → node overload
  - Performa drop, GC pressure naik, ingestion bisa gagal
```

---

### Siapa yang Mengatur? — Tiga Komponen Wajib

#### Komponen 1 — ISM Delete Policy (Satu-satunya Auto Manager)

ISM yang menghapus index lama sehingga shard lama hilang dan slot terbuka untuk index baru:

```
Ingestion berjalan terus → index baru dibuat tiap hari
ISM delete index > 90 hari → shard lama dihapus

Equilibrium tercapai kalau:
  shard masuk per hari == shard keluar per hari (via ISM delete)
```

Tanpa ISM delete policy, shard terus bertambah tanpa batas sampai cluster tidak bisa terima index baru.

**Kalkulasi kapasitas sebelum production:**

```
Config       : 3 primary shard, 1 replica → 6 shard per index
Retensi data : 90 hari
Index per hari: 1 (log_YYYYMMDD)
Maks shard/node: 240 (RAM 16GB, heap 8GB → 20 × 8 = 160, lebih aman pakai 240 jika 3 node)

Total shard aktif: 90 hari × 6 shard = 540 shard
Node yang dibutuhkan: 540 / 240 = 2.25 → minimal 3 node
```

---

#### Komponen 2 — Cluster Hard Limit (Safety Net)

OpenSearch punya setting untuk menolak alokasi shard baru kalau sudah melebihi batas:

```json
PUT _cluster/settings
{
  "persistent": {
    "cluster.max_shards_per_node": 240
  }
}
```

Kalau limit tercapai, OpenSearch throw error saat index baru mau dibuat:

```
[TOO_MANY_REQUESTS] Validation Failed:
this action would add [6] total shards,
but this cluster currently has [240]/[240] maximum shards open
```

> ⚠️ Ini safety net, bukan solusi. Ingestion akan gagal kalau tidak ditangani.
> Solusinya tetap ISM delete policy + kapasitas node yang cukup.

---

#### Komponen 3 — Monitoring & Alerting (Kamu yang Pantau)

OpenSearch tidak kirim notifikasi kalau shard mendekati batas. Harus setup sendiri:

```bash
# Cek jumlah shard per node
GET /_cat/nodes?v&h=name,shards,heap.percent,ram.percent

# Output contoh:
# name         shards  heap.percent  ram.percent
# hot-node-1   198     72            85       ← mendekati batas, perlu perhatian
# hot-node-2   201     68            80       ← mendekati batas
# cold-node-1  87      45            60       ← aman

# Cek detail shard per index
GET /_cat/shards?v&h=index,shard,prirep,state,node,store
```

Pasang alert di Prometheus/Grafana/CloudWatch:

```yaml
# Contoh alert rule Prometheus
- alert: OpenSearchShardCountWarning
  expr: opensearch_node_shards_total > 200
  labels:
    severity: warning
  annotations:
    summary: "Shard count mendekati batas di node {{ $labels.node }}"

- alert: OpenSearchShardCountCritical
  expr: opensearch_node_shards_total > 230
  labels:
    severity: critical
  annotations:
    summary: "Shard count kritis di node {{ $labels.node }}, segera investigasi"
```

---

### Alur Ideal di Production

```
Ingestion harian
  ↓
Index baru dibuat (log_20260124) → 6 shard masuk hot node
  ↓
ISM berjalan tiap malam:
  ├── Index > 7 hari   → pindah ke warm node
  ├── Index > 30 hari  → pindah ke cold node, replica → 0
  │                       (shard berkurang dari 6 → 3, hemat 50%)
  └── Index > 90 hari  → DELETE → 3 shard hilang, slot terbuka
  ↓
Slot shard terbuka untuk index hari berikutnya
↓
Jumlah shard di cluster relatif stabil (ada yang masuk, ada yang keluar)
```

---

### Checklist Sebelum Production

```
✅ Hitung total shard aktif: (retensi hari × shard per index)
✅ Hitung node yang dibutuhkan: total shard / maks shard per node
✅ Set cluster.max_shards_per_node sebagai safety net
✅ ISM delete policy aktif dan sudah ditest
✅ Monitoring shard count per node sudah terpasang
✅ Alert threshold sudah dikonfig (warning 80%, critical 95% dari maks)
```

---

## 8. Ringkasan

| Aspek | Primary Shard | Replica Shard |
|---|---|---|
| **Fungsi** | Distribusi data & write | Fault tolerance & percepat read |
| **Bisa diubah setelah index dibuat?** | ❌ Tidak | ✅ Bisa kapan saja |
| **Dikonfig di** | Index create / template | Index settings / ISM |
| **Mempercepat** | Write (paralel ingest) | Read (paralel query) |
| **Di cold tier** | Tetap ada | Dihapus (replica: 0) |

### Aturan Praktis Production

```
1. Ukuran shard ideal      : 10GB – 50GB per shard
2. Maks shard per node     : 20 × heap JVM (GB)
3. Heap JVM                : setengah dari total RAM
4. Selalu pakai template   : agar mapping & shard config konsisten
5. ISM wajib ada           : satu-satunya auto manager shard lama
6. Primary shard           : tentukan sebelum index dibuat, tidak bisa diubah
7. Replica di cold         : turunkan ke 0 untuk hemat storage 50%
8. Hitung kapasitas dulu   : (retensi hari × shard per index) / maks shard per node
9. Set cluster hard limit  : cluster.max_shards_per_node sebagai safety net
10. Monitoring wajib       : OpenSearch tidak notifikasi shard mendekati batas
```
