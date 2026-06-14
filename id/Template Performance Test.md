# Rencana & Laporan Pengujian Performa

**Kontrol Dokumen**
- **Nama Proyek:** [Masukkan Nama Proyek - Harus sama dengan BRD/FSD]
- **Lingkungan Target:** [misal: Lingkungan Performance Testing / Pre-Prod]
- **Versi FSD Terkait:** [misal: 1.0 (Ref: Bagian 4 Kebutuhan Non-Fungsional)]
- **Jendela Pengujian:** [misal: 22 Juni 2026 - 24 Juni 2026]
- **Penulis Dokumen:** [Masukkan Nama/Jabatan, misal: Lead Performance Engineer]
- **Status:** [Direncanakan / Sedang Berjalan / Menganalisis / Persetujuan Akhir]

---

## 1. Pendahuluan
### 1.1 Tujuan
Tujuan dari dokumen ini adalah untuk menguraikan strategi, eksekusi, dan hasil pengujian performa untuk aplikasi dan database yang telah di-deploy. Dokumen ini memvalidasi bahwa sistem memenuhi target throughput (TPS/TPH), waktu respons (response time), dan penggunaan sumber daya (resource utilization) yang didefinisikan dalam FSD di bawah kondisi beban normal dan puncak (peak load).

### 1.2 Ruang Lingkup Pengujian
**Fungsi Bisnis Dalam Ruang Lingkup (In-Scope):**
- **Autentikasi Pengguna:** Login, validasi MFA, penyegaran Token (Token refresh).
- **Transaksi Inti:** Pemrosesan pembayaran, pembuatan pesanan, pengambilan data.
- **Tugas Asinkron:** Pembuatan Ekspor Data CSV (Background Worker).

**Di Luar Ruang Lingkup (Out-of-Scope):**
- Performa rendering UI pada perangkat mobile kelas bawah (dicakup oleh pengujian Frontend/UX).
- Batas pemrosesan internal payment gateway pihak ketiga (kita akan melakukan mock respons gateway untuk pengukuran TPS murni).

---

## 2. Tujuan Performa & SLA
> *Panduan: Definisikan target kuantitatif pasti yang harus dipenuhi sistem. Target ini diturunkan dari perencanaan kapasitas bisnis.*

| Metrik | Definisi | Target / SLA |
| :--- | :--- | :--- |
| **Throughput (TPS)** | Transactions Per Second / Transaksi Per Detik (Kondisi Stabil) | Minimal **500 TPS** untuk Core API. |
| **Throughput (TPH)** | Transactions Per Hour / Transaksi Per Jam (Puncak Harian) | Minimal **1.800.000 TPH** dipertahankan selama 4 jam. |
| **Response Time (Rata-rata)**| Waktu rata-rata untuk memproses satu transaksi | **< 200 ms** |
| **Response Time (Persentil ke-95)**| Waktu di mana 95% permintaan selesai diproses | **< 500 ms** |
| **Error Rate** | Persentase permintaan yang gagal (HTTP 5xx / Timeout) | **< 0.1%** di bawah beban puncak. |
| **Resource Utilization** | Penggunaan CPU dan Memory pada server App/DB | Rata-rata **< 75%**, Puncak **< 85%**. |

---

## 3. Skenario Uji & Model Beban Kerja
> *Panduan: Definisikan jenis pengujian dan campuran perjalanan pengguna (user journey mix).*

- **Load Test (Uji Beban):** Mensimulasikan traffic harian normal dan puncak yang diharapkan (500 TPS) dalam periode berkelanjutan (4 jam) untuk memverifikasi kepatuhan SLA.
- **Stress Test (Uji Tekanan):** Meningkatkan beban secara bertahap melampaui puncak (hingga 1000 TPS) untuk mengidentifikasi titik hancur (breaking point) sistem dan mengamati pemulihan dari kegagalan.
- **Spike Test (Uji Lonjakan):** Menyuntikkan lonjakan traffic besar secara tiba-tiba (misal: 2000 TPS selama 5 menit) untuk menguji pemicu auto-scaling dan ketahanan sistem.
- **Soak/Endurance Test (Uji Ketahanan):** Menjalankan sistem pada 80% kapasitas (400 TPS) selama 24-48 jam untuk mengidentifikasi kebocoran memori (memory leak) atau kehabisan connection pool database.

**Campuran Beban Kerja (Profil Load Test):**
- 40% Login/Autentikasi Pengguna
- 40% Transaksi Inti (Read/Write)
- 15% Pengambilan Data Dashboard
- 5% Pemicu Ekspor CSV (Asinkron)

---

## 4. Lingkungan & Alat Uji
- **Infrastruktur:** Harus memiliki rasio 1:1 (atau rasio skala yang jelas) dengan infrastruktur Produksi. 
  - *App Servers:* 3x t3.large (Auto-scaling diaktifkan)
  - *Database:* AWS RDS PostgreSQL (db.r5.xlarge, deployment Multi-AZ)
- **Alat yang Digunakan:**
  - **Load Generation (Pembangkit Beban):** [misal: JMeter, Gatling, k6, atau LoadRunner]
  - **Monitoring (APM):** [misal: Datadog, Dynatrace, New Relic]
  - **Infrastructure Monitoring:** [misal: AWS CloudWatch, Prometheus/Grafana]
  - **Database Monitoring:** [misal: pg_stat_statements, RDS Performance Insights]

---

## 5. Hasil Eksekusi & Metrik
> *Panduan: Data tabular yang membandingkan performa aktual dengan SLA yang ditetapkan selama pengujian Load dan Stress.*

| Skenario Uji | Target TPS | Peak TPS Aktual | Rata-rata Response Time | Response Time Persentil ke-95 | Error Rate | CPU (App/DB) | Status (Lulus/Gagal) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Load Test (4 Jam)** | 500 | 500 | 145 ms | 320 ms | 0.02% | 65% / 70% | **LULUS** |
| **Stress Test** | N/A (Cari Batas) | 750 | 850 ms | 2100 ms | 4.5% | 95% / 98% | **INFO** (Titik hancur teridentifikasi) |
| **Spike Test** | 2000 (Lonjakan)| 2000 | 1200 ms | 3500 ms | 1.2% | 90% / 85% | **GAGAL** (Pelanggaran SLA saat lonjakan) |
| **Soak Test (24 Jam)**| 400 | 400 | 150 ms -> 450 ms | 350 ms -> 1200 ms| 0.05% | 60% / 75% | **GAGAL** (Penurunan performa seiring waktu) |

---

## 6. Analisis Bottleneck & Saran untuk Performa yang Lebih Baik
> *Panduan: Berdasarkan kegagalan dan bottleneck yang diidentifikasi di Bagian 5, berikan rekomendasi teknis yang dapat ditindaklanjuti untuk mengoptimalkan sistem.*

| Bottleneck yang Teridentifikasi | Analisis Akar Masalah (Root Cause) | Saran untuk Performa yang Lebih Baik (Optimasi) | Prioritas |
| :--- | :--- | :--- | :--- |
| **Latensi Spike Test** | Auto-scaling group membutuhkan waktu 4 menit untuk memutar pod baru; permintaan menumpuk (queue) dan timeout selama fase warm-up. | 1. Terapkan **Predictive Auto-scaling** berdasarkan pola traffic historis.<br>2. Turunkan ambang batas pemicu auto-scale (misal: scale pada CPU 50% alih-alih 70%).<br>3. Pertahankan baseline pod standby "warm" yang lebih tinggi. | Tinggi |
| **Penurunan Soak Test** | Kehabisan connection pool database dan kebocoran memori ringan pada layanan Background Worker. | 1. Terapkan **Connection Pooling** (misal: PgBouncer) di depan instance RDS.<br>2. Investigasi dan perbaiki kebocoran memori di kode Worker (periksa stream/objek yang tidak ditutup).<br>3. Tetapkan `maxLifetime` yang ketat untuk koneksi DB. | Tinggi |
| **Latensi Read Dashboard** | SQL join yang kompleks pada tabel `transactions` menyebabkan CPU DB tinggi selama beban read-heavy. | 1. Tambahkan **Read Replicas** untuk database dan arahkan semua kueri GET/Dashboard ke replica.<br>2. Terapkan **Redis Caching** untuk data agregasi dashboard yang sering diakses (TTL: 5 menit). | Sedang |
| **Pemblokiran Ekspor CSV** | Pemrosesan ekspor besar secara sinkron memblokir thread API. | 1. Pastikan semua ekspor > 10 ribu baris secara ketat di-push ke **Message Queue** (misal: RabbitMQ/SQS) dan diproses secara asinkron oleh Background Worker. | Sedang |

---

## 7. Pertimbangan Teknis & Non-Teknis

### 7.1 Pertimbangan Teknis
- **Bandwidth Jaringan:** Pastikan tautan jaringan antara Load Generators dan Application Servers memiliki bandwidth yang cukup (misal: 10 Gbps) agar jaringan tidak menjadi bottleneck.
- **Volume Data Uji:** Database harus diisi dengan volume data yang menyerupai produksi (misal: 50 juta catatan transaksi) sebelum pengujian. Pengujian pada DB yang kosong menghasilkan query execution plan yang tidak akurat.
- **Mocking Dependensi Eksternal:** API pihak ketiga (Payment Gateway, SSO) harus di-mock menggunakan alat seperti WireMock untuk mengembalikan respons 200ms yang tetap. Kita menguji performa sistem *kita*, bukan vendor.
- **Overhead Monitoring:** Pastikan agen APM (Datadog/New Relic) dikonfigurasi dengan sampling rate sehingga alat monitoring itu sendiri tidak menurunkan performa aplikasi.

### 7.2 Pertimbangan Non-Teknis
- **Manajemen Biaya:** Pengujian performa di cloud bisa mahal. Pastikan batas auto-scaling ditetapkan, dan lingkungan non-produksi dihentikan (terminated) segera setelah jendela pengujian selesai.
- **Komunikasi Bisnis:** Beri tahu NOC/SOC dan pemangku kepentingan bisnis bahwa alert CPU/Jaringan yang tinggi akan dipicu selama jendela pengujian untuk mencegah eskalasi insiden palsu.
- **Privasi Data:** Jika data produksi disalin ke lingkungan performa, data tersebut HARUS di-mask/anonimkan sesuai dengan regulasi privasi data (GDPR/UU PDP).

---

## 8. Peran & Tanggung Jawab (PIC)
> *Panduan: Matriks kontak untuk eksekusi pengujian performa dan fase tuning.*

| Peran / Tanggung Jawab | Nama | Kontak / Chat | Tanggung Jawab selama Pengujian |
| :--- | :--- | :--- | :--- |
| **Lead Performance Engineer** | [Nama] | [Email / Slack] | Merancang skrip uji, mengeksekusi beban, menganalisis metrik APM, menulis laporan. |
| **DevOps / Cloud Lead** | [Nama] | [Email / Slack] | Memantau scaling infrastruktur, menyesuaikan tipe instance, mengelola biaya cloud. |
| **Lead Backend Developer** | [Nama] | [Email / Slack] | Menganalisis bottleneck tingkat kode, menerapkan profiling, menerapkan optimasi kode. |
| **Database Administrator (DBA)** | [Nama] | [Email / Slack] | Memantau lock DB, kueri lambat, connection pool; menerapkan tuning indeks/konfigurasi. |
| **QA / Test Automation Lead** | [Nama] | [Email / Slack] | Memvalidasi bahwa integritas fungsional tetap terjaga di bawah beban (tidak ada false positive). |
| **Product Owner** | [Nama] | [Email / Slack] | Meninjau pencapaian SLA, menyetujui penerimaan risiko untuk deviasi performa minor. |

---

## 9. Persetujuan & Clearance Akhir
*Dengan menandatangani di bawah ini, para pemangku kepentingan mengonfirmasi bahwa pengujian performa telah selesai, sistem memenuhi SLA TPS/TPH dan response time yang diperlukan (atau deviasi diterima secara formal), dan aplikasi telah di-clear untuk deployment Produksi.*

| Nama | Jabatan / Peran | Keputusan (Lulus / Lulus Bersyarat / Gagal) | Tanda Tangan | Tanggal |
| :--- | :--- | :--- | :--- | :--- |
| [Nama] | Lead Performance Engineer | [misal: Lulus Bersyarat] | ____________________ | [Tanggal] |
| [Nama] | Lead Backend / DBA Architect | [misal: Lulus Bersyarat] | ____________________ | [Tanggal] |
| [Nama] | DevOps / Infrastructure Lead | [misal: Lulus] | ____________________ | [Tanggal] |
| [Nama] | Product Owner / Business Sponsor| [misal: Lulus Bersyarat] | ____________________ | [Tanggal] |
