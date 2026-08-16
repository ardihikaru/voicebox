# Setting Up Voicebox on This Laptop

A build-from-source walkthrough for **Ubuntu 24.04 (`workspace`)**, with a measured
comparison between what Voicebox needs and what this machine actually has.

Measured on **2026-08-16** against commit `51f49de` (Voicebox 0.5.0).

---

## Summary

| | |
| --- | --- |
| **Verdict** | Runs — but only after freeing disk space, and only on the small models. |
| **CPU** | AMD Ryzen 7 5800H, 8C/16T — comfortably enough. |
| **RAM** | 27 GB total, **~9.8 GB available right now** — enough for the small/mid engines. |
| **GPU** | RTX 3050 Ti Laptop, **4 GB VRAM** — the binding constraint. Rules out Qwen 1.7B, TADA 3B, Whisper Large. |
| **Disk** | **`/home` has 14 GB free (94% full).** A full setup needs ~20–25 GB. **This is the blocker.** |
| **Toolchain** | Node, Python 3.12, Docker, ffmpeg present. **Bun, Rust, and `just` are missing** and must be installed. |
| **Tauri deps** | 5 of the required `apt` packages are missing (webkit2gtk, gtk3, appindicator, rsvg, xdo, alsa). |

**The three things to do before anything else:**

1. Free space on `/home`, or point the model cache and build caches at `/` (83 GB free).
2. Install Bun, Rust, `just`, and the Tauri system libraries.
3. Plan on **Kokoro, LuxTTS, Chatterbox Turbo, Qwen 0.6B, and Whisper Turbo** — skip the
   6–8 GB VRAM models.

If you would rather not deal with the Rust/Tauri toolchain at all, use `just dev-web`
(browser UI, no desktop shell) or `docker compose up`. Both skip the Rust build entirely.

---

## 1. What this laptop has

Everything below was read off the machine, not assumed.

### Hardware

| Resource | Value |
| --- | --- |
| Host | `workspace` — Ubuntu 24.04.4 LTS, kernel 6.8.0-101-generic, x86_64 |
| CPU | AMD Ryzen 7 5800H — 8 cores / 16 threads, 400 MHz–4.46 GHz, 16 MB L3 |
| RAM | 27 GiB total · 17 GiB in use · **9.8 GiB available** |
| Swap | 13 GiB across `/swap.img` (4 G) and `/swapfile` (10 G), 714 MiB used |
| dGPU | NVIDIA GeForce RTX 3050 Ti Laptop — **4096 MiB VRAM**, compute capability 8.6 |
| iGPU | AMD Radeon Vega (Cezanne) — not usable for inference here |
| NVIDIA driver | 535.288.01 (CUDA 12.2), GPU currently idle at 8 MiB used |

### Disk

| Mount | Size | Used | Free | Note |
| --- | --- | --- | --- | --- |
| `/home` (`nvme0n1p5` … p2) | 232 G | 206 G | **14 G (94%)** | Where the repo lives. Tight. |
| `/` (`nvme0n1p5`) | 236 G | 141 G | **83 G (64%)** | Lots of room. Put the caches here. |

Two obvious reclaim targets on `/home`: `~/.cache` is **24 G** and `~/devel` is **98 G**.

### Toolchain already installed

| Tool | Status |
| --- | --- |
| Python | ✅ 3.12.3 (+ `python3.12-venv`) |
| Node.js | ✅ v26.5.1 |
| Docker + Compose | ✅ 29.2.1 / v5.1.0 |
| ffmpeg | ✅ 6.1.1 |
| git | ✅ 2.43.0 |
| uv | ✅ 0.11.27 |
| build-essential, pkg-config, libssl-dev, patchelf | ✅ |
| **Bun** | ❌ missing — required |
| **Rust / cargo** | ❌ missing — required for the desktop app |
| **just** | ❌ missing — every documented command uses it |
| `libwebkit2gtk-4.1-dev`, `libgtk-3-dev`, `libayatana-appindicator3-dev`, `librsvg2-dev`, `libxdo-dev`, `libasound2-dev` | ❌ missing — required by Tauri |

---

## 2. What Voicebox will consume

### Disk

| Item | Size |
| --- | --- |
| Repo checkout | ~200 MB |
| `bun install` across the 4 workspaces | ~1–2 GB |
| Python venv **with CUDA PyTorch** (torch + nvidia CUDA libs + transformers + friends) | **~8–10 GB** |
| Rust `target/` after a Tauri dev build | ~4–6 GB |
| Model cache (HuggingFace) — depends entirely on which engines you use | 0.4–25 GB |
| **Realistic total for a small-model setup** | **~20–25 GB** |

Against 14 GB free on `/home`, that does not fit. Section 3 handles this.

### Models — size on disk and VRAM at load

TTS engines (from `docs/content/docs/developer/model-management.mdx`):

| Model | Disk | VRAM | Fits in 4 GB? |
| --- | --- | --- | --- |
| Kokoro 82M | 350 MB | ~150 MB | ✅ easily |
| LuxTTS | 300 MB | ~1 GB | ✅ |
| Chatterbox Turbo | 1.5 GB | ~1.5 GB | ✅ |
| Qwen TTS 0.6B / CustomVoice 0.6B | 1.2 GB | ~2 GB | ✅ |
| Chatterbox Multilingual | 3.2 GB | ~3 GB | ⚠️ tight — nothing else on the GPU |
| TADA 1B | 4 GB | ~4 GB | ❌ won't fit alongside the driver |
| Qwen TTS 1.7B / CustomVoice 1.7B | 3.5 GB | ~6 GB | ❌ CPU only |
| TADA 3B Multilingual | 8 GB | ~8 GB | ❌ |

Whisper (STT):

| Model | Disk | Fits? |
| --- | --- | --- |
| Base / Small | 300–500 MB | ✅ |
| Turbo | ~1.5 GB | ✅ — the one to use |
| Medium | ~1.5 GB | ✅ |
| Large v3 | ~3 GB | ⚠️ only with nothing else loaded |

Local LLM (dictation refinement + voice personalities):

| Model | Disk |
| --- | --- |
| Qwen3 0.6B | ~400 MB (default) |
| Qwen3 1.7B | ~1.1 GB |
| Qwen3 4B | ~2.5 GB |

Voicebox unloads models per-engine, so you can hold a TTS engine *or* Whisper *or* the
LLM at a time — but stacking a 3 GB TTS model, Whisper Turbo, and the LLM will OOM 4 GB
of VRAM. Use the per-model unload in **Settings → Models**.

### RAM and CPU at runtime

Budget roughly **4–8 GB of RAM** with the small engines loaded, plus the Vite dev server
and the Tauri webview. With 9.8 GB currently available that works, but close a browser or
two first. The 16 threads make CPU-only inference viable as a fallback — LuxTTS claims
~150× realtime on CPU, and Kokoro is fast there too.

### Ports and paths

- Backend: **`127.0.0.1:17493`** — REST API, `/docs` (OpenAPI UI), `/mcp` (MCP server)
- Docker Compose maps the container to host **`127.0.0.1:17600`** so it can coexist
  with a native dev backend
- App data: `data/` in the repo (SQLite DB, voice profiles, generations) — gitignored
- Model cache: the standard HF cache (`~/.cache/huggingface`) unless `VOICEBOX_MODELS_DIR`
  is set

---

## 3. Fix the disk situation first

`/home` has 14 GB free; `/` has 83 GB. Move the heavy caches to `/`.

```bash
# Create a models directory on the roomy partition
sudo mkdir -p /opt/ai-models/huggingface
sudo chown -R "$USER:$USER" /opt/ai-models

# Point Voicebox (and huggingface_hub generally) at it
echo 'export VOICEBOX_MODELS_DIR=/opt/ai-models/huggingface' >> ~/.bashrc
echo 'export HF_HOME=/opt/ai-models/huggingface' >> ~/.bashrc
```

Fish shell (this machine's default):

```fish
set -Ux VOICEBOX_MODELS_DIR /opt/ai-models/huggingface
set -Ux HF_HOME /opt/ai-models/huggingface
```

`backend/config.py` reads `VOICEBOX_MODELS_DIR` at import time and sets `HF_HUB_CACHE`
from it, so this must be exported **before** the backend starts.

Optionally move the Rust build cache too — `target/` is the other multi-gigabyte offender:

```bash
sudo mkdir -p /opt/cargo-target && sudo chown -R "$USER:$USER" /opt/cargo-target
set -Ux CARGO_TARGET_DIR /opt/cargo-target   # fish
```

And reclaim some of that 24 GB `~/.cache`:

```bash
du -sh ~/.cache/* | sort -h | tail -20    # look before deleting
```

Even after this, the Python venv (~8–10 GB) still lands in `backend/venv` on `/home`.
Free at least **12 GB** there before running setup.

---

## 4. Install the missing toolchain

### System libraries (Tauri)

```bash
sudo apt update
sudo apt install -y \
  libwebkit2gtk-4.1-dev \
  libgtk-3-dev \
  libayatana-appindicator3-dev \
  librsvg2-dev \
  libxdo-dev \
  libasound2-dev \
  libssl-dev \
  build-essential curl wget file pkg-config
```

`libxdo-dev` and `libasound2-dev` matter specifically for the global dictation hotkey and
audio capture.

### Bun

```bash
curl -fsSL https://bun.sh/install | bash
```

Then add `~/.bun/bin` to `PATH` (fish: `fish_add_path ~/.bun/bin`). The repo pins
`bun@1.3.8` via `packageManager` and sets `engine-strict=true` — npm/yarn will refuse.

### Rust

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
```

### just

```bash
cargo install just          # after Rust is installed
# or: sudo apt install just  (24.04 ships a usable version)
```

Verify everything:

```bash
bun --version && rustc --version && just --version && python3 --version
```

---

## 5. Set up the repo

```bash
cd ~/devel/projects/voicebox
just setup
```

`just setup` runs two steps:

**`setup-python`** — creates `backend/venv`, then on Linux checks for
`/proc/driver/nvidia/version`. This laptop has it, so it installs CUDA PyTorch from
`https://download.pytorch.org/whl/cu128` before the rest of `backend/requirements.txt`.
It then installs `chatterbox-tts` and `hume-tada` with `--no-deps` (their pins conflict
with the rest of the stack), Qwen3-TTS from git, and the dev tools
(`pyinstaller ruff pytest pytest-asyncio`).

**`setup-js`** — `bun install` across the `app`, `tauri`, `web`, and `landing` workspaces.

Expect **15–40 minutes** on a first run; the CUDA torch wheels alone are ~2.5 GB of
download.

> **Driver note:** the setup script pulls `cu128` wheels while this machine runs driver
> 535.288.01 (CUDA 12.2). CUDA minor-version compatibility means these normally work, but
> if you hit `CUDA driver version is insufficient`, reinstall torch against an older
> index and re-run:
> ```bash
> backend/venv/bin/pip install torch torchaudio \
>   --index-url https://download.pytorch.org/whl/cu124 --force-reinstall
> ```

Initialize the database:

```bash
just db-init
```

---

## 6. Run it

Pick the mode that matches what you're doing.

| Goal | Command | Builds Rust? |
| --- | --- | --- |
| Full desktop app | `just dev` | yes — first build is slow |
| Frontend/backend work in the browser | `just dev-web` | no |
| Backend only | `just dev-backend` | no |
| Stop everything | `just kill` | — |

`just dev` and `just dev-web` reuse an already-running backend if one answers on
`127.0.0.1:17493/health`, so you can leave `just dev-backend` running in its own terminal.

**Start with `just dev-web`.** It gets you a working app in minutes instead of waiting on
a cold Rust compile, and it exercises the same frontend code.

### Docker alternative

Skips Bun, Rust, and the venv entirely:

```bash
docker compose up --build
```

Container-internal port 17493 is published on host **17600**. Note the compose file caps
the container at **4 CPUs and 8 GB RAM** and builds a **CPU-only** image — no CUDA. Fine
for Kokoro and LuxTTS, slow for anything larger. The `docker-compose.rocm.yml` overlay is
for AMD GPUs and does not apply to this NVIDIA laptop.

---

## 7. Verify

```bash
# Backend is alive
curl http://127.0.0.1:17493/health

# CUDA is actually visible to torch
backend/venv/bin/python -c "import torch; print(torch.cuda.is_available(), torch.cuda.get_device_name(0))"
# expect: True NVIDIA GeForce RTX 3050 Ti Laptop GPU

# Model status
curl http://127.0.0.1:17493/models/status

# Watch VRAM while generating
watch -n1 nvidia-smi
```

Open `http://127.0.0.1:17493/docs` for the full API surface.

Then, in the app: create a voice profile from a short audio sample, pick **Kokoro** as the
engine (smallest download, fastest first success), and generate a line. First generation
downloads the model, so it will be slower than the ones after it.

Run the test suite and linters:

```bash
just test     # pytest
just check    # Biome + ruff + tsc
```

---

## 8. Recommended configuration for this machine

| Setting | Pick | Why |
| --- | --- | --- |
| Everyday TTS | **Kokoro 82M** | 150 MB VRAM, 8 languages, 50 preset voices |
| Voice cloning | **LuxTTS** (English) or **Qwen 0.6B** (multilingual) | ~1–2 GB VRAM |
| Expressive / emotion tags | **Chatterbox Turbo** | ~1.5 GB VRAM, only engine reading `[laugh]`-style tags |
| Broad language coverage | **Chatterbox Multilingual** | ~3 GB — load nothing else alongside it |
| Transcription | **Whisper Turbo** | ~8× faster than Large, minimal quality loss |
| Refinement LLM | **Qwen3 0.6B** | 400 MB; step up to 1.7B only if VRAM allows |
| Avoid | Qwen 1.7B, TADA 1B/3B, Whisper Large | 4–8 GB VRAM — will OOM or fall back to CPU |

Unload models you're not using from **Settings → Models** rather than deleting the
download — it frees VRAM and keeps the files on disk.

---

## 9. Claude Code / MCP integration

The repo ships a pre-wired `.mcp.json` pointing at `http://127.0.0.1:17493/mcp`, and
`.claude/settings.local.json` already enables it. Start the dev backend, then run Claude
Code from inside this checkout and the `voicebox.speak`, `voicebox.transcribe`,
`voicebox.list_captures`, and `voicebox.list_profiles` tools are available.

Bind a voice to the `claude-code` client in **Settings → MCP**.

---

## 10. Troubleshooting

**`just setup` fails on disk space** — the venv needs ~10 GB on `/home`. Free space
(section 3) or relocate the venv to `/` and symlink `backend/venv` to it.

**`torch.cuda.is_available()` is `False`** — check `nvidia-smi` works, confirm the venv
got the CUDA wheels (`backend/venv/bin/pip show torch` should show a `+cu` build), and
re-run the `cu124` fallback from section 5.

**CUDA out of memory** — 4 GB fills fast. Unload other models in Settings → Models, drop
to a smaller engine, or shorten the text so auto-chunking generates smaller pieces.

**Tauri build fails on missing `.pc` files** — a `libwebkit2gtk-4.1-dev` / `libgtk-3-dev`
gap. Re-run the `apt install` in section 4, then `cd tauri/src-tauri && cargo clean`.

**Port 17493 already in use** — `just kill`, or `lsof -i :17493` to find the holder.

**Models re-download every run** — `VOICEBOX_MODELS_DIR` wasn't exported in the shell that
started the backend. It's read at import time in `backend/config.py`.

**Global dictation hotkey doesn't paste** — expected. Auto-paste is macOS-only today;
Linux `uinput`/AT-SPI support is on the roadmap. Transcription itself works, and the text
lands in the Captures tab.

---

## Reference

- `README.md` — features, API, tech stack
- `CONTRIBUTING.md` — contribution workflow
- `docs/content/docs/developer/setup.mdx` — upstream setup guide
- `docs/content/docs/developer/model-management.mdx` — model table source
- `docs/content/docs/overview/troubleshooting.mdx` — full troubleshooting guide
- `justfile` — every command, with the platform-specific branches
- `CLAUDE.md` — conventions and gotchas for working in this repo
