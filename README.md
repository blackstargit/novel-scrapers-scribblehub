# ScribbleHub Novel To EPUB Automator

A sleek, headless automation engine wrapped in a beautifully styled React frontend. Paste any **ScribbleHub** novel link and this tool will immediately extract the novel's core data, scrape every chapter, build a complete `.epub` file natively compatible with QuickNovel, and securely push the generated E-book straight to your designated Gmail inbox via SMTP.

## Repository Structure

This project uses a **nested Git submodule** hierarchy:

```
novel-main/               ← Root monorepo (parent)
└── scrapers/             ← Submodule: blackstargit/novel-scrapers
    └── scribblehub/      ← Submodule: blackstargit/novel-scrapers-scribblehub
        ├── backend/      ← Sub-submodule: blackstargit/novel-scrapers-scribblehub-backend
        └── frontend/     ← Sub-submodule: blackstargit/novel-scrapers-scribblehub-frontend
```

> ⚠️ Always clone with `git clone --recurse-submodules` to initialize all nested submodules.  
> After pulling updates, run `git submodule update --init --recursive`.

## Features

- **Blazing Fast**: Directly polls WordPress AJAX core endpoints, instantly returning chapter logic without fully parsing heavy site overhead.
- **Auto-Repairing HTML Parsing**: Powered by `lxml`, aggressively fixing messy ScribbleHub un-closed text paragraphs, preventing data duplication.
- **Idempotent Resume**: Interrupted connection? Power tripped? The tool stores data by `Post ID` and caches existing sections using `skip_existing=True`. It only scrapes brand-new chapters, instantly piecing them into your updated EPUB!
- **QuickNovel Perfect Support**: Injects description data into standard metadata and specifically generates a **Preface Chapter** strictly to render plot hooks gracefully inside mobile readers.
- **Modern React UI**: Glassmorphic glassmorphism UI built with React + Vite + TypeScript + Tailwind CSS v4, polling job updates in real time.
- **E-Book Emailing**: Securely routes the finished file right into your Gmail library using SMTP App-Passwords.

## Installation & Setup

### 1. Clone (with all submodules)

```bash
git clone --recurse-submodules <your-repo>
cd scrapers/scribblehub
```

### 2. Backend — Python Virtual Environment

The backend uses `.venv` (inside `backend/`) as its Python virtual environment. It is git-ignored and must be created locally.

```bash
cd backend/

# Create virtual environment
python -m venv .venv

# Activate:
.venv\Scripts\activate      # Windows (CMD / PowerShell)
source .venv/bin/activate   # Linux / macOS / VPS

# Install dependencies
pip install -r requirements.txt
```

### 3. Configure Environment Secrets

Create a `.env` file inside the `backend/` directory (copy from `.env.example`):

```env
GMAIL_USER=your-email@gmail.com
GMAIL_APP_PASSWORD=xxxx xxxx xxxx xxxx

# Security: restrict which hostnames/origins can reach the API
ALLOWED_HOSTS=mynovelapp.duckdns.org
ALLOWED_ORIGINS=https://mynovelapp.duckdns.org

# FlareSolverr URL (only needed if not using docker-compose)
FLARESOLVERR_URL=http://localhost:8191
```

> You must generate an App Password from Google: *Manage Your Google Account → Security → 2-Step Verification → App Passwords*.

### 4. Frontend — Node.js Dev Server

```bash
cd frontend/
npm install
npm run dev          # starts at http://localhost:5173
```

Set `VITE_API_URL=http://localhost:8600` in `frontend/.env.development` so the frontend proxies to the local FastAPI server.

## Running The Application (Local Dev)

1. Start the FastAPI backend (from `backend/` with `.venv` activated):
   ```bash
   uvicorn app.main:app --reload --port 8600
   ```
2. Start the React frontend (from `frontend/`):
   ```bash
   npm run dev
   ```
3. Navigate your browser to `http://localhost:5173`.
4. Paste a ScribbleHub novel URL and an email address, then watch the progress!

## Docker Deployment (VPS)

From the `backend/` directory:

```bash
docker compose up -d --build
```

This starts two services:
- `scribblehub-scraper` — FastAPI app on port `8600`
- `flaresolverr` — Cloudflare bypass proxy (internal only)

> After any code change, always re-run `docker compose up -d --build` to rebuild the image.

## Security & Privacy

- `.env` is listed in `.gitignore` and will never be committed.
- All secrets are read dynamically via `os.getenv()` / `pydantic-settings`. No hardcoded strings exist inside the repository.
- `.venv/` is also git-ignored — only `requirements.txt` is committed.
