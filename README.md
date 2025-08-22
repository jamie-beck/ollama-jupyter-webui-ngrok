## Ollama + Open WebUI Stack (with Watchtower, Dozzle, Jupyter, and ngrok)

This repository contains a Docker Compose stack for running:

- Ollama API server
- Open WebUI (frontend for local/remote LLMs)
- Jupyter (datascience notebook image)
- Watchtower (automatic image updates)
- Dozzle (container logs viewer)
- ngrok (optional public tunnel to Open WebUI)

This stack is intended to run as a standard Docker Compose deployment (non‑Swarm).

### What you get

- Ollama served on `http://localhost:11434` (API)
- Open WebUI on `http://localhost:3000` (UI)
- Dozzle on `http://localhost:9999` (logs)
- ngrok inspector on `http://localhost:4040` (when enabled)
- Jupyter container is included; by default it is not exposed on a host port (see notes below)

---

## Prerequisites

- Docker and Docker Compose v2
- (Optional, recommended) NVIDIA GPU with recent drivers
- (Optional, GPU) NVIDIA Container Toolkit installed and working with Docker
- For ngrok: an ngrok account and auth token

On Debian/Ubuntu for NVIDIA Toolkit:

```bash
sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

Verify Docker can see your GPU:

```bash
docker run --rm --gpus all nvidia/cuda:12.3.2-base-ubuntu22.04 nvidia-smi
```

---

## Files

- `docker-compose.yaml`: service definitions
- `env.example`: template for required environment variables
- `ollama-model-update.service`: systemd unit to update all downloaded Ollama models on a schedule
- `ollama-model-update.timer`: systemd timer definition for the above service

---

## Data storage

- Ollama data is a host-mapped folder via `OLLAMA_DATA_PATH` → mounted at `/root/.ollama` in the `ollama` container. This is ideal for placing large model files on an external SSD/HDD. Set `OLLAMA_DATA_PATH` to an absolute path on the host (e.g., `/mnt/nas/ollama/ollama-data`, `/mnt/c/Users/<you>/ollama-data`). Ensure the path exists and is shared/accessible to Docker.
- Open WebUI data is stored in a Docker named volume `open-webui`. This data (users, settings, small assets, chat logs) is typically much smaller than Ollama model data. You can keep it as a named volume or convert it to a host bind if you prefer.

If you want Open WebUI to also use a host path instead of a named volume, replace its volume entry with a bind mount, for example:

```yaml
  open-webui:
    volumes:
      - /path/to/openwebui-data:/app/backend/data
```

---

## Configuration

1) Create your `.env` file

```bash
cp env.example .env
```

Then edit `.env` and set:

- `NGROK_AUTHTOKEN`: your ngrok auth token
- `NGROK_URL`: a reserved domain or let ngrok choose one (e.g. leave empty and comment the `--url` line, or set to something like `my-ollama.ngrok.app` if you have a reserved domain)
- `OLLAMA_DATA_PATH`: absolute path on the host for Ollama data, e.g. `/mnt/nas/ollama/ollama-data`

2) Create the external volume used by Open WebUI (first time only)

```bash
docker volume create open-webui
```

3) GPU notes

- This runs on regular Docker Compose (non‑Swarm). GPU access is enabled via `gpus: all` on the `ollama` service in `docker-compose.yaml`.
- If your GPU is not detected, try one of the following:
  - Ensure NVIDIA Container Toolkit is installed and Docker is configured for NVIDIA
  - Uncomment `runtime: nvidia` under the `ollama` service
  - Add env vars on the service: `NVIDIA_VISIBLE_DEVICES=all` and `NVIDIA_DRIVER_CAPABILITIES=compute,utility`

If you run CPU-only, the stack still works; Ollama will use CPU.

4) Jupyter exposure (optional)

By default, Jupyter does not expose a port to the host. This is fine for Open WebUI integration because Open WebUI can reach Jupyter over the internal Compose network via `http://jupyter:8888`.

To access Jupyter from your host browser, add to the `jupyter` service in `docker-compose.yaml`:

```yaml
    ports:
      - "8888:8888"
```

Also change the token in the compose file or via env to something unique:

```yaml
    environment:
      - JUPYTER_TOKEN=<secure-token>
```

---

## Quick start

```bash
# 1) Prepare env
cp env.example .env
$EDITOR .env

# 2) Create external volume for Open WebUI
docker volume create open-webui

# 3) Launch stack
docker compose up -d

# 4) Verify containers
docker compose ps
```

Open:

- Open WebUI: `http://localhost:3000`
- Ollama API: `http://localhost:11434`
- Dozzle: `http://localhost:9999` (view logs)
- ngrok inspector: `http://localhost:4040` (when ngrok is running)

Example API checks:

```bash
curl http://localhost:11434/api/tags

curl -s http://localhost:11434/api/generate -d '{
  "model": "llama3.1:8b",
  "prompt": "Say hello"
}' | jq -r 'select(.response != null) | .response'
```

To pull a model inside the container:

```bash
docker exec -it ollama ollama pull llama3.1:8b
```

### Configure Jupyter as the Code Executor in Open WebUI

You can use the Jupyter container to execute code blocks within Open WebUI chats.

1) Sign in to Open WebUI (`http://localhost:3000`) as an admin
2) Go to Admin Panel → Settings → Tools (or Code Execution)
3) Enable Code Execution, select Jupyter as the engine, and configure:
   - Base URL: `http://jupyter:8888`
   - Token: the value of `JUPYTER_TOKEN` from `docker-compose.yaml` (default is `ollama`)
   - Working directory (optional): `/home/jovyan/work`
   - Timeout / limits (optional): set to your preference
4) Save and Test the connection. It should succeed without exposing any additional host ports because services can talk over the default Compose network.
5) In a chat, enable tools or code execution as needed; code blocks will run inside the Jupyter container.

---

## Service descriptions

- Ollama (`ollama/ollama`)
  - API on port `11434`
  - Data persisted at host path `OLLAMA_DATA_PATH` (via `ollama` volume)
  - Default env tuned for a single loaded model / parallel job to reduce VRAM

- Open WebUI (`ghcr.io/open-webui/open-webui:main`)
  - UI on port `3000`
  - Stores its data in the external Docker volume `open-webui`
  - First run will prompt for admin account creation

- Jupyter (`jupyter/datascience-notebook`)
  - Persists notebooks in the `jupyter-data` volume (`/home/jovyan/work`)
  - To expose on host, add a ports mapping as described above
  - Default token is set in the compose file; change it for security

- Watchtower (`containrrr/watchtower`)
  - Checks for image updates every 10 minutes
  - Cleans up old images automatically

- Dozzle (`amir20/dozzle`)
  - Minimal web UI to tail/view container logs on port `9999`

- ngrok (`ngrok/ngrok`)
  - Tunnels Open WebUI to the public internet
  - Requires `NGROK_AUTHTOKEN` and optional `NGROK_URL`

---

## Systemd: scheduled Ollama model updates (optional)

This repository includes a oneshot systemd service and a timer to periodically update all downloaded Ollama models.

1) Copy files and reload systemd

```bash
sudo cp ollama-model-update.service /etc/systemd/system/
sudo cp ollama-model-update.timer /etc/systemd/system/
sudo systemctl daemon-reload
```

2) Enable and start the timer

```bash
sudo systemctl enable --now ollama-model-update.timer
systemctl list-timers | grep ollama-model-update
```

3) Check logs

```bash
journalctl -u ollama-model-update.service -f
```

Notes:

- In the provided timer file, ensure the unit name matches the service (should be `Unit=ollama-model-update.service`).
- In the provided service file, `ExecStart` runs `ollama pull` on the host. If Ollama is only available inside the container, use a wrapped `docker exec` to perform pulls inside the `ollama` container, e.g.:

```ini
ExecStart=/usr/bin/docker exec ollama bash -lc 'ollama list | awk '\''NR>1 {print $1}'\'' | xargs -I {} bash -lc 'echo Updating model: {}; ollama pull {}; echo --''
```

Adapt the user and paths to your environment as needed.

---

## Environment variables

Defined in `.env` (copied from `env.example`):

- `NGROK_AUTHTOKEN`: ngrok auth token
- `NGROK_URL`: reserved domain or leave unset
- `OLLAMA_DATA_PATH`: absolute host path to store Ollama models and data

Other important settings are in `docker-compose.yaml`:

- `OLLAMA_KEEP_ALIVE`, `OLLAMA_NUM_PARALLEL`, `OLLAMA_MAX_LOADED_MODELS`
- Jupyter `JUPYTER_TOKEN`

---

## Maintenance

- Update images manually: `docker compose pull && docker compose up -d`
- Update models manually: `docker exec -it ollama ollama pull <model:tag>`
- Clean dangling images: `docker image prune -f`

Watchtower will automatically update running containers on a schedule. If you prefer manual control, remove or disable the Watchtower service.

---

## Troubleshooting

- Ports already in use: change host ports in `docker-compose.yaml`
- Permission denied on `OLLAMA_DATA_PATH`: ensure the directory exists and is writable by Docker
- GPU not detected: verify `nvidia-smi` in a CUDA container and that Docker is configured for NVIDIA runtime
- Open WebUI shows empty models: pull models inside the `ollama` container, then refresh
- ngrok does not start: ensure `NGROK_AUTHTOKEN` is set; if using a reserved domain, ensure `NGROK_URL` is correct

---

## License

No license is specified in this repository. Consider adding one if you plan to share or collaborate.


