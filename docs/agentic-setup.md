# Setup Agentic AI Skorcast (POC)

> Ditulis saat implementasi pertama agent Python (LangGraph), 30 Aug 2026.
> Penjelasan: apa yang dibuat, KENAPA dipakai pendekatan itu, dan dampaknya.

## Yang ditambahkan
1. Folder `agent/` — Python service (FastAPI + LangGraph) yang **berjalan terpisah**
   dari Laravel (proses uvicorn sendiri di port 8000).
2. `agent/agent/` — isi graph: `state.py`, `nodes.py`, `graph.py`, `db.py`.
3. `agent/api.py` — endpoint `/chat`, `/approve`, `/health` + CORS + API key.
4. `public/agent-chat.html` — halaman testing chat (langsung fetch ke :8000).
5. `docs/` — folder dokumentasi ini.

## Kenapa struktur & teknologi ini dipilih

### FastAPI + uvicorn (bukan LangServe/WebSocket)
- **Kenapa**: Pola paling umum & ringan untuk expose LangGraph ke production.
  Cukup 2 endpoint: `/chat` (sinkron, balik draft) dan `/approve` (resume).
- **Kenapa bukan WebSocket**: alur kita request-response + jeda konfirmasi,
  bukan stream token dua arah. SSE/WebSocket bisa ditambah nanti kalau mau
  efek "mengetik", tapi tidak wajib untuk POC.
- **Dampak**: agent di-compile sekali di module level; tiap request pakai
  `thread_id` sebagai kunci checkpoint → percakapan antar admin terpisah.

### LangGraph StateGraph + `interrupt()` (human-in-the-loop)
- **Kenapa**: ini inti kebutuhan — admin MINTA konfirmasi sebelum insert.
  `interrupt()` menghentikan eksekusi graph setelah node `make_draft`, menyimpan
  state ke checkpointer, dan menunggu keputusan dari luar (`/approve`) lewat
  `Command(resume=...)`.
- **Alur**: `START → parse → make_draft → wait_confirm(interrupt) → insert → END`.
- **Dampak**: tidak ada dobel insert walau agent restart, karena state persisten.

### PostgresSaver sebagai checkpointer
- **Kenapa**: `MemorySaver` (default dev) tidak persisten & bocor antar user
  (dua admin barengan bisa lihat state satu sama lain). Karena skorcast sudah
  pakai Postgres, kita pakai DB yang sama. `.setup()` otomatis bikin tabel
  `checkpoints` saat startup.
- **Dampak**: aman untuk multi-user; state bertahan walau service di-restart.

### Parse tanpa LLM (regex sederhana) di POC
- **Kenapa**: agar bisa jalan & kamu bisa pelajari alur TANPA butuh API key LLM.
  `nodes.parse`/`make_draft` ekstrak field (nama, format, jumlah, sistem skor)
  dari teks via regex.
- **Dampak**: saat mau pakai LLM sungguhan, cukup ganti `parse` dengan pemanggilan
  model (OpenAI-compatible) — graph & interrupt tetap sama.

### Akses DB langsung dari Python (psycopg)
- **Kenapa**: agent tulis langsung ke tabel `tournaments` (sama seperti Laravel).
  Lebih sederhana daripada lewat API Laravel untuk POC.
- **Catatan**: kolom `code` wajib NOT NULL → node `insert` generate kode 6-char
  acak (seperti `LN67LN`).

### File HTML testing terpisah
- **Kenapa**: kamu mau testing lewat browser tanpa harus bangun panel Laravel dulu.
  HTML fetch langsung ke `:8000` (CORS diaktifkan). Nanti panel chatbot resmi
  akan ada di dalam Laravel (`AgentService`) sebagai jalur produksi.
- **Dampak**: feedback cepat; bisa buka `http://skorcast.online/agent-chat.html`.

## Cara menjalankan (dev)
```bash
cd agent
uv venv && uv pip install -r requirements.txt   # atau copy .venv yg sudah ada
cp .env.example .env   # isi DATABASE_URL & AGENT_API_KEY
bash run.sh            # jalanin uvicorn di 127.0.0.1:8000
```
Buka `public/agent-chat.html` di browser → ketik permintaan → klik Setuju/Batal.

## Verifikasi (sudah diuji)
- `POST /chat` "Buat turnamen Tunggal 8 orang 3x15" → draft + needs_confirmation.
- `POST /approve` decision=approve → insert ke Postgres (id=3, code=LN67LN).
- `POST /approve` decision=reject → "Tidak ada data yang ditulis" (aman).

## Yang belum / next step
- `AgentService.php` di Laravel + panel chatbot resmi (jalur produksi).
- Ganti `parse` dengan LLM sungguhan (butuh OPENAI_API_KEY).
- Insert peserta/teams/bracket otomatis (sekarang baru tournament header).
- Deploy agent sebagai systemd service (bukan `run.sh` manual).
- Ganti `CORS_ORIGINS=*` ke domain saat produksi + API key kuat.

## Tambahan: Nginx Reverse Proxy (agen diakses via domain, bukan 127.0.0.1)

**Masalah**: HTML testing dibuka lewat domain publik (skorcast.online), tapi JS fetch
ke `http://127.0.0.1:8000` -> "Failed to fetch". Penyebab: 127.0.0.1 merujuk ke
perangkat browser sendiri, bukan server.

**Solusi**: tambah `location /agent/` di nginx (`/etc/nginx/sites-available/skorcast.online`)
yang `proxy_pass` ke `http://127.0.0.1:8000/`. Maka:
- HTML fetch ke `window.location.origin + "/agent"` -> same-origin.
- Tidak perlu buka port 8000 ke publik / tidak butuh CORS ribet.
- Cloudflare di depan tetap menangani TLS (nginx listen 80, CF termination).

**Konfigurasi nginx (inject sebelum location /):**
```nginx
location /agent/ {
    proxy_pass http://127.0.0.1:8000/;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_read_timeout 60s;
}
```
Setelah itu `sudo nginx -t && sudo systemctl reload nginx`.

**Verifikasi**: `curl http://skorcast.online/agent/health` -> `{"status":"ok"}`.

**Catatan**: file nginx ada di luar repo git (/etc/nginx), jadi perubahan ini
didokumentasikan di sini, tidak ter-commit otomatis.
