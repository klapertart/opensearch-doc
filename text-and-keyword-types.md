# OpenSearch: Text vs Keyword

> Dua tipe field untuk data string di OpenSearch — berbeda fundamental 
dalam cara penyimpanan dan penggunaannya.

---

## 1. Text

Field di-**analisis** sebelum disimpan — dipecah jadi token, di-lowercase, 
dihilangkan stop word-nya.

```
Input  : "Error connecting to Database Server"
Disimpan: ["error", "connecting", "database", "server"]
```

### Cocok Untuk
```
- Kalimat panjang yang butuh full-text search
- Log message, deskripsi, artikel, komentar
- User mengetik kata sebagian dan tetap ketemu hasilnya
```

### Contoh Query yang Bisa Dilakukan
```
User ketik: "database error"
→ ketemu dokumen yang mengandung "Error connecting to Database Server"
→ karena token "error" dan "database" ada di index

User ketik: "connecting"
→ tetap ketemu, karena token "connecting" ada
```

### Tidak Bisa Untuk
```
❌ Sorting
❌ Aggregation (group by, count distinct)
❌ Exact match
```

---

## 2. Keyword

Field disimpan **apa adanya** tanpa analisis — satu nilai utuh.

```
Input  : "Error connecting to Database Server"
Disimpan: "Error connecting to Database Server"  ← satu string utuh
```

### Cocok Untuk
```
- Nilai yang butuh exact match
- ID, status, kode, enum, nama field terstruktur
- Butuh sorting atau aggregation
```

### Contoh Query yang Bisa Dilakukan
```
filter status == "active"         ✅
filter level == "ERROR"           ✅
aggregation: count per level      ✅
sort by service_name              ✅
```

### Tidak Bisa Untuk
```
❌ Full-text search (harus exact match)
❌ User ketik sebagian kata → tidak ketemu
```

---

## 3. Perbandingan

| | Text | Keyword |
|---|---|---|
| **Analisis** | Ya, dipecah jadi token | Tidak, disimpan utuh |
| **Full-text search** | ✅ | ❌ |
| **Exact match** | ❌ | ✅ |
| **Aggregation** | ❌ | ✅ |
| **Sorting** | ❌ | ✅ |
| **Contoh field** | message, description, body | status, level, service, 
host |

---

## 4. Multi-field Mapping — Keduanya Digabung

Untuk field yang butuh **full-text search sekaligus aggregation**, gunakan 
keduanya sekaligus:

```json
PUT _index_template/logs_template
{
  "index_patterns": ["log_*"],
  "template": {
    "mappings": {
      "properties": {
        "message": {
          "type": "text",
          "fields": {
            "raw": {
              "type": "keyword"
            }
          }
        }
      }
    }
  }
}
```

```
Query full-text search → pakai field: "message"
Aggregation / exact   → pakai field: "message.raw"
```

---

## 5. Contoh Mapping untuk Log

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
        "@timestamp" : { "type": "date"    },
        "message"    : { "type": "text"    },
        "level"      : { "type": "keyword" },
        "service"    : { "type": "keyword" },
        "host"       : { "type": "keyword" },
        "trace_id"   : { "type": "keyword" },
        "error"      : {
          "type": "text",
          "fields": {
            "raw": { "type": "keyword" }
          }
        }
      }
    }
  }
}
```

### Alasan Tiap Field

| Field | Tipe | Alasan |
|---|---|---|
| `@timestamp` | date | Range query by waktu |
| `message` | text | Kalimat bebas, butuh full-text search |
| `level` | keyword | Nilai enum: ERROR, WARN, INFO — exact match & 
aggregation |
| `service` | keyword | Nama service terstruktur — filter & count per 
service |
| `host` | keyword | Nama server — filter per node |
| `trace_id` | keyword | ID unik — exact match untuk distributed tracing |
| `error` | text + keyword | Butuh search pesan error sekaligus 
aggregation jenis error |

---

## 6. Bahaya Dynamic Mapping

Kalau mapping tidak didefinisikan, OpenSearch akan **menebak tipe field 
secara otomatis** (dynamic mapping):

```
Field "level" dengan nilai "ERROR"
→ OpenSearch tebak sebagai: text  ← salah, harusnya keyword
→ Akibat: aggregation per level tidak bisa dilakukan
          filter level == "ERROR" tidak akurat
```

```json
// Matikan dynamic mapping untuk mencegah tebakan salah
PUT _index_template/logs_template
{
  "template": {
    "mappings": {
      "dynamic": "strict",    // tolak field yang tidak ada di mapping
      "properties": {
        // definisikan semua field secara eksplisit
      }
    }
  }
}
```

| Dynamic Setting | Perilaku |
|---|---|
| `true` (default) | Field baru otomatis ditambah dengan tipe hasil 
tebakan |
| `false` | Field baru diabaikan, tidak diindex |
| `strict` | Field baru ditolak, throw error |

> Rekomendasi production: gunakan `strict` agar tidak ada field yang salah 
tipe masuk tanpa disadari.

---

## 7. Aturan Praktis

```
Kalimat bebas yang diketik manusia  →  text
  contoh: message, description, body, comment

Nilai terstruktur, kode, enum       →  keyword
  contoh: status, level, service, host, trace_id, user_id

Butuh keduanya sekaligus            →  text + keyword (multi-field)
  contoh: error message yang juga ingin di-aggregasi

Angka                               →  integer, long, float
Waktu                               →  date
Boolean                             →  boolean
```
