# Goal2Opportunity AI

> **Do not search by opportunity name. Describe what you want to achieve.**

Goal2Opportunity AI is a goal-first opportunity discovery platform for students. A student describes an outcome—such as gaining practical AI experience, finding a free weekend event, or building a portfolio—and the system converts that goal into structured constraints, searches verified opportunities, ranks suitable results, and explains every recommendation.

This repository includes a researched dataset of **44 current opportunities collected on 2 September 2026** from Unstop, Devpost, and All College Event. The original Excel workbook and a machine-readable copy are included in `data/` for transparent review and reproducible import.

## The problem

Most opportunity portals follow this model:

```text
Keyword → Category → Filters → Results
```

That assumes students already know what to search for. A request such as “I need practical AI experience for internships, I am a beginner, I am free on weekends, and I can spend ₹500” contains intent, eligibility, time, budget, and format constraints that ordinary keyword search misses.

Goal2Opportunity reverses the flow:

```text
Goal → Need → Constraints → Verified opportunities → Explainable matches
```

## What the system does

1. Accepts a natural-language student goal.
2. Parses intent, skills, opportunity types, budget, location, mode, dates, eligibility, and exclusions.
3. Applies hard constraints before ranking so invalid results cannot receive attractive scores.
4. Scores eligible opportunities using deterministic, inspectable factors.
5. Explains why each result matches.
6. Surfaces a hidden opportunity when a less obvious event can advance the student's underlying goal.
7. If there is no exact match, proposes the smallest useful constraint change instead of returning an empty screen.

## Key features

- Goal-first natural-language search
- Strict budget, location, mode, date, eligibility, and exclusion handling
- Explainable deterministic ranking
- Hidden-opportunity discovery
- Smart zero-result recovery
- Student profiles and saved opportunities
- Verified-only discovery for newly ingested records
- Deterministic fallback parsing when no OpenAI key is configured
- Optional semantic retrieval with OpenAI embeddings and PostgreSQL `pgvector`
- Privacy-conscious discovery analytics

## Submission dataset

| File | Purpose |
|---|---|
| `data/event_opportunities_2026-09-02.xlsx` | Original human-readable research workbook |
| `data/event_opportunities_2026-09-02.json` | Exact extracted rows used by the repeatable importer |

### Coverage

| Platform | Records | Collection note |
|---|---:|---|
| Unstop | 9 | Accessible current and indexed listings |
| Devpost | 12 | Current and upcoming hackathons from category/search pages |
| All College Event | 23 | Accessible current, search, and featured pages |
| **Total** | **44** | Snapshot collected on 2 September 2026 |

Each row includes the event name, platform, type, organizer, dates, deadline, mode, location, eligibility, prize or benefit, fee, source URL, and status. This is a researched snapshot, not a guaranteed exhaustive crawl of dynamically paginated platforms.

The importer normalizes workbook values into the application schema and upserts records with stable `xlsx-...` slugs. Re-running it updates existing dataset records instead of duplicating them.

## Explainable ranking

Hard constraints are checked first. A result that exceeds a maximum budget or violates an explicit exclusion is removed rather than merely ranked lower.

| Ranking factor | Weight |
|---|---:|
| Goal relevance | 35% |
| Intent fit | 20% |
| Skill fit | 10% |
| Budget fit | 10% |
| Eligibility fit | 10% |
| Availability fit | 5% |
| Preference fit | 5% |
| Freshness | 5% |

The final percentage is computed from recorded attributes; it is not an unsupported AI-generated score.

## Architecture

```text
Student goal
    │
    ▼
Query parser + Zod validation
    │  OpenAI parser when configured
    │  deterministic parser otherwise
    ▼
Structured SearchQueryModel
    │
    ├── hard-constraint filtering
    ├── lexical / semantic retrieval
    ├── deterministic weighted ranking
    ├── hidden-opportunity detection
    └── smallest-useful-change resolver
    │
    ▼
Grounded results + match explanations
```

### Important modules

| Module | Responsibility |
|---|---|
| `lib/ai/queryParser.ts` | Converts natural language into a validated search model |
| `lib/db/prisma.ts` | Reads verified opportunities and maps database records |
| `lib/search/hybridSearch.ts` | Coordinates filtering and ranking |
| `lib/search/ranker.ts` | Computes the weighted match score |
| `lib/search/semanticSearch.ts` | Optional embedding retrieval with safe fallback |
| `lib/search/hiddenDiscovery.ts` | Finds non-obvious goal-aligned opportunities |
| `lib/search/constraintResolver.ts` | Suggests the smallest useful relaxation |
| `lib/ai/explanation.ts` | Produces grounded match explanations |
| `scripts/import-event-opportunities.mjs` | Imports the supplied 44-row dataset |

## Technology stack

- Next.js 14 App Router, React 18, and TypeScript
- Tailwind CSS and Lucide React
- PostgreSQL with Prisma ORM
- PostgreSQL `pgvector` for optional semantic search
- Zod runtime validation
- OpenAI API as an optional enhancement

## Run locally

### Prerequisites

- Node.js 20+
- npm
- Docker Desktop or an accessible PostgreSQL instance

### 1. Clone and install

```bash
git clone <YOUR_GITHUB_REPOSITORY_URL>
cd Goal2Opportunity-AI
npm install
```

Replace the placeholder with the repository URL created by GitHub.

### 2. Configure the environment

```bash
cp .env.example .env
```

Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

Relevant variables:

```env
DATABASE_URL="postgresql://g2o:g2o_local_dev_password@localhost:5433/g2o?schema=public"
OPENAI_API_KEY=""
OPENAI_MODEL="gpt-4o-mini"
OPENAI_EMBEDDING_MODEL="text-embedding-3-small"
NEXT_PUBLIC_APP_URL="http://localhost:3000"
```

`OPENAI_API_KEY` is optional. Without it, deterministic parsing and lexical/structured search remain available.

### 3. Start PostgreSQL

```bash
docker compose -f docker-compose.postgres.yml up -d
```

### 4. Apply the schema

```bash
npx prisma migrate deploy
npx prisma generate
```

For a new development database, `npx prisma db push` may be used instead of migration deployment.

### 5. Load demo and submission data

```bash
npm run db:seed
npm run db:import-events
```

`db:seed` creates the demo user and built-in test opportunities. `db:import-events` upserts the 44 verified opportunities supplied with this submission.

### 6. Launch

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). If port 3000 is occupied, Next.js prints the next available localhost URL.

## Demo account

```text
Email:    alex.rivers@college.edu
Password: password123
```

These credentials are only for local demonstration.

## Suggested judging flow

### Goal understanding and hidden discovery

```text
I want practical AI experience for internships. I am a beginner,
available this weekend, and my budget is under ₹500.
```

Inspect the parsed intent, constraints, ranked explanations, and any hidden opportunity supporting the career goal.

### Negative constraints

```text
Find a free beginner cybersecurity opportunity. Do not show workshops.
```

Workshop records should be excluded before scoring.

### Zero-result recovery

```text
I want a free beginner AI event this weekend, offline in Coimbatore.
```

If no verified record matches exactly, the interface explains the conflict and proposes a minimal change, such as allowing online events.

### Supplied dataset

Search for `hackathon`, `AI`, `coding`, `Coimbatore`, `online`, or a budget requirement. Open a result to verify its organizer, dates, eligibility, and original source link.

## API overview

| Endpoint | Purpose |
|---|---|
| `POST /api/discovery/search` | Parse a goal and return ranked, explained matches |
| `GET /api/opportunities` | List verified opportunities |
| `GET /api/opportunities/:id` | Read one opportunity |
| `POST /api/ingestion/opportunities` | Ingest records under the verification contract |
| `GET/POST /api/profile` | Read or update the student profile |

## Verification

```bash
npm run test:engine
npm run build
```

The engine test covers representative query and constraint behavior. The production build validates route compilation and TypeScript integration.

## Optional semantic search

With a valid OpenAI key, `pgvector`, and applied migrations:

```bash
npm run db:embed
```

If semantic retrieval is unavailable, discovery falls back to deterministic lexical and structured matching instead of failing.

## Data trust and responsible use

- Every imported row retains its original source URL.
- Newly ingested records are non-searchable by default until reviewed.
- The bundled workbook is marked verified specifically for this hackathon dataset import.
- Event details can change after collection; users should confirm deadlines, fees, and eligibility at the source before registering.
- Explanations must remain grounded in recorded attributes and computed factors.

## Repository structure

```text
app/          Next.js pages and API routes
components/   Reusable interface components
data/         Submission workbook and extracted dataset
docs/         Supporting documentation
lib/ai/       Query parsing and explanations
lib/db/       Prisma access and seed data
lib/search/   Filtering, ranking, discovery, and recovery
prisma/       Schema and migrations
scripts/      Seeding, tests, embeddings, and dataset import
types/        Shared TypeScript models
workflows/    Ingestion and automation resources
```

## Future improvements

- Scheduled source refresh with reviewer approval queues
- Location geocoding and travel-distance preferences
- Calendar conflict detection and deadline reminders
- Feedback-aware personalization without weakening hard constraints
- Multilingual goal parsing
- Production authentication, rate limiting, monitoring, and secret management

