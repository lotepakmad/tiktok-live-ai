# PANDUAN — project-baru (Demo Website TikTok Live + AI Host)

Demo website berstyle TikTok Live: host AI (LLM beneran + suara), penonton bot,
komentar real-time, gift + combo, like. Tujuan akhir Farhan: live TikTok otomatis
dengan AI (video source + connector TikTok + RTMP = phase 2, BELUM dibuat).

---

## Menjalankan

```bash
cd project-baru
node server.js            # buka http://127.0.0.1:21060
```

Tidak ada dependensi npm (zero-dep, Node >= 22, pakai fetch + node:http bawaan).

### Konfigurasi LLM host (config.json)

```json
{
  "base_url": "http://127.0.0.1:21050/v1",   // gateway OpenAI-compatible
  "api_key":  "sk-...",
  "model":    "mimo-v2.6-flash-free"
}
```

Prioritas: env var `AI_BASE_URL` / `AI_API_KEY` / `AI_MODEL` > config.json >
fallback "balasan cadangan" (kata-kata siap, tanpa LLM).

Saat ini menumpang di **freeOpencode playground (port 21050)**. Kalau gateway itu
mati, host otomatis pakai balasan cadangan (server TIDAK crash). Bisa dialihkan
ke gateway mana pun (farhanhana, 9routerLite, MiniMax dsb) — cukup edit
config.json, tidak perlu ubah kode.

**Kebersihan balasan:** prefix "VANTA online" (disuntik gateway opencode)
di-strip otomatis; `**` markdown dibuang (biar TTS tidak membacanya).

---

## Struktur file

```
project-baru/
├── server.js            Semua backend: HTTP + SSE + bot penonton + AI host
├── config.json          Kredensial LLM host (RAHASIA — jangan dibagikan)
├── public/
│   └── index.html       Seluruh UI + CSS + JS client (satu file, tanpa build)
├── cloudflared.exe      Cloudflare Tunnel (v2026.10.3, quick tunnel jalan)
└── PANDUAN.md           File ini
```

---

## Arsitektur (alur data)

```
[Bot penonton]──┐                                ┌── browser (SSE stream)
[POST /api/*] ──┼──► server.js ──broadcast SSE──┤
[LLM gateway] ◄─┘        │                       └── browser lain (nonton bareng)
   (config.json)         └── state: likes, viewers, topGifters, recent chat
```

- **SSE** (`GET /events`): satu arah server → client. Event: `state` (awal),
  `comment`, `host_reply`, `host_typing` / `host_typing_end`, `gift`,
  `presence` (join), `stats`. Reconnect otomatis (built-in EventSource).
- **Kirim interaksi** via POST biasa (`/api/comment`, `/api/gift`, `/api/like`).
- Multi-tab / multi-penonton sudah jalan (semua yang terkoneksi dapat event sama).

---

## Endpoint

| Method | Path | Fungsi |
|---|---|---|
| GET | `/` | Dashboard TikTok Live (public/index.html) |
| GET | `/events` | Stream SSE (state awal + semua event live) |
| GET | `/api/state` | Snapshot: host, likes, viewers, gifts[], topGifters[], recent[] |
| POST | `/api/comment` | `{name, text}` — tampilkan komentar + (85%) host membalas |
| POST | `/api/gift` | `{gift, name, combo}` — gift event + (combo≥8) host berterima kasih |
| POST | `/api/like` | Tambah 1 like |

---

## Fitur yang sudah ada

- **UI TikTok Live**: header host + Follow, badge LIVE, penonton, chat mengalir,
  rail aksi (like 💗 hati melayang / komentar / gift / suara 🔊 / share),
  gift sheet 8 item (Mawar 🪙1 → Istana 🪙5000), combo popup, top gifter,
  join pill, input komentar (nama tersimpan di localStorage).
- **Host AI**: LLM via gateway (persona: host live TikTok, maks 2 kalimat,
  gaul, sapa nama penonton), indikator "sedang mengetik", fallback balasan
  cadangan kalau LLM gagal (timeout 20 dtk, ada log `[host] LLM gagal`).
- **Suara host**: Web Speech API — auto-pilih suara id-ID perempuan
  (Gadis/Damayanti di Windows), avatar "berbicara" (glow cyan + detak cepat),
  tombol 🔊/🔇 (preferensi tersimpan di localStorage).
- **Bot penonton**: 20 nama + 20 kalimat, komentar acak tiap 2–6 dtk,
  gift acak tiap 20–70 dtk, join tiap 4–13 dtk, viewers & likes naik sendiri.

---

## Hal yang perlu diketahui / jebolan

- **Port 21060** — kalau bentrok: `netstat -ano | grep 21060` lalu `taskkill -F -PID <pid>`.
- **Autoplay suara**: browser memblokir suara sebelum interaksi pengguna —
  klik sekali di halaman mana pun untuk mengaktifkan.
- **config.json = RAHASIA** (berisi API key). Jangan commit/dibagikan.
- **freeOpencode (21050) mati** → host pakai balasan cadangan, bukan error.
- **Image host cuma emoji** 👩‍🎤 — image generation tidak tersedia dari sini
  (FAL_KEY belum di-set). Alternatif: SVG avatar gerak, atau Farhan sediakan gambar.
- **Belum ada persistensi** — restart server = likes/top gifter/sesi reset.
- **Belum ada auth/room** — siapa pun yang bisa akses port bisa komen.
  Kalau mau publik (tunnel), tambah dulu identitas room + anti-spam.

---

## Rencana lanjutan (belum dikerjakan)

1. Avatar host bergerak (SVG/gambar asli) — bukan emoji
2. Efek gift besar layar penuh (Singa/Istana) — sekarang banner kecil semua
3. Sound effects (notif gift, pop like) via Web Audio API
4. Avatar/nama warna + level di komentar bot
5. Timer live + statistik sesi (penonton tertinggi, total gift)
6. Persistensi state ke file JSON
7. Control panel operator (pause bot, ganti model/persona, kirim sebagai host)
8. Multi-penonton publik via Cloudflare tunnel — ✅ SUDAH JALAN (quick tunnel
   gratis, tanpa login): `./cloudflared.exe tunnel --url http://127.0.0.1:21060`
   URL acak *.trycloudflare.com, berubah tiap restart. Verifikasi 200 OK via publik.
9. TTS lebih bagus (MiniMax speech-2.8-turbo atau Edge TTS) — lihat CHANGELOG
10. Phase 2 besar: sumber video asli + TikTok Live connector + RTMP balik

---

## Verifikasi cepat

```bash
curl -s http://127.0.0.1:21060/api/state | head -c 200                    # state OK
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"name":"Tes","text":"halo"}' http://127.0.0.1:21060/api/comment    # komen OK
curl -s -N http://127.0.0.1:21060/events | head -c 300                    # SSE stream OK
```

Balasan host berasal dari LLM jika jawabannya spesifik terhadap pertanyaan
(bukan salah satu dari ~15 kalimat balasan cadangan di server.js `REPLIES`).
