# Rencana & Laporan Eksekusi Uji Penerimaan Pengguna (SIT)

**Kontrol Dokumen**
- **Nama Proyek:** [Masukkan Nama Proyek - Harus sama dengan BRD/FSD]
- **Versi FSD Terkait:** [misal: 1.0]
- **Versi BRD Terkait:** [misal: 1.0]
- **Tanggal:** [Masukkan Tanggal, misal: 14 Juni 2026]
- **Versi:** [misal: 1.0]
- **Koordinator SIT / Penulis:** [Masukkan Nama/Jabatan]
- **Status:** [Draf / Sedang Berjalan / Selesai / Ditandatangani]

---

## 1. Pendahuluan
### 1.1 Tujuan
Tujuan dari dokumen ini adalah untuk menguraikan strategi, kasus uji, dan hasil eksekusi Uji Penerimaan Pengguna (SIT). Dokumen ini memvalidasi bahwa solusi yang dikembangkan memenuhi kebutuhan bisnis (BRD) dan spesifikasi fungsional (FSD) dari perspektif pengguna akhir sebelum deployment ke produksi.

### 1.2 Ruang Lingkup SIT
SIT ini mencakup alur kerja fungsional, antarmuka pengguna (UI), dan integrasi yang didefinisikan dalam FSD. SIT ini secara eksplisit **tidak mencakup** pengujian performa/beban (load testing) dan pengujian penetrasi keamanan teknis yang mendalam, yang mana telah dicakup dalam fase pengujian QA/InfoSec yang terpisah.

---

## 2. Prasyarat & Lingkungan
> *Panduan: Pastikan semua kondisi terpenuhi sebelum eksekusi SIT dimulai.*

- **Lingkungan Pengujian:** [misal: Lingkungan SIT (URL: https://SIT.example.com)]
- **Data Uji:** [misal: Dump data produksi yang telah dianonimkan dari 10 Juni 2026, termasuk 50 akun pengguna uji dan 100 sampel transaksi.]
- **Persyaratan Akses:** Penguji harus memiliki peran [misal: "Pengguna Standar" dan "Pengguna Admin"] yang telah disediakan di lingkungan SIT.
- **Browser/Perangkat yang Didukung:** [misal: Chrome v115+, Edge v115+, Safari v16+]

---

## 3. Ringkasan Eksekusi SIT
> *Panduan: Dasbor tingkat tinggi dari kemajuan pengujian. Perbarui tabel ini setiap hari atau mingguan.*

| Modul / Area Fitur | Total Kasus Uji | Lulus | Gagal | Terblokir | Belum Dijalankan | Tingkat Kelulusan (%) |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| Autentikasi & Otorisasi Pengguna | 5 | 5 | 0 | 0 | 0 | 100% |
| Ekspor Data & Pelaporan | 4 | 3 | 1 | 0 | 0 | 75% |
| [Tambah Modul] | [X] | [X] | [X] | [X] | [X] | [X]% |
| **TOTAL KESELURUHAN** | **[Total]** | **[Total]** | **[Total]** | **[Total]** | **[Total]** | **[Total %]** |

---

## 4. Detail Kasus Uji SIT (Data Tabular)
> *Panduan: Ini adalah tabel eksekusi inti. Setiap baris mewakili skenario uji spesifik yang dipetakan langsung ke FSD dan BRD. Perbarui "Hasil Aktual", "Status", dan "ID Defek" selama eksekusi.*

| ID Kasus Uji | Ref FSD | Ref BRD | Skenario Uji / Deskripsi | Prasyarat | Langkah Uji | Hasil yang Diharapkan | Hasil Aktual | Status (Lulus/Gagal/Terblokir) | Nama Penguji | Tanggal | ID Defek / Bug (jika Gagal) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **TC-001** | FUNC-001 | REQ-001 | Verifikasi login pengguna berhasil dengan kredensial dan MFA yang valid. | Akun pengguna ada dan aktif. MFA diaktifkan. | 1. Navigasi ke halaman login.<br>2. Masukkan username/password valid.<br>3. Masukkan kode MFA valid.<br>4. Klik "Login". | Pengguna diarahkan ke Dashboard. Token JWT yang valid dibSIT. | Pengguna berhasil diarahkan ke Dashboard. | **LULUS** | [Nama] | [Tanggal] | N/A |
| **TC-002** | FUNC-001 | REQ-001 | Verifikasi penguncian akun setelah beberapa kali percobaan login gagal. | Akun pengguna aktif dan tidak terkunci. | 1. Masukkan username valid tetapi password salah 5 kali berturut-turut. | Sistem menampilkan pesan "Login gagal" generik. Akun terkunci setelah percobaan ke-5. | Akun terkunci, tetapi pesan error menunjukkan "Password salah" secara spesifik, bukan pesan generik. | **GAGAL** | [Nama] | [Tanggal] | BUG-104 |
| **TC-003** | FUNC-002 | REQ-002 | Verifikasi ekspor CSV asinkron untuk dataset besar (>10 ribu baris). | Pengguna sudah login. Dataset dengan 15.000 baris tersedia. | 1. Terapkan filter tanggal.<br>2. Klik "Ekspor ke CSV".<br>3. Tunggu pemrosesan. | Sistem menampilkan spinner "Memproses". Pengguna menerima email dengan tautan unduhan dalam waktu 2 menit. | Email diterima dengan tautan CSV valid. Data sesuai dengan kriteria filter. | **LULUS** | [Nama] | [Tanggal] | N/A |
| **TC-004** | FUNC-00X | REQ-00X | [Tambahkan skenario uji spesifik Anda di sini] | [Prasyarat] | 1. Langkah 1<br>2. Langkah 2 | [Hasil yang diharapkan] | [Apa yang sebenarnya terjadi] | **[Status]** | [Nama] | [Tanggal] | [ID Bug] |

*(Tambahkan baris tambahan sesuai kebutuhan untuk semua item fungsional yang didefinisikan dalam FSD)*

---

## 5. Log Defek / Masalah
> *Panduan: Lacak semua kegagalan atau hambatan yang diidentifikasi selama SIT. Ini harus selaras dengan sistem pelacakan bug Anda (misal: Jira, Azure DevOps).*

| ID Defek | ID Kasus Uji Terkait | Tingkat Keparahan (Kritis/Tinggi/Sedang/Rendah) | Deskripsi Masalah | Ditugaskan Kepada | Status Saat Ini | Target Tanggal Perbaikan |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **BUG-104** | TC-002 | Sedang | Pesan error login mengungkapkan terlalu banyak informasi (menyatakan "Password salah" alih-alih pesan generik "Login gagal"). | Tim Dev | Terbuka / Dalam Proses | [Tanggal] |
| **BUG-105** | [TC-ID] | Tinggi | [Deskripsi bug lainnya] | [Nama] | [Status] | [Tanggal] |

---

## 6. Risiko yang Belum Selesai & Rekomendasi
- **Risiko:** [misal: BUG-105 memiliki tingkat keparahan Tinggi dan harus diselesaikan sebelum Go-Live. Pengujian performa pada fitur ekspor masih tertunda.]
- **Rekomendasi:** [misal: Lanjutkan ke keputusan Go/No-Go dengan syarat penyelesaian BUG-105 dan keberhasilan pengujian ulang TC-002.]

---

## 7. Persetujuan & Tanda Tangan SIT
*Dengan menandatangani di bawah ini, para pemangku kepentingan mengonfirmasi bahwa Uji Penerimaan Pengguna telah selesai, sistem memenuhi kebutuhan bisnis dan fungsional, serta solusi disetujui untuk deployment ke produksi (dengan syarat penyelesaian defek kritis yang telah disepakati, jika ada).*

| Nama | Jabatan / Peran | Keputusan (Setuju / Tolak / Setuju dengan Syarat) | Tanda Tangan | Tanggal |
| :--- | :--- | :--- | :--- | :--- |
| [Nama] | Business Owner / Product Owner | [misal: Setuju dengan Syarat] | ____________________ | [Tanggal] |
| [Nama] | SIT Lead / QA Manager | [misal: Setuju] | ____________________ | [Tanggal] |
| [Nama] | Project Sponsor | [misal: Setuju] | ____________________ | [Tanggal] |
