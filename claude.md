# ScribbleHub Novel Automation — Project Guide

This document is the single source of truth for the **ScribbleHub → EPUB → Gmail** automation pipeline. Every new agent or developer should read this file before touching anything.

---

## Goal

Build a full end-to-end automation that:
1. Accepts a ScribbleHub novel URL via a simple web form.
2. Scrapes all chapters and novel metadata (title, description, cover) idempotently based on the novel's Post ID.
3. Converts the scraped chapters (Markdown files) into a single uniquely named `.epub`.
4. Embeds the novel's description both as native EPUB metadata (`DC:description`) and as a "Chapter 0" (Preface) to ensure visibility in readers like Quick Novel.
5. Emails the `.epub` as an attachment to a specified Gmail address.
6. Runs headlessly on a VPS, accessible via a browser at `http://<vps-ip>:8600` (or via reverse proxy at the configured domain).

---

## Project Layout

```
scrapers/scribblehub/
├── claude.md              ← YOU ARE HERE — read before coding
├── README.md              ← Public instructions for setting up the project
├── docs/
│   └── VPS_DEPLOYMENT_GUIDE.md  ← Full guide: Docker + DuckDNS + NGINX + SSL
│
├── backend/               ← Python FastAPI application
│   ├── .env               ← secrets (never commit)
│   ├── .env.example       ← Template of required env vars (safe to commit)
│   ├── .gitignore
│   ├── Dockerfile         ← python:3.11-slim, port 8600
│   ├── docker-compose.yml ← Orchestrates scraper + FlareSolverr
│   ├── main.py            ← FastAPI app (API endpoints)
│   ├── scraper.py         ← ScribbleHub HTML scraper (lxml + two-tier fetch)
│   ├── emailer.py         ← Gmail SMTP sender (port 587, STARTTLS)
│   ├── md_to_epub.py      ← EPUB Builder
│   ├── requirements.txt
│   └── data/              ← (git-ignored) Downloaded novels keyed by Post ID
│
└── frontend/              ← React + Vite + TypeScript + Tailwind CSS v4
    ├── .env.development   ← VITE_API_URL for local dev proxy
    ├── index.html         ← Vite entry HTML
    ├── vite.config.ts     ← Vite config (@ alias → src/, Tailwind plugin)
    ├── tsconfig.app.json  ← TypeScript config (@ alias paths)
    └── src/
        ├── main.tsx
        ├── App.tsx            ← Root layout — responsive 2-col card grid
        ├── index.css          ← Global styles + Tailwind v4 import
        ├── types/index.ts     ← Shared TypeScript types (Job, JobStatus)
        ├── lib/api.ts         ← All fetch calls to the FastAPI backend
        ├── hooks/
        │   └── useScrapeJob.ts  ← Custom hook: submit, poll, cleanup
        └── components/
            ├── ScraperCard.tsx    ← Main scrape form + job status panel
            ├── JobHistoryCard.tsx ← Recent jobs (future: SQLite API)
            ├── SystemInfoCard.tsx ← Deployment config info panel
            ├── LibraryCard.tsx    ← Downloaded novels browser (future: /api/library)
            └── ui/
                ├── Card.tsx        ← Glassmorphic base card + CardHeader
                ├── StatusBadge.tsx ← Colour-coded job status pill
                └── ProgressBar.tsx ← Gradient progress bar
```

> **Backend virtual environment** (for running the Python backend locally without Docker):
> ```bash
> cd backend/
> python -m venv .venv
> source .venv/Scripts/activate  # Windows
> source .venv/bin/activate      # Linux / VPS
> pip install -r requirements.txt
> uvicorn main:app --reload
> ```

> **Frontend dev server** (Vite — separate from the backend):
> ```bash
> cd frontend/
> npm install
> npm run dev          # dev server at http://localhost:5173
> npm run build        # production build → dist/
> ```
> Set `VITE_API_URL=http://localhost:8600` in `.env.development` so the frontend talks to the local FastAPI during development.

---

## Technical Decisions & Accomplishments

### 1. Robust HTML Parsing (`lxml`)
ScribbleHub's HTML (specifically `#chp_raw`) often has unclosed `<p>` tags or messy structural nesting. When using standard `html.parser` with BeautifulSoup, this leads to aggressive text duplication. We explicitly enforce `lxml` since it implements aggressive tree-repair mapping, natively fixing unclosed `<p>` elements and restoring pristine chapter contents.

### 2. Idempotent Scraping via ScribbleHub Post IDs
Instead of ephemeral UUIDs, every job is keyed to the novel's unique **Post ID** (extracted directly from the URL, e.g. `scribblehub.com/series/311974/...`).
- **Resumable**: If a URL is submitted twice, it uses `skip_existing=True` to heavily optimize performance—downloading only new chapters.
- **Concurrent-Safe**: If the Frontend submits a request for a job already in motion, `main.py` detects the active state and smoothly attaches the frontend to the existing background task.

### 3. Two-Tier Chapter Fetching (Direct → FlareSolverr Fallback)
Chapter content pages are fetched using a **tiered strategy**:
1. **Primary – Direct GET** (`fetch_direct`): Fast, no overhead. Attempts a plain `requests.Session` GET with browser-like headers. Used for chapter pages which often don't trigger Cloudflare JS challenges.
2. **Fallback – FlareSolverr** (`fetch_via_flaresolverr`): If the direct fetch raises a `RuntimeError` (e.g. Cloudflare blocks), the scraper automatically retries via the FlareSolverr sidecar container.

The **series page** and **AJAX chapter list** are always fetched via FlareSolverr since the main series page reliably triggers Cloudflare challenges.

### 4. FlareSolverr Sidecar Container
`docker-compose.yml` runs **FlareSolverr** (`ghcr.io/flaresolverr/flaresolverr:latest`) as a companion Docker service. It is a headless Chrome proxy that can solve Cloudflare JS/turnstile challenges automatically. The scraper communicates with it at `http://flaresolverr:8191` (Docker internal network). If running locally, set `FLARESOLVERR_URL=http://localhost:8191` and run the FlareSolverr container separately.

### 5. Quick Novel EPUB Optimization
Quick Novel expects explicit items in the EPUB's HTML spine to track visual progress and correctly display the book description.
- We directly inject `DC:description` standard EPUB metadata.
- We artificially construct a **"Description"** chapter (acting as _Chapter 0 / Preface_) formatted strictly as HTML and placed at the very top of the `.xhtml` spine array.
- Cover images are **downloaded at EPUB build time** from the novel's `cover_url` metadata field.

### 6. Gmail SMTP — Port 587 (STARTTLS)
`emailer.py` connects to `smtp.gmail.com` on **port 587** using `STARTTLS` (not SSL/port 465). This is the current, widely-supported Gmail App Password flow. The sender requires a **Gmail App Password** — not the account's real password.

### 7. Dynamic UI with No Build Step
The frontend (`index.html`) is written natively in highly-modern CSS/JS focusing on rich UI aesthetics (gradients, dynamic glassy backgrounds, pulsing statuses, smooth transition bars). It queries the unified backend state via asynchronous javascript polling (`setInterval`) to give a flawless real-time view into the Scrape → EPUB → Email task queue.

### 8. Consolidated Flat Directory
All application aspects are cleanly grouped into `/scrapers/scribblehub` replacing the initial nested abstractions, allowing `uvicorn main:app` to safely resolve, compile, and execute dependencies without advanced python `$PYTHONPATH` hacking or nested import confusions.

---

## Environment Variables

Create a `.env` file directly beside `main.py` (copy from `.env.example`):

```env
# Gmail credentials for sending EPUBs
GMAIL_USER=your-email@gmail.com
GMAIL_APP_PASSWORD=xxxx xxxx xxxx xxxx

# Security: restrict which hostnames/origins can reach the API
# For local dev, leave as * (the defaults)
ALLOWED_HOSTS=mynovelapp.duckdns.org
ALLOWED_ORIGINS=https://mynovelapp.duckdns.org

# FlareSolverr URL — only needed if NOT using docker-compose (compose sets this automatically)
FLARESOLVERR_URL=http://localhost:8191
```

| Variable | Purpose | Default |
|---|---|---|
| `GMAIL_USER` | Gmail address to send from | *(required)* |
| `GMAIL_APP_PASSWORD` | Gmail App Password (not account password) | *(required)* |
| `ALLOWED_HOSTS` | FastAPI TrustedHost middleware whitelist | `*` |
| `ALLOWED_ORIGINS` | FastAPI CORS origin whitelist | `*` |
| `FLARESOLVERR_URL` | URL of the FlareSolverr instance | `http://localhost:8191` |

**Gmail App Password** — generated at:
`https://myaccount.google.com/security` → 2-Step Verification → App Passwords

> ⚠️ **Never commit `.env` to git.** It is explicitly ignored inside `.gitignore`.

---

## Docker Deployment

The project runs as **two Docker services** orchestrated by `docker-compose.yml`:

| Service | Image | Port | Role |
|---|---|---|---|
| `scribblehub-scraper` | Built from `./Dockerfile` | `8600` (host) | FastAPI app |
| `flaresolverr` | `ghcr.io/flaresolverr/flaresolverr:latest` | `8191` (internal only) | Cloudflare bypass proxy |

### Key Docker facts
- **App port**: `8600` (exposed to host). The Dockerfile `CMD` runs `uvicorn main:app --host 0.0.0.0 --port 8600`.
- **`.env` is mounted**, not baked in: `.env:/app/.env` — secrets stay outside the image.
- **`data/` is mounted**: `./data:/app/data` — downloaded novels and EPUBs persist on the host even if the container is stopped or rebuilt.
- `scribblehub-scraper` **depends on** `flaresolverr` starting first.

### Commands
```bash
# First run / after code changes — always rebuild:
docker compose up -d --build

# View live logs:
docker compose logs -f scribblehub-scraper
docker compose logs -f flaresolverr

# Stop everything:
docker compose down
```

> 🔑 **After changes to `index.html` or any Python file**, you MUST run `docker compose up -d --build` — a simple restart will NOT pick up the new files.

---

## API Documentation

**Framework**: FastAPI + uvicorn
**Pattern**: Asynchronous Background Tasks with Thread Locking

| Method | Path | Description |
|---|---|---|
| `GET` | `/` | Serve `index.html` FileResponse |
| `POST` | `/api/scrape` | Validates Post ID and starts OR attaches to existing Background Scrape |
| `GET` | `/api/status/{job_id}` | Polls backend memory store dict for live progress + status |

**POST Body:**
```json
{
  "url": "https://www.scribblehub.com/series/311974/invincible-me/",
  "email": "user@gmail.com"
}
```

**Job Status Flow:**
```
queued → scraping → converting → emailing → done
                                          ↘ error
```

---

## VPS Deployment (High-Level)

Full details are in `docs/VPS_DEPLOYMENT_GUIDE.md`. Summary:

1. **DuckDNS**: Register a free subdomain (e.g. `mynovelapp.duckdns.org`) and point it to your VPS public IP.
2. **Upload code**: `git clone` or SFTP the project to the VPS. Create `backend/.env` with real credentials and set `ALLOWED_HOSTS`/`ALLOWED_ORIGINS` to your DuckDNS domain.
3. **Frontend build**: Run `npm run build` in `frontend/`. The `dist/` folder contains the compiled static assets. Configure FastAPI (or NGINX) to serve these.
4. **Docker**: In `backend/`, run `docker compose up -d --build`. App is now listening on `<VPS-IP>:8600` (internal).
5. **NGINX reverse proxy**: Install NGINX and point port 80/443 → `127.0.0.1:8600`. UFW rules block direct access to port 8600 from the internet.
6. **SSL via Certbot + DuckDNS**: Run `sudo certbot --nginx -d mynovelapp.duckdns.org`. Certbot automatically provisions a **Let's Encrypt** certificate and patches the NGINX config to redirect all HTTP → HTTPS. Auto-renewal via `certbot renew` cron runs silently.
7. **Firewall (UFW)**:
   ```bash
   sudo ufw allow "Nginx Full"
   sudo ufw allow ssh
   sudo ufw deny 8600       # prevent bypassing NGINX
   sudo ufw enable
   ```

> The live application is served at `https://mynovelapp.duckdns.org` over TLS. All traffic goes through NGINX which terminates SSL and reverse-proxies to the Docker container on port 8600.

---

## TODO / Future Expansions
- [ ] Google Drive integration fallback for extremely large novels (> 25MB).
- [ ] Frontend Authentication (HTTP Basic Auth or Tokens) for public VPS hosting.
- [ ] Add persistence to job histories via SQLite if job logging is heavily required beyond pure RAM.
