# Agent Notes — raindragon14

## Identity

This is the **personal portfolio and AI infrastructure hub** of Muhammad Reihan Pandanarang (raindragon14). It hosts README, deployment configs, and documentation for all linked projects. AI Engineer profile: hybrid open-model systems, RAG, agents, ML engineering.

---

## Sources (read in this order)

1. **This README** — current profile, project summaries, tech stack
2. **Individual project READMEs** — each sub-project has its own architecture and quick-start
3. **`/home/reihan/github/` sibling repos:**
   - `omnigate/` — OpenAI-compatible gateway (Bun/TypeScript)
   - `sentinel/` — personal intelligence brief (Go)
   - `axos/` — AI OS for edge (Rust, 13 crates)
   - `doc-intelligence-hub/` — enterprise RAG (Python/FastAPI)
   - `OpenData-Jatim/` — risk scoring (Python/ML)
4. **VPS deployment guide:** `DEPLOY-VPS.md` (if present)
5. **GitHub Actions:** check `.github/workflows/` in each repo for CI/CD patterns

---

## Commands

```sh
# Clone all projects
for repo in omnigate sentinel axos doc-intelligence-hub OpenData-Jatim; do
  git clone https://github.com/raindragon14/$repo.git ../$repo 2>/dev/null || echo "$repo already exists"
done

# Portfolio deployment (VPS)
ssh reihan@203.24.92.154 "bash -s" < deploy-hybrid-model.sh
```

---

## Hybrid Open-Model Architecture (VPS: 203.24.92.154)

```
┌─────────────────────────────────────────────────────────┐
│                    VPS 203.24.92.154                     │
│                                                          │
│  ┌──────────┐    ┌──────────────┐    ┌────────────────┐  │
│  │  Ollama   │    │  OmniGate    │    │  Caddy Reverse │  │
│  │ :11434    │───>│  :8787       │───>│  Proxy :80/443 │  │
│  │           │    │  (routing)   │    │                │  │
│  │ Qwen 2.5  │    │              │    │  Routes:       │  │
│  │ Llama 3.2 │    │  Auto-fallback│   │  /api/* → AI   │  │
│  │ Mistral   │    │  Cost-track  │    │  /sentinel → 7780││
│  │ Phi-3     │    │              │    │  /doc-intel → 8000││
│  └──────────┘    └──────────────┘    └────────────────┘  │
│         │                                       │         │
│    Cloud Fallback                            Docker      │
│    (OpenAI/GLM)                             Compose     │
└─────────────────────────────────────────────────────────┘
```

### Design Principles

- **Local-first:** Ollama handles 80%+ of requests (cost $0, privacy, latency)
- **Cloud fallback:** OmniGate routes to OpenAI/GLM when local model fails or is inadequate
- **Model size constraint:** VPS has 3.8GB RAM → max model ~1.5B params (Qwen 2.5 1.5B, Phi-3)
- **Evaluation-first:** Every model switch is measured (latency, quality, cost) before routing

---

## Tech Stack Rules

| Decision | Rule |
|---|---|
| New AI project | Must support hybrid local+cloud from day 1 |
| Orchestration | Custom Python (FastAPI) or Bun (Hono) — no LangChain monolith |
| Embeddings | Ollama embeddings or TEI — no closed-source embeddings |
| Vector DB | pgvector (Postgres) for production, FAISS for dev |
| Evaluation | Must have eval harness before deploying any model |
| Containerization | Docker Compose minimum; no bare-metal installs |
| Secret management | `.env` files only, never committed. `.env.example` required |
| Testing | pytest (Python) / bun test (Bun) / cargo test (Rust) — must pass before merge |

---

## VPS Operations

### Quick Start (Hybrid Model)

```sh
# SSH to VPS
ssh reihan@203.24.92.154

# Install Ollama (one-time)
curl -fsSL https://ollama.com/install.sh | sh

# Pull lightweight models for 3.8GB RAM
ollama pull qwen2.5:1.5b      # primary chat model
ollama pull llama3.2:1b       # fast fallback
ollama pull mistral:7b-instruct-v0.3  # quality fallback (if RAM allows)
ollama pull nomic-embed       # embedding model

# Verify
ollama list
curl http://localhost:11434/api/tags
```

### Docker Compose (AI Services)

```sh
# Navigate to AI deployment directory on VPS
cd /home/reihan/projects/ai-hybrid/

# Start all services
docker compose up -d --build

# Check status
docker compose ps

# View logs
docker compose logs -f omnigate
docker compose logs -f ollama-bridge
```

### Maintenance

```sh
# Pull latest models
ollama pull qwen2.5:1.5b

# Restart AI stack
docker compose restart omnigate ollama-bridge

# Check resource usage
docker stats --no-stream

# Backup configs
tar -czf backup-$(date +%Y%m%d).tar.gz /home/reihan/docker/ /home/reihan/.env
```

---

## Repository Structure

```
raindragon14/
├── README.md          ← THIS FILE: profile + links + deployment
├── AGENTS.md          ← Agent notes (you are here)
├── DEPLOY-VPS.md      ← VPS deployment guide (detailed)
└── .git/
```

---

## Key Contacts & URLs

- **VPS:** 203.24.92.154 (user: reihan, SSH key in ~/.ssh/id_ed25519)
- **GitHub:** https://github.com/raindragon14
- **LinkedIn:** https://linkedin.com/in/mreihanpandanarang
- **Email:** reihanpandanarang@gmail.com
- **Website:** reihanpandanarang.my.id

---

## Gotchas

- **RAM is scarce (3.8GB):** Do not run 7B+ models locally. Stick to ≤1.5B or use cloud fallback.
- **No GPU:** CPU inference only. Model choice matters — Qwen 2.5 1.5B is the sweet spot.
- **Ollama is not persistent:** After VPS reboot, Ollama service must be restarted manually unless using systemd service.
- **Caddy is host-network:** It binds to ports 80/443 directly (network_mode: host in compose). Do not conflict.
- **SSH keys:** If `~/.ssh/id_ed25519` is corrupted, regenerate: `ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519` and re-add to `~/.ssh/authorized_keys`.
- **Disk:** 35GB total, 45% used. Monitor with `df -h /`. Clean Docker images regularly: `docker image prune -f`.
- **Swap:** 1GB swap available but slow. If OOM, reduce model size or add zram.
