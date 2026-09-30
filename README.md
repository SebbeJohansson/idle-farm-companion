# idle-game-companion

The local AI services behind idle-game's digger: the content it generates
and the checks that content has to pass. For now this is a development
setup; the plan's player-facing companion (one port, installer, health
checks) grows out of it later.

| Service | Role | Runs | Address (from WSL) |
| --- | --- | --- | --- |
| [Laya](https://github.com/NandhaKishorM/laya) | Decides: scores generated biomes (fits its mood? cozy? safe?) | Docker, in WSL, CPU | `http://localhost:8000` |
| [Ollama](https://ollama.com) + Gemma 4 | Writes: generates biome JSON | Natively on Windows, GPU | `http://<windows host>:11434` |

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
