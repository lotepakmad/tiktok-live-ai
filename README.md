# tiktok-live-ai

Demo website berstyle **TikTok Live** dengan host AI: LLM beneran + suara TTS,
penonton bot, komentar real-time, gift + combo, like. Zero-dependency
(Node >= 22, tanpa npm install).

> Tujuan proyek: live TikTok otomatis dengan AI. Fase sekarang = demo website
> (host AI + bot penonton). Fase 2 (belum dibuat): sumber video asli +
> connector TikTok Live + RTMP balik ke TikTok.

## Menjalankan

```bash
node server.js            # buka http://127.0.0.1:21060
```

Tidak ada dependensi npm — butuh Node.js >= 22 (pakai fetch + node:http bawaan).

### Konfigurasi LLM host

```bash
cp config.example.json config.json
```

Lalu isi `config.json`:

```json
{
  "base_url": "http://127.0.0.1:21050/v1",
  "api_key": "sk-...",
  "model": "nama-model"
}
```

`base_url` = gateway apa pun yang OpenAI-compatible (`/v1/chat/completions`).
Prioritas: env var `AI_BASE_URL` / `AI_API_KEY` / `AI_MODEL` > config.json >
fallback balasan cadangan (tanpa LLM, server tidak crash).

Kebersihan balasan: prefix injection gateway di-strip otomatis; markdown `**`
dibuang agar tidak dibacakan TTS.

## Publik via Cloudflare Tunnel (gratis)

```bash
# Download cloudflared (Windows amd64):
# https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-windows-amd64.exe
./cloudflared.exe tunnel --url http://127.0.0.1:21060
```

Quick tunnel gratis tanpa login → URL acak `*.trycloudflare.com`
(berubah tiap restart). Multi-penonton real-time (SSE) didukung.

## Endpoint

| Method | Path | Fungsi |
|---|---|---|
| GET | `/` | Dashboard TikTok Live |
| GET | `/events` | Stream SSE (state awal + semua event live) |
| GET | `/api/state` | Snapshot: host, likes, viewers, gifts, topGifters, recent |
| POST | `/api/comment` | `{name, text}` — komentar + (85%) host membalas |
| POST | `/api/gift` | `{gift, name, combo}` — gift event + host berterima kasih |
| POST | `/api/like` | Tambah 1 like |

## Fitur

- UI TikTok Live: header host + Follow, badge LIVE, viewer count, chat
  mengalir, rail aksi (like 💗 / komentar / gift / suara / share), gift sheet
  8 item, combo popup, top gifter, join pill
- Host AI: LLM via gateway (persona host live, maks 2 kalimat), indikator
  "sedang mengetik", fallback balasan cadangan
- Suara host: Web Speech API (auto-pilih suara id-ID perempuan di Windows),
  avatar "berbicara", toggle 🔊/🔇
- Bot penonton: 20 nama + 20 kalimat, komentar/gift/join acak, viewers & likes
  naik sendiri

## Catatan

- Port default **21060**
- Browser memblokir suara sebelum interaksi — klik sekali di halaman
- `config.json` berisi API key → **jangan pernah di-commit** (sudah di .gitignore)
- Belum ada persistensi — restart = sesi reset
- Belum ada auth/room — siapa pun yang bisa akses bisa komen

## Kontribusi

Pull request terbuka — template UI baru, efek gift, TTS engine lain, dsb.
Lihat bagian "Rencana lanjutan" di PANDUAN.md untuk ide.

## Lisensi

MIT
