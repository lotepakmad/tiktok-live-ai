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

### Konfigurasi penuh (config.json) — 2026-10-24

Semua perilaku demo sekarang configurable. Struktur config.json:

```json
{
  "port": 21060,                  // port server
  "host_name": "AI Host",         // nama host yang tampil di chat
  "base_url": "...",              // LLM host (Pollinations langsung, via proxy)
  "api_key": "...",               // key LLM
  "model": "openai",              // model LLM
  "host_persona": "Kamu {name}, ...",  // kepribadian host (system prompt LLM)
  "host_welcome": ["...{name}..."],    // sambutan join (template, {name} diganti nama)
  "tts_api_key": "...",           // Cartesia TTS
  "tts_base_url": "https://api.cartesia.ai",
  "tts_version": "2026-08-14",
  "tts_model": "sonic-3.6",
  "tts_voice_id": "...",
  "tts_lang": "id",
  "proxy_file": "proxy.txt",      // file proxy (host:user:pass@host:port per baris)
  "bot_names": [...],             // (opsional) nama bot penonton
  "bot_lines": [...],             // (opsional) komentar bot
  "bot_gifters": [...],           // (opsional) nama pemberi gift bot
  "canned_replies": {...},        // (opsional) balasan cadangan per kategori
  "demo": {
    "bot_comments_enabled": false,   // komentar bot penonton
    "bot_gifts_enabled": false,      // gift bot
    "bot_joins_enabled": true,       // join bot
    "host_welcome_on_join": true,    // host nyambut yang join (prioritas antrean)
    "bot_comment_delay": [1800, 4200],  // delay komentar bot (ms, [min,max] diacak)
    "bot_gift_delay": [20000, 50000],
    "bot_join_delay": [60000, 60000],  // join bot — 1 menit sekali
    "host_reply_chance": 0.35,       // peluang host bales komentar BOT (user selalu 100%)
    "host_answer_gap": [2500, 5000], // jeda antar jawaban host saat antrean panjang
    "queue_max": 20,                 // max antrean host (yang terlama dibuang)
    "host_reply_max_len": 200,       // potongan maksimal balasan host
    "viewers_start": 1243,           // penonton awal
    "viewers_min": 800,              // batas bawah fluktuasi penonton
    "likes_start": 128400,           // like awal
    "combo_thanks_min_combo": 8,     // host ucap makasih gift: min combo
    "combo_thanks_min_coin": 1000,   // ...atau min koin gift
    "viewer_fluctuation_interval": 5000
  }
}
```

Semua delay = `[min, max]` ms — server mengacak di antaranya biar natural.
Env var (`PORT`, `AI_*`, `TTS_*`, `HOST_NAME`, `HOST_PERSONA`) menimpa config.
Ubah config → restart `node server.js`.

Riwayat backend host: dulu freeOpencode (21050), sempat 9router (20128),
sekarang **Pollinations LANGSUNG via curl.exe + proxy Bright Data** (proxy.txt).

**Kebersihan balasan:** prefix "VANTA online" (disuntik gateway opencode)
di-strip otomatis; `**` markdown dibuang (biar TTS tidak membacanya).

### Pengaturan live — /setting

```
http://127.0.0.1:21060/setting?key=<settings_key>
```

Halaman kontrol (toggle bot, delay, persona host, angka live, LLM/TTS, daftar
bot). Perubahan **langsung aktif tanpa restart** (kecuali `port` — berlaku di
boot berikutnya) dan **tersimpan ke config.json**; file lama selalu di-backup
dulu ke `backup/<tanggal>/config.json.pre-settings-<jam>`.

- **Key**: 32-hex acak, dibuat otomatis saat pertama start, ada di
  `config.json → settings_key` dan di-log server (`[setting] http://…/key=…`).
  Path lama `/settings` otomatis redirect ke `/setting` (bookmark lama aman).
- **Auth**: GET/POST `/api/settings` wajib `?key=` atau header `x-settings-key`;
  tanpa/salah key → **403**. API key LLM/TTS tidak pernah ikut dikirim balik
  (hanya flag `api_key_set`), kosongkan input = tidak diubah.
- Input di-whitelist + di-clamp server-side (field asing dibuang).
- Delay baru berlaku untuk jadwal bot berikutnya; toggle on/off langsung.

---

## Struktur file

```
project-baru/
├── server.js            Semua backend: HTTP + SSE + bot penonton + AI host
├── config.json          Kredensial LLM host (RAHASIA — jangan dibagikan)
├── public/
│   ├── index.html       Seluruh UI + CSS + JS client (satu file, tanpa build)
│   └── settings.html    Halaman pengaturan (/setting, wajib key)
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
| GET | `/setting` | Halaman pengaturan (kirim `?key=`; `/settings` redirect ke sini) |
| GET | `/api/settings` | Ambil pengaturan aktif — wajib key (rahasia ditandai `*_set`) |
| POST | `/api/settings` | Terapkan pengaturan (whitelist+clamp) → live-apply + tulis config |
| POST | `/api/comment` | `{name, text}` — tampilkan komentar + (85%) host membalas |
| POST | `/api/gift` | `{gift, name, combo}` — gift event + (combo≥8) host berterima kasih |
| POST | `/api/like` | Tambah 1 like |

---

## Fitur yang sudah ada

- **UI TikTok Live**: header host + Follow, badge LIVE, penonton, chat mengalir,
  rail aksi (like 💗 hati melayang / komentar / gift / suara 🔊 / share),
  gift sheet 8 item (Mawar 🪙1 → Istana 🪙5000), combo popup, top gifter,
  join pill, input komentar (nama tersimpan di localStorage).
- **Visual host: inti JARVIS** — orb cyan + 3 ring + radar sweep + waveform,
  murni CSS/SVG-less (tanpa gambar). State `talking` (saat TTS/suara jalan)
  → ring ngebut, core flash, waveform muncul; mini orb di header ikut
  bereaksi via sibling selector `#avatar-big.talking ~ #header`.
- **Host AI**: LLM via gateway (persona: host live TikTok, maks 2 kalimat,
  gaul, sapa nama penonton), indikator "sedang mengetik", fallback balasan
  cadangan kalau LLM gagal (timeout 20 dtk, ada log `[host] LLM gagal`).
- **Self-talk**: kalau host diam ≥ `selftalk_sec` (default 25 dtk, 0 = mati,
  diatur di `/setting` → kartu Host AI), dia ngomong sendiri isi jeda —
  lewat LLM (prompt `[SEPI]`) atau kalimat jadian `SELF_LINES` kalau LLM
  gagal/mati. Timer reset tiap host bersuara (balas/sambutan/self-talk).
  Pesan masuk event `comment` biasa → tampilan: cuma suara (chat host
  disembunyikan), orb JARVIS ikut reaksi `talking`.
- **Anti-ngulang**: provider menentistikkan/caching request identik
  (terbukti: 2 POST identik + temperature 0.9 → jawaban byte-identik), jadi:
  `temperature: 0.9` + **nuansa acak anti-berturut** (`RESP_ANGLES`, disisipkan
  ke system message tiap panggilan → badan prompt selalu beda), **topik acak**
  (`SELF_TOPICS`) + dilarang ulang 3 topik terakhir, dan **shuffle-bag**
  `SELF_BAG` — 14 kalimat cadangan habis dulu sebelum mulai lagi.
  Log: `[host] self-talk → <jawaban final>`, `[host] LLM balas kosong → …`.
- **Retry LLM**: selain retry HTTP, `llmChat` retry 1x kalau HTTP 200 tapi
  konten kosong/JSON aneh (log `[host] LLM konten kosong/aneh → retry 1x`).
- **Uji koneksi LLM**: tombol ⚡ di `/setting` kartu 🧠 → `GET /api/llmtest?key=`
  → `{ok, latencyMs, reply}`; tanpa key = 403; memakai konfigurasi tersimpan.
- **Suara host (browser)**: kartu 🔊 di `/setting` — voice picker + rate/pitch/
  volume + tombol test. Tersimpan di localStorage (`dtl_voice/rate/pitch/vol`,
  satu origin dengan halaman live) → dipakai Web Speech. TTS Cartesia tetap
  pakai Voice ID tersimpan.
- **Shock orb**: event gift → `orbShock()` → `.shock` → gelombang kejut 0,75 dtk
  dari tepi orb (`#avatar-big.shock::after`).
- **Git**: repo di root project-baru. `.gitignore`: `config.json`, `proxy.txt`,
  `backup/`, `node_modules/`, `*.log` — rahasia jangan pernah `git add -f`;
  `config.example.json` = template placeholder.
- **Suara host**: Web Speech API — auto-pilih suara id-ID perempuan
  (Gadis/Damayanti di Windows), avatar "berbicara" (glow cyan + detak cepat),
  tombol 🔊/🔇 (preferensi tersimpan di localStorage).
- **Bot penonton**: 20 nama + 20 kalimat, komentar acak tiap 2–6 dtk,
  gift acak tiap 20–70 dtk, join tiap 1 menit, viewers & likes naik sendiri.

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
