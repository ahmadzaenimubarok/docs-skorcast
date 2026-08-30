# Agentic AI Skorcast — Dokumentasi Setup & Cara Kerja

> Ditulis saat implementasi fitur agent pembuat turnamen (Python + LangGraph), 30 Aug 2026.
> Tujuannya: menjelaskan **apa itu agent ini, apa saja yang ada di dalamnya,
> bagaimana alurnya, dan metode yang dipakai** — bukan sekadar catatan commit.
>
> **Catatan konteks (untuk Hermes):** file ini juga berfungsi sebagai *context store*
> agar tidak perlu membaca seluruh source setiap sesi. Jika ada perubahan pada
> tools, graph, atau flow, UPDATE bagian terkait di sini, bukan hanya commit log.

---

## 1. Gambaran Besar (Big Picture)

Agent ini adalah asisten yang membantu **admin membuat turnamen bulu tangkis lewat
obrolan natural** di web Skorcast. Admin mengetik apa yang diinginkan, agent menangkap
maksudnya, menyusun draf turnamen, lalu **berhenti dan minta konfirmasi** sebelum
menulis apa pun ke database. Setelah admin menyetujui, agent menyelesaikan pembuatan
dan memberi tahu detail + URL turnamen.

Dua prinsip utama:

1. **Agentic, bukan bot.** Agent (LLM) yang menentukan kapan info cukup, apa balasannya,
   dan bagaimana menyusun kalimat — bukan template statis yang kaku.
2. **Human-in-the-loop (HITL).** Agent boleh menyusun draf, tapi **menulis ke database
   hanya boleh setelah admin menyetujui**. Ini mencegah agent bertindak sepihak.

---

## 2. Anatomi Agent (Apa saja yang ada di dalamnya)

Seperti agent pada umumnya, ini terdiri dari beberapa komponen yang bekerja bersama:

| Komponen | Di project ini | Fungsi |
|----------|----------------|--------|
| ** Reasoning engine ** (otak) | LLM `hy3-free` via OpenCode Zen | Mengerti pesan user, memutuskan kapan info cukup, menyusun balasan |
| ** Memory / state ** | `PostgresSaver` (checkpointer) | Menyimpan riwayat percakapan tiap `thread_id`, agar bisa lanjut dari titik berhenti |
| ** Tools ** (tangan ke dunia luar) | `create_tournament_draft` | Satu-satunya "aksi" yang bisa diajukan agent: menyimpan draf turnamen |
| ** Reflection / loop ** | node `agent → tools → agent` | Agent berpikir, panggil tool bila perlu, lihat hasil, lalu respons |
| ** Guardrail (pintu aman) ** | node `wait_confirm` (interrupt) | Menghentikan eksekusi sebelum write DB, menunggu persetujuan admin |
| ** Executor ** | node `insert` + `finish` | Menulis ke Postgres (bila disetujui) lalu susun pesan selesai |

Siklus kerjanya mengikuti pola **ReAct**: *Think → Act (tool) → Observe → Repeat*
sampai kondisi berhenti tercapai (entah butuh klarifikasi, atau draf siap dikonfirmasi).

---

## 3. Flow / Alur Kerja

```
                 ┌─────────────────────────────────────────────┐
                 │            ADMIN MENGETIK PESAN              │
                 └───────────────────────┬─────────────────────┘
                                         │  POST /chat
                                         ▼
        ┌────────────────────────────────────────────────────────┐
        │  NODE agent  — LLM baca riwayat + tool                   │
        │  • jika info kurang  → balas minta klarifikasi (END)     │
        │  • jika sudah lengkap → panggil tool create_tournament_  │
        │    draft (tool_call)                                     │
        └───────────────────────┬──────────────────────────────────┘
                                 │ ada tool_call?
                    ┌────────────┴────────────┐
                    │ TIDAK                   │ YA
                    ▼                         ▼
              balas ke user          NODE tools — simpan draft ke state
              (END)                        │
                                          ▼
                                  NODE agent (lagi) — respons:
                                  "draf siap, menunggu konfirmasi"
                                          │
                                          ▼
                          ┌───────────────────────────────────┐
                          │  NODE wait_confirm (interrupt ⏸)    │
                          │  graph BERHENTI, state di-checkpoint│
                          │  frontend tampilkan tombol Setuju   │
                          └─────────────────┬───────────────────┘
                                            │ admin klik Setuju
                                            │ POST /approve  Command(resume)
                                            ▼
                                  NODE insert — tulis ke Postgres
                                  (HANYA bila decision == approve)
                                            │
                                            ▼
                                  NODE finish — LLM susun pesan
                                  "selesai dibuat + detail + URL"
                                            │
                                            ▼
                                         END ✅
```

**Poin penting:**
- Agent **tidak langsung insert**. Ia berhenti di `wait_confirm` (interrupt).
- State tersimpan di Postgres, jadi walau server restart atau admin refresh halaman,
  percakapan tetap bisa dilanjutkan dari titik yang sama.
- Write ke DB terjadi **hanya** di node `insert` dan **hanya** bila admin approve.

---

## 4. Komponen per Modul (Struktur Kode)

Project berada di folder `agent/` (Python service, berjalan terpisah dari Laravel
lewat proses `uvicorn` di port 8000). Berikut susunannya, mirip pola modul yang umum
di dokumentasi LangGraph:

### MODULE 1 — State (`agent/agent/state.py`)
Mendefinisikan "ingatan" graph: `messages` (riwayat chat), `draft` (draf turnamen),
`decision` (approve/reject dari admin), `result` (hasil insert).

### MODULE 2 — LLM Client (`agent/agent/llm.py`)
Helper `get_client()` + `MODEL`. Membaca env `OPENCODE_ZEN_API_KEY`,
`OPENAI_BASE_URL=https://opencode.ai/zen/v1`, `AGENT_MODEL=hy3-free`.
OpenAI-compatible, jadi gampang ganti provider.

### MODULE 3 — Tools & Agent Node (`agent/agent/agent_node.py`)

**Tools yang dibuat (1 buah saja): `create_tournament_draft`**

- **Lokasi**: `agent/agent/agent_node.py` (didekorasi `@tool`, sekitar baris 52).
- **Tanda tangan**:
  ```python
  @tool
  def create_tournament_draft(
      name: str,
      play_mode: str,
      participant_count: int,
      points_to_win: int,
      cap: int,
      mid_game_interval_at: int,
      switch_end_game3: int,
  ) -> str:
      """Panggil saat semua info turnamen sudah lengkap. Menyimpan draf (belum insert)."""
  ```
- **Field & nilai valid**:
  | Field | Tipe | Nilai |
  |-------|------|-------|
  | `name` | str | bebas |
  | `play_mode` | str | `"doubles"` / `"singles"` |
  | `participant_count` | int | > 0 |
  | `points_to_win` | int | 21 atau 15 |
  | `cap` | int | 30 (jika 21) / 21 (jika 15) |
  | `mid_game_interval_at` | int | 11 (jika 21) / 8 (jika 15) |
  | `switch_end_game3` | int | 11 (jika 21) / 8 (jika 15) |
- **Yang DILAKUKAN**: menyimpan field ke `state["draft"]` (lewat `tools_handler`).
- **Yang TIDAK dilakukan**: **tidak** menulis ke Postgres, **tidak** mengubah data
  produksi. Ia murni "ajukan draf".
- **Kapan dipanggil**: LLM memanggilnya sendiri bila yakin info sudah lengkap
  (diarahkan oleh `SYSTEM_PROMPT` di file ini).
- **Setelah dipanggil**: graph berhenti di `wait_confirm` (interrupt) menunggu admin
  approve; baru node `insert` yang tulis ke DB.

**Node lain di modul ini:**
- `agent_node` — memanggil LLM dengan riwayat + tool. LLM yang menentukan balasan
  dan kapan memanggil tool.
- `tools_handler` — menjalankan tool, menyimpan hasil ke `state["draft"]`.
- `finish_node` — setelah insert, LLM menyusun pesan penutup natural
  (detail + URL), bukan template.

> **Catatan**: `create_tournament_draft` adalah **satu-satunya tool** di agent ini.
> Tools lain (edit/cancel) belum ada — ruang lingkup saat ini hanya pembuat turnamen.

### MODULE 4 — Executor (`agent/agent/nodes.py`)
`insert_node` — menulis ke `tournaments` (Postgres) **hanya bila** `decision == "approve"`.
Mengembalikan dict berisi `id`, `code`, `url` (`/t/{code}`), dan detail turnamen.

### MODULE 5 — Graph (`agent/agent/graph.py`)
Merangkai node jadi alur:
`START → agent → tools → agent → wait_confirm(interrupt) → insert → finish → END`.
Checkpointer `PostgresSaver` dipasang agar state persisten.

### MODULE 6 — API (`agent/api.py`, FastAPI)
- `POST /chat` — terima pesan, jalankan graph sampai butuh input/interrupt.
  Balas `reply` (balasan terakhir agent) + `needs_confirmation`.
- `POST /approve` — terima `approve`/`reject`, resume graph lewat `Command(resume=...)`.
  Balas `result` + `url` + `reply`.
- `GET /health` — healthcheck.

### MODULE 7 — Frontend Testing (`public/agent-chat.html`)
Halaman chat sederhana untuk uji agent. Fetch ke `/agent/chat` & `/agent/approve`
(lewat nginx reverse proxy, same-origin). Saat klik Setuju tampil "⏳ Sedang diproses…",
lalu menampilkan pesan selesai + URL sebagai link.

---

## 5. Metode yang Dipakai & Kenapa

### FastAPI + uvicorn (bukan LangServe/WebSocket)
- **Kenapa**: alur kita request-response + jeda konfirmasi, bukan stream token dua arah.
  Cukup 2 endpoint (`/chat`, `/approve`). Ringan & mudah di-production.
- **Dampak**: agent di-compile sekali; tiap request pakai `thread_id` sebagai kunci
  checkpoint → percakapan antar admin terpisah.

### LangGraph StateGraph + `interrupt()` (inti HITL)
- **Kenapa**: admin wajib konfirmasi sebelum insert. `interrupt()` menghentikan graph,
  menyimpan state ke checkpointer, menunggu `Command(resume=...)` dari luar.
- **Dampak**: agent tidak bisa write DB tanpa restu admin. Aman untuk data produksi.

### PostgresSaver (checkpointer persisten)
- **Kenapa**: `MemorySaver` hilang saat restart & bocor antar user. PostgresSaver
  menyimpan state di DB sama yang dipakai Laravel.
- **Dampak**: percakapan bisa dilanjutkan walau server restart; aman multi-admin.

### LLM sebagai pengambil keputusan (bukan template)
- **Kenapa**: user mau agent terasa hidup, bukan bot. LLM yang menilai "info cukup?"
  dan menyusun tiap kalimat (klarifikasi, konfirmasi, pesan selesai).
- **Dampak**: tidak ada teks hardcode; konsisten dengan prinsip agentic.

### Best Practices HITL (dari dokumentasi LangChain)
Diterapkan di UI kita:
- **Show clear context** — draft lengkap ditampilkan sebelum tombol konfirmasi.
- **Make approve the easiest path** — cukup 1 klik "Setuju".
- **Persist interrupt state** — state di Postgres, bisa refresh & tetap resume.
- **Feedback saat proses** — "⏳ Sedang diproses…" sebelum response datang.

---

## 6. Provider LLM

- Dev: **OpenCode Zen `hy3-free`** (slug pakai dash, bukan `hy3:free` yang sudah
  404 di OpenRouter sejak 2026-08).
- Ganti provider cukup ubah `AGENT_MODEL` + `OPENAI_BASE_URL` di `agent/.env`
  (OpenAI-compatible).

---

## 7. Bugs yang Ditemukan & Diperbaiki saat Testing

- Recursion limit: rute `agent` tanpa tool_call/draft harus `END`, bukan balik ke `agent`.
- `load_dotenv()` di CWD salah muat `.env` Laravel → arahkan ke `agent/.env`.
- `AIMessage(tool_calls=None)` tidak valid → pakai `[]` bila kosong.
- `choice.message.content` → `choice.content` (OpenAI SDK).

---

## 8. Cara Menjalankan (ringkas)

```bash
cd /var/www/skorcast.online
sudo -u www-data bash -c 'agent/.venv/bin/python -m uvicorn agent.api:app --host 127.0.0.1 --port 8000'
# akses lewat nginx: http://skorcast.online/agent/chat
# testing UI:          http://skorcast.online/agent-chat.html
```

Agent diakses dari web lewat nginx `location /agent/` (reverse proxy ke :8000),
sehingga tidak butuh CORS/port terbuka.
