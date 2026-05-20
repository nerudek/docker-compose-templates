---
name: docker-compose
description: Production-ready Docker Compose templates for web apps, databases, reverse proxies, and AI model serving — one command to spin up your entire stack
version: 1.0.0
author: nerudek
compatible-with: hermes-agent, claude-code, openclaw
---

# Docker Compose — Ready-to-Deploy Stack Templates

## Problem

Setting up a development or production stack means installing PostgreSQL, Redis, Nginx, and your app separately. Configuration drift between dev and prod causes bugs. Onboarding new developers takes hours of setup. Containerizing AI models with GPU access is a nightmare.

## Solution

Pre-built Docker Compose templates for common stacks. One `docker compose up -d` and everything runs. Identical environments on Mac, Linux, and VPS. GPU passthrough for LLM inference included.

## Templates

### 1. Web App + PostgreSQL + Redis + Nginx

```yaml
version: '3.8'
services:
  app:
    build: .
    ports: ['3000:3000']
    environment:
      DATABASE_URL: postgresql://user:pass@db:5432/app
      REDIS_URL: redis://cache:6379
    depends_on: [db, cache]
    restart: unless-stopped
    volumes: ['./uploads:/app/uploads']

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: app
    volumes: ['pgdata:/var/lib/postgresql/data']
    restart: unless-stopped

  cache:
    image: redis:7-alpine
    volumes: ['redisdata:/data']
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports: ['80:80', '443:443']
    volumes: ['./nginx.conf:/etc/nginx/nginx.conf', './ssl:/etc/nginx/ssl']
    depends_on: [app]
    restart: unless-stopped

volumes:
  pgdata:
  redisdata:
```

### 2. LM Studio / llama.cpp with GPU

```yaml
version: '3.8'
services:
  llama:
    image: ghcr.io/ggerganov/llama.cpp:full
    ports: ['8080:8080']
    volumes:
      - /Volumes/2TB_APFS/models:/models:ro
    command: >
      --server --port 8080
      --model /models/qwen3.5-27b.Q4_K_M.gguf
      --n-gpu-layers 99
      --ctx-size 32768
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: unless-stopped
```

### 3. Monitoring Stack (Prometheus + Grafana)

```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:latest
    ports: ['9090:9090']
    volumes: ['./prometheus.yml:/etc/prometheus/prometheus.yml', 'prometheus_data:/prometheus']
    command: ['--config.file=/etc/prometheus/prometheus.yml', '--storage.tsdb.path=/prometheus']

  grafana:
    image: grafana/grafana:latest
    ports: ['3001:3000']
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes: ['grafana_data:/var/lib/grafana']
    depends_on: [prometheus]

volumes:
  prometheus_data:
  grafana_data:
```

### 4. Development Hot-Reload Setup

```yaml
services:
  dev:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports: ['5173:5173']
    volumes:
      - ./:/app
      - /app/node_modules  # anonymous volume — don't overwrite
    environment:
      NODE_ENV: development
    command: npm run dev -- --host
```

## Key Patterns

- **Health checks**: `curl -f http://localhost:3000/health || exit 1` 
- **Secrets**: Use `.env` file, never hardcode passwords
- **Restart policy**: `unless-stopped` for production, `no` for one-shot tasks
- **Volumes**: Named volumes for databases, bind mounts for code in dev
- **Networks**: Docker Compose auto-creates a network; all services reach each other by name

## Common Pitfalls

1. Running as root inside container — use `USER node` in Dockerfile
2. Forgetting `--host` flag in Vite dev server
3. Database password in docker-compose.yml instead of .env
4. Port already in use — check with `lsof -i :PORT`
5. Bind mount on Mac is slow — use `:cached` or `:delegated` flag

## FAQ

**Q: How do I manage secrets properly?**
Use `.env` file for local dev, Docker secrets or external secret manager in production. NEVER commit `.env`.

**Q: Can I run this on a VPS with 1GB RAM?**
Yes. Use Alpine images, limit container memory with `mem_limit: 256m`, and avoid running PostgreSQL (use SQLite or external DB).

**Q: How do I update containers?**
`docker compose pull && docker compose up -d --build`.

**Q: My database migrations — when do they run?**
Add an init container or entrypoint script: `npm run migrate && npm start`.

**Q: GPU passthrough not working on Mac?**
Correct — Docker on Mac has no GPU passthrough. Use this template on Linux with NVIDIA Container Toolkit.

**Q: How to back up database volumes?**
`docker compose exec db pg_dump -U user app > backup.sql` or use `pg_dump` from host.

**Q: Can I run multiple instances on different ports?**
Use `docker compose -p project2 up -d` with a different project name.

**Q: How to expose to the internet securely?**
Put Cloudflare Tunnel or Tailscale Funnel in front, never expose raw Docker ports to 0.0.0.0.

**Q: What about Let's Encrypt for HTTPS?**
Use Caddy or Traefik as reverse proxy instead of Nginx — they auto-handle TLS certs.

**Q: Containers keep restarting in a loop — how to debug?**
`docker compose logs app --tail 50` shows the error. Check that dependencies are ready (use `depends_on` with `condition: service_healthy`).

**Q: How to share code between projects via Docker?**
Use a shared Docker network: `docker network create shared` then `networks: [shared]` in compose files.

**Q: Can I run cron jobs inside Docker?**
Use `ofelia` — a Docker-native cron scheduler. Or create a separate container with a cron daemon.

**Q: How to limit disk usage for a container?**
Use `storage_opt: size: 10G` in the service definition.

**Q: Do I need Docker Compose v2?**
Yes. All templates use v3.8 syntax. Install via `brew install docker-compose` or Docker Desktop.

**Q: How to run Docker without sudo?**
`sudo usermod -aG docker $USER`, then log out/in. On Mac, Docker Desktop handles this automatically.

---

If this saved you time: [PayPal.me/nerudek](https://www.paypal.me/nerudek)
GitHub: [github.com/nerudek](https://github.com/nerudek)
