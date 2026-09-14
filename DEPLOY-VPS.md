# DEPLOY-VPS.md — Hybrid Open-Model AI Infrastructure

> Deployment guide for **VPS 203.24.92.154** (user: reihan)
> Installs Ollama, pulls lightweight models, and sets up Docker Compose for hybrid AI serving.

---

## VPS Specs

| Resource | Value |
|---|---|
| CPU | 2 vCPU (Intel Xeon Cascadelake) |
| RAM | 3.8 GB total, ~2.2 GB available |
| Disk | 35 GB (45% used, 19 GB free) |
| OS | Debian 13 (trixie) |
| Virtualization | KVM |
| Existing Services | Caddy (80/443), desa-depok-website (8080), phishing-detector (8000), adguardhome |
| Ports Free | 11434 (Ollama), 8787 (OmniGate), 7780 (Sentinel) |

---

## Prerequisites

- SSH access: `ssh reihan@203.24.92.154`
- Docker already installed and running (verify: `docker ps`)
- ~2 GB RAM available for AI services

---

## Step 1: Install Ollama

```bash
ssh reihan@203.24.92.154 << 'EOF'
# Install Ollama (systemd service)
curl -fsSL https://ollama.com/install.sh | sh

# Verify
ollama --version
systemctl status ollama
EOF
```

---

## Step 2: Pull Lightweight Models

> **Critical:** VPS has only 3.8GB RAM. Choose models carefully.

```bash
ssh reihan@203.24.92.154 << 'EOF'
# Primary chat model — good balance of speed and quality
ollama pull qwen2.5:1.5b

# Fast fallback — very quick responses
ollama pull llama3.2:1b

# Embedding model for RAG
ollama pull nomic-embed:latest

# List all models
ollama list

# Check memory usage after pulling
free -h
EOF
```

### Model Sizes Reference

| Model | RAM Needed | Use Case |
|---|---|---|
| `llama3.2:1b` | ~0.8 GB | Ultra-fast fallback, simple tasks |
| `qwen2.5:1.5b` | ~1.2 GB | **Primary model** — best quality/speed ratio |
| `mistral:7b-instruct` | ~4.5 GB | ❌ Too large for this VPS — use cloud fallback |
| `nomic-embed` | ~0.3 GB | Embeddings for RAG |

---

## Step 3: Create Docker Compose for Hybrid AI Stack

### Create directory

```bash
ssh reihan@203.24.92.154 << 'EOF'
mkdir -p /home/reihan/projects/ai-hybrid
cd /home/reihan/projects/ai-hybrid
EOF
```

### `docker-compose.yml`

```yaml
version: "3.8"

services:
  # Ollama runs natively (not in Docker) for better RAM access
  # Docker container below is an API bridge for HTTP access

  ollama-bridge:
    image: alpine:latest
    container_name: ollama-bridge
    restart: unless-stopped
    command: >
      sh -c "
        while ! wget -qO- http://host.docker.internal:11434/health 2>/dev/null; do
          sleep 2
        done
        echo 'Ollama ready'
      "
    depends_on: []
    network_mode: host

  omnigate:
    build:
      context: /home/reihan/github/omnigate
      dockerfile: Dockerfile
    container_name: omnigate
    restart: unless-stopped
    env_file:
      - /home/reihan/github/omnigate/.env
    environment:
      - OMNIGATE_API_KEY=${OMNIGATE_API_KEY}
      - PORT=8787
    ports:
      - "127.0.0.1:8787:8787"
    volumes:
      - omnigate_data:/app/.data
    depends_on:
      - ollama-bridge
    healthcheck:
      test: ["CMD", "wget", "-q", "-O", "-", "http://127.0.0.1:8787/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 10s

networks:
  default:
    driver: bridge

volumes:
  omnigate_data:
```

### `.env` (at `/home/reihan/projects/ai-hybrid/.env`)

```env
OMNIGATE_API_KEY=your-secure-api-key-here
```

---

## Step 4: Configure Caddy Reverse Proxy

Add AI routes to existing Caddy config:

```bash
ssh reihan@203.24.92.154 << 'EOF'
cat /home/reihan/docker/caddy/Caddyfile
EOF
```

Add these routes to the Caddyfile (or a new site block):

```caddy
ai.reihanpandanarang.my.id {
    reverse_proxy 127.0.0.1:8787
}

ollama.reihanpandanarang.my.id {
    reverse_proxy 127.0.0.1:11434
}
```

Or for local-only access (no domain needed), the services are already accessible at `http://203.24.92.154:8787` and `http://203.24.92.154:11434`.

---

## Step 5: Verify Everything Works

```bash
# Ollama check
curl -s http://203.24.92.154:11434/api/tags | python3 -m json.tool

# Test local model
curl -s http://203.24.92.154:11434/api/generate \
  -d '{"model": "qwen2.5:1.5b", "prompt": "Halo, siapa kamu?", "stream": false}' \
  | python3 -m json.tool

# Test embedding
curl -s http://203.24.92.154:11434/api/embeddings \
  -d '{"model": "nomic-embed", "prompt": "hello world"}' \
  | python3 -c "import sys,json; d=json.load(sys.stdin); print(f'Embedding dims: {len(d[\"embedding\"])}')"

# OmniGate health
curl -s http://203.24.92.154:8787/health

# Docker status
docker compose -f /home/reihan/projects/ai-hybrid/docker-compose.yml ps
docker ps
```

---

## Step 6: OmniGate Provider Config for Hybrid Routing

Edit `/home/reihan/github/omnigate/src/config/provider.registry.yaml` to add Ollama as primary provider:

```yaml
providers:
  - name: "ollama-local"
    base_url: "http://127.0.0.1:11434/v1"
    models:
      - "qwen2.5:1.5b"
      - "llama3.2:1b"
    api_key_env: "OLLAMA_API_KEY"   # Ollama has no key, use dummy "ollama"
    family: "chat"
    quality_score: 0.75
    rate_limit: 100
    rpd: 60
    cooldown_threshold: 3
    features:
      - chat
      - stream
    allow_paid: false

  - name: "openai-cloud"
    base_url: "https://api.openai.com/v1"
    models:
      - "gpt-4o-mini"
      - "gpt-4o"
    api_key_env: "PROVIDER_A_API_KEY"
    family: "chat"
    quality_score: 0.95
    rate_limit: 1000
    rpd: 1000
    features:
      - chat
      - stream
      - json
      - tools
    allow_paid: true
```

Edit `/home/reihan/projects/ai-hybrid/.env` (or wherever OmniGate .env is):

```env
OLLAMA_API_KEY=ollama
OPENAI_API_KEY=your-openai-key-here
PROVIDER_A_API_KEY=your-openai-key-here
```

---

## Step 7: Service Management

```bash
# Start all AI services
cd /home/reihan/projects/ai-hybrid/
docker compose up -d

# Check logs
docker compose logs -f
docker compose logs -f omnigate

# Restart
docker compose restart
docker compose down && docker compose up -d

# Monitor resources
watch -n 5 'docker stats --no-stream; echo "---"; free -h; echo "---"; ollama list'

# Stop everything
docker compose down
```

---

## Step 8: Systemd Service for Ollama (ensure it starts on boot)

```bash
ssh reihan@203.24.92.154 << 'EOF'
# Check if Ollama service exists
systemctl status ollama

# If not, create it
sudo tee /etc/systemd/system/ollama.service > /dev/null << 'SERVICE'
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
ExecStart=/usr/bin/ollama serve
User=reihan
Restart=always
RestartSec=3
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

[Install]
WantedBy=default.target
SERVICE

sudo systemctl daemon-reload
sudo systemctl enable ollama
sudo systemctl start ollama
EOF
```

---

## Monitoring & Maintenance

```bash
# Daily: check disk (Docker images accumulate)
docker image prune -f

# Weekly: check model performance
curl -s http://127.0.0.1:8787/v1/models \
  -H "Authorization: Bearer $OMNIGATE_API_KEY" | python3 -m json.tool

# Monthly: update models
ollama pull qwen2.5:1.5b
ollama pull llama3.2:1b

# Quarterly: review cost/quality metrics
# Check OmniGate SQLite stats: .data/omnigate.sqlite
```

---

## Troubleshooting

| Problem | Solution |
|---|---|
| OOM during model load | Reduce to `llama3.2:1b` or stop other Docker containers |
| Ollama not responding | `systemctl restart ollama` |
| OmniGate can't reach Ollama | Check `curl http://127.0.0.1:11434/health` from host |
| Slow responses | Normal for CPU inference with 1.5B models (2-10 tokens/sec) |
| Port conflict | Change OmniGate port in `docker-compose.yml` |
| Caddy not routing | Verify Caddyfile syntax: `caddy validate` |
| Disk full | `docker image prune -a -f` + remove unused models |

---

## Architecture Diagram

```
 User Request
      │
      ▼
 Caddy :80/443
      │
 ┌────┴────┐
 │ Routing │
 └────┬────┘
      │
 ┌────┴──────────┐     ┌──────────────┐
 │ OmniGate :8787 │────>│ Ollama :11434│
 │ (router)       │     │ (local models│
 │                │     │  qwen2.5:1.5b│
 │ If local fails │────>│              │
 │  ↓             │     └──────────────┘
 │ Cloud API      │
 │ (OpenAI etc.)  │     ┌──────────────┐
 └────────────────┘     │ FastAPI apps │
                        │ :8000, :8080 │
                        └──────────────┘
```

---

## Cost Estimate

| Component | Cost |
|---|---|
| VPS (existing) | ~$5-10/month (included) |
| Ollama models | Free (local) |
| Cloud fallback (OpenAI) | Pay-per-use, ~$0.15/1M tokens (gpt-4o-mini) |
| **Total baseline** | **~$5-10/month** |

---

*Last updated: September 2026*
