# OpenSearch Documentation & Technical Notes

Repositori ini berisi dokumentasi dan rangkuman teknis mengenai implementasi serta optimasi OpenSearch.

## Daftar Isi
- [Hot-Cold Architecture & Index Naming Convention](hot-and-cold.md)
- [Shard and Replica Management](shard-and-replica.md)

## Deskripsi Singkat
Dokumentasi di dalam repositori ini mencakup:
1. **Strategi Penyimpanan:** Penjelasan mengenai Hot-Warm-Cold architecture untuk optimasi biaya dan performa.
2. **Optimasi Query:** Pentingnya *Index Pruning* menggunakan *Naming Convention* (seperti `log_YYYYMMDD`) untuk mempercepat pencarian di layer aplikasi.
3. **Index State Management (ISM):** Bagaimana OpenSearch mengelola siklus hidup data secara otomatis.
4. **Sharding & Replicas:** Konsep dasar, strategi sizing (10GB-50GB per shard), dan cara kerja replikasi untuk fault tolerance serta performa query.

---
*Dibuat untuk referensi teknis implementasi OpenSearch.*
