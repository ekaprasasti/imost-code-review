# Checklist Review Source Code iMOST (On-Site di PLN NR)

**Tujuan:** Menilai kerapian kode, pola arsitektur, dan struktur DB sebagai **indikator risiko awal** untuk meneruskan codebase vendor sebelumnya.
**Batasan waktu:** Target 30 menit, maksimal 1 jam.
**Disclaimer (sudah disampaikan ke management):** Ini *skimming* pola & struktur, **bukan** debug dan **bukan** scoring kerapian setara standar Badr (SonarQube). Hasilnya = sinyal risiko, bukan jaminan kualitas.

> **Strategi inti:** Jangan browsing semua folder dangkal. **Telusuri 1 slice vertikal** (1 fitur dari route sampai DB) + sampling 2–3 area berisiko. Lebih cepat dan lebih jujur.

---

## 0. Persiapan Sebelum Berangkat (biar nggak buang waktu nanya "ini di mana")

- [ ] Minta tim PLN NR / vendor lama **menyiapkan repo sudah ter-clone & bisa di-search** (IDE / grep)
- [ ] Minta **akses lihat DB** (Prisma Studio / pgAdmin / DBeaver) atau minimal `schema.prisma` terbuka
- [ ] **Pastikan kebijakan foto/NDA** — boleh foto layar atau tidak? (Tentukan strategi output di bawah berdasarkan ini)
- [ ] Idealnya **didampingi developer yang paham repo** supaya navigasi cepat
- [ ] Tentukan slice yang akan ditelusuri di depan: **Approval MoM** (paling bernilai) + 1 CRUD biasa (mis. ESG / Pendanaan)

---

## A. Higiene Repo & Project (±5 menit — sinyal cepat)

- [ ] README ada & bermakna? Ada instruksi setup?
- [ ] `.env.example` ada? Secret **tidak** ter-commit ke git?
- [ ] Git: kualitas commit history, strategi branch, jumlah kontributor, **commit terakhir kapan**, `.gitignore` waras?
- [ ] `package.json`: versi dependency di-pin? Ada dep aneh/abandoned? Node version? Lockfile ada?
- [ ] Struktur folder sesuai dokumentasi? (`index.ts → api → routes → middleware → controllers → validator`) — konsisten?
- [ ] TypeScript benar-benar dipakai atau `any` di mana-mana? `tsconfig` strict?
- [ ] Config lint/format (ESLint/Prettier) ada & ditegakkan?
- [ ] **Ada test sama sekali?** (Kemungkinan besar tidak — catat sebagai temuan)

**Bagus:** README jelas, env terkelola, commit rapi, TS strict.
**Red flag:** Secret ter-commit, `any` di mana-mana, tidak ada lockfile, commit "fix fix fix final2".

---

## B. Arsitektur & Pola — Telusuri 1 Slice Vertikal (±10 menit)

Ambil fitur **Approval MoM**. Telusuri: `route → middleware → controller → logic/service → Prisma → tabel`.

- [ ] Pemisahan tanggung jawab jelas (controller tipis, logic di service) atau **fat controller** dengan query DB inline?
- [ ] **Di mana otorisasi ditegakkan?** Di middleware/terpusat, atau ad-hoc per-endpoint? **Konsisten?**
      *(Ini langsung menjelaskan bug approval MoM — approval-3 bisa lihat/edit BA milik level 1–2)*
- [ ] Error handling: ada centralized error handler atau `try/catch` berserakan tidak konsisten?
- [ ] Validasi input: layer validator dipakai di **semua** route atau selektif?
- [ ] Konsistensi penamaan, panjang fungsi, kode mati / komentar nyampah, magic number/string
- [ ] **Duplikasi kode:** logic yang sama di-copy-paste antar modul?

**Bagus:** Cek ownership/role di server tiap endpoint terproteksi, struktur berlapis konsisten.
**Red flag:** Otorisasi cuma di frontend (hide tombol), tiap modul punya gaya sendiri, copy-paste masif.

---

## C. Database & Prisma (±8 menit — area paling bernilai)

Bandingkan dengan `migration.sql` yang sudah kita punya. Buka `schema.prisma`.

- [ ] **FK constraint benar-benar didefinisikan** atau cuma kolom `integer` lepas? (banyak `project_id` nullable — cek integritas referensial DB vs aplikasi)
- [ ] Asal-usul skema: tampak **eks-Laravel** (`guard_name`, `sessions`, `cache`, `remember_token`). **Tanya:** Prisma hasil *introspect* DB lama atau dikelola Prisma Migrate? → menentukan apakah DB bisa direproduksi saat handover
- [ ] **Migration history utuh** (Prisma Migrate) atau SQL manual? *(Risiko handover: bisakah kita rebuild DB dari nol?)*
- [ ] Index pada FK / kolom yang sering di-query?
- [ ] Pola nullable di mana-mana? Soft delete? Audit trail (`audit_logs` sudah ada — dipakai konsisten?)
- [ ] Risiko N+1: query pakai `include`/`select` wajar atau looping query?
- [ ] Penggunaan ENUM berat (sudah terlihat) — catat: nambah value butuh migration

**Bagus:** FK & index lengkap, migration history bersih, relasi Prisma rapi.
**Red flag:** FK cuma di level app, tidak ada migration history, naming DB↔Prisma berantakan (`@map` ad-hoc).

---

## D. Security / RBAC Cepat (±5 menit — terkait BIT ITD & pen-test)

- [ ] RBAC ditegakkan **server-side tiap endpoint** atau cuma sembunyi-tampil di frontend?
- [ ] JWT: secret di env? Ada expiry/refresh? Disimpan di mana di frontend?
- [ ] Password hashing pakai bcrypt/argon? (kolom `users.password` — cek algoritmanya)
- [ ] Ada `$queryRaw` dengan string interpolation (risiko SQL injection)?
- [ ] File upload: middleware "File Validator" benar-benar memvalidasi tipe/ukuran?
- [ ] Bug approval MoM: **sistemik** (tidak ada cek ownership/row-level) atau terisolasi?

**Red flag:** Otorisasi hanya di FE, raw query dengan interpolasi, password hashing lemah/plaintext.

---

## E. Kesiapan Operasional & Handover (±3 menit)

- [ ] Dockerfile / docker-compose ada & up-to-date?
- [ ] Config CI/CD ada (Vercel untuk FE, sesuatu untuk BE)?
- [ ] Pengelolaan env/secret/config jelas?
- [ ] Dokumentasi lain selain 1 PDF? API docs (Swagger)?
- [ ] Bisa di-clone & jalan, atau bergantung *tribal knowledge*? (tanya lead dev)

---

## Skema Penilaian (per bagian A–E)

Beri label cepat, **bukan angka presisi**:

| Label | Arti |
|---|---|
| 🟢 Rapi | Sesuai praktik baik, risiko rendah |
| 🟡 Cukup | Bisa diteruskan, perlu sedikit penyesuaian |
| 🟠 Perlu Perhatian | Ada utang teknis, perlu buffer effort |
| 🔴 Red Flag | Risiko tinggi, perlu mitigasi/diskusi komersial |

---

## OUTPUT YANG DIBAWA PULANG

### 1. Checklist terisi + label per bagian
Isi A–E di atas dengan 🟢/🟡/🟠/🔴 + 1–2 kalimat alasan. Beri header: *"Indikator risiko awal berbasis skimming ~30 menit, bukan audit penuh."*

### 2. Bukti / catatan (sesuaikan dengan kebijakan foto)
- **Jika boleh foto:** ambil 5–7 artefak kunci → `schema.prisma`, folder tree, 1 controller/service representatif, `package.json`, middleware otorisasi.
- **Jika tidak boleh foto:** catat manual struktur + tulis ulang contoh verbatim singkat (nama pola/fungsi, bukan seluruh kode).

### 3. Risk Register & Syarat Handover (output paling penting)
Hal yang **tidak bisa diverifikasi penuh dalam 30 menit** → harus diamankan secara kontrak/teknis saat handover:

| # | Item | Status saat dilihat | Yang harus dipastikan saat handover |
|---|---|---|---|
| 1 | Source code lengkap via Github | (sudah di brief) | Akses repo penuh + history |
| 2 | Reproduksibilitas DB (migration history) | ? | Migration Prisma utuh / dump skema |
| 3 | Daftar env/secret/config | ? | Dokumen env lengkap |
| 4 | Area tanpa dokumentasi (tribal knowledge) | ? | Sesi knowledge transfer |
| 5 | Otorisasi RBAC server-side | ? | Konfirmasi sebelum pen-test |

### 4. Dampak ke Estimasi (mandays)
Topan sudah setuju **bug-fix masuk hitungan MD dan plus-minus = bagian risiko**. Output kamu cukup menyatakan **postur risiko**, contoh:
- *"Struktur cukup rapi → buffer bug-fix di MD eksisting memadai."* atau
- *"Banyak red flag (otorisasi ad-hoc, tanpa migration history) → rekomendasikan kontingensi MD / audit kode berbayar sebagai item discovery terpisah."*

### 5. Rekomendasi singkat (1 paragraf)
Postur risiko + tindak lanjut, dengan **disclaimer eksplisit** bahwa 30 menit skimming bukan jaminan kualitas.

---

*Tip akhir: tetapkan ekspektasi di awal kunjungan (kebijakan foto/NDA, siapa yang mendampingi). Kalau kamu telusuri slice Approval MoM dengan benar, sebagian besar penilaian A–E akan terjawab dari satu alur itu saja.*
