# Rencana Implementasi & Deployment

**Kontrol Dokumen**
- **Nama Proyek:** [Masukkan Nama Proyek - Harus sama dengan BRD/FSD]
- **Versi Rilis:** [misal: v1.0.0]
- **Versi FSD Terkait:** [misal: 1.0]
- **Lingkungan Target:** Produksi (Production)
- **Tanggal Deployment:** [Masukkan Tanggal & Waktu, misal: Sabtu, 20 Juni 2026, 00:00 - 04:00 WIB]
- **Penulis Dokumen:** [Masukkan Nama/Jabatan, misal: Release Manager / DevOps Lead]
- **Status:** [Draf / Disetujui / Dieksekusi]

---

## 1. Pendahuluan
### 1.1 Tujuan
Dokumen ini berfungsi sebagai runbook resmi untuk men-deploy Rilis v[1.0.0] ke lingkungan Produksi. Dokumen ini memberikan instruksi langkah demi langkah bagi tim Production Support dan DevOps untuk men-deploy komponen aplikasi dan database, mengelola integrasi, dan mengeksekusi rollback jika diperlukan.

### 1.2 Jendela Deployment (Deployment Window)
- **Waktu Mulai:** [misal: 2026-06-20 00:00 WIB]
- **Waktu Selesai:** [misal: 2026-06-20 04:00 WIB]
- **Perkiraan Downtime:** [misal: 45 menit untuk migrasi database; aplikasi akan berada dalam mode read-only/pemeliharaan]

---

## 2. Komponen & Artefak Perangkat Lunak yang akan di-Deploy
> *Panduan: Daftar pasti artefak, versi, dan konfigurasi yang akan di-push ke produksi.*

| Tipe Komponen | Nama Komponen | Versi / Tag | Lokasi Artefak / Repositori | Deskripsi |
| :--- | :--- | :--- | :--- | :--- |
| **Database** | Skrip Migrasi Skema | v1.0.0_DB | `db/migrations/prod/v1.0.0/` | Membuat tabel baru untuk fitur Ekspor Data; menambahkan indeks. |
| **Database** | Seed Data / Data Konfigurasi | v1.0.0 | `db/seed/prod/` | Data konfigurasi awal untuk parameter sistem baru. |
| **Aplikasi Backend** | Core API Service | `v1.0.0-prod` | `docker-registry.internal/core-api:v1.0.0` | Microservice backend Node.js. |
| **Aplikasi Backend** | Background Worker | `v1.0.0-prod` | `docker-registry.internal/worker:v1.0.0` | Pemroses pekerjaan asinkron untuk ekspor CSV. |
| **Aplikasi Frontend** | Web UI | `v1.0.0-prod` | `s3://prod-static-assets/web-ui/v1.0.0/` | Build frontend React.js. |
| **Infrastruktur** | Skrip Terraform / IaC | `main-branch` | `infra/terraform/prod/` | Menyediakan bucket S3 baru dan memperbarui peran IAM. |
| **Konfigurasi** | App Config / Secrets | `prod-v1.0.0` | AWS Secrets Manager / HashiCorp Vault | Kunci API yang diperbarui untuk integrasi Payment Gateway. |

---

## 3. Aplikasi Terkait & Analisis Dampak
> *Panduan: Referensi ke Bagian 8 (Kebutuhan Integrasi) dari FSD. Identifikasi sistem upstream/downstream yang terpengaruh.*

| Sistem Terkait | Tipe Integrasi | Dampak Deployment | Tindakan yang Diperlukan (jika ada) |
| :--- | :--- | :--- | :--- |
| **Payment Gateway** | REST API (Outbound) | Endpoint API baru akan dipanggil untuk pemrosesan transaksi. | Pastikan kunci API Produksi diperbarui di Vault sebelum deployment. |
| **CRM Lama (Legacy)** | SFTP/SOAP (Inbound) | Field pelanggan baru akan disinkronkan. | Tim CRM harus memvalidasi pemetaan field baru di UAT sebelum rilis Prod. |
| **SSO Perusahaan (Okta)** | SAML 2.0 | Aplikasi baru terdaftar di Okta. | Admin Okta harus mengaktifkan integrasi aplikasi Prod dan menetapkan grup pengguna. |
| **Monitoring (Datadog)** | APM / Logs | Metrik kustom dan pola log baru diperkenalkan. | Tim Datadog untuk mengimpor JSON dashboard baru yang disediakan dalam paket rilis. |

---

## 4. Daftar Periksa Pra-Deployment (Pre-Deployment Checklist)
> *Panduan: Harus diselesaikan 24 jam sebelum jendela deployment.*

- [ ] **Persetujuan UAT (UAT Sign-off):** Laporan UAT akhir disetujui (Ref: Dokumen UAT v1.0).
- [ ] **Code Freeze:** Branch produksi dibekukan dan di-tag dengan `v1.0.0`.
- [ ] **Build Artefak:** Semua image Docker dan build frontend berhasil dikompilasi dan di-push ke registry Prod.
- [ ] **Backup:** Backup otomatis penuh dari Database Produksi dan bucket S3 telah diverifikasi.
- [ ] **Komunikasi:** Email pemeliharaan (maintenance) dikirim ke semua pengguna bisnis dan pemangku kepentingan.
- [ ] **Kesiapan Dukungan:** Helpdesk L1/L2 telah dibriefing tentang fitur baru dan masalah yang diketahui (known issues).

---

## 5. Instruksi Deployment Langkah demi Langkah
> *Panduan: Perintah dan langkah terperinci yang dapat dieksekusi untuk tim DevOps/Prod Support. Sertakan estimasi waktu untuk setiap langkah.*

### Fase 1: Infrastruktur & Database (Est. 30 menit)
| Langkah | Tindakan / Perintah | Output yang Diharapkan / Verifikasi | Penanggung Jawab |
| :--- | :--- | :--- | :--- |
| 1.1 | Aktifkan Mode Pemeliharaan pada Load Balancer. | Pengguna melihat halaman "Sistem Sedang Dalam Pemeliharaan". Traffic turun ke 0. | DevOps |
| 1.2 | Jalankan Infrastructure as Code (Terraform).<br>`terraform apply -var-file="prod.tfvars"` | Bucket S3 baru dibuat; peran IAM diperbarui. | DevOps |
| 1.3 | Eksekusi Migrasi Database.<br>`flyway migrate -url=jdbc:postgresql://prod-db...` | Skema diperbarui. Verifikasi tabel baru ada menggunakan DBeaver/pgAdmin. | DBA |
| 1.4 | Eksekusi Seed Data Database.<br>`psql -f seed_data.sql` | Baris konfigurasi berhasil dimasukkan. | DBA |

### Fase 2: Aplikasi Backend (Est. 20 menit)
| Langkah | Tindakan / Perintah | Output yang Diharapkan / Verifikasi | Penanggung Jawab |
| :--- | :--- | :--- | :--- |
| 2.1 | Perbarui Secrets di Vault/Secrets Manager. | Kunci API Payment Gateway baru sudah aktif. | SecOps |
| 2.2 | Deploy Background Worker.<br>`kubectl set image deployment/worker worker=docker-reg/worker:v1.0.0` | Pod restart dan masuk ke status `Running`. Periksa log untuk keberhasilan startup. | DevOps |
| 2.3 | Deploy Core API Service (Rolling Update).<br>`kubectl set image deployment/core-api core-api=docker-reg/core-api:v1.0.0` | Pod roll out tanpa memutus koneksi aktif (jika ada). Health check lulus. | DevOps |

### Fase 3: Aplikasi Frontend (Est. 10 menit)
| Langkah | Tindakan / Perintah | Output yang Diharapkan / Verifikasi | Penanggung Jawab |
| :--- | :--- | :--- | :--- |
| 3.1 | Sinkronisasi Aset Frontend ke S3.<br>`aws s3 sync ./build s3://prod-static-assets/web-ui/v1.0.0/` | File berhasil diunggah. | DevOps |
| 3.2 | Invalidasi Cache CloudFront.<br>`aws cloudfront create-invalidation --distribution-id EXXX --paths "/*"` | ID Invalidasi dikembalikan. | DevOps |

### Fase 4: Pasca-Deployment & Smoke Testing (Est. 20 menit)
| Langkah | Tindakan / Perintah | Output yang Diharapkan / Verifikasi | Penanggung Jawab |
| :--- | :--- | :--- | :--- |
| 4.1 | Nonaktifkan Mode Pemeliharaan pada Load Balancer. | Traffic dilanjutkan. | DevOps |
| 4.2 | Eksekusi Suite Automated Smoke Test.<br>`npm run test:e2e:prod` | Semua uji coba jalur kritis lulus. | QA |
| 4.3 | Verifikasi Manual: Login, Ekspor Data, Periksa Dashboard. | UI dimuat dengan benar; ekspor data berhasil. | QA / Prod Support |

---

## 6. Skenario Rollback
> *Panduan: Definisikan kriteria "Go/No-Go" yang pasti dan langkah-langkah pembalikan jika deployment gagal.*

### 6.1 Pemicu Rollback (Kriteria No-Go)
- Migrasi database gagal dan tidak dapat diselesaikan dalam waktu 15 menit.
- Core API gagal dalam health check setelah 3 kali percobaan restart.
- Critical Smoke Test (misal: Login Pengguna, Pemrosesan Pembayaran) gagal.
- **Waktu Keputusan:** Jika deployment belum selesai dan diverifikasi pada pukul [03:00 WIB], inisiasi rollback.

### 6.2 Langkah Eksekusi Rollback
| Langkah | Tindakan / Perintah | Output yang Diharapkan | Penanggung Jawab |
| :--- | :--- | :--- | :--- |
| **R1** | Aktifkan Mode Pemeliharaan pada Load Balancer. | Traffic diblokir. | DevOps |
| **R2** | Kembalikan Aplikasi Backend ke versi sebelumnya.<br>`kubectl rollout undo deployment/core-api`<br>`kubectl rollout undo deployment/worker` | Aplikasi kembali ke `v0.9.0`. Health check lulus. | DevOps |
| **R3** | Kembalikan Aset Frontend.<br>Arahkan ulang CloudFront ke versi S3 sebelumnya / Invalidasi cache. | UI lama disajikan. | DevOps |
| **R4** | **Rollback Database:** Kembalikan DB dari snapshot pra-deployment ATAU jalankan skrip migrasi balik (reverse).<br>`flyway undo` atau AWS RDS Point-in-Time Recovery. | Skema DB cocok dengan `v0.9.0`. Integritas data diverifikasi. | DBA |
| **R5** | Nonaktifkan Mode Pemeliharaan. | Sistem aktif kembali pada versi sebelumnya. | DevOps |
| **R6** | Beri tahu pemangku kepentingan tentang kegagalan deployment dan linimasa yang direvisi. | Komunikasi terkirim. | Release Mgr |

---

## 7. Pertimbangan Teknis & Non-Teknis

### 7.1 Pertimbangan Teknis
- **Caching:** Cache Redis harus di-flush pasca-deployment untuk mencegah masalah data usang (stale data) (`redis-cli FLUSHDB`).
- **DNS/CDN:** Pastikan TTL untuk record DNS diturunkan ke 300 detik 24 jam sebelum deployment untuk memungkinkan propagasi cepat jika perubahan routing diperlukan.
- **Feature Toggles:** Fitur "Ekspor Data" baru dibungkus dalam feature flag (`ENABLE_EXPORT_V2`). Fitur ini akan di-deploy dalam status `disabled` (nonaktif) dan diaktifkan melalui UI Admin pasca-deployment.

### 7.2 Pertimbangan Non-Teknis
- **Komunikasi Pengguna:** Spanduk (banner) akan ditempatkan di halaman login 24 jam sebelumnya yang mengumumkan jendela pemeliharaan.
- **Kesiapan Helpdesk:** Dukungan L1 harus memiliki dokumen FAQ yang diperbarui (v1.1) mengenai fitur Ekspor Data baru.
- **Pembekuan Bisnis (Business Freeze):** Tim Keuangan dan Operasi diberitahu untuk tidak menjalankan pekerjaan batch akhir bulan selama jendela deployment.

---

## 8. Daftar Kontak (Person In Charge - PIC)
> *Panduan: Matriks eskalasi untuk jendela deployment. Semua PIC harus tersedia di bridge call yang ditunjuk.*

| Peran / Tanggung Jawab | Nama | Telepon / Kontak | Tautan Bridge / Chat |
| :--- | :--- | :--- | :--- |
| **Release Manager (Ketua Keseluruhan)** | [Nama] | [+62-812-XXXX-XXXX] | [Tautan Zoom/Teams] |
| **DevOps / Infrastructure Lead** | [Nama] | [+62-812-XXXX-XXXX] | [Tautan Zoom/Teams] |
| **Database Administrator (DBA)** | [Nama] | [+62-812-XXXX-XXXX] | [Tautan Zoom/Teams] |
| **Lead Backend Developer** | [Nama] | [+62-812-XXXX-XXXX] | [Tautan Zoom/Teams] |
| **Lead Frontend Developer** | [Nama] | [+62-812-XXXX-XXXX] | [Tautan Zoom/Teams] |
| **QA / Smoke Test Lead** | [Nama] | [+62-812-XXXX-XXXX] | [Tautan Zoom/Teams] |
| **Business / Product Owner** | [Nama] | [+62-812-XXXX-XXXX] | [Tautan Zoom/Teams] |
| **InfoSec / Perwakilan Keamanan** | [Nama] | [+62-812-XXXX-XXXX] | [Tautan Zoom/Teams] |

---

## 9. Persetujuan Deployment
*Dengan menandatangani di bawah ini, masing-masing pemimpin mengonfirmasi bahwa area spesifik mereka siap untuk deployment dan menyetujui eksekusi rencana ini.*

| Peran | Nama | Tanda Tangan | Tanggal & Waktu |
| :--- | :--- | :--- | :--- |
| Release Manager | [Nama] | ____________________ | [Tanggal/Waktu] |
| DevOps Lead | [Nama] | ____________________ | [Tanggal/Waktu] |
| DBA Lead | [Nama] | ____________________ | [Tanggal/Waktu] |
| Product Owner | [Nama] | ____________________ | [Tanggal/Waktu] |
