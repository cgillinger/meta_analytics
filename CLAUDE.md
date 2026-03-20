# Meta Analytics — Persistent SQLite Version

## Project Overview

Convert the existing browser-only Meta Analytics app (React + IndexedDB) into a locally-run web application with a persistent SQLite database. The app runs in a Docker container — the user runs `docker compose up` and opens localhost in any browser. CSV data is uploaded once and stored permanently in a SQLite file on a mounted volume.

It must also be possible to run natively with `npm start` (no Docker) for development.

**Repo:** `cgillinger/meta_analytics`

**Branch strategy:**
- `main` — existing browser-only app. **Do not modify.** Not a single commit on main.
- `metadb` — new branch, created from `main`. All work happens here.

**First step before any code changes:**
```
git checkout -b metadb
```

The `metadb` branch will diverge significantly from `main` — that is expected. The two variants share visual design and UI components but have fundamentally different data layers. Later, the branch can be extracted to a standalone repo if desired.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  Docker container                                   │
│                                                     │
│   Browser (React frontend)                          │
│       ↕ fetch('/api/...')                           │
│   Express server (0.0.0.0:3001 inside container)    │
│       ↕ better-sqlite3                              │
│   /data/analytics.db ← mounted volume               │
│                                                     │
└─────────────────────────── port 3001 → localhost ───┘
```

**Stack:**
- Frontend: Existing React 18 + Tailwind + shadcn/ui (Vite for dev/build)
- Backend: Express + better-sqlite3 (synchronous SQLite)
- Container: Docker (multi-stage build, Node.js 22 Alpine)
- Dev (without Docker): Vite dev server proxies `/api/*` to Express
- Prod (Docker or native): Express serves built static files + API from same port

**Startup options:**
- `docker compose up` — Production. Builds image, mounts `./data` volume, exposes `localhost:3001`
- `docker compose up --build` — Rebuild after code changes
- `npm run dev` — Native dev mode. Vite + Express concurrently (hot reload, no Docker)
- `npm start` — Native production mode. Express serves built frontend + API
- `npm run build` — Vite builds frontend to `dist/`

---

## Database Schema

File: `server/db/schema.sql`

```sql
CREATE TABLE IF NOT EXISTS schema_version (
  version INTEGER PRIMARY KEY,
  applied_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE IF NOT EXISTS imports (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  filename TEXT NOT NULL,
  platform TEXT NOT NULL CHECK(platform IN ('facebook', 'instagram')),
  month TEXT NOT NULL,            -- '2026-01' format, derived from post dates
  imported_at TEXT DEFAULT (datetime('now')),
  row_count INTEGER DEFAULT 0,
  account_count INTEGER DEFAULT 0,
  date_range_start TEXT,
  date_range_end TEXT
);

CREATE TABLE IF NOT EXISTS posts (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  import_id INTEGER NOT NULL REFERENCES imports(id) ON DELETE CASCADE,
  post_id TEXT NOT NULL,
  account_id TEXT,
  account_name TEXT,
  account_username TEXT,          -- Instagram only
  description TEXT,
  publish_time TEXT,
  post_type TEXT,
  permalink TEXT,
  platform TEXT NOT NULL CHECK(platform IN ('facebook', 'instagram')),
  -- Metrics
  views INTEGER DEFAULT 0,
  reach INTEGER DEFAULT 0,
  likes INTEGER DEFAULT 0,
  comments INTEGER DEFAULT 0,
  shares INTEGER DEFAULT 0,
  -- Facebook-specific
  total_clicks INTEGER DEFAULT 0,
  link_clicks INTEGER DEFAULT 0,
  other_clicks INTEGER DEFAULT 0,
  -- Instagram-specific
  saves INTEGER DEFAULT 0,
  follows INTEGER DEFAULT 0,
  -- Computed at import time
  interactions INTEGER DEFAULT 0,  -- likes + comments + shares
  engagement INTEGER DEFAULT 0,    -- FB: interactions + total_clicks, IG: interactions + saves + follows
  -- Collab detection
  is_collab BOOLEAN DEFAULT 0,

  UNIQUE(post_id, platform)       -- Dedup: same post_id on same platform = update
);

CREATE INDEX idx_posts_import ON posts(import_id);
CREATE INDEX idx_posts_account ON posts(account_id);
CREATE INDEX idx_posts_publish_time ON posts(publish_time);
CREATE INDEX idx_posts_platform ON posts(platform);
CREATE INDEX idx_posts_month ON posts(publish_time);  -- For month-based queries
```

**Migration system:** `server/db/migrations/` folder with numbered SQL files. On startup, check `schema_version` and apply missing migrations.

---

## API Design

All endpoints under `/api/`. Express router in `server/routes/`.

### Imports management

```
GET    /api/imports                    → list all imports (with month, platform, row_count)
POST   /api/imports                    → upload CSV (multipart/form-data), parse, insert into DB
DELETE /api/imports/:id                → delete import + all its posts (CASCADE)
GET    /api/imports/coverage           → which months have data, which are missing
```

### Posts (server-side pagination, sorting, filtering)

```
GET /api/posts?page=1&pageSize=20&sort=publish_time&order=desc&account=X&platform=facebook&month=2026-01
```

Response:
```json
{
  "data": [...posts],
  "total": 4521,
  "page": 1,
  "pageSize": 20,
  "totalPages": 227
}
```

### Account aggregation (SQL-based)

```
GET /api/accounts?fields=views,reach,likes,comments&sort=views&order=desc&platform=facebook
```

Returns per-account sums/averages computed in SQL, not in JS.
- `views`, `likes`, `comments`, `shares`, `total_clicks`, `link_clicks`, `other_clicks`, `saves`, `follows`, `interactions`, `engagement` → SUM
- `reach` → AVG (räckvidd cannot be summed — this is a firm rule)
- `post_count` → COUNT
- `posts_per_day` → COUNT / date_range_days

### Post type aggregation

```
GET /api/post-types?account=all&fields=views,reach,likes
```

Returns per-post-type averages and counts, computed in SQL.

### Trend analysis

```
GET /api/trends?metric=interactions&accounts=id1,id2&granularity=month
```

Returns monthly time series per account for the selected metric.

### Database maintenance

```
POST /api/maintenance/vacuum           → VACUUM the database
POST /api/maintenance/redetect-collab  → Re-run collab detection across all data
GET  /api/maintenance/stats            → DB file size, row counts, date range
GET  /api/maintenance/backup           → Stream .db file as download
```

### Server entry point (server/index.js)

Must read environment variables with sensible defaults:

```js
const HOST = process.env.HOST || '127.0.0.1';  // Docker sets to 0.0.0.0
const PORT = process.env.PORT || 3001;
const DB_PATH = process.env.DB_PATH || './data/analytics.db';
```

In production mode (`NODE_ENV=production`), Express also serves static files from `dist/`.

---

## CSV Processing — Server Side

Move CSV parsing from browser to server. Flow:

1. Frontend sends raw CSV file via `POST /api/imports` (multipart)
2. Server receives file, parses with PapaParse (Node version)
3. Server detects platform from headers (existing `detectPlatform()` logic)
4. Server maps columns (existing `FB_COLUMN_MAPPINGS` / `IG_COLUMN_MAPPINGS`)
5. Server calculates `interactions`, `engagement` per row
6. Server derives `month` from publish_time dates in the file
7. Server inserts into `imports` + `posts` tables (using transaction)
8. Dedup: `INSERT OR REPLACE` on `(post_id, platform)` unique constraint
9. After insert: re-run collab detection for affected accounts
10. Returns import summary to frontend

**Collab detection** runs as a post-import step across ALL data (not just the new import), since an account's status can change when new data arrives.

---

## Frontend Changes

### Remove entirely:
- `src/utils/storageService.js` — replace with `src/utils/apiClient.js`
- `src/utils/memoryUtils.js` — no longer needed
- `src/utils/electronApiEmulator.js` — no longer needed
- `src/renderer/components/StorageIndicator/` — no longer needed
- All localStorage/IndexedDB code
- 12h auto-cleanup logic in `App.jsx`

### New file: `src/utils/apiClient.js`
Thin wrapper around fetch:
```js
export const api = {
  getImports: () => fetch('/api/imports').then(r => r.json()),
  uploadCSV: (file) => { /* FormData POST */ },
  deleteImport: (id) => fetch(`/api/imports/${id}`, { method: 'DELETE' }),
  getPosts: (params) => fetch(`/api/posts?${new URLSearchParams(params)}`).then(r => r.json()),
  getAccounts: (params) => fetch(`/api/accounts?${new URLSearchParams(params)}`).then(r => r.json()),
  getPostTypes: (params) => fetch(`/api/post-types?${new URLSearchParams(params)}`).then(r => r.json()),
  getTrends: (params) => fetch(`/api/trends?${new URLSearchParams(params)}`).then(r => r.json()),
  vacuum: () => fetch('/api/maintenance/vacuum', { method: 'POST' }),
  getStats: () => fetch('/api/maintenance/stats').then(r => r.json()),
};
```

### Modified components:

**FileUploader.jsx** — Simplified. No longer parses CSV in browser. Just sends file to server and shows progress. No memory checks needed.

**AccountView.jsx** — Remove `summarizeByAccount()` (200+ lines). Fetch pre-aggregated data from `/api/accounts`. Keep sorting/pagination UI but drive it via API params instead of client-side sort.

**PostView.jsx** — Server-side pagination. Remove client-side sort/filter. Pass sort/filter/page params to API.

**PostTypeView.jsx** — Remove `aggregateByPostType()`. Fetch from `/api/post-types`.

**TrendAnalysisView.jsx** — Remove `monthlyAccountData` computation. Fetch from `/api/trends`.

**MainView.jsx** — Remove memory-related state. Add "Database" tab or section for import management (list imports, delete, see coverage). The field selector and platform filter remain.

**LoadedFilesInfo.jsx** — Adapt to use `/api/imports` instead of localStorage metadata. Add month display, delete-per-import. Add "coverage map" showing which months have data.

**App.jsx** — Remove IndexedDB init, stale data check, memory warnings. On mount, check `/api/maintenance/stats` to see if DB has data → show MainView or FileUploader.

### New component: ImportManager
Shows:
- Table of imports: month, platform, filename, row count, date imported
- Delete button per import
- Coverage visualization: which months have data (calendar/grid)
- "Vacuum database" button
- DB file size display

### Keep unchanged:
- All `src/renderer/components/ui/` (shadcn components)
- `src/lib/utils.js`
- `src/renderer/styles/globals.css`
- `tailwind.config.js`, `postcss.config.js`
- `src/utils/columnConfig.js` — shared between server and client (move to `shared/`)

---

## File Structure (target)

```
├── Dockerfile                    # Multi-stage build
├── docker-compose.yml            # Volume mount + port mapping
├── .dockerignore
├── server/
│   ├── index.js                  # Express app entry (reads HOST env var)
│   ├── db/
│   │   ├── connection.js         # better-sqlite3 init + migration runner
│   │   ├── schema.sql            # Initial schema
│   │   └── migrations/           # Numbered SQL migrations
│   ├── routes/
│   │   ├── imports.js            # POST/GET/DELETE /api/imports
│   │   ├── posts.js              # GET /api/posts
│   │   ├── accounts.js           # GET /api/accounts
│   │   ├── postTypes.js          # GET /api/post-types
│   │   ├── trends.js             # GET /api/trends
│   │   └── maintenance.js        # Vacuum, stats, backup
│   ├── services/
│   │   ├── csvProcessor.js       # Parse CSV, map columns, calculate fields
│   │   └── collabDetector.js     # Collab detection logic
│   └── middleware/
│       └── errorHandler.js
├── shared/
│   └── columnConfig.js           # Moved from src/utils/ — used by both server and client
├── data/
│   └── .gitkeep                  # analytics.db created here at runtime
├── src/                          # React frontend (modified)
│   ├── utils/
│   │   ├── apiClient.js          # NEW — replaces storageService.js
│   │   ├── dataProcessing.js     # Simplified — field definitions only
│   │   └── version.js
│   └── renderer/                 # Components (modified as described above)
├── vite.config.js                # Add proxy: '/api' → 'http://127.0.0.1:3001'
├── package.json                  # Add: express, better-sqlite3, multer, concurrently
└── README.md                     # Updated startup instructions
```

---

## Docker

### Dockerfile (multi-stage)

```dockerfile
# Stage 1: Build frontend
FROM node:22-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Production
FROM node:22-alpine
WORKDIR /app

# better-sqlite3 is a native module — needs build tools
RUN apk add --no-cache python3 make g++

COPY package.json package-lock.json ./
RUN npm ci --omit=dev && apk del python3 make g++

# Copy server + shared code
COPY server/ ./server/
COPY shared/ ./shared/

# Copy built frontend from stage 1
COPY --from=builder /app/dist ./dist

# Data directory (will be mounted as volume)
RUN mkdir -p /data

ENV NODE_ENV=production
ENV HOST=0.0.0.0
ENV PORT=3001
ENV DB_PATH=/data/analytics.db

EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s \
  CMD wget -qO- http://localhost:3001/api/health || exit 1

CMD ["node", "server/index.js"]
```

### docker-compose.yml

```yaml
services:
  meta-analytics:
    build: .
    container_name: meta-analytics
    ports:
      - "127.0.0.1:3001:3001"    # localhost only — not exposed to network
    volumes:
      - ./data:/data              # SQLite database persists here
    environment:
      - NODE_ENV=production
      - DB_PATH=/data/analytics.db
      - HOST=0.0.0.0
      - PORT=3001
    restart: unless-stopped
```

### .dockerignore

```
node_modules/
dist/
data/analytics.db
.git/
.DS_Store
*.log
.vscode/
_old_facebook/
_old_instagram/
old_csv/
```

### Key Docker considerations

1. **Volume mount is critical.** Without `./data:/data`, the database dies with the container. The `data/` directory on the host is the single source of truth.

2. **better-sqlite3 native compilation.** This C++ addon must be compiled inside the container (Alpine Linux), not on the host. The Dockerfile installs build tools, compiles, then removes them to keep the image small.

3. **DB_PATH environment variable.** `server/db/connection.js` must read `process.env.DB_PATH` to know where the database file is:
   - Docker: `/data/analytics.db` (inside the mounted volume)
   - Native: `./data/analytics.db` (relative to project root)
   - Default fallback: `./data/analytics.db`

4. **HOST environment variable.** `server/index.js` must read `process.env.HOST`:
   - Docker: `0.0.0.0` (required — container networking needs this)
   - Native: `127.0.0.1` (safe default — localhost only)

5. **File uploads in Docker.** Multer temp directory must be inside the container (default `/tmp` is fine). Only the parsed data goes to the SQLite volume — raw CSVs are not stored.

6. **VACUUM in Docker.** Works normally. SQLite creates a temp file during VACUUM — ensure the `/data` volume has enough space (2x current DB size briefly).

7. **Backup from Docker.** The `POST /api/maintenance/backup` endpoint should stream the .db file as a download. The user can also just copy `./data/analytics.db` from the host since it's volume-mounted.

8. **Image size target.** With multi-stage build and Alpine: ~150–200 MB. The SQLite database is not in the image.

---

## package.json changes

Add dependencies:
```json
{
  "dependencies": {
    "express": "^4.21.0",
    "better-sqlite3": "^11.0.0",
    "multer": "^1.4.5-lts.1",
    "papaparse": "^5.4.1"
  },
  "devDependencies": {
    "concurrently": "^9.0.0"
  },
  "scripts": {
    "dev": "concurrently \"node server/index.js\" \"vite\"",
    "start": "node server/index.js",
    "build": "vite build",
    "preview": "vite preview",
    "docker:up": "docker compose up --build -d",
    "docker:down": "docker compose down",
    "docker:logs": "docker compose logs -f"
  }
}
```

PapaParse moves from client-only to also server-side (it supports Node natively).

---

## vite.config.js changes

Add API proxy for development:
```js
server: {
  port: 5173,
  proxy: {
    '/api': {
      target: 'http://127.0.0.1:3001',
      changeOrigin: true
    }
  }
}
```

Production: Express serves static files from `dist/` AND handles `/api/*`.

Remove `base: '/meta_analytics/'` from vite.config.js — the app is no longer deployed to GitHub Pages subpath. Set `base: '/'`.

Remove `.github/workflows/deploy.yml` on the `metadb` branch — GitHub Pages deploy is not applicable for this variant. (It remains on `main` where the browser-only version uses it.)

---

## Important Business Rules (from existing codebase)

1. **Reach cannot be summed.** Always AVG for account-level reach. This is non-negotiable.
2. **Engagement differs by platform.** FB: likes+comments+shares+total_clicks. IG: likes+comments+shares+saves+follows. Store the computed value at import time.
3. **Interactions = likes + comments + shares** on both platforms.
4. **Meta timezone:** CSV exports use Pacific Time (PST/PDT). Convert to Stockholm time during import, handling DST transitions correctly.
5. **Collab detection:** Accounts with ≤2 posts that are NOT known SR accounts (see `COLLAB_SAFE_TERMS` and `KNOWN_ACCOUNTS` in webDataProcessor.js) are flagged `is_collab=1`. Must re-run across all data after each import.
6. **Dedup by post_id + platform.** Same post_id on same platform = same post. Use UPSERT.
7. **FB-only fields** (total_clicks, link_clicks, other_clicks) show as N/A for IG accounts.
8. **IG-only fields** (saves, follows) show as N/A for FB accounts.

---

## Migration Strategy

No need to migrate data from the old IndexedDB version. Users re-upload their CSVs once into the new SQLite version, and they're permanent.

The old browser-only app continues to work on the `main` branch and on the existing GitHub Pages deploy. The two versions are independent.

---

## Security

- **Native mode:** Express binds to `127.0.0.1` only (not `0.0.0.0`)
- **Docker mode:** Express binds to `0.0.0.0` inside the container (required for Docker networking), but `docker-compose.yml` maps the port to `127.0.0.1:3001` on the host — so it's still localhost-only
- No authentication needed (single-user local app)
- CORS not needed (same origin)
- Sanitize SQL inputs via parameterized queries (better-sqlite3 does this by default)
- Use environment variable `HOST=0.0.0.0` (Docker) vs `HOST=127.0.0.1` (native) in `server/index.js`

---

## Testing

- Keep `validate.js` but adapt it to call the API endpoints instead of shimming browser APIs
- Add a simple health check: `GET /api/health` → `{ status: "ok", dbSize: "42 MB", posts: 12345 }`
- Docker health check uses the same endpoint (configured in Dockerfile `HEALTHCHECK`)
- Verify both startup paths work: `docker compose up` and `npm start`
- Verify volume persistence: `docker compose down` then `docker compose up` — data must survive

---

## Implementation Order

0. **Create branch:** `git checkout -b metadb` from `main`. All subsequent work on this branch only.
1. **Server skeleton:** Express + better-sqlite3 + schema + migrations + `DB_PATH`/`HOST` env vars
2. **CSV import endpoint:** POST /api/imports with full parsing pipeline
3. **Read endpoints:** GET /api/posts, /api/accounts, /api/post-types, /api/trends
4. **Frontend apiClient.js:** Replace storageService.js
5. **Adapt components** one by one: FileUploader → AccountView → PostView → PostTypeView → TrendAnalysisView → MainView
6. **Import manager UI:** List/delete imports, coverage view
7. **Maintenance endpoints:** Vacuum, stats, backup (download .db)
8. **Docker:** Dockerfile (multi-stage), docker-compose.yml, .dockerignore, health check
9. **Cleanup:** Remove dead code (IndexedDB, localStorage, memoryUtils, StorageIndicator, 12h cleanup, electronApiEmulator)
10. **Update README** with both native and Docker startup instructions

---

## What NOT to change

- **The `main` branch.** Zero commits on main. All work on `metadb` only.
- Visual design, colors, layout
- shadcn/ui components
- Column display names (Swedish)
- Export to CSV/Excel functionality (stays client-side)
- Tooltip mode preference (nearest, not index)
- Professional Swedish tone in UI text
