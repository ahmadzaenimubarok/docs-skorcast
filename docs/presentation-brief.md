# Skor Cast — Presentation Brief

**Tujuan:** Brief ini dibagikan ke panitia, lalu diserahkan ke teman-teman, agar memberikan feedback yang *relevan* dengan arah pembangunan Skor Cast.

---

## 1. Latar Belakang & Kenapa Membuat

Skor Cast adalah aplikasi pencatat skor bulu tangkis **publik dan real-time**. Dibuat karena:

- **Masalah:** Pencatatan skor turnamen badminton masih manual (kertas / spreadsheet), tidak transparan, dan tidak bisa diakses penonton jarak jauh.
- **Kebutuhan nyata:** Turnamen butuh sistem yang menampilkan skor secara live, bisa diakses siapa saja (penonton, wasit, admin), tanpa perlu login untuk melihat skor.
- **Produk publik-facing:** Skor Cast bukan untuk diri sendiri — ditujukan untuk komunitas badminton secara luas. Keputusan desain, branding, dan server selalu diarahkan untuk mendukung audiens publik.

---

## 2. Kenapa Stack yang Sekarang

| Layer | Pilihan | Alasan |
|-------|---------|--------|
| Backend | **Laravel 13** | Ekosistem PHP dewasa, routing, queue, artisan — cocok untuk CRUD turnamen & skor |
| Real-time UI | **Livewire 4.3.3** | Real-time tanpa JavaScript manual — polling otomatis, state sinkron antara admin & penonton |
| Styling | **Tailwind 4** | Utility-first, desain konsisten, cepat iterasi UI |
| Interaktivitas | **Alpine.js** | Component ringan di blade (toggle, timer, queue) — tidak perlu build step terpisah |
| Server | **nginx + php-fpm 8.3** | Proven di produksi, ringan, Cloudflare di depan untuk TLS & caching |
| Database | **SQLite → Postgres** (migrasi berjalan) | SQLite cukup untuk tahap awal; Postgres dipilih untuk skalabilitas & reliability produksi |
| CDN/TLS | **Cloudflare** | TLS terminates di Cloudflare, nginx tinggal serve HTTP internal — sederhana |

**Prinsip pemilihan:** stack ini dipilih karena **paling efisien untuk menghadirkan produk publik yang real-time** dengan tim kecil. Bukan trend, tapi pragmatic — Laravel+Livewire menyelesaikan masalah real-time scoreboard tanpa JavaScript kompleks, dan Cloudflare mengurus TLS tanpa perlu certbot.

---

## 3. Fitur Inti yang Sudah Dibangun

### 3.1. Public Bracket & Scoreboard
- Live bracket yang diperbarui setiap 3 detik saat match ongoing
- Scoreboard standalone (`/s`) tanpa kode — siapa pun bisa akses
- Format scoring **parametrik**: 3×21, 3×15, 1-21 (BWF compliant)

### 3.2. Interval System
- Jeda otomatis ber-timer: mid-game (11 poin untuk 3×21) dan between-game (2 minit)
- Server-authoritative: countdown persisten walaupun halaman di-refresh
- **Sinkron admin ↔ publik:** ketika admin melompati jeda, penonton langsung melihat perubahan

### 3.3. Optimistic Scoreboard Update
- Tap skor → angka langsung jalan (tanpa tunggu server)
- Queue background + idempotency: tidak ada double-count walaupun koneksi lambat
- Offline-resilient: antrian tetap jalan, sinkron saat reconnect

### 3.4. Role System (Admin & Wasit)
- Admin: kontrol penuh (generate tim, input skor, atur turnamen)
- Wasit: input skor saja (tanpa akses ke fitur administratif)
- Login bersama (`/login`), redirect berdasarkan role

### 3.5. Blog Module
- Template blog publik (`/blog`) untuk berbagi artikel
- Setiap artikel mengundang pembaca pakai Skor Cast (content rule)

---

## 4. Inisiatif Agentic AI — Kenapa & Kenapa Sekarang

### 4.1. Visi
Skor Cast akan memiliki **lapisan AI** sehingga admin bisa membuat turnamen **via bahasa alami** — cukup ketik "buat turnamen bulu tangkis 3×15 dengan 16 peserta", dan sistem akan men-draft turnamen tersebut.

### 4.2. Arsitektur yang Dipilih
- **Agent = Python (LangGraph)**, berjalan sebagai proses terpisah (FastAPI + uvicorn)
- **Bukan bot — asisten:** agen *mendraft* rencana, **menunggu konfirmasi admin** sebelum menulis ke database (human-in-the-loop)
- **Frontend sementara:** `public/agent-chat.html` untuk testing, produksi via Laravel bridge nanti

### 4.3. Alasan Opsi B (LLM di dalam agent node)
- Awalnya pakai regex tanpa LLM (POC tanpa API key)
- Dipilih Opsi B karena LLM bisa **memahami intent** dan mengompresi multi-turn percakapan, bukan sekadar parse pola
- Model: `inclusionai/ling-3.0-flash-fin:free` (OpenAI-compatible, gratis)
- Fallback graceful: kalau model tidak merespons, tampilkan pesan minta diulang — bukan error kosong

### 4.4. Yang Sudah Dibangun
- Graph: `START → agent → (tools) → agent → [interrupt: wait approve] → insert → finish → END`
- Sistem fallback: kalau `content:null` (free model sering return), agen tetap kasih pesan bermakna
- Prompt domain-bound: agen hanya paham topik Skor Cast/badminton

---

## 5. Fitur Depan — Roadmap

| Prioritas | Fitur | Status |
|-----------|-------|--------|
| **Tinggi** | BWF 3×15 parametric scoring | ✅ **Built** (migration + model + UI + `/s`) |
| **Tinggi** | Interval countdown (3×21) | ✅ **Built** (jeda pertengahan + between-game) |
| **Tinggi** | Optimistic scoreboard update | ✅ **Built** (queue + idempotency + offline) |
| **Tinggi** | Agentic AI — draft turnamen via bahasa alami | ✅ **Built** (LangGraph + FastAPI, human-in-the-loop) |
| **Sedang** | Postgres migration (dari SQLite) | 🔄 Sedang berjalan |
| **Sedang** | Admin chatbot panel (Livewire) | 📋 Rencana — produksi via Laravel bridge |
| **Rendah** | Sistem klasemen / standings | 📋 Rencana |
| **Rendah** | Notifikasi push ke penonton | 📋 Rencana |
| **Rendah** | Multi-tournament management | 📋 Rencana |

---

## 6. Area yang Butuh Feedback

Ini bagian yang paling penting — berikut **5 pertanyaan** yang kami minta masukkan ke teman-teman:

### 6.1. Desain & UX
- Apakah layout skorboard sudah cukup *readable* dari jauh (ponsel, projector, layar besar)?
- Ukuran tombol & interaksi sudah sesuai untuk wasit yang sedang bertanding?

### 6.2. Real-time & Performa
- Apakah polling 3 detik terasa cukup cepat, atau masih ada *lag* yang terasa?
- Apakah ada skenario di mana skor tidak sinkron antara admin dan penonton?

### 6.3. Stack & Teknis
- Apakah memilih Laravel+Livewire (daripada Next.js/React atau Flutter) sudah tepat untuk produk publik?
- Apakah migration ke Postgres akan berjalan mulus di lingkungan produksi?

### 6.4. Agentic AI
- Apakah konsep "draft dulu, konfirmasi baru tulis" terasa tepat, atau terlalu hati-hati dan memperlambat proses?
- Apakah fitur "buat turnamen lewat bahasa alami" punya nilai bagi pengguna turnamen?

### 6.5. Produk Publik
- Apakah Skor Cast punya *unique value* yang cukup untuk bersaing dengan skorboard turnamen yang sudah ada?
- Apakah audiens yang tepat: klub lokal, panitia turnamen, atau komunitas online?

---

## 7. Catatan Teknis untuk Reviewer

- **Server:** nginx + php-fpm 8.3, Cloudflare TLS, domain `skorcast.online`
- **Repo:** `/var/www/skorcast.online` (www-data owned)
- **Doc repo terpisah:** `/var/www/docs-skorcast` (MkDocs Material, github-ahmad)
- **Testing:** PHPUnit (48 test pass), browser verification via HP
- **No push tanpa konfirmasi:** setiap perubahan produksi diverifikasi dulu via curl + HP sebelum `git push`

---

*Brief ini akan disempurnakan setelah feedback masuk dari teman-teman.*
