# Rencana & Laporan Pengujian Keamanan (Pengujian Kerentanan & Penetrasi)

**Kontrol Dokumen**
- **Nama Proyek:** [Masukkan Nama Proyek - Harus sama dengan BRD/FSD]
- **Lingkungan Target:** [misal: Staging / Pre-Prod / Produksi]
- **Versi FSD Terkait:** [misal: 1.0 (Ref: Bagian 7 Kebutuhan Keamanan)]
- **Jendela Pengujian:** [misal: 15 Juni 2026 - 18 Juni 2026]
- **Penulis Dokumen:** [Masukkan Nama/Jabatan, misal: Lead Application Security Engineer]
- **Status:** [Direncanakan / Sedang Berjalan / Remediasi / Persetujuan Akhir]

---

## 1. Pendahuluan
### 1.1 Tujuan
Tujuan dari dokumen ini adalah untuk menguraikan strategi, eksekusi, dan temuan dari Penilaian Kerentanan (Vulnerability Assessment/VA) dan Pengujian Penetrasi (Penetration Testing/PT) untuk aplikasi dan database yang telah di-deploy. Hal ini memastikan bahwa semua kontrol keamanan yang didefinisikan dalam FSD telah diimplementasikan dengan benar dan bahwa sistem tangguh terhadap serangan berbahaya sebelum go-live ke produksi.

### 1.2 Ruang Lingkup Pengujian
**Aset Dalam Ruang Lingkup (In-Scope):**
- **Web UI (Frontend):** `https://staging.example.com` (Build React.js)
- **Core API (Backend):** `https://api.staging.example.com` (Microservice Node.js)
- **Background Worker:** Layanan pemrosesan asinkron (Jaringan internal)
- **Database:** PostgreSQL (Utama & Read-Replicas)
- **Integrasi:** Endpoint Payment Gateway & Corporate SSO (Okta) SAML yang di-mock.

**Di Luar Ruang Lingkup (Out-of-Scope):**
- Serangan Denial of Service (DoS/DDoS).
- Keamanan fisik pusat data/penyedia cloud (AWS/Azure).
- Infrastruktur penyedia SSO pihak ketiga (Okta).

---

## 2. Metodologi & Alat Pengujian
> *Panduan: Definisikan kerangka kerja dan alat yang digunakan untuk memastikan pengujian yang terstandarisasi dan dapat diulang.*

- **Kerangka Kerja (Frameworks):** 
  - OWASP Top 10 (Keamanan Aplikasi Web)
  - PTES (Penetration Testing Execution Standard)
  - CIS Benchmarks (untuk Database dan Infrastruktur Cloud)
- **Alat yang Digunakan:**
  - **DAST (Pengujian Dinamis):** Burp Suite Professional, OWASP ZAP
  - **SAST/Dependency:** SonarQube, Snyk, Trivy (Pemindaian container)
  - **Database/Jaringan:** Nmap, SQLMap, Nessus, pgAudit
  - **Pengujian API:** Postman, SwaggerSpy

---

## 3. Hasil Penilaian Kerentanan (VA)
> *Panduan: Hasil pemindaian otomatis dan tinjauan konfigurasi. Berfokus pada kerentanan yang diketahui, patch yang hilang, dan kesalahan konfigurasi pada aplikasi dan DB.*

| ID VA | Aset / Komponen | Nama Kerentanan | Tingkat Keparahan (CVSS) | Alat yang Digunakan | Deskripsi & Dampak | Status Remediasi |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **VA-01** | Docker Containers | Kerentanan Base Image | Tinggi (7.5) | Trivy | Base image Node.js mengandung CVE-2025-XXXX yang memungkinkan RCE. | **Terbuka** - DevOps untuk memperbarui base image ke `node:20-alpine`. |
| **VA-02** | PostgreSQL DB | Data Tidak Terenkripsi saat Diam (At Rest) | Sedang (5.3) | Nessus | Parameter enkripsi AWS RDS disetel ke `false` di Terraform. | **Diperbaiki** - Terraform diperbarui, snapshot DB dipulihkan dengan AES-256. |
| **VA-03** | Core API | Security Headers Hilang | Rendah (3.1) | OWASP ZAP | `X-Frame-Options` dan `Content-Security-Policy` tidak ada. | **Diperbaiki** - Ditambahkan ke konfigurasi Nginx/Ingress. |
| **VA-04** | S3 Bucket | Peran IAM Terlalu Longgar | Tinggi (8.1) | Prowler | Peran IAM Background Worker memiliki `s3:*` alih-alih akses bucket spesifik. | **Terbuka** - DevOps untuk membatasi kebijakan IAM ke hak istimewa paling minimal (least privilege). |

---

## 4. Hasil Pengujian Penetrasi (PT)
> *Panduan: Pengujian manual berbasis eksploitasi yang mensimulasikan penyerang dunia nyata. Berfokus pada logika bisnis, bypass autentikasi, dan rantai serangan yang kompleks.*

| ID PT | Ref FSD | Skenario / Vektor Serangan | Tingkat Eksploitasi | Dampak | Temuan & Bukti | Status Remediasi |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **PT-01** | FUNC-001 (Auth) | **Kebingungan Algoritma JWT & Kedaluwarsa**<br>Mencoba mengubah algoritma JWT menjadi `none` atau memanipulasi klaim kedaluwarsa. | Rendah | Tinggi | Sistem dengan benar menolak algoritma `none`. Namun, kedaluwarsa JWT disetel ke 24 jam, bukan 1 jam seperti yang diwajibkan FSD. | **Terbuka** - Dev untuk memperbarui konfigurasi signing JWT ke 3600 detik. |
| **PT-02** | FUNC-002 (Export) | **Insecure Direct Object Reference (IDOR) pada Unduhan CSV**<br>Mencegat tautan unduhan email dan mengubah `file_id=101` menjadi `file_id=102`. | Tinggi | Kritis | Penyerang berhasil mengunduh laporan keuangan PII pengguna lain dengan menebak ID file yang berurutan. | **Terbuka** - Dev untuk mengimplementasikan UUID untuk nama file dan menegakkan pemeriksaan kepemilikan di API. |
| **PT-03** | INT-001 (API) | **SQL Injection melalui Sinkronisasi CRM Lama**<br>Menyuntikkan payload berbahaya ke dalam payload SOAP XML yang disinkronkan dari CRM. | Sedang | Tinggi | WAF memblokir SQLi standar, tetapi blind SQLi order-by sekunder melewati filter dan mengekstrak versi DB. | **Terbuka** - Implementasikan parameterized queries di lapisan ingest SOAP. |
| **PT-04** | N/A (UI) | **Cross-Site Scripting (Stored XSS) di Profil Pengguna**<br>Menyuntikkan tag `<script>` ke dalam field "Nama Perusahaan". | Tinggi | Sedang | Payload dieksekusi ketika Admin melihat daftar pengguna, mencuri cookie sesi Admin. | **Terbuka** - Implementasikan pengkodean output kontekstual React dan DOMPurify. |

---

## 5. Pertimbangan Teknis & Non-Teknis

### 5.1 Pertimbangan Teknis (Aturan Keterlibatan / Rules of Engagement)
- **Privasi Data:** Pengujian HARUS dilakukan menggunakan data sintetis/mock. Tidak ada PII (Informasi Identifikasi Pribadi) produksi nyata yang boleh digunakan atau diekstraksi selama pengujian.
- **Pengujian Destruktif:** Perintah `DELETE` atau `DROP` sangat dilarang. Penguji harus menggunakan `SELECT` atau `UPDATE` untuk membuktikan dampak tanpa menghancurkan data.
- **Pembatasan Rate & WAF:** Web Application Firewall (WAF) dan pembatas rate API akan tetap **AKTIF** selama pengujian untuk mengevaluasi efektivitasnya. Penguji harus mendokumentasikan bypass WAF apa pun.
- **Snapshot Database:** Snapshot baru dari DB Staging harus diambil segera sebelum PT-03 (pengujian SQLi) untuk memungkinkan pemulihan cepat jika terjadi korupsi data.

### 5.2 Pertimbangan Non-Teknis
- **Otorisasi Legal (Surat Izin Resmi):** Semua penguji penetrasi harus membawa Surat Otorisasi (Letter of Authorization/LoA) yang ditandatangani oleh CISO dan departemen Hukum yang secara eksplisit mengizinkan kegiatan ini.
- **Notifikasi SOC / Monitoring:** Security Operations Center (SOC) dan tim monitoring DevOps (Datadog/Splunk) HARUS diberitahu tentang jendela pengujian. 
  - *Opsi A (Uji Buta / Blind Test):* SOC TIDAK diberitahu untuk menguji kemampuan respons insiden mereka.
  - *Opsi B (Uji Diumumkan / Announced Test):* SOC diberitahu untuk mencegah kepanikan positif-palsu dan eskalasi insiden yang tidak perlu. *(Rencana Saat Ini: Uji Diumumkan)*.
- **Koordinasi Pihak Ketiga:** Lingkungan sandbox Okta dan Payment Gateway harus dikosongkan untuk pengujian agresif agar IP perusahaan tidak masuk daftar hitam (blacklisted) oleh vendor.

---

## 6. Peran & Tanggung Jawab (PIC)
> *Panduan: Matriks kontak untuk jendela pengujian. Sangat penting untuk unblocking yang cepat dan manajemen insiden.*

| Peran / Tanggung Jawab | Nama | Kontak / Chat | Tanggung Jawab selama Pengujian |
| :--- | :--- | :--- | :--- |
| **Lead Penetration Tester** | [Nama] | [Email / Slack] | Mengeksekusi skenario PT, mendokumentasikan eksploit, menulis laporan akhir. |
| **AppSec Engineer (Internal)** | [Nama] | [Email / Slack] | Meninjau pemindaian VA, memilah false positive, membantu Dev dalam remediasi. |
| **Database Administrator (DBA)** | [Nama] | [Email / Slack] | Memantau performa DB selama pemindaian, memulihkan snapshot jika diperlukan. |
| **DevOps / Cloud Lead** | [Nama] | [Email / Slack] | Mengelola aturan WAF, menyediakan lingkungan uji, memperbaiki temuan VA Infra. |
| **SOC Lead (Monitoring)** | [Nama] | [Email / Slack] | Memantau SIEM untuk IP penguji, memvalidasi pembuatan alert. |
| **CISO / Security Sponsor** | [Nama] | [Email / Slack] | Otoritas akhir pada penerimaan risiko dan penandatanganan LoA. |

---

## 7. Rencana Remediasi & Penerimaan Risiko
> *Panduan: Bagaimana kerentanan akan ditangani pasca-pengujian.*

- **Keparahan Kritis & Tinggi:** Harus diremediasi dan diuji ulang **sebelum** Go-Live Produksi. Tidak ada pengecualian.
- **Keparahan Sedang:** Harus diremediasi dalam waktu 14 hari. Jika Go-Live mendesak, memerlukan Formulir Penerimaan Risiko formal yang ditandatangani oleh Product Owner dan CISO.
- **Rendah / Informasional:** Dicatat dalam backlog utang teknis (technical debt) untuk ditangani di sprint mendatang.

---

## 8. Persetujuan & Clearance Akhir
*Dengan menandatangani di bawah ini, para pemimpin keamanan dan bisnis mengonfirmasi bahwa aplikasi telah menjalani pengujian keamanan yang ketat, semua kerentanan Kritis/Tinggi telah dimitigasi, dan risiko sisa (residual risk) dapat diterima untuk deployment Produksi.*

| Nama | Jabatan / Peran | Keputusan (Clear / Bersyarat / Diblokir) | Tanda Tangan | Tanggal |
| :--- | :--- | :--- | :--- | :--- |
| [Nama] | Lead Penetration Tester | [misal: Clear] | ____________________ | [Tanggal] |
| [Nama] | Application Security Lead | [misal: Clear] | ____________________ | [Tanggal] |
| [Nama] | Product Owner (Penerima Risiko) | [misal: Clear] | ____________________ | [Tanggal] |
| [Nama] | CISO / InfoSec Director | [misal: Clear] | ____________________ | [Tanggal] |
