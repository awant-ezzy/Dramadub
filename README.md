# DramaDub AI — Fullstack (Server Edition)

Aplikasi **Dubbing & Voice-Over Drama** dengan **auto-clip 30 detik – 5 menit**,
**watermark**, **subtitle burn-in**, dan **render server-side** memakai ffmpeg sistem.
Backend Python **FastAPI** + frontend HTML statis — dirancang untuk **VM Ubuntu**.

> Pemrosesan 100% di server Anda (VM). Tidak ada unggahan ke pihak ketiga.
> TTS offline via **espeak-ng** (gratis); opsional hubungkan API premium (OpenAI/ElevenLabs).

---

## Arsitektur

```
┌────────────┐     HTTP      ┌──────────────────────────┐
│  Browser   │ ◄──────────►  │  FastAPI (uvicorn :8000) │
│ index.html │               │  - REST API              │
└────────────┘               │  - Static frontend       │
                             │  - Job queue (threading) │
                             └────────────┬─────────────┘
                                          │ subprocess
                             ┌────────────▼─────────────┐
                             │  ffmpeg + espeak-ng      │
                             │  (apt install)           │
                             └────────────┬─────────────┘
                                          │ file I/O
                             ┌────────────▼─────────────┐
                             │  /var/lib/dramadub       │
                             │   projects/{pid}/...     │
                             └──────────────────────────┘
```

**Stack:** Python 3.12 · FastAPI · uvicorn · pydantic · ffmpeg 7.x · espeak-ng · nginx (reverse proxy) · systemd.

---

## Struktur Paket

```
dramadub-fullstack/
├── install.sh                # Installer 1-klik untuk Ubuntu (sudo)
├── Dockerfile                # Container image
├── docker-compose.yml        # Orkestrasi Docker
├── .env.example              # Template konfigurasi
├── nginx/dramadub.conf       # Site config nginx (port 80 → 8000)
├── systemd/dramadub.service  # Unit systemd (auto-start, restart)
├── backend/
│   ├── main.py               # FastAPI app (single-file, ~500 baris)
│   └── requirements.txt
├── frontend/
│   └── index.html            # UI lengkap (inline CSS+JS)
├── scripts/
│   └── dev.sh                # Jalankan tanpa install (dev mode)
├── README.md                 # Dokumen ini
└── INSTALL.md                # Panduan instalasi detail
```

---

## Quick Start — 3 Cara

### A. Installer 1-klik (disarankan untuk VM Ubuntu)

```bash
# 1. Salin paket ke VM
scp dramadub-fullstack.tar.gz user@vm:/tmp/

# 2. Di VM:
ssh user@vm
sudo apt update && sudo apt install -y tar
tar xzf /tmp/dramadub-fullstack.tar.gz -C /tmp
cd /tmp/dramadub-fullstack
sudo bash install.sh

# 3. Buka browser
# http://<IP-VM>/
```

Installer otomatis: apt install ffmpeg+espeak-ng+nginx+python, buat venv, install deps, systemd service, nginx reverse proxy. Selesai ~2 menit.

### B. Docker

```bash
tar xzf dramadub-fullstack.tar.gz
cd dramadub-fullstack
cp .env.example .env
docker compose up -d --build
# http://<IP-VM>:8000
```

### C. Tanpa install (dev)

```bash
cd dramadub-fullstack
# pastikan ffmpeg & espeak-ng terinstall: sudo apt install -y ffmpeg espeak-ng
bash scripts/dev.sh
# http://127.0.0.1:8000
```

---

## Fitur (Server-Side, Nyata)

| Modul | Implementasi Server |
|-------|---------------------|
| **Upload Video** | chunked upload, ffprobe durasi/resolusi/fps |
| **Auto-Clip** | ffmpeg `select=gt(scene,X)` showinfo → deteksi scene nyata; mode even/silence juga |
| **Dubbing** | proxy API terjemahan (OpenAI/Google/DeepL/LibreTranslate) atau manual |
| **Voice-Over** | **espeak-ng** (offline, gratis, 19 bahasa) atau API premium (OpenAI/ElevenLabs); audio MP3 nyata |
| **Watermark** | ffmpeg `drawtext` (teks) / `overlay` (gambar); posisi 9-titik, opasitas, gaya |
| **Subtitle** | editor per klip, ekspor SRT, **burn-in** via `subtitles` filter |
| **Export** | render ffmpeg: clip + watermark + VO + subtitle dalam **satu pass**; format MP4/WebM/MOV |
| **Jobs** | antrian render async (threading) + polling status via `/api/jobs/{id}` |
| **Project** | persistensi JSON di `/var/lib/dramadub/projects/{pid}/` |

---

## API Endpoints

| Method | Path | Fungsi |
|--------|------|--------|
| GET | `/api/health` | status server + ffmpeg + espeak |
| GET/PUT | `/api/settings` | baca/tulis konfigurasi API |
| GET/POST | `/api/projects` | daftar / buat project |
| GET/PUT/DELETE | `/api/projects/{pid}` | detail / update / hapus |
| POST | `/api/projects/{pid}/upload` | upload video (multipart) |
| POST | `/api/projects/{pid}/autoclip` | generate klip (job async) |
| GET/POST | `/api/projects/{pid}/clips` | daftar / tambah klip |
| PUT/DELETE | `/api/projects/{pid}/clips/{cid}` | update / hapus klip |
| POST | `/api/projects/{pid}/clips/{cid}/translate` | terjemahkan |
| POST | `/api/projects/{pid}/clips/{cid}/tts` | generate voice-over |
| POST | `/api/projects/{pid}/clips/{cid}/render` | render klip (job async) |
| GET | `/api/jobs/{jid}` | status job |
| GET | `/api/files/{pid}/{cid}/{name}` | unduh file hasil |

Dokumentasi interaktif (Swagger UI) otomatis tersedia di `/docs`.

---

## Konfigurasi API (opsional)

Tanpa API, app tetap berfungsi: TTS via **espeak-ng** (offline), terjemahan manual.

Atur via UI (tab **Settings**) atau edit `DATA_DIR/settings.json`:

| Provider | Endpoint | Catatan |
|----------|----------|--------|
| OpenAI TTS | `https://api.openai.com/v1/audio/speech` | voices: alloy/nova/onyx/shimmer |
| ElevenLabs | `https://api.elevenlabs.io/v1/text-to-speech/{voice_id}` | voice_id dari dashboard |
| OpenAI Translate | `https://api.openai.com/v1/chat/completions` | model gpt-4o-mini |
| DeepL | `https://api-free.deepl.com/v2/translate` | target_lang uppercase |

API key disimpan di server (DATA_DIR/settings.json), tidak diekspos ke browser.

---

## Operasional

```bash
# status / restart / log
systemctl status dramadub
systemctl restart dramadub
journalctl -u dramadub -f

# nginx
sudo nginx -t && sudo systemctl reload nginx

# docker
docker compose logs -f
docker compose restart

# cek ffmpeg & espeak
ffmpeg -version | head -1
espeak-ng --version
curl http://localhost/api/health | python3 -m json.tool

# backup data
sudo tar czf dramadub-backup-$(date +%F).tar.gz /var/lib/dramadub
```

---

## Performa & Batasan

- **Render** via ffmpeg sistem (bukan WASM): klip 1 menit 1080p ~5–15 detik di VM 2 vCPU.
- **Upload besar**: nginx `client_max_body_size 2G` (edit conf jika perlu lebih).
- **Disk**: video sumber + render disimpan di `DATA_DIR`; sesuaikan kapasitas VM.
- **espeak-ng**: kualitas robotic; untuk voice-over profesional gunakan API premium.
- **Workers**: uvicorn `--workers 1` (job queue internal multi-thread). Naikkan jika perlu, tapi ffmpeg sudah multi-thread via `ffmpeg_threads` setting.
- **RAM**: ~200MB idle, naik saat render (ffmpeg butuh memori sesuai resolusi).

---

## Troubleshooting

| Gejala | Solusi |
|--------|--------|
| `502 Bad Gateway` | `systemctl status dramadub` — pastikan service jalan |
| Upload gagal `413` | naikkan `client_max_body_size` di nginx, reload |
| TTS error | `espeak-ng` terinstall? `which espeak-ng`; atau atur API TTS |
| Render stuck | cek `journalctl -u dramadub`; ffmpeg error ada di log |
| Scene detection kosong | turunkan `scene_sens` (lebih rendah = lebih sensitif) |
| Port 80 sibuk | edit `nginx/dramadub.conf` listen port, atau akses langsung :8000 |

---

## Privasi

Semua pemrosesan terjadi di VM Anda. Video, audio, transkrip, dan render
disimpan lokal di `/var/lib/dramadub`. API eksternal hanya dipanggil jika Anda
mengisi endpoint + key di Settings.
"# Dramadub" 
