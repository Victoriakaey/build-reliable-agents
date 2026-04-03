---
name: devops
description: "Use when deploying an application to production, setting up CI/CD, or moving from local development to a hosted environment. Guides through deployment target selection (PaaS, Docker+VPS, cloud platform), containerization, environment configuration, CI/CD pipeline design, monitoring, and rollback strategy. Covers both simple and complex deployment scenarios."
---

# DevOps & Deployment

## The Core Problem

Most projects work locally and then break in production in ways that are hard to debug — missing environment variables, dependency mismatches, different OS behavior, secrets not configured, no rollback plan. The gap between "it works on my machine" and "it works reliably in production" is where most time gets wasted.

**The discipline:** Treat deployment as a system, not an afterthought. Define the environment, containerize early, automate what you repeat, and always have a rollback plan.

---

## Step 1: Choose Your Deployment Target

Answer these questions first:

```
What is the team size and technical depth?
(solo / small team / large team with DevOps)

What is the expected traffic?
(personal project / low traffic / high traffic / variable/spiky)

What is the budget?
(free / $10-50/month / $100+/month / cost is not a constraint)

Do you need auto-scaling?
(traffic is unpredictable / need to scale to zero / fixed load)

How much infrastructure management do you want to do?
(as little as possible / comfortable with Docker / comfortable with Kubernetes)

Are there compliance or data residency requirements?
(HIPAA, GDPR, data must stay in specific region)
```

### Deployment target comparison

| Target | Best For | Monthly Cost | Complexity | Auto-scale |
|---|---|---|---|---|
| **PaaS (Railway, Render, Fly.io)** | Solo/small team, fast iteration, simple apps | $5-50 | Low | Yes (Fly, Render) |
| **Docker + VPS (DigitalOcean, Hetzner)** | Full control, predictable load, cost-sensitive | $6-20 | Medium | Manual |
| **GCP Cloud Run** | Variable traffic, scale to zero, serverless containers | Pay per use | Medium | Yes |
| **AWS ECS / Fargate** | AWS ecosystem, team familiar with AWS | $20-100+ | Medium-High | Yes |
| **Kubernetes (GKE, EKS, AKS)** | Large teams, complex microservices, high traffic | $70-200+ | High | Yes |
| **Serverless (Lambda, Cloud Functions)** | Event-driven, short-lived tasks, low traffic | Pay per use | Low-Medium | Yes |

### Recommended starting points

**Solo project or MVP:** Railway or Render — deploy from GitHub in minutes, handles SSL, env vars, databases.

**Cost-sensitive with stable load:** Hetzner VPS ($5/month) + Docker Compose — cheapest reliable option, requires manual setup.

**Variable/spiky traffic:** GCP Cloud Run — scales to zero (free when idle), scales up automatically, pay only for usage.

**Enterprise / team with DevOps:** AWS ECS or Kubernetes depending on complexity.

---

## Step 2: Containerize Your Application

Docker is the foundation for consistent deployments. If you're not containerized, do this first.

### Basic Dockerfile for a Python app

```dockerfile
# Use specific version — never use :latest in production
FROM python:3.11.12-slim

# Set working directory
WORKDIR /app

# Install dependencies first (layer caching)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Don't run as root
RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1
# Note: for non-HTTP apps (WebSocket, gRPC), replace curl with a custom script
# or use CMD ["python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]

# Start command
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Basic Dockerfile for a Node.js app

```dockerfile
FROM node:20.18.1-slim

WORKDIR /app

# Install dependencies
COPY package*.json .
RUN npm ci --only=production

COPY . .

RUN useradd -m appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
```

### .dockerignore (always include this)

```
.git
.env
.env.*
node_modules/
__pycache__/
*.pyc
.pytest_cache/
logs/
*.log
README.md
.DS_Store
```

### Docker Compose for local development

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@db:5432/mydb
    env_file:
      - .env
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - .:/app  # for development hot reload only — remove in production

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password  # development only — use env vars or secrets manager in production
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

---

## Step 3: Environment Configuration

### The golden rule: no secrets in code

All secrets and environment-specific config must come from environment variables — never hardcoded, never in version control.

```bash
# .env (local development only — gitignore this)
DATABASE_URL=postgresql://localhost:5432/mydb
OPENAI_API_KEY=sk-...
SECRET_KEY=local-dev-secret
ENVIRONMENT=development

# .env.example (commit this — shows what vars are needed, no values)
DATABASE_URL=
OPENAI_API_KEY=
SECRET_KEY=
ENVIRONMENT=
```

### Environment tiers

| Environment | Purpose | Config Source |
|---|---|---|
| `development` | Local dev, fast iteration | `.env` file |
| `staging` | Pre-production testing, mirrors production | Platform env vars |
| `production` | Live traffic | Platform env vars + secrets manager |

### Startup validation

Fail fast if required vars are missing — don't let the app start with missing config:

```python
# config.py
import os

def require_env(name: str) -> str:
    value = os.environ.get(name)
    if not value:
        raise RuntimeError(f"Required environment variable {name} is not set")
    return value

DATABASE_URL = require_env("DATABASE_URL")
API_KEY = require_env("OPENAI_API_KEY")
ENVIRONMENT = os.environ.get("ENVIRONMENT", "development")
```

---

## Step 4: Deployment by Target

### Option A: PaaS (Railway / Render / Fly.io)

**Railway:**
```bash
# Install Railway CLI
npm install -g @railway/cli

# Login and init
railway login
railway init

# Set environment variables
railway variables set OPENAI_API_KEY=sk-...
railway variables set DATABASE_URL=postgresql://...

# Deploy
railway up

# Add a database
railway add postgresql
```

**Render:** Connect GitHub repo in dashboard → set env vars → auto-deploys on push.

**Fly.io:**
```bash
# Install flyctl
curl -L https://fly.io/install.sh | sh

# Launch app (generates fly.toml)
fly launch

# Set secrets
fly secrets set OPENAI_API_KEY=sk-...

# Deploy
fly deploy

# Scale
fly scale count 2
```

### Option B: Docker + VPS (DigitalOcean / Hetzner)

```bash
# 1. Create VPS, SSH in
ssh root@your-server-ip

# 2. Install Docker
curl -fsSL https://get.docker.com | sh
systemctl enable docker

# 3. Create deploy user
useradd -m -G docker deploy
su - deploy

# 4. Clone repo
git clone https://github.com/you/your-app.git
cd your-app

# 5. Create .env with production values
cp .env.example .env
nano .env

# 6. Build and run
docker compose -f docker-compose.prod.yml up -d

# 7. Set up nginx as reverse proxy (see nginx.conf below)
```

**nginx.conf** (reverse proxy with SSL):
```nginx
events {
    worker_connections 1024;
}

http {
    upstream app {
        server app:8000;
    }

    server {
        listen 80;
        server_name your-domain.com;

        # Redirect HTTP to HTTPS
        location / {
            return 301 https://$host$request_uri;
        }

        # Let's Encrypt challenge
        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }
    }

    server {
        listen 443 ssl;
        server_name your-domain.com;

        ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem;

        location / {
            proxy_pass http://app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Timeout settings for long-running LLM requests
            proxy_read_timeout 120s;
            proxy_send_timeout 120s;
        }
    }
}
```

**docker-compose.prod.yml** (no volume mounts, restart policy):
```yaml
version: '3.8'
services:
  app:
    build: .
    restart: unless-stopped
    environment:
      - ENVIRONMENT=production
    env_file:
      - .env
    ports:
      - "8000:8000"
    depends_on:
      - db

  db:
    image: postgres:15
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - postgres_data:/var/lib/postgresql/data

  nginx:
    image: nginx:alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - certbot_data:/etc/letsencrypt

volumes:
  postgres_data:
  certbot_data:
```

### Option C: GCP Cloud Run

```bash
# 1. Build and push to Google Container Registry
gcloud auth configure-docker
docker build -t gcr.io/YOUR_PROJECT/your-app .
docker push gcr.io/YOUR_PROJECT/your-app

# 2. Deploy to Cloud Run
gcloud run deploy your-app \
  --image gcr.io/YOUR_PROJECT/your-app \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars ENVIRONMENT=production \
  --set-secrets OPENAI_API_KEY=openai-api-key:latest

# Note: create the secret first if it doesn't exist:
# gcloud secrets create openai-api-key --replication-policy="automatic"
# echo -n "sk-..." | gcloud secrets versions add openai-api-key --data-file=-

# 3. Set up Cloud SQL for database
gcloud sql instances create your-db \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=us-central1
```

### Option D: AWS ECS / Fargate

```bash
# 1. Push to ECR
aws ecr create-repository --repository-name your-app
aws ecr get-login-password | docker login --username AWS --password-stdin YOUR_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com
docker build -t YOUR_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/your-app .
docker push YOUR_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/your-app

# 2. Create ECS cluster, task definition, and service via console or Terraform
# Use AWS Copilot for simpler setup:
copilot init
copilot env init --name production
copilot deploy
```

---

## Step 5: CI/CD Pipeline

Automate: test → build → deploy. Never deploy manually in production.

### GitHub Actions — basic pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest tests/ -v
        env:
          DATABASE_URL: sqlite:///test.db

  deploy:
    needs: test  # only deploy if tests pass
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to Railway
        run: railway up --detach
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}

      # OR for Docker + VPS:
      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: deploy
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /home/deploy/your-app
            git pull origin main
            docker compose -f docker-compose.prod.yml up -d --build

      # OR for Cloud Run:
      - name: Deploy to Cloud Run
        uses: google-github-actions/deploy-cloudrun@v1
        with:
          service: your-app
          image: gcr.io/${{ secrets.GCP_PROJECT }}/your-app
          region: us-central1
```

### Branch strategy

```
main          → production (auto-deploy)
staging       → staging environment (auto-deploy)
feature/*     → no auto-deploy, PRs only
```

**Protection rules for `main`:**
- Require PR before merging
- Require CI to pass
- Require at least 1 review (for team projects)

---

## Step 6: Health Checks and Monitoring

### Health endpoint (required)

Every service needs a `/health` endpoint. Load balancers and orchestrators use it to route traffic.

```python
# FastAPI
@app.get("/health")
async def health():
    # check critical dependencies
    db_ok = await check_db_connection()
    return {
        "status": "ok" if db_ok else "degraded",
        "version": os.environ.get("APP_VERSION", "unknown"),
        "environment": os.environ.get("ENVIRONMENT"),
        "db": "ok" if db_ok else "error"
    }
```

### Logging

Structure your logs — plain text logs are hard to query in production:

```python
import logging
import json

class JSONFormatter(logging.Formatter):
    def format(self, record):
        return json.dumps({
            "level": record.levelname,
            "message": record.getMessage(),
            "time": self.formatTime(record),
            "module": record.module,
        })

# Use structured logging
logger = logging.getLogger(__name__)
logger.info("Request processed", extra={"user_id": user_id, "duration_ms": 150})
```

### Basic monitoring checklist

- [ ] Health endpoint returns 200 when healthy
- [ ] Error rate alerting (> 1% 5xx errors)
- [ ] Latency alerting (p99 > 5s)
- [ ] Disk space alerting (> 80% full)
- [ ] Memory alerting (> 90% used)
- [ ] Log aggregation set up (Papertrail, Logtail, or GCP/AWS native)

---

## Step 7: Rollback Strategy

Always know how to roll back before you deploy.

### PaaS rollback

```bash
# Railway
railway rollback

# Render: use dashboard → "Rollback to previous deploy"

# Fly.io
fly releases list
fly deploy --image registry.fly.io/your-app:v42  # specific version
```

### Docker + VPS rollback

```bash
# Tag your images with git SHA
docker build -t your-app:$(git rev-parse --short HEAD) .

# Rollback: restart with previous image
docker compose down
git checkout previous-commit
docker compose -f docker-compose.prod.yml up -d
```

### Database migrations rollback

**Zero-downtime migration order:** Deploy new code (backward-compatible with old schema) → run migration → deploy code that uses new schema. Never run a breaking migration before the new code is deployed.

Never deploy a breaking database migration without a rollback plan:

```python
# Use Alembic (Python) or similar — always write down() migrations
def upgrade():
    op.add_column('users', sa.Column('new_field', sa.Text()))

def downgrade():
    op.drop_column('users', 'new_field')
```

**Rule:** Test `downgrade()` in staging before deploying to production.

---

## Deployment Checklist

### Before first production deploy
- [ ] All secrets in environment variables, not code
- [ ] `.env` is in `.gitignore`
- [ ] `.env.example` committed with all required vars listed
- [ ] Health endpoint implemented
- [ ] Dockerfile built and tested locally
- [ ] Database migrations tested (upgrade + downgrade)
- [ ] Database backup strategy configured (automated daily backups at minimum)
- [ ] Backup restoration tested at least once
- [ ] Rollback procedure documented

### Before every deploy
- [ ] Tests pass locally
- [ ] CI pipeline passes
- [ ] Staging deploy tested (if staging exists)
- [ ] Database migration is backward-compatible (or deploy migration separately first)
- [ ] Know how to roll back

### After every deploy
- [ ] Health endpoint returns 200
- [ ] No spike in error rate
- [ ] Latency is normal
- [ ] Check logs for unexpected errors

---

## Step 8: Database Backup Strategy

Backups are non-negotiable for production. Data loss from a missing backup is unrecoverable.

### Managed database backups (PaaS / cloud)
Most managed databases (Railway, Render, Cloud SQL, RDS) include automated daily backups. Verify this is enabled and test a restore.

### Self-hosted backup (Docker + VPS)

```bash
# Automated daily backup via cron
# Add to crontab: crontab -e
0 3 * * * /home/deploy/backup-db.sh >> /var/log/db-backup.log 2>&1
```

```bash
#!/bin/bash
# backup-db.sh
BACKUP_DIR="/home/deploy/backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
KEEP_DAYS=30

# Dump database
docker compose exec -T db pg_dump -U postgres mydb | gzip > "$BACKUP_DIR/mydb_$TIMESTAMP.sql.gz"

# Clean old backups
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +$KEEP_DAYS -delete

# Optional: upload to remote storage (S3, GCS, etc.)
# aws s3 cp "$BACKUP_DIR/mydb_$TIMESTAMP.sql.gz" s3://your-backup-bucket/
```

### Restore from backup

```bash
# Restore from a compressed backup
gunzip -c backups/mydb_20260401_030000.sql.gz | docker compose exec -T db psql -U postgres mydb
```

### Backup checklist
- [ ] Automated daily backups configured
- [ ] Backups stored in a separate location from the database (different server or cloud storage)
- [ ] Backup restoration tested at least once
- [ ] Retention policy set (keep 30 days minimum)
- [ ] Alerts configured for backup failures