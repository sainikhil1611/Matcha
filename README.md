# Matcha

Matcha is an AI-powered hackathon teammate-matching app. A React 18 + Vite
frontend talks to a Flask backend that scrapes Devpost participant data
(Selenium + BeautifulSoup), stores it in MongoDB, and uses Google Gemini to
match teammates by skills/interests and generate project ideas.

> **Note on repo history:** this repo previously also contained a root
> `server/` (an Express API) and a root `src/` (a stale duplicate of
> `frontend/src/`). Both were dead code -- `server/` required three route
> files that never existed in git history and could never start, and root
> `src/` was an unreferenced older copy of the frontend. Both were removed;
> full history for either is still recoverable with `git log --all -- server/`
> / `git log --all -- src/` if ever needed. The only live app is
> `frontend/` (React/Vite) + `backend/` (Flask).

## Quick start (devcontainer — recommended)

1. Prereqs: [Docker](https://docs.docker.com/get-docker/), [VS Code](https://code.visualstudio.com/) + the "Dev Containers" extension.
2. Open this folder in VS Code → "Reopen in Container" when prompted. This starts the `backend`, `frontend`, and `mongo` services (from `docker-compose.yml`) and attaches VS Code to the `backend` service.
3. `cp backend/.env.example backend/.env` and `cp frontend/.env.example frontend/.env`, then fill in values (see table below). For local dev, `backend/.env`'s `MONGODB_URI` should stay pointed at the bundled `mongo` compose service (`mongodb://mongo:27017/devpost_data`) rather than an Atlas cluster.
4. Inside the devcontainer: `python app.py` (backend, from `/app`) → http://localhost:5000. In a separate terminal on the host (or another container shell), `cd frontend && npm install && npm run dev` → http://localhost:5173.

## Manual setup (no devcontainer)

Backend (requires **Python 3.12** specifically — 3.14 breaks
`google-generativeai`'s protobuf native extension):

```bash
cd backend
python3.12 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # fill in values
python app.py           # http://127.0.0.1:5000
```

Frontend:

```bash
cd frontend
npm ci
cp .env.example .env   # fill in values
npm run dev              # http://localhost:5173
npm run build             # production build to dist/
```

The Devpost scraper (`/api/scrape`) also needs a local Chrome/Chromium +
matching chromedriver on your PATH if you're running the backend outside
a container.

## Environment variables

| Variable | Required | Purpose | Where to get it |
|---|---|---|---|
| `MONGODB_URI` (backend) | Yes — app exits at startup if unset | MongoDB connection string used by scrape/stats/matching routes | Your MongoDB Atlas cluster, or `mongodb://mongo:27017/devpost_data` for the bundled compose service |
| `GEMINI_API_KEY` (backend) | Yes — app exits at startup if unset | Google Gemini API key for AI teammate matching and idea generation | [Google AI Studio](https://aistudio.google.com/app/apikey) |
| `DATABASE_NAME` (backend) | No, default `devpost_data` | Mongo database name | — |
| `COLLECTION_NAME` (backend) | No, default `participants` | Mongo collection name | — |
| `ALLOWED_ORIGIN` (backend) | No, default `http://localhost:5173` | Single CORS-allowed origin for browser requests | Your frontend's dev/prod URL |
| `FLASK_DEBUG` (backend) | No, default `false` | Enables the Werkzeug interactive debugger. **Leave off** except for local troubleshooting — reachable off localhost it allows remote code execution | — |
| `DEVPOST_JWT`, `DEVPOST_REMEMBER_USER_TOKEN`, `DEVPOST_SESSION`, `DEVPOST_AWS_WAF_TOKEN`, `DEVPOST_GA` (backend) | No — only needed for `/api/scrape` to authenticate against Devpost | Logged-in Devpost session cookies | Log in to Devpost in a browser → DevTools → Application → Cookies → devpost.com |
| `VITE_API_URL` (frontend) | No, default `http://127.0.0.1:5000/api` | Base URL of the Flask backend. Build-time Vite var — changing it requires a rebuild/restart. Shared by `src/services/api.js` and `src/pages/Ideas.jsx`; there is only one backend in this repo | — |
| `VITE_AUTH0_DOMAIN`, `VITE_AUTH0_CLIENT_ID`, `VITE_AUTH0_AUDIENCE` (frontend) | No | Auth0 tenant config for the optional Auth0 login button (`Sidebar.jsx`/`Landing.jsx`). The primary sign-in flow (`src/context/AuthContext.jsx`) is a localStorage mock login that does **not** require Auth0 | [Auth0 dashboard](https://manage.auth0.com/) |

## Testing

No test suite exists for this project (`frontend/`, `backend/`; confirmed
at triage — no test files, no `test` script). Verified manually instead:

- `docker compose build` succeeds for both `backend` and `frontend`.
- `docker compose up` brings up `backend` (200 on `GET /`), `frontend`
  (200, serves the SPA `index.html`), and `mongo`; `GET /api/stats`
  against the live compose Mongo returns `200 {"hackathons":[],
  "total_participants":0}` rather than hanging/erroring.
- CORS: a request with `Origin: http://localhost:5173` gets
  `Access-Control-Allow-Origin` echoed back; a request with an
  unrelated origin gets no such header (blocked).
- A forced exception (malformed JSON body to `/api/scrape`) returns
  the client a generic `{"error": "Internal server error."}` (500)
  while the full traceback is only visible in `docker logs` — confirms
  the info-leak fix without needing a real MongoDB/Gemini key.
- Running the backend entry point with no `.env` present exits 1 with
  `Missing required environment variables: MONGODB_URI, GEMINI_API_KEY`
  and no traceback.
- `npx @devcontainers/cli up --workspace-folder .` and
  `npx @devcontainers/cli exec --workspace-folder . python --version`
  succeed, confirming Python 3.12 inside the devcontainer.

## Deployment

Build the production images:
```bash
docker build -t matcha-backend ./backend
docker build -t matcha-frontend ./frontend
```
Run them (with a real `backend/.env`):
```bash
docker run --env-file backend/.env -p 5000:5000 matcha-backend
docker run -p 5173:80 matcha-frontend
```
These containers run on any host (Fly.io, Render, a VPS with Docker) — see
[DEPLOYMENT.md](../DEPLOYMENT.md) in the Projects root for the full walkthrough.

## Known limitations

- **Gemini/MongoDB Atlas flows are unverifiable without real keys.**
  Everything above was verified against a local `mongo:7.0` compose
  service and a placeholder Gemini key; the actual AI matching/idea
  generation and Devpost scraping features require a real
  `GEMINI_API_KEY` and Devpost session cookies to exercise end-to-end.
- **The `/ideas` page (`frontend/src/pages/Ideas.jsx`) is
  non-functional.** It calls `POST {VITE_API_URL}/ideas/generate`,
  which the Flask backend does not implement (it has a differently-shaped
  `/api/generate-ideas` instead). This predates this cleanup — the page's
  original target was the Express server that was removed (see repo
  history note above); that server's three required route files never
  existed in git history, so the page was never functional as shipped.
  Wiring it up would mean either adding an `/ideas/generate` route to
  the Flask backend or rewriting the page to call `/api/generate-ideas`
  — out of scope here.
- **Devpost scraping is inherently fragile.** It drives headless Chrome
  against a live third-party site's DOM structure and requires
  currently-valid, logged-in Devpost session cookies; it will break
  silently if Devpost changes its markup or the cookies expire.
- **Auth0 env vars are present but the primary auth flow doesn't use
  them.** `src/context/AuthContext.jsx` is a localStorage mock-login
  flow; `useAuth0()` is called directly in `Sidebar.jsx`/`Landing.jsx`
  without an `<Auth0Provider>` ancestor in `App.jsx`, so the Auth0 login
  button's behavior was not verified as part of this cleanup.
- **npm audit**: 10 vulnerabilities remain in `frontend/` after removing
  the Express-only dependencies that used to inflate this to 40 (1
  low/2 moderate/7 high). Not triaged further here.
