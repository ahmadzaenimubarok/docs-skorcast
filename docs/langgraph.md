# LangGraph di Skorcast

> Dokumentasi: kenapa LangGraph dipakai, bagaimana flow-nya di project ini, dan
> dampaknya. (Ditulis saat mulai eksplorasi agentic AI, 29 Aug 2026)

## Apa itu LangGraph?
LangGraph adalah framework **orchestration** open-source (MIT) dari LangChain Inc.
untuk membangun agen AI yang **stateful** dan **long-running**. Alur kerja
direpresentasikan sebagai **graph**: tiap langkah = *node*, transisi = *edge*
(bisa conditional/loop, bukan cuma linear A→B→C). Node berisi kode biasa
(Python/TS) — bisa campur logika deterministic dengan langkah LLM.

## Kenapa pakai LangGraph (bukan LangChain biasa / framework lain)?
Kebutuhan Skorcast untuk admin bikin turnamen via chat:
1. **Human-in-the-loop wajib** — agen harus draft rencana insert, lalu MINTA
   KONFIRMASI sebelum menulis ke database. Ini fitur bawaan LangGraph
   (`interrupt()` / checkpoint), tidak perlu kita bangun manual.
2. **State eksplisit & persisten** — saat jeda konfirmasi, state tersimpan di
   checkpointer (PostgresSaver). Agen bisa resume persis di titik yang sama,
   tidak dobel insert.
3. **Alur bercabang** — parse → draft → (setuju|batal) → insert. Graph cocok
   untuk cabang & loop, berbeda dengan LangChain yang lebih ke rantai linear.
4. **Durable execution + streaming** — cocok untuk chat yang responsif.

Alternatif yang dipertimbangkan:
- *LangChain biasa*: terlalu linear, state implisit → kurang pas untuk HITL.
- *Framework ringan/subagent*: cukup kalau kasus simpel, tapi kehilangan
  checkpoint & kontrol graph yang kita butuhkan.

## Flow penerapan di project ini
```
[admin chat] "Buat turnamen Ganda 16 orang 3x21"
      │  (Laravel panel / HTML testing → POST /chat ke FastAPI)
      ▼
┌─ LangGraph StateGraph (Python, proses terpisah) ──┐
│ START → parse → draft → [interrupt: tunggu approve]│
│                        ↑_________________│         │
│              approve → insert → END                 │
│              reject  → END                          │
└────────────────────────────────────────────────────┘
      │ draft (JSON) balik ke Laravel/HTML
      ▼
[admin: SETUJU / BATAL]
      │ POST /approve {thread_id, decision}
      ▼
agent resume → eksekusi insert ke Postgres
```
- Agent dijalankan **terpisah** (FastAPI + uvicorn, port 8000), 1 repo dengan
  Laravel tapi proses sendiri.
- Checkpointer = `PostgresSaver` ke DB Postgres yang sama (fallback MemorySaver
  saat POC tanpa DB).
- HTML testing (`public/agent-chat.html`) langsung hubungi FastAPI lewat CORS
  untuk coba cepat; Laravel bridge (`AgentService`) menyusul sebagai jalur produksi.

## Dampak
- **Positif**: admin bisa buat turnamen dengan bahasa natural; risiko salah tulis
  DB kecil karena ada gerbang konfirmasi; state aman walau agent restart.
- **Yang perlu dijaga**:
  - Agent jangan di-expose ke publik tanpa auth (pakai API key + bind localhost).
  - Checkpointer wajib Postgres di produksi (MemorySaver bocor antar user).
  - Insert nyata ke `tournaments` baru dilakukan setelah alur konfirmasi
    kamu validasi aman di testing.
- **Konvensi**: tiap tambahan terkait agent didokumentasikan di `docs/` ini.

## Revisi 30 Aug 2026 — `participant_count` jadi opsional (default bebas)
**Yang ditambah/diubah:**
- `create_tournament_draft` (tool agent): `participant_count` kini **opsional**
  (default `None` = bebas tanpa batas peserta). Dikeluarkan dari `required`.
- SYSTEM_PROMPT: jumlah peserta tidak lagi wajib dikumpulkan; agent BOLEH
  menanyakan tapi TIDAK memaksa. Info wajib cukup: nama, format, sistem skor.
- `insert_node` (`nodes.py`): menulis `participant_count` ke kolom
  `max_participants` (nullable) supaya nilai bebas tersimpan; juga
  mengembalikannya di `result` (termasuk `max_participants`).
- `finish_node`: peserta tampil "bebas" bila `participant_count` kosong.

**KENAPA:**
- Keputusan produk: jumlah peserta tidak selalu diketahui di awal; turnamen
  boleh dibuat dulu lalu peserta ditambah belakangan (sesuai alur form admin
  yang mengizinkan `max_participants` null = tanpa batas baku).
- Menjaga keselarasan antara tool agent dan model `Tournament`
  (`max_participants` nullable) agar turnamen dari chat tidak terkunci angka.
