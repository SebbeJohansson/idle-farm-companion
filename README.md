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

Docker Engine installed directly in the WSL distro, started by systemd:

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER      # then open a new terminal
sudo systemctl enable --now docker
```

Laya then runs whenever WSL is running. If Docker Desktop is also installed
on Windows, keep its WSL integration for this distro turned off so the two
don't fight over the `docker` command.

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
