# Deployment Information

Two Railway services live from this repo — one per lab part:

| Part | Purpose | URL | Features |
|------|---------|-----|----------|
| Part 3 | Simple agent (first public deploy) | https://feisty-nourishment-production.up.railway.app | `/health`, `/ask` (no auth) |
| **Part 6** | **Production agent (graded submission)** | **https://remarkable-mercy-production-d435.up.railway.app** | Full stack: auth, rate limit, cost guard, health, ready, JSON logs, security headers |

Both are auto-deployed from GitHub pushes to `main`.

---

## Part 6 — Production Agent (graded submission)

### URL

**https://remarkable-mercy-production-d435.up.railway.app**

### Platform

Railway, Dockerfile builder, GitHub integration.

- Root directory: `06-lab-complete/`
- Config file: `06-lab-complete/railway.toml`
- Health check path: `/health`
- Replicas: 1 (single container, `--workers 2` inside)

### Environment Variables (Railway → Variables tab)

| Key | Value | Notes |
|-----|-------|-------|
| `PORT` | auto-injected | Railway assigns per deploy |
| `ENVIRONMENT` | `production` | Triggers strict validation in `config.py` |
| `AGENT_API_KEY` | `<secret>` | Required — default `dev-key...` is rejected in production |
| `JWT_SECRET` | `<secret>` | Required |
| `APP_NAME` | `Day 12 Final Project` | Shown in `/health` response |
| `RATE_LIMIT_PER_MINUTE` | `10` | Per API-key sliding window |
| `DAILY_BUDGET_USD` | `10.0` | Global cost guard |
| `ALLOWED_ORIGINS` | `*` | CORS |

### Test Commands

Set the URL and key locally:

```bash
URL="https://remarkable-mercy-production-d435.up.railway.app"
KEY="<paste AGENT_API_KEY from Railway Variables>"
```

**Health (public, no auth):**
```bash
curl $URL/health
```

Expected:
```json
{
  "status": "ok",
  "version": "1.0.0",
  "environment": "production",
  "uptime_seconds": 408.8,
  "total_requests": 6,
  "checks": {"llm": "mock"},
  "timestamp": "2026-04-17T13:16:34.410059+00:00"
}
```

**Readiness (public):**
```bash
curl $URL/ready
# {"ready":true}
```

**/ask without key → 401:**
```bash
curl -i -X POST $URL/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'
# HTTP/2 401
# {"detail":"Invalid or missing API key. Include header: X-API-Key: <key>"}
```

**/ask with key → 200:**
```bash
curl -X POST $URL/ask \
  -H "X-API-Key: $KEY" \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello from production!"}'
```

Expected:
```json
{
  "question": "Hello from production!",
  "answer": "...",
  "model": "gpt-4o-mini",
  "timestamp": "..."
}
```

**Rate limit (10 req/min per key; 2 uvicorn workers → effective ~20 combined):**
```bash
for i in {1..30}; do
  curl -s -o /dev/null -w "%{http_code}\n" \
    -X POST $URL/ask \
    -H "X-API-Key: $KEY" \
    -H "Content-Type: application/json" \
    -d "{\"question\":\"t$i\"}"
done | sort | uniq -c
```

Expect a mix of `200` and `429` as the sliding window fills.

---

## Part 3 — Simple Agent (first deploy, kept for Part 3 reference)

### URL

**https://feisty-nourishment-production.up.railway.app**

### Platform

Railway, NIXPACKS builder, GitHub integration.

- Root directory: `03-cloud-deployment/railway/`
- Config file: `03-cloud-deployment/railway/railway.toml`
- Health check path: `/health`

### Test Commands

```bash
curl https://feisty-nourishment-production.up.railway.app/health

curl -X POST https://feisty-nourishment-production.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello from the internet!"}'
```

Response:
```json
{
  "question": "Hello from the internet!",
  "answer": "...",
  "platform": "Railway"
}
```

### Environment Variables

| Key | Notes |
|-----|-------|
| `PORT` | auto-injected |
| `ENVIRONMENT` | `production` |
| `AGENT_API_KEY` | set but unused in this version |

---

## Deployment verification

- Part 3 first deploy: 2026-04-17 08:52 UTC
- Part 6 first successful deploy: 2026-04-17 13:16 UTC (after fixing 4 bugs in lab source — see `MISSION_ANSWERS.md` Part 6)
- Both URLs tested via `curl` from local machine; responses match expected JSON shape
