# Denah Compliance UU PDP → Eksekusi Development & Operasional
### Platform Social Trading (MT5 Integration + Signal Sharing)

> **Catatan status hukum (per Juli 2026):** UU No. 27/2022 sudah berlaku penuh. Peraturan Pemerintah (PP) turunannya **masih dalam proses pengesahan** (belum diundangkan), demikian juga Badan Pengawas PDP. Ketiadaan PP **tidak menghapus kewajiban** yang sudah eksplisit di level UU — dokumen ini fokus ke kewajiban yang sudah mengikat sekarang, plus catatan area yang menunggu detail teknis dari PP nanti.

---

## Cara Membaca Dokumen Ini

| Kolom | Arti |
|---|---|
| **Tahap** | `DEV` = dikerjakan saat development, `PRE-LAUNCH` = sebelum go-live, `LIVE` = operasional berkelanjutan, `INCIDENT` = hanya saat insiden terjadi |
| **Prioritas** | 🔴 Wajib sebelum launch · 🟡 Wajib tapi bisa menyusul di Phase 2 · 🟢 Best practice, tidak wajib tapi disarankan |

---

## 1. Consent & Dasar Pemrosesan Data

| Pasal | Ketentuan Inti | Tahap | Langkah Teknis/Eksekusi | Prioritas |
|---|---|---|---|---|
| Pasal 20 ayat (2) huruf a | Dasar pemrosesan = persetujuan eksplisit dari user | DEV | Buat halaman onboarding dengan checkbox **terpisah per tujuan** (bukan satu checkbox "Setuju S&K") | 🔴 |
| Pasal 22 ayat (1)-(3) | Persetujuan wajib tertulis/terekam, berkekuatan hukum sama baik elektronik/nonelektronik | DEV | Simpan setiap consent ke tabel `user_consents` dengan kolom: `user_id`, `consent_type`, `version_kebijakan`, `timestamp`, `ip_address`, `status` (granted/revoked) | 🔴 |
| Pasal 22 ayat (4) | Jika consent memuat tujuan lain, harus dibedakan jelas, format mudah dipahami, bahasa sederhana | DEV | Pisahkan consent minimal jadi 3 jenis: (1) koneksi MT5 read-only, (2) tampil di dashboard pribadi, (3) share ke signal feed publik/PAMM | 🔴 |
| Pasal 22 ayat (5) | Consent yang tidak memenuhi syarat di atas **batal demi hukum** | PRE-LAUNCH | Legal/QA review teks consent sebelum launch — pastikan tidak ada bundling consent | 🔴 |
| Pasal 21 | Wajib sampaikan info: legalitas, tujuan, jenis data, retensi, durasi, hak subjek data — SEBELUM consent diminta | DEV | Buat halaman "Informasi Pemrosesan Data" yang wajib dibaca/scroll sebelum tombol consent aktif | 🔴 |
| Pasal 21 ayat (2) | Jika ada perubahan tujuan/kebijakan, wajib beritahu user **sebelum** perubahan berlaku | LIVE | Bangun sistem notifikasi in-app + email untuk perubahan kebijakan privasi, dengan re-consent flow jika perlu | 🟡 |
| Pasal 24 | Pengendali wajib bisa **menunjukkan bukti** consent yang diberikan | LIVE | Buat admin panel untuk export riwayat consent per user (untuk audit/permintaan regulator) | 🔴 |
| Pasal 9 | User berhak menarik kembali persetujuan kapan saja | DEV | Buat toggle di setting profil: "Cabut izin share ke signal feed" / "Putuskan koneksi MT5" | 🔴 |
| Pasal 40 | Setelah penarikan consent, pemrosesan wajib dihentikan **maks 3x24 jam** | DEV | Buat job/queue otomatis yang trigger stop-processing begitu status consent = revoked (bukan proses manual) | 🔴 |

---

## 2. Data Spesifik & Penilaian Risiko (DPIA)

| Pasal | Ketentuan Inti | Tahap | Langkah Teknis/Eksekusi | Prioritas |
|---|---|---|---|---|
| Pasal 4 ayat (2) huruf f | Data keuangan pribadi (saldo, P/L, posisi trading) = **Data Pribadi spesifik** | DEV | Tandai kolom terkait (balance, equity, open positions, transaction history) sebagai "sensitive field" di schema — beda level enkripsi & akses dari data umum (nama, email) | 🔴 |
| Pasal 34 ayat (1)-(2) | Wajib DPIA jika pemrosesan berisiko tinggi: data spesifik, skala besar, profiling otomatis, teknologi baru | PRE-LAUNCH | Susun dokumen DPIA internal sebelum launch: identifikasi risiko (kebocoran saldo, penyalahgunaan signal, dsb) + mitigasi. Tidak perlu format resmi (PP belum terbit) — cukup terdokumentasi & bisa ditunjukkan | 🔴 |
| Pasal 34 ayat (2) huruf d | Auto-signal generation (mendeteksi posisi baru otomatis) = "pemantauan sistematis" → risiko tinggi | DEV (Phase 2) | Saat bangun fitur auto-signal (Phase 2), masukkan ke DPIA sebagai item risiko baru sebelum fitur live | 🟡 |
| Pasal 4 ayat (3) huruf f | Data yang dikombinasikan untuk identifikasi (nomor telp, IP address) juga termasuk Data Pribadi | DEV | Pastikan IP address/device fingerprint yang disimpan untuk fraud detection juga masuk kebijakan retensi & consent yang sama | 🟡 |

---

## 3. Keamanan Teknis Data (Pasal 35, 38, 39)

| Pasal | Ketentuan Inti | Tahap | Langkah Teknis/Eksekusi | Prioritas |
|---|---|---|---|---|
| Pasal 35 huruf a | Wajib langkah teknis operasional melindungi data dari gangguan pemrosesan | DEV | Enkripsi **at-rest** untuk kolom sensitif (investor password MT5, data saldo) — gunakan AES-256 atau setara, bukan plaintext di database | 🔴 |
| Pasal 35 huruf b | Tingkat keamanan disesuaikan sifat & risiko data | DEV | Investor password MT5 = risiko tinggi → simpan di secrets manager (mis. HashiCorp Vault/AWS Secrets Manager), bukan kolom biasa di tabel `mt5_accounts` | 🔴 |
| Pasal 36 | Wajib jaga kerahasiaan data | DEV | Terapkan **role-based access control (RBAC)**: admin support tidak bisa lihat investor password mentah, hanya sistem yang bisa decrypt saat connect ke MT5 bridge | 🔴 |
| Pasal 37 | Wajib awasi semua pihak yang terlibat dalam pemrosesan | LIVE | Audit log setiap akses ke data sensitif (siapa akses data siapa, kapan) — termasuk akses dari tim internal/support | 🟡 |
| Pasal 38-39 | Wajib cegah akses tidak sah, proses via sistem elektronik "andal, aman, bertanggung jawab" | DEV | - TLS 1.2+ untuk semua komunikasi (Laravel API ↔ Python bridge ↔ MT5)<br>- MFA untuk akun admin/special user<br>- Rate limiting & WAF di API endpoint<br>- Vulnerability scanning berkala | 🔴 |
| Pasal 39 (bridge Windows VPS spesifik) | Sistem elektronik harus aman | DEV | VPS Windows tempat MT5 Python bridge jalan: firewall ketat (hanya port yang perlu terbuka), disable RDP publik, patch OS rutin | 🔴 |
| Pasal 31 | Wajib rekam seluruh kegiatan pemrosesan data (mirip ROPA - Record of Processing Activities) | PRE-LAUNCH | Buat dokumen internal ROPA: daftar semua tempat data pribadi diproses (DB tabel, log file, cache, third-party API) | 🟡 |

---

## 4. Hak Subjek Data & SLA Teknis

| Pasal | Ketentuan Inti | Tahap | Langkah Teknis/Eksekusi | Prioritas |
|---|---|---|---|---|
| Pasal 7 | User berhak akses & salinan data pribadinya | DEV | Endpoint `GET /api/user/my-data/export` — generate JSON/PDF berisi semua data pribadi user | 🟡 |
| Pasal 32 ayat (2) | Akses wajib diberikan maks **3x24 jam** sejak diminta | DEV | Jika belum bisa full-automated, minimal buat tiket otomatis ke admin dengan SLA countdown 72 jam | 🟡 |
| Pasal 6, 30 | User berhak koreksi data; wajib diperbaiki maks **3x24 jam** | DEV | Fitur edit profil self-service untuk data umum; untuk data yang butuh verifikasi, buat request queue dengan SLA 72 jam | 🟡 |
| Pasal 8, 43 | User berhak minta hapus data; wajib dihapus jika diminta | DEV | Endpoint "Delete Account" — hard delete atau anonymize data pribadi, sesuaikan dengan kewajiban retensi lain (misal data transaksi untuk audit keuangan) | 🔴 |
| Pasal 13 | User berhak dapat data dalam format yang bisa dibaca sistem lain (portabilitas) | DEV | Export format terstruktur (JSON/CSV), bukan screenshot atau PDF read-only saja | 🟢 |
| Pasal 14 | Pelaksanaan hak user diajukan via "permohonan tercatat" | DEV | Semua request (akses/hapus/koreksi) harus melalui sistem yang tercatat (bukan chat WhatsApp informal) — misal in-app request form + ticket ID | 🟡 |

---

## 5. Kegagalan Perlindungan Data (Data Breach)

| Pasal | Ketentuan Inti | Tahap | Langkah Teknis/Eksekusi | Prioritas |
|---|---|---|---|---|
| Pasal 46 ayat (1) | Wajib notifikasi tertulis maks **3x24 jam** ke user & lembaga (saat ini: Kemkomdigi, karena Badan PDP belum berdiri) | INCIDENT | Siapkan **incident response playbook**: template notifikasi, kontak Kemkomdigi untuk pelaporan, eskalasi internal | 🔴 |
| Pasal 46 ayat (2) | Notifikasi minimal memuat: data yang bocor, kapan/bagaimana bocor, upaya pemulihan | INCIDENT | Siapkan template pre-drafted supaya tidak menyusun dari nol saat panik — isi tinggal disesuaikan | 🔴 |
| Pasal 46 ayat (3) | Dalam hal tertentu wajib beritahu masyarakat luas | INCIDENT | Tentukan threshold internal: breach > berapa user → wajib pengumuman publik (blog/press release) | 🟡 |
| — | (praktik terbaik, tidak eksplisit di UU) | DEV | Setup monitoring/alerting (mis. anomaly detection untuk akses data massal, failed login spike) supaya breach terdeteksi cepat, bukan setelah 3 bulan | 🟢 |

---

## 6. Transfer Data ke Pihak Ketiga (Broker Lain, PAMM Manager, Signal Follower)

| Pasal | Ketentuan Inti | Tahap | Langkah Teknis/Eksekusi | Prioritas |
|---|---|---|---|---|
| Pasal 16 ayat (1) huruf e | Transfer/penyebarluasan data = bagian dari "pemrosesan" yang diatur UU | DEV | Setiap fitur baru yang mengirim data keluar sistem (share ke broker/PAMM/follower) → wajib cek ulang consent flow (lihat Bagian 1) | 🔴 |
| Pasal 55 | Transfer dalam wilayah hukum RI — kedua pihak (pengirim & penerima) wajib tetap terapkan perlindungan sesuai UU ini | DEV | Jika integrasi dengan broker lokal/PAMM lokal, pastikan ada **Data Processing Agreement (DPA)** — kontrak yang menyatakan mereka juga terapkan standar perlindungan yang sama | 🟡 |
| Pasal 56 ayat (2)-(4) | Transfer ke luar negeri — wajib pastikan negara tujuan setara/lebih tinggi, atau ada binding safeguard, atau consent eksplisit tambahan | PRE-LAUNCH | Dokumentasikan lokasi hosting semua mitra (broker asing, Pusher jika bukan self-hosted, dsb). Jika tidak yakin levelnya setara → minta consent eksplisit tambahan khusus untuk ini | 🔴 |
| Pasal 65 ayat (2) | Larangan mengungkapkan data melawan hukum — pidana hingga 4 tahun | DEV | **Anonimisasi/agregasi untuk signal feed publik**: pertimbangkan tampilkan username/alias, bukan nama asli, kecuali user secara eksplisit setuju identitas penuh ditampilkan | 🟡 |

---

## 7. Data Anak & Penyandang Disabilitas (jika relevan ke target user)

| Pasal | Ketentuan Inti | Tahap | Langkah Teknis/Eksekusi | Prioritas |
|---|---|---|---|---|
| Pasal 25 | Pemrosesan data anak wajib persetujuan orang tua/wali | DEV | Tambahkan age-gate di registrasi (platform trading umumnya sudah exclude di bawah 18, tapi pastikan eksplisit di T&C + validasi umur) | 🟢 |

---

## 8. Retensi & Penghapusan Data

| Pasal | Ketentuan Inti | Tahap | Langkah Teknis/Eksekusi | Prioritas |
|---|---|---|---|---|
| Pasal 16 ayat (2) huruf g | Data wajib dihapus/dimusnahkan setelah masa retensi berakhir, kecuali diatur lain oleh peraturan lain | DEV | Definisikan retention policy per jenis data: investor password (hapus segera saat disconnect), data transaksi (retensi sesuai kewajiban pajak/OJK jika ada), log akses (misal 1-2 tahun) | 🟡 |
| Pasal 44 | Wajib musnahkan data jika sudah habis masa retensi & tidak terkait proses hukum | LIVE | Buat scheduled job (cron) untuk purge data sesuai retention policy di atas | 🟡 |

---

## 9. Governance & Kelembagaan Internal

| Pasal | Ketentuan Inti | Tahap | Langkah Teknis/Eksekusi | Prioritas |
|---|---|---|---|---|
| Pasal 53 | Wajib tunjuk petugas Pelindungan Data Pribadi (DPO-like) jika: layanan publik, skala besar, atau data spesifik skala besar | PRE-LAUNCH | Untuk startup kecil, cukup tunjuk 1 orang (bisa founder/CTO di awal) sebagai penanggung jawab compliance data — formalkan saat tim bertambah besar | 🟡 |
| Pasal 54 | Tugas petugas: informasi & saran ke pengendali, pantau kepatuhan, DPIA, jadi narahubung | LIVE | Petugas ini yang bertanggung jawab update dokumen compliance ini secara berkala | 🟡 |

---

## Ringkasan Prioritas per Fase Roadmap Anda

### MVP (Portfolio Dashboard + Manual Signal Sharing)
🔴 **Wajib sebelum launch:**
- Consent granular (Bagian 1)
- Enkripsi investor password + secrets management (Bagian 3)
- DPIA dasar (Bagian 2)
- Delete account flow (Bagian 4)
- Incident response playbook (Bagian 5)
- Dokumentasi lokasi hosting untuk cek Pasal 56 (Bagian 6)

### Phase 2 (Auto-Signal + Regular User MT5 Connection)
🟡 **Tambahan sebelum Phase 2 live:**
- Update DPIA untuk cover auto-detection (profiling otomatis = risiko tinggi, Pasal 34 ayat 2 huruf a)
- Data access/koreksi self-service API (Bagian 4)
- Audit log akses data sensitif (Bagian 3)

### Phase 3 (Auto-Copy Trading)
🟡 **Evaluasi ulang DPIA** — auto-copy trading melibatkan eksekusi otomatis atas dasar data pihak lain, kemungkinan naik level risiko & butuh consent yang lebih spesifik lagi (bukan sekadar "lihat signal" tapi "eksekusi otomatis berdasarkan signal").

---

## Yang Perlu Dipantau ke Depan

RPP PDP dan Badan PDP masih dalam proses finalisasi (per info April-Mei 2026, tinggal tahap akhir pengesahan). Setelah terbit, kemungkinan ada:
- Format DPIA resmi/baku
- Parameter pasti kapan wajib menunjuk DPO formal
- Kanal pelaporan breach resmi ke Badan PDP (bukan lagi ke Kemkomdigi)

**Rekomendasi:** cek berkala ke `jdih.komdigi.go.id` atau `pdp.id` untuk update pengesahan, dan sesuaikan dokumen mapping ini begitu PP resmi terbit.
