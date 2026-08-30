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

## Tambahan: Parse pakai LLM (OpenRouter, model free)

**Konteks**: POC awal `parse` pakai regex supaya jalan tanpa API key. Setelah alur
human-in-the-loop terbukti, kita sambungkan LLM agar admin bisa mengetik bebas
(bahasa natural) bukan format kaku.

**Kenapa OpenRouter + `z-ai/glm-5.2:free`?**
- OpenRouter itu **OpenAI-compatible** (endpoint `https://openrouter.ai/api/v1`,
  format request sama dengan OpenAI). Cukup ganti `base_url` + `api_key`, tidak
  ubah graph/interrupt sama sekali.
- Satu key akses banyak model, termasuk tier **gratis** (`z-ai/glm-5.2:free`)
  cocok untuk development tanpa biaya. Bisa ganti model kapan saja via env
  `AGENT_MODEL` (mis. `tencent/hy3`, `deepseek/deepseek-v4-flash-latest`).
- Key sudah tersedia di Hermes (`OPENROUTER_API_KEY`), dipakai ulang di agent.

**Implementasi**:
- File baru `agent/agent/llm.py`: `extract_tournament(text)` panggil chat completion,
  minta model balik JSON field turnamen. Bersihkan code fence bila ada.
- `nodes.parse` prioritas LLM; **fallback ke regex** (`_parse_intent`) bila key
  kosong / request gagal → graph tetap jalan tanpa API key.
- Dep `openai` ditambah ke venv.

**Env agent (`.env`)**:
```
OPENROUTER_API_KEY=sk-or-...
OPENAI_BASE_URL=https://openrouter.ai/api/v1
AGENT_MODEL=z-ai/glm-5.2:free
```
(NB: key diambil dari Hermes, disalin ke agent/.env. Jangan commit .env.)

**Verifikasi**: chat "turnamen beregu ganda 24 pemain 3x15" → LLM ekstrak
play_mode=doubles, count=24, points_to_win=15 → approve → insert id=6.

**Dampak**: admin lebih leluasa mengetik; ekstraksi lebih robust dari regex.
Risiko: model free bisa rate-limit/throttle — fallback regex menjamin tetap jalan.

## Pergantian provider: OpenRouter -> OpenCode Zen

**Alasan**: model free OpenRouter (`z-ai/glm-5.2:free`) sering kena rate-limit 429
dari sisi upstream (is_byok=false), sehingga `extract_tournament` gagal & agent
selalu jatuh ke fallback regex (draft statis). User memutuskan pakai **OpenCode Zen**
sebagai provider LLM.

**Kenapa OpenCode Zen?**
- Provider LLM kurasi (GPT/Claude/Gemini/GLM/Kimi), **OpenAI-compatible** -> cukup
  ganti base_url + api_key, tidak ubah graph/interrupt.
- Key `OPENCODE_ZEN_API_KEY` sudah tersedia di Hermes, dipakai ulang di agent.
- Base URL: `https://opencode.ai/zen/v1`.
- Model dev: `glm-5.2` (gratis di Zen, cukup untuk ekstraksi JSON turnamen).
  Alternatif: `kimi-k2.5`, atau model berbayar (claude/gpt) kalau butuh kualitas.

**Perubahan**:
- `.env` agent: `OPENROUTER_API_KEY` -> `OPENCODE_ZEN_API_KEY`, `OPENAI_BASE_URL`
  -> `https://opencode.ai/zen/v1`, `AGENT_MODEL=glm-5.2`.
- `llm.py`: baca `OPENCODE_ZEN_API_KEY` (bukan OpenRouter).
- Fallback regex tetap ada bila key kosong / request error.

**Verifikasi**: chat "turnamen tunggal 10 orang 3x15" -> glm-5.2 ekstrak benar
(singles,10,3x15) -> approve -> insert id=7.

**Dampak**: agent tidak lagi bergantung pada model free yang tidak stabil; ekstraksi
natural language lebih andal. Biaya: model gratis di Zen = $0 untuk dev.

## Tambahan: Validasi input (tolak draft tidak lengkap)

**Masalah**: saat user mengetik teks tanpa info turnamen (mis. "hy, skor"), LLM gagal
ekstrak -> fallback regex -> draft default statis dengan `participant_count: null`
tetap tampil tombol Setuju. Itu tidak wajar: agent seharusnya minta klarifikasi,
bukan siapkan konfirmasi insert dari data kosong.

**Solusi**:
- `nodes._validate_intent(intent)`: cek field wajib — `name`, `play_mode`
  (doubles/singles), `participant_count`, `points_to_win`. Bila kurang -> set
  `_incomplete=True` + `_missing` (daftar yang kurang).
- `nodes.draft`: bila `_incomplete`, balik pesan klarifikasi (bukan ringkasan
  konfirmasi) + set `_skip_confirm=True`.
- `api.chat`: kirim `needs_confirmation = not skip` dan `incomplete` flag.
- `agent-chat.html`: bila `incomplete`, sembunyikan tombol Setuju/Batal (hanya
  tampilkan pesan minta klarifikasi).

**Verifikasi**:
- "hy, skor" -> `incomplete:True`, pesan "Mohon sebutkan: jumlah peserta".
- "Buat turnamen Ganda 16 orang 3x21" -> `incomplete:False`, draft lengkap.

**Dampak**: mencegah konfirmasi insert dari data tidak valid; UX lebih jelas
(admin tahu field apa yang kurang). Insert tetap aman di belakang (node insert
juga guard `decision=='approve'`).

# Perubahan Opsi B — Agentic Murni (2026-08-30)

## Masalah sebelumnya (Opsi A / template)
Agent memakai node `parse` (regex/LLM ekstrak JSON) + `draft` (template statis).
Saat input tidak lengkap, agent membalas teks HARDCODE:
"Maaf, informasi belum cukup..." — terasa seperti bot, bukan agent.
Frontend juga menampilkan SELURUH riwayat pesan (bukan hanya balasan terakhir),
makanya chat lama ikut muncul berulang.

## Yang diubah (Opsi B — sesuai keinginan user)
1. **Satu node `agent` yang memanggil LLM dengan tool.**
   - LLM diberi riwayat + 1 tool `create_tournament_draft`.
   - LLM yang MENENTUKAN: apa balasannya (klarifikasi natural / jawaban biasa),
     dan kapan memanggil tool (saat info turnamen lengkap).
   - Tidak ada lagi template statis — agent menyusun tiap kalimat sendiri.
2. **Tool `create_tournament_draft`** hanya menyimpan draft ke state, BUKAN insert.
3. **Human-in-the-loop tetap**: setelah LLM panggil tool, graph `interrupt()` menunggu
   admin approve (`/approve`) baru node `insert` tulis ke Postgres.
4. **Frontend** hanya menampilkan `reply` (balasan terakhir agent), bukan riwayat.
   Tombol konfirmasi muncul hanya bila `needs_confirmation=true`.

## Kenapa begini (dampak)
- Agent benar-benar "agentic": berpikir, bertanya balik, menyusun pesan — tidak kaku.
- Admin tidak pernah melihat template bot; pengalaman terasa seperti chat dengan asisten.
- Write ke DB tetap aman (hanya setelah persetujuan) — compliance Skor Cast.

## File
- `agent/agent/agent_node.py` (BARU): node `agent`, `tools_handler`, tool `create_tournament_draft`.
- `agent/agent/graph.py` (REWRITE): `agent → tools → agent → [interrupt] → insert → END`.
- `agent/agent/nodes.py` (REWRITE): hanya `insert_node` (guard decision=='approve').
- `agent/agent/llm.py` (UPDATE): `get_client()` + `MODEL`, load_dotenv dari `agent/.env`.
- `agent/api.py` (REWRITE): balas `reply` (last message), `needs_confirmation` dari draft.
- `public/agent-chat.html` (UPDATE): tampilkan `data.reply` saja.

## Provider LLM
- User minta OpenCode Zen `hy3:free`. Faktanya di Zen slug-nya `hy3-free` (dash, bukan colon).
- `tencent/hy3:free` sudah 404 di OpenRouter sejak 2026-08; `hy3-free` ada di Zen.
- `.env` agent: `OPENCODE_ZEN_API_KEY`, `OPENAI_BASE_URL=https://opencode.ai/zen/v1`,
  `AGENT_MODEL=hy3-free`.

## Bugs yang ditemukan & diperbaiki saat testing
- Recursion limit: rute `agent` tanpa tool_call/draft harus `END`, bukan balik ke `agent`.
- `load_dotenv()` di CWD salah muat `.env` Laravel → arahkan ke `agent/.env`.
- `AIMessage(tool_calls=None)` tidak valid → pakai `[]` bila kosong.
- `choice.message.content` → `choice.content` (OpenAI SDK).
## Feedback setelah pembuatan turnamen (2026-08-30)
- User minta: saat Setuju -> tampilkan "sedang diproses"; setelah selesai -> pesan
  konfirmasi + detail + URL turnamen.
- Implementasi:
  - `nodes.insert_node` kembalikan dict `result` berisi id, code, name, play_mode,
    participant_count, points_to_win, url (`/t/{code}`), message.
  - Node BARU `finish_node` (agent_node.py): panggil LLM susun pesan penutup natural
    (bukan template) berisi detail + URL. Graph: `insert -> finish -> END`.
  - `api.approve` balik `result` (dict) + `url`.
  - `agent-chat.html`: klik Setuju -> "⏳ Sedang diproses..."; tampilkan `url` sbg link.
- Kenapa agentic: pesan selesai disusun LLM, bukan hardcode -> konsisten dgn prinsip
  "agent yang menentukan, bukan template".
