# Deep Word — Complete Architecture & System Design Guide

**Version:** 1.0  
**Last Updated:** 2025-05-15  
**Project:** Bible Study Web Application  
**Tech Stack:** Next.js 15, TypeScript, SQLite, Tailwind CSS v4, Anthropic API (optional)

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [System Architecture](#system-architecture)
3. [Technology Stack](#technology-stack)
4. [File Structure](#file-structure)
5. [Database Schema](#database-schema)
6. [Data Flow](#data-flow)
7. [Components & Pages](#components--pages)
8. [API Routes](#api-routes)
9. [Import Process](#import-process)
10. [Deployment](#deployment)
11. [Development Workflow](#development-workflow)
12. [Key Design Decisions](#key-design-decisions)
13. [Future Enhancements](#future-enhancements)

---

## Project Overview

### What is Deep Word?

A modern Bible study web application that enables users to explore scripture in three dimensions:

1. **Original Languages** — Greek New Testament (Nestle 1904) + Hebrew Old Testament (Westminster Leningrad Codex)
2. **English Translation** — King James Version (KJV) for direct comparison
3. **Lexical Analysis** — Strong's definitions, transliterations, morphology, cross-references, and optional AI theological insights

### Core User Journeys

| Journey | User Actions | Data Involved |
|---------|--------------|---------------|
| **Discover** | Search "John 3:16" or keyword "love" | KJV text search + verse reference resolution |
| **Study** | Navigate to verse, tap Greek/Hebrew word | Original language text + Strong's lookup + cross-refs |
| **Understand** | Request AI insight on a word | Claude API call (cached, never repeated for same word/verse) |
| **Remember** | Add personal notes to study journal | Browser localStorage (no server writes) |

### Key Constraints

- **Cost:** Completely free (no paid APIs except optional Anthropic)
- **Data:** All public domain (KJV, Nestle 1904, WLC, Strong's)
- **Deployment:** Works on single-server or Docker (no scaling needed)
- **Offline:** Can work with pre-imported database + no AI features

---

## System Architecture

### High-Level Architecture Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                     BROWSER / CLIENT                            │
├────────────────────────────────────────────────────────────────┤
│ - Search Bar (verse ref or keyword)                             │
│ - Verse Display (Greek/Hebrew + KJV, clickable words)          │
│ - Word Panel (Strong's + cross-refs + AI insight)              │
│ - Study Journal (localStorage-backed notes)                     │
└────────────────────────────────────────────────────────────────┘
                              ↓
                     (HTTP/API Calls)
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                   NEXT.JS SERVER (Node.js)                      │
├────────────────────────────────────────────────────────────────┤
│ Pages:                                                          │
│  - / (home + search)                                           │
│  - /verse/[book]/[chapter]/[verse] (word-by-word study)       │
│  - /verse/[book]/[chapter] (chapter view)                      │
│  - /search?q=... (keyword results)                             │
│  - /study (journal, localStorage UI)                           │
│                                                                 │
│ API Routes:                                                    │
│  - /api/word (Strong's lookup)                                 │
│  - /api/insight (Claude AI, with cache check)                 │
│  - /api/health (status)                                        │
│                                                                 │
│ Libs:                                                          │
│  - lib/db.ts (SQLite queries)                                  │
│  - lib/anthropic.ts (Claude SDK)                               │
│  - lib/books.ts (book name/slug mapping)                       │
└────────────────────────────────────────────────────────────────┘
                              ↓
                         (SQLite)
                              ↓
┌────────────────────────────────────────────────────────────────┐
│                    LOCAL SQLITE DATABASE                        │
├────────────────────────────────────────────────────────────────┤
│ Tables:                                                         │
│  - verses (31K+ rows) — KJV, Greek, Hebrew texts              │
│  - words (1M+ rows) — clickable words with Strong's           │
│  - strongs (15K rows) — definitions, pronunciation            │
│  - ai_cache (grows) — Claude responses (cached, never deleted) │
└────────────────────────────────────────────────────────────────┘
```

### Data Flow Sequence

#### 1. User Searches for a Verse
```
User types "John 3:16"
    ↓
SearchBar parses reference
    ↓
Navigate to /verse/john/3/16
    ↓
Server-side: Query SQLite for KJV + Greek/Hebrew
    ↓
VerseDisplay renders with clickable words
    ↓
WordPanel ready to accept clicks
```

#### 2. User Taps a Word
```
User clicks Greek word "λόγος" with Strong's "G3056"
    ↓
WordPanel opens (slide from right)
    ↓
Fetch /api/word?strongs=G3056&verseId=123
    ↓
Server returns:
   - Strong's entry (definition, pronunciation, morphology)
   - 5 random cross-references (same Strong's number elsewhere)
    ↓
Display in panel
    ↓
User can click "Generate theological insight"
```

#### 3. User Requests AI Insight
```
User clicks "Generate theological insight"
    ↓
Fetch /api/insight?strongs=G3056&word=logos&book=John&chapter=3&verse=16
    ↓
Server checks: Is this in ai_cache?
    ├─ YES → Return cached response immediately
    └─ NO → Call Claude API with:
         "In 3-4 sentences, explain the theological significance of 
          the word 'logos' (Strong's G3056) in the context of John 3:16..."
    ↓
Claude responds in ~1 second (max_tokens: 200)
    ↓
Cache the response (never call API again for this word/verse)
    ↓
Display in panel with "Claude · cached" badge
```

#### 4. User Adds Journal Note
```
User navigates to /study
    ↓
Form for: Reference (e.g., "John 3:16") + Note text
    ↓
On save: Store in localStorage as JSON array
    ↓
Browser persists notes — survives page reloads but not browser clear
    ↓
Notes searchable/editable/deletable in the UI
```

---

## Technology Stack

### Frontend
| Layer | Tech | Purpose |
|-------|------|---------|
| **Framework** | Next.js 15 (App Router) | Server-side rendering, file-based routing, optimized builds |
| **Language** | TypeScript | Type safety, better DX, fewer runtime errors |
| **Styling** | Tailwind CSS v4 | Utility-first CSS, dark mode support, responsive mobile-first |
| **State** | React 19 | Client-side interactivity (word panel, journal) |
| **Dark Mode** | CSS vars + `dark:` class | Automatic OS theme detection + user override via localStorage |

### Backend
| Layer | Tech | Purpose |
|-------|------|---------|
| **Runtime** | Node.js 20+ | JavaScript on server |
| **Framework** | Next.js (App Router) | API routes, server-side queries, middleware |
| **Database** | SQLite (better-sqlite3) | Single-file DB, no external server, perfect for read-heavy |
| **ORM** | Raw SQL (transactions) | Direct control, no abstraction overhead |
| **AI** | Anthropic SDK (Claude Sonnet 4) | Theological insights (optional, cached) |

### Data & DevOps
| Component | Tech | Purpose |
|-----------|------|---------|
| **Package Manager** | npm | Dependency management |
| **Build** | Next.js standalone | Single executable, minimal size |
| **Container** | Docker + docker-compose | Easy one-command deployment |
| **Database** | SQLite WAL mode | Concurrent reads, crash-safe |
| **Caching** | In-memory + SQLite | API responses cached forever |

---

## File Structure

```
deep-word/
├── app/                              # Next.js App Router
│   ├── globals.css                   # Tailwind imports + CSS variables
│   ├── layout.tsx                    # Root layout (metadata, theme script)
│   ├── page.tsx                      # Home page (search bar + examples)
│   ├── not-found.tsx                 # 404 page
│   ├── error.tsx                     # Error boundary
│   │
│   ├── search/
│   │   └── page.tsx                  # Search results (/search?q=...)
│   │
│   ├── verse/[book]/[chapter]/[verse]/
│   │   ├── page.tsx                  # Verse view (main study page)
│   │   └── loading.tsx               # Loading skeleton (optional)
│   │
│   ├── verse/[book]/[chapter]/
│   │   └── page.tsx                  # Chapter view (all verses)
│   │
│   ├── study/
│   │   └── page.tsx                  # Study journal (client-side notes)
│   │
│   └── api/
│       ├── word/route.ts             # GET /api/word (Strong's + cross-refs)
│       ├── insight/route.ts          # GET /api/insight (Claude AI, cached)
│       └── health/route.ts           # GET /api/health (status check)
│
├── components/                        # React components
│   ├── SearchBar.tsx                 # Search input with live submission
│   ├── VerseDisplay.tsx              # Stacked Greek/Hebrew + KJV
│   ├── WordPanel.tsx                 # Slide-in side panel for words
│   ├── WordPanelWrapper.tsx          # Client wrapper for word state
│   └── Header.tsx                    # Sticky header with nav
│
├── lib/
│   ├── db.ts                         # SQLite queries + types
│   ├── anthropic.ts                  # Claude API wrapper
│   └── books.ts                      # 66-book canonical names + slugs
│
├── scripts/
│   ├── init-db.ts                    # Create tables + indexes
│   ├── import-kjv.ts                 # Fetch & parse KJV JSON
│   ├── import-strongs.ts             # Fetch & parse Strong's JS files
│   ├── import-greek.ts               # Fetch & parse Nestle1904 XML
│   ├── import-hebrew.ts              # Fetch & parse WLC OSIS XML
│   ├── setup.ts                      # Orchestrate all imports (npm run db:import-all)
│   ├── helpers.ts                    # Retry logic, logging, progress bars
│   └── check-db.ts                   # Verify import success
│
├── public/                            # Static assets (favicon, etc)
├── .env.local                         # Environment: ANTHROPIC_API_KEY (optional)
├── .gitignore                         # Exclude node_modules, .next, *.db
├── .dockerignore                      # Exclude from Docker build
│
├── package.json                       # Dependencies + scripts
├── tsconfig.json                      # TypeScript config
├── next.config.ts                     # Next.js config (standalone, headers)
├── postcss.config.mjs                 # Tailwind + PostCSS
│
├── Dockerfile                         # Multi-stage build (deps → builder → runner)
├── docker-compose.yml                 # Compose file with volume for DB
├── entrypoint.sh                      # Docker entry (auto-import on first run)
│
├── SETUP.md                           # Quick start guide
├── ARCHITECTURE.md                    # This file
└── deep-word.db                       # SQLite database (created on first import)
```

---

## Database Schema

### Table: `verses`

**Purpose:** Store Bible text in three languages/versions

```sql
CREATE TABLE verses (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  book TEXT NOT NULL,
  chapter INTEGER NOT NULL,
  verse INTEGER NOT NULL,
  text TEXT NOT NULL,
  language TEXT NOT NULL CHECK (language IN ('english','greek','hebrew')),
  version TEXT NOT NULL CHECK (version IN ('kjv','nestle1904','wlc')),
  UNIQUE(book, chapter, verse, version)
);
```

| Column | Type | Notes |
|--------|------|-------|
| `id` | PK | Auto-increment row ID |
| `book` | TEXT | Canonical name: "John", "Genesis", etc. |
| `chapter` | INT | 1-150 range |
| `verse` | INT | 1-176 range (Psalm 119 is longest) |
| `text` | TEXT | Full verse text (e.g., "In the beginning was the Word...") |
| `language` | ENUM | 'english', 'greek', or 'hebrew' |
| `version` | ENUM | 'kjv', 'nestle1904', or 'wlc' |

**Indexes:**
```sql
CREATE INDEX idx_verses_ref ON verses(book, chapter, verse, version);
```

**Example Rows:**
```
id=1: John, 3, 16, "For God so loved the world...", english, kjv
id=2: John, 3, 16, "ἦν λόγος καὶ ὁ λόγος...", greek, nestle1904
id=3: John, 3, 16, "בְרֵאשִׁית בָּרָא אֱלֹהִים...", hebrew, wlc
```

---

### Table: `words`

**Purpose:** Store individual words with morphological data + Strong's linkage

```sql
CREATE TABLE words (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  verse_id INTEGER NOT NULL,
  position INTEGER NOT NULL,
  surface_form TEXT NOT NULL,
  strongs_number TEXT,
  transliteration TEXT,
  morphology TEXT,
  FOREIGN KEY (verse_id) REFERENCES verses(id) ON DELETE CASCADE
);
```

| Column | Type | Notes |
|--------|------|-------|
| `verse_id` | FK | Points to `verses.id` |
| `position` | INT | Word order in verse (0-indexed) |
| `surface_form` | TEXT | Actual word: "λόγος", "הַדָּבָר", "word" |
| `strongs_number` | TEXT | Strong's reference: "G3056", "H1697", or NULL |
| `transliteration` | TEXT | Romanization: "logos", "ha-dabar", or NULL |
| `morphology` | TEXT | Grammatical code: "N-NSM", "HR/Ncfsa", etc. |

**Indexes:**
```sql
CREATE INDEX idx_words_verse ON words(verse_id);
CREATE INDEX idx_words_strongs ON words(strongs_number);
```

**Example Row:**
```
id=12345
verse_id=1
position=2
surface_form="λόγος"
strongs_number="G3056"
transliteration="logos"
morphology="N-NSM"  (Noun, Nominative, Singular, Masculine)
```

---

### Table: `strongs`

**Purpose:** Strong's lexicon (definitions, pronunciations, usage counts)

```sql
CREATE TABLE strongs (
  number TEXT PRIMARY KEY,
  original_word TEXT,
  transliteration TEXT,
  pronunciation TEXT,
  definition TEXT,
  kjv_usage TEXT,
  part_of_speech TEXT
);
```

| Column | Type | Notes |
|--------|------|-------|
| `number` | PK | "G3056", "H1697", etc. |
| `original_word` | TEXT | Greek/Hebrew: "λόγος", "דָּבָר" |
| `transliteration` | TEXT | "logos", "dabar" |
| `pronunciation` | TEXT | "LOG-os", "dah-VAHR" |
| `definition` | TEXT | Full definition from Strong's |
| `kjv_usage` | TEXT | "KJV: word (218x), ...other forms..." |
| `part_of_speech` | TEXT | "noun", "verb", "adjective", etc. |

**Example Row:**
```
number="G3056"
original_word="λόγος"
transliteration="logos"
pronunciation="LOG-os"
definition="a word, speech, divine utterance, reason, motive"
kjv_usage="KJV: word (218), ...speak (7), ...utterance (2)..."
part_of_speech="noun"
```

---

### Table: `ai_cache`

**Purpose:** Cache Claude API responses (never repeat the same call)

```sql
CREATE TABLE ai_cache (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  strongs_number TEXT NOT NULL,
  book TEXT NOT NULL,
  chapter INTEGER NOT NULL,
  verse INTEGER NOT NULL,
  response TEXT NOT NULL,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(strongs_number, book, chapter, verse)
);
```

| Column | Type | Notes |
|--------|------|-------|
| `strongs_number` | TEXT | "G3056", "H1697" |
| `book` | TEXT | "John", "Genesis" |
| `chapter` | INT | Chapter number |
| `verse` | INT | Verse number |
| `response` | TEXT | Full Claude response (max_tokens=200) |
| `created_at` | DATETIME | Timestamp of first request |

**Why UNIQUE:** Prevents duplicate API calls for the same word in the same verse

**Example Row:**
```
strongs_number="G3056"
book="John"
chapter="3"
verse="16"
response="In John 3:16, 'logos' (Greek for 'word') refers to the 
          divine utterance or eternal Word of God, emphasizing 
          God's complete expression of himself through Christ. 
          This theological concept is foundational to Johannine 
          Christology..."
```

---

## Components & Pages

### Page: `/` (Home)

**File:** `app/page.tsx`

**Purpose:** Entry point, search discovery, example verses

**Structure:**
```
┌─────────────────────────────┐
│      Deep Word (logo)       │
│  Explore scripture in the   │
│  original Greek & Hebrew    │
├─────────────────────────────┤
│  [  Search Bar  ] [Magnify] │
├─────────────────────────────┤
│ Examples (clickable links):  │
│  John 3:16  Genesis 1:1     │
│  Psalm 23:1  Isaiah 40:31   │
├─────────────────────────────┤
│  [Study Journal] [Dark Mode]│
└─────────────────────────────┘
```

**Key Logic:**
- `handleSearch(formData)` → Parse reference OR redirect to /search?q=
- `hasData()` → Show warning banner if DB is empty (import not run yet)
- Dark mode toggle via localStorage + `document.documentElement.classList`

---

### Page: `/verse/[book]/[chapter]/[verse]`

**File:** `app/verse/[book]/[chapter]/[verse]/page.tsx`

**Purpose:** Core study experience — side-by-side original language + English with clickable words

**Layout:**
```
┌─────────────────────────────────────────────┐
│  Verse Header: John 3:16  [← prev] [next →]│
├─────────────────────────────────────────────┤
│                                             │
│  Greek (Nestle 1904)                       │
│  ἦν [λόγος] καὶ ὁ [λόγος] ἦν πρὸς...     │
│                                             │
│  ─────────────────────────────────────────  │
│                                             │
│  KJV English                                │
│  For God so loved the world, that he       │
│  gave his only begotten Son...             │
│                                             │
├─────────────────────────────────────────────┤
│  [+ Add to journal] [Chapter 3] [Footer]    │
└─────────────────────────────────────────────┘

  ╔════════════════════════════╗
  ║   Word Panel (on click)    ║
  ║  λόγος                     ║
  ║                            ║
  ║  G3056 | noun              ║
  ║  logos | LOG-os            ║
  ║                            ║
  ║  Definition: ...           ║
  ║                            ║
  ║  Also in Scripture:        ║
  ║  John 1:1, Mark 4:14, ...  ║
  ║                            ║
  ║  [Generate AI Insight]     ║
  ╚════════════════════════════╝
```

**Data Flow:**
1. Extract route params: `book="john"`, `chapter=3`, `verse=16`
2. Slug → canonical name: `"john"` → `"John"`
3. Query DB: `getVerse("John", 3, 16, "kjv")` + `getVerse("John", 3, 16, "nestle1904")`
4. If Greek/Hebrew exists: `getWordsForVerse(verseId)` → Array of Word objects
5. Render VerseDisplay + WordPanelWrapper with interactive state
6. User clicks word → setState → WordPanel slides in + fetches /api/word

---

### Page: `/verse/[book]/[chapter]`

**File:** `app/verse/[book]/[chapter]/page.tsx`

**Purpose:** Chapter overview — list all verses in chapter

**Layout:**
```
┌─────────────────────────────────┐
│ John — Chapter 3                │
├─────────────────────────────────┤
│                                 │
│ 1  Now there was a man of      │
│    the Pharisees named...      │
│                                 │
│ 2  The same came to Jesus by   │
│    night and said unto him...  │
│                                 │
│ ... (all verses in chapter)    │
│                                 │
├─────────────────────────────────┤
│ [← Chapter 2] [Chapter 4 →]     │
└─────────────────────────────────┘
```

**Key Feature:** Each verse is clickable → navigate to /verse/john/3/{verse}

---

### Page: `/search`

**File:** `app/search/page.tsx`

**Purpose:** Full-text search results

**Flow:**
1. User types "love" in search bar
2. Redirected to `/search?q=love`
3. Server queries: `searchVerses("love", 25)` → List of verses containing "love"
4. Highlight matching text in results

---

### Page: `/study`

**File:** `app/study/page.tsx`

**Purpose:** Personal journal — save notes per verse

**Features:**
- Input: Verse reference (e.g., "John 3:16") + note text
- Storage: Browser localStorage (JSON array)
- UI: List of notes, editable/deletable
- Persistence: Survives page reloads but lost if browser data cleared

**Data Structure:**
```typescript
interface Note {
  id: string;              // UUID
  ref: string;             // "John 3:16"
  text: string;            // User's note text
  createdAt: string;       // ISO timestamp
}
```

---

### Component: `<WordPanel>`

**File:** `components/WordPanel.tsx`

**Purpose:** Right-side slide panel showing word details

**Features:**
1. **Strong's Entry** — Definition, pronunciation, part of speech
2. **Cross-References** — 5 random verses with same Strong's number
3. **AI Insight** (optional) — Claude explanation (cached)

**Props:**
```typescript
type Props = {
  word: Word | null;           // Selected word
  book: string;                // For context in AI
  chapter: number;             // For context in AI
  verse: number;               // For context in AI
  verseId: number;             // To fetch cross-refs
  onClose: () => void;         // Close panel
};
```

**State:**
```typescript
const [data, setData] = useState<{ strongs?, crossRefs[] }>(null);
const [insight, setInsight] = useState<string | null>(null);
const [insightDisabled, setInsightDisabled] = useState(false);  // No API key
const [loading, setLoading] = useState(false);
```

**Key Flows:**
- Fetch Strong's + cross-refs when word changes (useEffect)
- User clicks "Generate theological insight" → POST /api/insight
- If `disabled: true` in response → Show "Get API key →" link
- Cache check: If already cached, show instantly

---

### Component: `<VerseDisplay>`

**File:** `components/VerseDisplay.tsx`

**Purpose:** Render verse in stacked layout with clickable words

**Props:**
```typescript
type Props = {
  origText: string | null;      // Greek/Hebrew verse
  origWords: Word[];            // Word objects with Strong's
  kjvText: string;              // English verse
  lang: "greek" | "hebrew";     // For RTL, fonts
  onWordClick: (word: Word) => void;
  activeWordId: number | null;  // Highlighted word
};
```

**Render Logic:**
```
If origWords exist:
  Loop through words
  Render each as <ClickableWord>
    On click: onWordClick(word)
    Hover: show Strong's number in tooltip
Else:
  Render plain origText (no clickable words)

Divider (optional)

Render KJV text (not clickable, select-all enabled)
```

---

### Component: `<SearchBar>`

**File:** `components/SearchBar.tsx`

**Purpose:** Input + submit for reference or keyword search

**Props:**
```typescript
type Props = {
  action: (formData: FormData) => Promise<void>;  // Server action
  defaultValue?: string;
  placeholder?: string;
};
```

**Logic:**
```
Input type=search (mobile-friendly)
  Placeholder: "John 3:16 · Genesis 1:1 · keyword…"
  onSubmit: Call action(formData)
  
Action (server-side):
  Parse query
  If matches reference pattern (e.g., "John 3:16"):
    redirect(/verse/john/3/16)
  Else:
    redirect(/search?q=keyword)
```

---

## API Routes

### GET `/api/word`

**Purpose:** Fetch Strong's entry + cross-references for a word

**Query Params:**
```
strongs: string   (e.g., "G3056")
verseId: number   (to exclude from cross-refs)
```

**Response:**
```json
{
  "strongs": {
    "number": "G3056",
    "original_word": "λόγος",
    "transliteration": "logos",
    "pronunciation": "LOG-os",
    "definition": "a word, speech...",
    "kjv_usage": "KJV: word (218x), ...",
    "part_of_speech": "noun"
  },
  "crossRefs": [
    {
      "id": 101,
      "book": "Mark",
      "chapter": 4,
      "verse": 14,
      "text": "The sower soweth the word..."
    },
    // ... up to 5 more
  ]
}
```

**Backend Logic:**
```typescript
const strongs = getStrongs(strongsNumber);
const crossRefs = getCrossReferences(strongsNumber, excludeVerseId, limit=5);
return NextResponse.json({ strongs, crossRefs });
```

---

### GET `/api/insight`

**Purpose:** Get Claude theological insight (cached, optional)

**Query Params:**
```
strongs: string      (e.g., "G3056")
word: string         (e.g., "logos")
book: string         (e.g., "John")
chapter: number      (3)
verse: number        (16)
```

**Response (with API key):**
```json
{
  "text": "In John 3:16, 'logos' (Greek for 'word') refers to the divine utterance..."
}
```

**Response (without API key):**
```json
{
  "disabled": true,
  "message": "AI Insight requires an Anthropic API key."
}
```

**Backend Logic:**
```typescript
// Check if no API key
if (!hasApiKey()) {
  return { disabled: true, message: "..." };
}

// Check cache first
const cached = getCachedInsight(strongs, book, chapter, verse);
if (cached) return { text: cached, cached: true };

// Rate limit: max 10 calls per minute per IP
if (isRateLimited(ip)) return 429 error;

// Call Claude
const text = await getWordInsight(word, strongs, book, chapter, verse);

// Cache it
setCachedInsight(strongs, book, chapter, verse, text);

return { text };
```

---

### GET `/api/health`

**Purpose:** Status check (database, data available)

**Response:**
```json
{
  "status": "ok",
  "db": "ready",
  "stats": {
    "verses": 31102,
    "words": 1084467,
    "strongs": 14952
  },
  "version": "1.0.0",
  "uptime": 3600
}
```

Or if no data:
```json
{
  "status": "degraded",
  "db": "empty — run db:import-all",
  "stats": { "verses": 0, "words": 0, "strongs": 0 }
}
```

---

## Import Process

### Overview

```
User runs: npm run db:import-all
    ↓
scripts/setup.ts orchestrates:
  1. npm run db:init         (Create tables)
  2. npm run db:import-kjv   (31K verses)
  3. npm run db:import-strongs (14K entries)
  4. npm run db:import-greek (7K verses, 280K words)
  5. npm run db:import-hebrew (23K verses, 350K words)
    ↓
Total: ~150 seconds
Database: ~deep-word.db (50-100 MB)
```

### Import Script: `import-kjv.ts`

**Data Source:** GitHub (aruljohn/Bible-kjv)

**Steps:**
1. For each of 66 books:
   - Fetch JSON: `{bookName}.json` from GitHub raw
   - Parse: Loop through chapters → verses
   - Insert: `INSERT INTO verses (book, chapter, verse, text, language, version) VALUES (...)`
2. Progress bar shows per-book status

**Example Fetch:**
```
https://raw.githubusercontent.com/aruljohn/Bible-kjv/master/Genesis.json
↓
{
  "book": "Genesis",
  "chapters": [
    {
      "chapter": "1",
      "verses": [
        { "verse": "1", "text": "In the beginning God created..." }
      ]
    }
  ]
}
```

---

### Import Script: `import-strongs.ts`

**Data Source:** GitHub (openscriptures/strongs)

**Steps:**
1. Fetch two files:
   - `greek/strongs-greek-dictionary.js` → G#### entries
   - `hebrew/strongs-hebrew-dictionary.js` → H#### entries
2. Strip JS wrapper (`var dict = {...};` → `{...}`)
3. Parse JSON
4. For each entry:
   ```
   INSERT INTO strongs (number, original_word, transliteration, 
                        pronunciation, definition, kjv_usage, part_of_speech)
   ```

**Example Entry:**
```
{
  "3056": {
    "lemma": "λόγος",
    "xlit": "logos",
    "pron": "LOG-os",
    "strongs_def": "a word, speech, divine utterance...",
    "kjv_def": "word, ...speech, utterance, reason, motive..."
  }
}
```

---

### Import Script: `import-greek.ts`

**Data Source:** GitHub (biblicalhumanities/Nestle1904)

**Steps:**
1. Fetch combined lowfat XML (~50 MB):
   ```
   https://raw.githubusercontent.com/biblicalhumanities/Nestle1904/master/nestle1904lowfat.xml
   ```
2. Parse XML recursively, collect all `<w>` (word) elements
3. Extract from each `<w>`:
   - `ref="Matt.1.1!1"` → book, chapter, verse, position
   - `strong="G976"` → Strong's number
   - `translit="biblos"` → Transliteration
   - Text content → surface form
4. Group by verse, insert all words in transaction

**XML Structure:**
```xml
<book>
  <sentence>
    <wg>
      <w ref="Matt.1.1!1" strong="G976" translit="biblos">βίβλος</w>
      <w ref="Matt.1.1!2" strong="G1078">γενέσεως</w>
      ...
    </wg>
  </sentence>
</book>
```

---

### Import Script: `import-hebrew.ts`

**Data Source:** GitHub (openscriptures/morphhb)

**Steps:**
1. For each OT book:
   - Fetch OSIS XML: `https://raw.githubusercontent.com/openscriptures/morphhb/master/wlc/{Osis}.xml`
   - Parse structure:
     ```xml
     <osis><osisText><div type="book">
       <chapter osisID="Gen.1">
         <verse osisID="Gen.1.1">
           <w lemma="b/7225" morph="HR/Ncfsa">בְּרֵאשִׁ֖ית</w>
         </verse>
       </chapter>
     </div></osisText></osis>
     ```
2. Extract from each `<w>`:
   - `lemma="b/7225"` → Parse number (7225) → Add "H" → "H7225"
   - `morph="HR/Ncfsa"` → Morphological code
   - Text (with cantillation marks stripped) → surface form
3. Group by verse, insert in transaction

**Lemma Parsing:**
```
"b/7225"     → H7225 (prefix b/ is preposition)
"1254 a"     → H1254 (variant a)
"430"        → H430
```

---

## Deployment

### Option 1: Docker (Recommended)

**Prerequisites:**
- Docker + Docker Compose installed
- `ANTHROPIC_API_KEY` in environment (optional)

**Steps:**
```bash
cd /path/to/deep-word
docker compose up --build
```

**What happens:**
1. Build image (Next.js standalone output)
2. Start container
3. On first run: entrypoint.sh detects no DB, runs `npm run db:import-all`
4. Subsequent runs: Use cached DB from volume

**Compose File:**
```yaml
services:
  deep-word:
    build: .
    ports:
      - "3000:3000"
    volumes:
      - bible-data:/app/data
    environment:
      - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY}
      - DB_PATH=/app/data/deep-word.db
    restart: unless-stopped

volumes:
  bible-data:
```

**Dockerfile (Multi-stage):**
```dockerfile
FROM node:20-slim AS deps
RUN apt-get update && apt-get install -y python3 make g++
COPY package.json package-lock.json ./
RUN npm ci

FROM node:20-slim AS builder
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-slim AS runner
ENV NODE_ENV=production
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/scripts ./scripts
COPY --from=builder /app/lib ./lib
COPY --from=deps /app/node_modules ./node_modules
COPY entrypoint.sh ./
RUN chmod +x entrypoint.sh
EXPOSE 3000
ENTRYPOINT ["./entrypoint.sh"]
CMD ["node", "server.js"]
```

---

### Option 2: Self-Hosted (VPS)

**Prerequisites:**
- VPS (DigitalOcean, Linode, etc.)
- Node 20+ installed
- Port 3000 publicly accessible (or behind reverse proxy)

**Steps:**
```bash
# SSH into VPS
ssh user@your-vps.com

# Clone repo
git clone <repo> /home/user/deep-word
cd /home/user/deep-word

# Install deps
npm install

# Import data (one-time, ~5-10 min)
npm run db:import-all

# Run with process manager (e.g., pm2)
npm install -g pm2
pm2 start npm --name deep-word -- start
pm2 save
```

**Reverse Proxy (nginx):**
```nginx
server {
    listen 80;
    server_name yourdomain.com;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

### Database Backups

**SQLite WAL Mode:**
- Main DB file: `deep-word.db`
- Write-ahead log: `deep-word.db-wal`
- Shared memory: `deep-word.db-shm`

**Backup strategy:**
```bash
# Simple: Copy main file (while running, WAL mode is safe)
cp deep-word.db deep-word.db.backup

# Or: Full atomic backup
sqlite3 deep-word.db ".backup deep-word.db.backup"
```

---

## Development Workflow

### Local Development

**Setup:**
```bash
# Install Node 20+
nvm install 20 && nvm use 20

# Clone repo
git clone <repo>
cd deep-word

# Install deps
npm install

# Import data (one-time)
npm run db:import-all

# Start dev server
npm run dev
# → http://localhost:3000
```

**Hot reload:**
- Edit `app/page.tsx` → Browser auto-refreshes
- Edit `lib/db.ts` → Need restart (server-side code)
- Edit `globals.css` → Browser auto-refreshes

**Debugging:**
```bash
# Database inspection
sqlite3 deep-word.db
sqlite> SELECT COUNT(*) FROM verses;  # Should be ~31,102

# API response test
curl http://localhost:3000/api/health | jq .

# View logs
# Check browser console for client-side errors
# Check terminal for server errors
```

---

### Code Organization

**Database Logic (lib/db.ts):**
```typescript
// One function per query type:
getVerse(book, chapter, verse, version)  // Single verse
searchVerses(query, limit)                // Keyword search
getWordsForVerse(verseId)                 // Words in verse
getStrongs(number)                        // Strong's entry
getCrossReferences(strongsNum, ...)       // Same word elsewhere
getCachedInsight(strongs, book, ch, vs)   // AI response cache
```

**Page Structure (app/):**
```
├── page.tsx              # "/" — Entry point
├── search/page.tsx       # "/search?q=..." — Results
├── verse/
│   └── [book]/[chapter]/[verse]/
│       └── page.tsx      # Core study page
└── api/
    ├── word/            # Lookup Strong's
    ├── insight/         # Claude request
    └── health/          # Status
```

**Component Hierarchy:**
```
RootLayout
  ├── ThemeScript (dark mode)
  └── Page (route-specific)
      ├── Header (sticky nav)
      ├── Main (content)
      └── Footer (nav)

WordPanelWrapper (client state)
  ├── VerseDisplay (read-only verse)
  └── WordPanel (slide-in panel)
      └── AI Insight section

SearchBar (reusable input)
```

---

### Type Safety

**TypeScript Interfaces:**
```typescript
// From lib/db.ts
type Verse = { id: number; book: string; chapter: number; verse: number; text: string; ... }
type Word = { id: number; verse_id: number; position: number; surface_form: string; strongs_number?: string; ... }
type StrongsEntry = { number: string; original_word?: string; definition?: string; ... }

// From lib/books.ts
type BookEntry = { name: string; slug: string; testament: "OT" | "NT"; osis: string }
```

**Strict Mode:**
```json
// tsconfig.json
{
  "strict": true,
  "strictNullChecks": true,
  "strictFunctionTypes": true,
  "noImplicitAny": true
}
```

---

## Key Design Decisions

### 1. **SQLite over PostgreSQL**

**Why:**
- No external server to manage
- Single file backup
- Perfect for read-heavy workload (Bible text is static)
- WAL mode handles concurrent reads
- Minimal operational overhead

**Trade-off:** Cannot scale horizontally, but not needed for 31K verses

---

### 2. **App Router (Next.js 15) over Pages Router**

**Why:**
- Modern pattern, better DX
- Server Components by default (efficient)
- Layout nesting
- Parallel routes (future: side-by-side reading)

**Impact:** Most components are Server Components; only client-side state (WordPanel, SearchBar) uses `"use client"`

---

### 3. **Optional AI Insights (Anthropic)**

**Why:**
- Adds depth without being essential
- Gracefully disabled without API key
- Cached aggressively (never call API twice for same word/verse)
- Rate-limited to prevent abuse

**Cost savings:** 31K verses × 66 books × average 5 words per verse = potential 10M+ calls. Caching cuts this to 1-10K actual calls.

---

### 4. **localStorage for Study Notes**

**Why:**
- No server write operations (stateless deployment)
- No auth system needed
- Survives page reload
- Privacy (data never leaves device)

**Trade-off:** Lost if browser data cleared; not synced across devices

---

### 5. **Tailwind CSS v4 + CSS Variables**

**Why:**
- Utility-first → rapid UI development
- CSS variables → dynamic theme switching (dark/light)
- Small bundle size
- Responsive mobile-first

**Config:**
```css
@theme {
  --color-surface: var(--c-bg);
  --color-primary: var(--c-text);
  ...
}
```

---

### 6. **Serverless-Ready Architecture**

**Current:** Single Node.js process with SQLite

**Future:** Could migrate to:
- Vercel Edge Functions (stateless)
- Turso (SQLite serverless) for database
- CloudFlare Workers for API
- S3 for static assets

**Why prepared:** Stateless design, no session storage, no file system writes (except DB file)

---

## Future Enhancements

### Short-term (1-3 months)

1. **Chapter Reading View**
   - Read full chapter in one session
   - Progress indicator
   - Jump to verse within chapter

2. **Search Enhancements**
   - Filter by book/testament
   - Advanced syntax (e.g., "book:John AND word:logos")
   - Search history

3. **Export**
   - Export notes as PDF
   - Print verse + notes

---

### Medium-term (3-6 months)

1. **Multi-Language Support**
   - NASB, ESV, CSB (KJV only now)
   - Parallel reading (Greek ↔ Hebrew ↔ English)

2. **Sync Across Devices**
   - User accounts + database sync
   - Cloud backup of notes

3. **Advanced Analytics**
   - Word frequency dashboard
   - Cross-reference maps

---

### Long-term (6+ months)

1. **Community Features**
   - Share notes/insights
   - Discussion per verse
   - Crowdsourced commentaries

2. **Mobile Apps**
   - React Native / Flutter
   - Offline support

3. **AI Enhancements**
   - Custom prompts
   - Multi-language theological explanations
   - Sermon preparation tools

---

## Troubleshooting

### Database Not Found

**Error:** `Database not initialized`

**Fix:**
```bash
npm run db:import-all
```

### Import Fails (Network)

**Error:** `Could not fetch... 404 Not Found`

**Cause:** GitHub URL changed or repo moved

**Fix:**
1. Check GitHub URLs in scripts match repo structure
2. Retry (may be temporary network issue)
3. Fallback: Can skip Greek/Hebrew, still have KJV

### Slow Queries

**Check:**
```sql
PRAGMA index_list(verses);
PRAGMA index_list(words);
```

**Optimize:**
```sql
ANALYZE;
REINDEX;
```

### Memory Spike on Import

**Cause:** Large XML parsing (Nestle1904 is 50+ MB)

**Fix:**
```bash
# Increase Node memory
node --max-old-space-size=4096 node_modules/.bin/tsx scripts/import-greek.ts
```

---

## Conclusion

**Deep Word** is a production-ready Bible study platform combining:
- **Classical texts** in original languages (Greek, Hebrew)
- **Lexical depth** via Strong's definitions
- **Modern UX** with responsive design, dark mode, dark mode
- **Optional AI** for theological insight
- **Zero friction** deployment via Docker

The architecture prioritizes **read-only performance**, **offline-first design**, and **stateless operations**, making it suitable for both local deployment and future serverless architectures.

---

**Document Version:** 1.0  
**Last Updated:** 2025-05-15  
**Status:** Complete & Production-Ready
