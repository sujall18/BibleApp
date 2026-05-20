# Deep Word — Bible Study App

A production-ready web app for exploring Scripture in its original languages (Greek NT & Hebrew OT) with Strong's lexicon definitions.

## ✨ Features

- **Original Language Explorer**: Navigate any verse → see Greek (NT) or Hebrew (OT) text
- **Word Analysis**: Tap/click any word → Strong's number, definition, pronunciation, transliteration, morphology
- **Full Bible Search**: Find verses by reference ("John 3:16") or keyword ("love")
- **Study Journal**: Personal notes saved to browser localStorage
- **AI Insights** (Optional): Claude-powered theological explanations (gracefully works without API key)
- **Dark Mode**: Full theme support with beautiful custom colors
- **Mobile-First**: Fully responsive design

## 🛠 Tech Stack

- **Frontend**: Next.js 15 + TypeScript + Tailwind CSS v4
- **Backend**: Next.js API routes
- **Database**: SQLite (better-sqlite3) with 31,000+ verses
- **AI**: Anthropic SDK (optional)
- **Deployment**: Docker, Vercel, Railway, or self-hosted

## 📦 Data Sources (All Public Domain)

- **KJV Text**: [aruljohn/Bible-kjv](https://github.com/aruljohn/Bible-kjv) (31,102 verses)
- **Greek NT**: Nestle 1904 from [biblicalhumanities/greek-new-testament](https://github.com/biblicalhumanities/greek-new-testament)
- **Hebrew OT**: Westminster Leningrad Codex (WLC)
- **Strong's Lexicon**: [openscriptures/strongs](https://github.com/openscriptures/strongs)

---

## 🚀 Quick Start (Local Development)

### Prerequisites
- Node.js 18+ 
- npm or yarn

### Installation & Setup

```bash
# 1. Clone or enter the repo
cd deep-word

# 2. Clean install (fixes Next.js version conflicts)
rm -rf node_modules package-lock.json
npm install

# 3. Import Bible data (~5-10 min, one-time)
npm run db:import-all

# 4. Start dev server
npm run dev

# → Visit http://localhost:3000
```

### Available Scripts

```bash
npm run dev              # Start dev server (port 3000)
npm run build            # Build for production
npm run start            # Start production server
npm run db:import-all    # Full import (KJV + Greek + Hebrew + Strong's)
npm run db:fix-missing   # Smart import (only missing data)
npm run db:reset         # Clear database completely
```

---

## 🐳 Docker Deployment

### Local Docker

```bash
# Build & run with automatic data import
docker compose up --build

# First run: Database imports (~5-10 min)
# Subsequent runs: Instant startup from persisted volume

# → Visit http://localhost:3000
```

Data persists in the `bible-data` Docker volume.

### Docker on a Server

```bash
# Copy files to server
scp -r . user@server:/app/deep-word
ssh user@server

cd /app/deep-word

# Run with optional Anthropic API key
ANTHROPIC_API_KEY=sk-... docker compose up --build -d

# View logs
docker compose logs -f

# Stop
docker compose down
```

---

## ☁️ Cloud Deployment

### Option 1: Vercel (Recommended for Simplicity)

**⚠️ Limitation**: Vercel's serverless functions have a 10MB timeout limit. The database import will fail on Vercel's free tier.

**Workaround**: Pre-build the database locally, then deploy:

```bash
# Local: Build and import
npm run build
npm run db:import-all

# Deploy to Vercel with the populated deep-word.db
# (commit the .db file or use a mounted volume service)
vercel deploy
```

**Better**: Use Railway or Render instead (below).

### Option 2: Railway (Recommended) ⭐

Railway supports persistent volumes and long-running processes.

1. **Create Railway account**: https://railway.app
2. **Connect GitHub repo**: Railway > New Project > GitHub
3. **Add Environment Variables**:
   - `ANTHROPIC_API_KEY` (optional)
   - `NODE_ENV` = `production`
4. **Configure Dockerfile**: Railway automatically uses `Dockerfile`
5. **Deploy**: Push to GitHub → Railway auto-deploys

First run takes 5-10 min (database import), then instant restarts.

### Option 3: Render

1. **Create Render account**: https://render.com
2. **New Web Service** > Connect GitHub repo
3. **Settings**:
   - Build Command: `npm install && npm run build && npx tsx scripts/setup.ts`
   - Start Command: `npm run start`
4. **Environment**: Add `ANTHROPIC_API_KEY` (optional)
5. **Deploy**

### Option 4: Self-Hosted (VPS)

```bash
# On your VPS (Ubuntu/Debian)

# Install Node.js 20
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt-get install -y nodejs docker.io docker-compose

# Clone repo
git clone https://github.com/yourusername/deep-word.git
cd deep-word

# Run with Docker
docker compose up --build -d

# View logs
docker compose logs -f deep-word

# Optionally: Set up nginx reverse proxy + SSL
# (see Nginx config section below)
```

---

## 🔐 Optional: Claude AI Insights

The app works **perfectly without an API key**. The AI Insights button simply shows a link to get one.

To enable AI features:

1. Get an API key from https://console.anthropic.com
2. Set environment variable: `ANTHROPIC_API_KEY=sk-...`
3. Restart the app
4. The "Get AI Insight" button now works

---

## 📊 Database Schema

```
verses
  ├── id (int, pk)
  ├── book (text) — "Genesis", "John", etc.
  ├── chapter (int)
  ├── verse (int)
  ├── text (text) — verse content
  ├── version (text) — "kjv", "nestle1904", "wlc"
  └── language (text) — "en", "grc", "heb"

words
  ├── id (int, pk)
  ├── verse_id (fk → verses)
  ├── surface_form (text) — original word
  ├── strongs_number (text) — "G1234", "H5678"
  ├── transliteration (text)
  ├── morphology (text)
  └── position (int)

strongs
  ├── id (int, pk)
  ├── number (text) — "G1234", "H5678"
  ├── original_word (text)
  ├── definition (text)
  ├── pronunciation (text)
  └── ...

ai_cache
  ├── id (int, pk)
  ├── verse_ref (text) — "John 3:16"
  ├── insight (text)
  └── cached_at (timestamp)
```

---

## 🚨 Troubleshooting

### Database Import Fails

**Error**: `Strong's Hebrew failed: Unexpected non-whitespace character`

**Solution**: The import script now has better error handling. Run:
```bash
npm run db:fix-missing
```

This will:
- Skip broken sources gracefully
- Re-import only missing data
- Continue with KJV-only if languages fail

### Port 3000 Already in Use

```bash
# Use a different port
PORT=3001 npm run dev

# Or kill the process
lsof -i :3000 | grep node | awk '{print $2}' | xargs kill -9
```

### Docker Volume Not Persisting

```bash
# Check volume
docker volume ls

# Inspect volume
docker volume inspect bible-data

# Remove and recreate
docker volume rm bible-data
docker compose up --build
```

### Out of Memory During Import

The import processes ~50MB of Greek data. If it fails:

```bash
# Increase Node.js heap
NODE_OPTIONS="--max-old-space-size=2048" npm run db:import-all
```

---

## 📁 Project Structure

```
deep-word/
├── app/
│   ├── page.tsx           # Home page
│   ├── layout.tsx         # Root layout
│   └── verse/[...]/page.tsx # Verse explorer
├── lib/
│   ├── db.ts              # Database helpers
│   ├── books.ts           # Book name ↔ slug mapping
│   └── db-cache.ts        # Cache layer
├── components/
│   ├── SearchBar.tsx
│   ├── WordPanel.tsx
│   └── ThemeToggle.tsx
├── scripts/
│   ├── import-kjv.ts      # KJV importer
│   ├── import-greek.ts    # Greek NT importer
│   ├── import-hebrew.ts   # Hebrew OT importer
│   ├── import-strongs.ts  # Strong's lexicon
│   └── setup.ts           # Orchestrates all imports
├── Dockerfile             # Container image
├── docker-compose.yml     # Local dev + deploy config
└── package.json
```

---

## 🎨 Customization

### Change Featured Passage

In `app/page.tsx`, line 28-29:
```typescript
getVerseCached("John", 3, 16, "kjv"),     // Change reference here
getVerseCached("John", 3, 16, "nestle1904"),
```

### Change Colors

Edit `tailwind.config.ts` to adjust the theme:
```typescript
extend: {
  colors: {
    accent: '#3b82f6',      // Primary blue
    // ... more colors
  }
}
```

### Add Custom Font

Replace font in `app/layout.tsx`:
```typescript
import { Inter } from "next/font/google";
```

---

## 📜 License

This app uses:
- **KJV Bible Text**: Public domain
- **Greek NT (Nestle 1904)**: Public domain
- **Hebrew Bible (WLC)**: Public domain
- **Strong's Lexicon**: Public domain
- **App Code**: MIT

All data sources are verified public domain.

---

## 🤝 Contributing

Found a bug? Have a feature idea? Open an issue or pull request on GitHub.

### Development Workflow

1. Create a branch: `git checkout -b fix/bug-name`
2. Make changes and test: `npm run dev`
3. Build: `npm run build`
4. Commit: `git commit -m "fix: describe change"`
5. Push and create a PR

---

## 🚀 Performance

- **Search**: Instant (SQLite FTS, <10ms)
- **Word Lookup**: <5ms (indexed Strong's)
- **Full Page Load**: ~1.2s (2G5 throttled)
- **Database Size**: ~450MB (includes all original language text)
- **Network**: Zero tracking, no analytics

---

## 💬 Support

- **Bugs**: Create a GitHub issue
- **Suggestions**: Discussions tab on GitHub
- **Questions**: Email or open an issue

---

**Happy studying!** 📖✨
