# idle-farm-companion

The local AI services behind [Idle Farm](https://github.com/SebbeJohansson/idle-farm)'s digger: the content it generates
and the checks that content has to pass. For now this is a development
setup; the plan's player-facing companion (one port, installer, health
checks) grows out of it later.

| Service | Role | Runs | Address (from WSL) |
| --- | --- | --- | --- |
| [Laya](https://github.com/NandhaKishorM/laya) | Decides: scores generated biomes (fits its mood? cozy? safe?) | Docker Desktop via WSL, CPU | `http://localhost:8000` |
| [Ollama](https://ollama.com) + Gemma 4 | Writes: generates biome JSON | Natively on Windows, GPU | `http://<windows host>:11434` |
| Bridge (Caddy) | One port for the game: `/ollama`, `/laya`, `/art`, `/status` | Docker via WSL | `http://127.0.0.1:7870` |
| [stable-diffusion.cpp](https://github.com/leejet/stable-diffusion.cpp) + SDXL + [pixel-art-xl](https://huggingface.co/nerijs/pixel-art-xl) | Draws: pixel-art sprites (Automatic1111 API, `/sdapi/v1/txt2img`) | Docker via WSL, GPU | `http://localhost:7860` |

## Laya

Needs Docker in WSL (see below). Then:

```bash
docker compose up -d --wait
curl -s localhost:8000/health
```

The first start builds the image from Laya's pinned release and downloads
the English checkpoint (a few GB), so allow several minutes. After that it
starts in seconds, and `restart: unless-stopped` brings it back whenever
Docker starts, until you `docker compose down`.

Settings go in a `.env` file next to `compose.yaml` (see `.env.example`).

### Docker in WSL

Docker Desktop with its WSL integration turned on for this distro
(Settings → Resources → WSL integration → your distro → Apply & restart).
The container runs in Docker Desktop's VM, and its published port shows up
on WSL's `localhost`. Also turn on Settings → General → "Start Docker
Desktop when you sign in": `restart: unless-stopped` only brings Laya back
once Docker itself is running.

Don't run `docker compose` from PowerShell instead: the port would land on
Windows' loopback, which WSL can't reach.

## Art (sprites)

`docker compose up -d` starts it with Laya. On first start `art-models`
downloads ~7.5 GB into a volume — SDXL base 1.0, its fp16-safe VAE, the
pixel-art-xl LoRA (style) and the LCM LoRA (8 steps instead of 25+) — and
`art` loads SDXL onto the GPU (~6.6 GB VRAM) and serves on `localhost:7860`.
LoRAs are chosen per request: the game sends them in the API's `lora`
field (this server ignores `<lora:…>` tags in prompts).

It's stable-diffusion.cpp rather than ComfyUI on purpose: current PyTorch
builds no longer support Pascal GPUs (GTX 10-series), and ComfyUI needs
PyTorch. stable-diffusion.cpp is built on ggml, like Ollama, which runs on
them fine. GPU access needs Docker Desktop's NVIDIA support (a quick check:
`docker run --rm --gpus all ubuntu nvidia-smi`). If the CUDA image doesn't
work on a card, switch the image tag to `master-vulkan`.

Measured on a GTX 1080 Ti: ~26 s per 768×768 image (LCM, 8 steps); 1024 px
~45 s. An SD 1.5 pixel-art model was tried first: 12 s, but it couldn't
draw named things (wheat came out as a signpost).

Licenses: SDXL and the LCM LoRA are CreativeML OpenRAIL++-M, pixel-art-xl
CreativeML OpenRAIL-M — output can be used in a game.

## Bridge

`bridge` (Caddy, `127.0.0.1:7870`) puts every service behind one local
port, so the **downloaded desktop app** can use them without the game's dev
server: `/ollama/*` (Ollama on the host, via `host.docker.internal`),
`/laya/*`, `/art/*`, and `/status` (which model to use). Only the game's own
origins may call it (`tauri://localhost`, `tauri.localhost`, the dev server);
any other website gets 403, so a page open in your browser can't use your
local AI. It strips the services' own CORS headers (stable-diffusion.cpp
sends `*`, and two Allow-Origin headers make browsers refuse the answer)
and the Origin header (Ollama refuses origins it doesn't know). Settings:
`BRIDGE_PORT`, `OLLAMA_UPSTREAM`, `OLLAMA_MODEL` in `.env`.

## Ollama (Windows)

Runs outside WSL for GPU speed. For WSL to reach it:

1. Set the Windows user environment variable `OLLAMA_HOST=0.0.0.0` and
   restart Ollama from the tray.
2. Allow the port from WSL only (PowerShell as admin):

   ```powershell
   New-NetFirewallRule -DisplayName "Ollama (WSL dev)" -Direction Inbound -Protocol TCP -LocalPort 11434 -RemoteAddress 172.16.0.0/12 -Action Allow
   ```

3. From WSL, the Windows host is the default gateway, which can change
   between reboots:

   ```bash
   curl -s $(ip route | awk '/default/{print $3}'):11434/api/tags
   ```

idle-game finds it that way on its own; set `OLLAMA_URL` to override.
