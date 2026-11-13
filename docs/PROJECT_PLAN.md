# Movie Recommendation Web App — Project Plan

Nov 8, 2026 · @Abrar

## 1. How to use this document

This is the single source of truth for building the app: every architecture, data, algorithm and infrastructure decision is recorded here, with the reasoning behind it. It was written for Claude Code, from a long planning conversation with the project owner, so that Claude Code starts with the same context the planning had.

**Instructions for Claude Code**

1. Read the whole document before writing any code. Then summarize the project back to the owner and list anything that seems unclear, contradictory or missing.
2. Build in the order of Section 20 (Build order). Do not start a milestone until the previous one is done and verified.
3. Break each milestone into small subtasks. Complete one subtask at a time, verify it (tests pass, behavior checked), and get the owner's confirmation before moving on.
4. Follow decisions marked **Confirmed** exactly. Do not substitute libraries, services or approaches.
5. Decisions marked **Proposed default** during planning were all approved by the owner on 2026-11-13 and are now Confirmed. None remain.
6. Items marked **Deferred** must not be decided by you. Build them so the decision can be made later (usually a setting).
7. If something is not covered here, ask. Do not invent requirements.
8. The owner must be able to explain every part of this code in technical interviews. Prefer clear, conventional, well-named code over clever code. Explain non-obvious choices in short comments or in the PR/commit description.
9. Never commit secrets. Never commit the MovieLens dataset (its license forbids redistribution).

**Decision status legend**

| Status | Meaning |
| --- | --- |
| Confirmed | Explicitly decided by the owner. Follow exactly. |
| Proposed default | No longer used: every proposed default was approved on 2026-11-13. |
| Deferred | Intentionally left open. Build it configurable; do not decide. |

Everything in Sections 2 to 20 is **Confirmed** unless marked **Deferred**. Section 21 lists the Deferred items, the decision log and a glossary.

## 2. Project overview

A full-stack movie recommendation web app where users sign in, browse and search movies, like and unlike them instantly, and get personalized recommendations from a content-based recommender designed to become a hybrid with collaborative filtering later.

**Purpose.** A portfolio project the owner will be asked about in technical interviews. Clean, well-structured, explainable engineering is preferred over cleverness.

**Fresh project.** This is built from scratch. It is not based on, and must not try to match, any earlier movie recommendation app the owner built.

### Functional requirements

- Real accounts via Supabase Auth: Google sign-in (primary), email + password (secondary) with email confirmation, forgot/reset password, in-app account deletion, and a demo account for interviewers.
- Each user's likes and recommendations belong to their account.
- Movie data comes from TMDB.
- Users can like and unlike movies, and it must feel instant: the UI updates immediately with no visible network wait.
- Personalized recommendations ("For You") from a content-based recommender using movie attributes, with a reason per recommendation ("Because you liked Heat").
- Browse and search all of TMDB; see movie details and similar movies ("More like this").
- Recommendation quality is measured offline with MovieLens and tracked over time.

### Non-functional requirements

- Runs within free tiers: Google Cloud Run (backend), Vercel (frontend), Supabase (database, auth, storage), GitHub Actions (jobs and CI).
- Recommendations computed on request in tens of milliseconds.
- Collaborative filtering can be added later as a hybrid without a rewrite.
- Accessible, responsive UI with a distinctive movie-app visual style.
- Secrets never in the repo (the repo is public).

### Pages

| Page | Purpose |
| --- | --- |
| Home | Search bar (live TMDB search) and rows such as Trending and Top Rated; a For You row once the user has likes |
| Movie detail | Overview, cast, like button, "More like this" |
| Liked | The user's liked movies, newest first, unlike in place |
| For You | Personalized recommendations with a reason each; cold-start prompt under 3 usable likes |
| Auth pages | Sign in, sign up, confirm email notice, forgot/reset password |
| Settings | Delete account |
| About | TMDB attribution and project info |
| Dev-only comparison view | Diversity side-by-side with slider (development builds only) |

The page set is Confirmed.

## 3. Tech stack

Python backend, TypeScript/React frontend, Supabase for Postgres + Auth + Storage, TMDB for movie data. PyTorch is not used anywhere in the project.

| Layer | Choice | Status | Why |
| --- | --- | --- | --- |
| Backend API | Python, FastAPI, Pydantic, pydantic-settings | Confirmed | Typed models, auto OpenAPI docs, async I/O for TMDB calls, dependency injection for auth |
| Database access | SQLAlchemy 2.0 (async) + asyncpg | Confirmed | Explicit, typed queries; works with pgvector; easy to test |
| Database | Supabase Postgres + pgvector extension (HNSW index, cosine) | Confirmed | Nearest-movie search inside the database |
| Schema changes | Supabase CLI migrations (plain SQL files) | Confirmed | One source of truth for tables, RLS policies and extensions |
| Auth | Supabase Auth | Confirmed | Google, email/password, confirmation, reset |
| File storage | Supabase Storage, behind a swappable interface | Confirmed | Weekly fitted-pieces bundle (Section 8) |
| Movie data | TMDB API | Confirmed | Weekly ingestion + live search |
| Recommender math | numpy, scikit-learn (TF-IDF, TruncatedSVD) | Confirmed | Standard and explainable |
| Plot-summary embeddings | fastembed (ONNX Runtime) with `sentence-transformers/all-MiniLM-L6-v2`, 384 dims | Confirmed | Same model in weekly job and API; small enough for a 1 GB server; no PyTorch |
| Frontend | React + TypeScript + Vite + React Router | Confirmed | SPA; no SSR needed for a logged-in app |
| Frontend data | TanStack Query | Confirmed | Optimistic likes and hash-keyed recommendation caching |
| Frontend auth | supabase-js (auth only) | Confirmed | Login flows only; data goes through the API |
| API types | openapi-typescript generated from FastAPI's schema | Confirmed | Frontend and backend types cannot drift |
| Styling | Tailwind CSS + shadcn/ui | Confirmed | Owned, restylable components |
| Icons | lucide-react | Confirmed | Pairs with shadcn/ui |
| Charts (dev view, if needed) | Recharts | Confirmed | Only if a chart is needed |
| Design guidance | Claude skills: frontend-design + UI/UX Pro Max (split roles, Section 12) | Confirmed |  |
| Evaluation data | MovieLens 32M (offline only) | Confirmed | Section 14 |
| Python tooling | uv, ruff, mypy | Confirmed | Fast, standard |
| Backend tests | pytest, httpx, respx | Confirmed |  |
| Frontend tests | Vitest, React Testing Library, MSW | Confirmed |  |
| End-to-end tests | Playwright | Confirmed |  |
| CI and scheduled jobs | GitHub Actions | Confirmed | Public repo: free minutes |
| Backend hosting | Google Cloud Run, free tier | Confirmed | Section 18 |
| Frontend hosting | Vercel | Confirmed | Section 18 |
| Email delivery | A transactional email provider via SMTP, single verified sender (no domain) | Approach Confirmed; provider Deferred | Section 13 |
| Monitoring | No telemetry platform | Confirmed | Structured logs, /health, job and eval reports |

### Languages

| Language | Used for |
| --- | --- |
| Python | API, recommender, weekly jobs, evaluation |
| TypeScript | Frontend (React/JSX), generated API types |
| SQL | Migrations, RLS policies, complex queries |
| YAML, Dockerfile, CSS via Tailwind | Configuration and styling |

## 4. System architecture

The browser uses Supabase only to sign in; all data goes through the FastAPI backend, which owns every read and write. Heavy work (ingestion, vector building, evaluation) runs in weekly GitHub Actions jobs, never in the API's request path, except one small case: building a single vector for an out-of-catalog like.

```text
Browser (React SPA on Vercel)
  | supabase-js -------------------------> Supabase Auth (Google, email/password; issues JWT)
  | fetch + Bearer JWT
  v
FastAPI on Google Cloud Run (scale to zero, 1 GB)
  |-- verifies JWT (Supabase JWKS / signing keys)
  |-- TMDB API (search, trending, details; key server-side; short cache)
  |-- Supabase Postgres + pgvector (catalog, vectors, likes, reports)
  |-- Supabase Storage (fitted-pieces bundle, loaded lazily)
  '-- fastembed / ONNX model (baked into the image; used for out-of-catalog likes)

GitHub Actions (scheduled)
  |-- weekly: ingestion -> feature build -> evaluation -> cleanup -> reports
  '-- every 2-3 days: keep-alive query (prevents Supabase free-tier pause)
```

### Key architecture decisions

- **Backend-mediated data access** (Confirmed): one place for business logic, the TMDB key never reaches the browser, the recommender must run server-side anyway, and the API is a clean, testable surface. RLS is still enabled on every table as defense in depth, so Supabase's auto-generated public API cannot leak data.
- **No ML framework in the request path except ONNX** (Confirmed): the API loads the small ONNX model and, lazily, the fitted-pieces bundle. The weekly job does all catalog-wide fitting.
- **Recommendations computed on request** (Confirmed): about 10k movies makes this tens of milliseconds and always current. Precomputing per user only pays off at much larger scale.
- **Recommendations cached in the browser** keyed by a hash of the liked set (Confirmed, Section 12).

### Request flows

**Like (movie in catalog):** heart fills instantly in the browser → `PUT /api/v1/me/likes/{id}` → API verifies token → insert like (idempotent) → 204. The browser's liked set changes, so the For You cache key changes automatically.

**Like (movie not in catalog):** see Section 10. The API fetches the movie from TMDB, stores it, builds its vector, saves the like, then responds. All work finishes before the response because Cloud Run throttles CPU after responding.

**Open For You:** browser checks its cache for the current liked-set hash → on a miss, `GET /api/v1/me/recommendations` → API loads liked vectors, averages them, retrieves 200 nearest catalog movies via pgvector, scores, picks 10, attaches reasons, logs impressions → response cached under that hash.

**Search:** browser → API → TMDB search (cached briefly) → results with each movie's liked state from the browser's liked set.

## 5. Data sources

TMDB supplies all movie data; Supabase Auth supplies accounts; users supply likes; MovieLens is used only for offline evaluation and never appears in the app.

### TMDB (all movie data)

- **Two modes.** (1) The weekly ingestion job stores a curated catalog (about 10k movies) in Postgres; recommendations come only from this catalog. (2) Live API calls serve search and Trending, so users can find any TMDB movie.
- **One call per movie:** `GET /movie/{id}?append_to_response=keywords,credits` returns details, keywords, cast and crew together. The director is the crew entry with job "Director".
- **Thin data is normal** for new or obscure movies (few keywords, short or missing overview, incomplete credits). Every feature block must handle empty input.
- **Terms:** free for non-commercial use with attribution. This portfolio app qualifies; adding ads or payments would make it commercial and require a separate agreement.
- **Attribution (required):** show the TMDB logo and the notice "This product uses the TMDB API but is not endorsed or certified by TMDB." The logo must be less prominent than the app's own branding. Place it in the footer and on the About page.
- **Rate limit:** about 40 requests per second per IP. The weekly job throttles to about 20 per second with retries and exponential backoff on HTTP 429 and 5xx.
- **Images:** posters and backdrops load directly from TMDB's image CDN using stored paths. Never copy or rehost images.
- **Key application:** describe the project honestly as "a non-commercial portfolio movie recommendation app". TMDB's terms treat AI-model training on its data as needing approval; this project is a non-commercial content recommender, and the application should give TMDB the full picture.
- **Titles and taglines are not used in vectors** (Confirmed): titles create false matches (Heat vs The Heat) and taglines are mostly marketing.

### Supabase Auth (accounts)

Users sign in through supabase-js in the browser. The backend never sees passwords; it reads the user id (`sub`) from the verified JWT.

### Users (likes)

The only user-generated data in the app: likes, plus shown-recommendation logs. These live only in the database; there is no backup (Confirmed, accepted risk).

### MovieLens 32M (evaluation only)

- Real people's movie ratings, used to measure recommendation quality offline (Section 14).
- About 32 million ratings, about 200k users, about 87k movies, ratings up to 2023 (figures approximate).
- Its `links.csv` maps MovieLens movie ids to TMDB ids.
- **License:** research use; must not be redistributed. Download it during evaluation (cached in GitHub Actions). Never commit it to the public repo.
- **Not used to build the catalog** (Confirmed). The catalog is defined purely by TMDB.

## 6. Database schema

Movie attributes are normalized into lookup and link tables; vectors are versioned; user data is tied to `auth.users` and removed with the account. All columns below are Confirmed.

### Catalog tables

| Table | Columns | Notes |
| --- | --- | --- |
| `movies` | `id` bigint PK (= TMDB id), `title`, `overview`, `release_date`, `original_language`, `runtime`, `popularity`, `vote_average`, `vote_count`, `poster_path`, `backdrop_path`, `source` (`top_list` or `user_like`), `is_active` bool, `tmdb_updated_at`, `ingested_at`, `updated_at` | `source` records how the movie first entered and never changes afterward. `is_active=false` when TMDB returns 404; never delete (likes point here). |
| `genres` | `id` (TMDB genre id) PK, `name` | One row per genre |
| `movie_genres` | `movie_id`, `genre_id`, PK both | Link table |
| `keywords` | `id` (TMDB keyword id) PK, `name` | One row per keyword |
| `movie_keywords` | `movie_id`, `keyword_id`, PK both | Link table |
| `people` | `id` (TMDB person id) PK, `name` | Cast and directors |
| `movie_credits` | `movie_id`, `person_id`, `role` (`cast` or `director`), `cast_order` | Store top 5 cast + directors |

### Vector tables

| Table | Columns | Notes |
| --- | --- | --- |
| `feature_versions` | `version` int PK, `status` (`building`, `active`, `retired`), `dims`, `block_weights` jsonb, `bundle_path`, `embedding_model`, `created_at`, `activated_at` | Exactly one `active` row. API and jobs read the active version here. |
| `movie_features` | `movie_id`, `version`, `embedding` vector(D), `built_by` (`weekly` or `instant`), `computed_at`; PK (`movie_id`, `version`) | Keep the active and the previous version (rollback). D is fixed by settings. |
| `overview_embeddings` | `movie_id` PK, `model_name`, `overview_hash`, `embedding` vector(384), `computed_at` | Cache: reuse when the overview text and model are unchanged. |

Indexing: an HNSW cosine index on `movie_features.embedding`, scoped to the active version (a partial index created by the job before activation). At about 10k rows an exact scan may also be fast enough; measure and pick the simpler option that meets latency.

### User and reporting tables

| Table | Columns | Notes |
| --- | --- | --- |
| `likes` | `user_id` uuid FK `auth.users` ON DELETE CASCADE, `movie_id` FK `movies`, `created_at`; PK (`user_id`, `movie_id`) | Composite PK makes likes idempotent. `created_at` is used only to sort the Liked page, never in recommendations. |
| `rec_impressions` | `id`, `user_id` (cascade), `movie_id`, `rank`, `model_version`, `shown_at` | What was shown; for evaluation and future collaborative filtering |
| `job_runs` | `id`, `job` (`ingest`, `features`, `eval`, `keepalive`, `cleanup`), `status`, `started_at`, `finished_at`, `stats` jsonb | Counts added, updated, failed; last successful ingest drives the 14-day fallback |
| `eval_runs` | `id`, `run_at`, `git_sha`, `feature_version`, `catalog_size`, `sample` (`ci-2k`, `weekly-20k`, `full`), `settings` jsonb, `metrics` jsonb, `overlap_rate`, `most_missed` jsonb | One row per evaluation run |
| `catalog_reports` | `id`, `week_start`, `out_of_catalog_like_rate`, `top_out_of_catalog` jsonb | Weekly real-user out-of-catalog report |

### Security

- RLS enabled on every table, with no policies for the `anon` or `authenticated` roles: the browser never queries tables directly.
- The backend and jobs connect with a privileged role from secrets (Section 18).
- The Supabase service key is backend-only and never reaches the frontend.

### Migrations

- Numbered SQL files in `supabase/migrations/`, applied in order; each runs once.
- Applied to the hosted database by CI on merge to `main`.
- **Backward-compatible migrations:** the previous code version must still work with the new schema, because code can be rolled back but schema changes cannot. Example: add a new column in one deploy, remove the old one in a later deploy.

## 7. Catalog ingestion job

Once a week (Confirmed: weekly, not nightly), the job rebuilds the current top-movie list from TMDB, adds movies not yet stored, and refreshes stored movies that TMDB reports as changed. It never removes movies. A typical weekly run takes a minute or two; the first-ever run fetches about 10k movies in about 10 minutes (estimates).

### Steps

1. **Check the last successful run** in `job_runs`. If it was more than about 14 days ago (TMDB's changes endpoint only covers about two weeks), do a full refresh of all stored movies instead of step 3.
2. **Build the current top list** from `/discover/movie` sorted three ways: by vote count, by popularity (catches new releases), and by rating with a minimum vote count. Union, de-duplicate, drop adult titles, keep about 10,000 ids. This list is held in memory only and discarded after the run.
3. **Find changed movies** via TMDB's changes endpoint for the past week, intersected with stored ids.
4. **Fetch** = (top-list ids not stored) ∪ (stored ids that changed). Typically dozens to a few hundred; this churn estimate is unmeasured, so log it every run.
5. **Fetch and store each movie** with one call (`append_to_response=keywords,credits`). In one transaction per movie: upsert the `movies` row and replace its genre, keyword and credit links. New movies from the top list get `source = 'top_list'`.
6. **Record the run** in `job_runs` with counts: added, updated, failed, marked inactive.

### Rules and edge cases

| Situation | Behavior |
| --- | --- |
| Movie falls out of the top list | Stays stored and recommendable |
| TMDB returns 404 for a stored movie | `is_active = false`; excluded from recommendations; likes stay valid |
| Adult title | Skipped |
| One movie fails to fetch | Log, skip, continue; retried next week |
| Job crashes midway | Re-run; every write is an upsert, so redoing work is harmless |
| Rate limiting | Throttle to about 20 requests/second; retry with backoff on 429/5xx |
| Missed weekly run | Next run catches up via the changes endpoint; gaps over about 14 days trigger a full refresh |

### Shared code

The "fetch one movie from TMDB and store it" function is shared by this job and the API's out-of-catalog like path (Section 10). There is one ingestion implementation, tested once.

### Local seed

A fixture of about 500 movies (JSON, in the repo, generated once from TMDB) is loaded by `make seed` so local development needs no TMDB key and no long ingestion run.

## 8. Movie vectors and the feature build job

Each movie becomes one vector of about 550 numbers, its content fingerprint, built from five weighted blocks so that cosine similarity between two movies equals the weighted sum of their per-block similarities. There is **one recommender version** (Confirmed: no v1/v2 split), and it includes the plot-summary block from the start.

### The five blocks

| Block | Input | Encoding | Size | Starting weight |
| --- | --- | --- | --- | --- |
| Genres | TMDB genres | Multi-hot (1 per genre the movie has) | about 19 | 0.30 |
| Keywords | TMDB keywords | TF-IDF (binary term frequency; the IDF rarity weight is what matters), then TruncatedSVD | 64 | 0.25 |
| People | Top 5 cast + director(s), by TMDB person id | TF-IDF, then TruncatedSVD | 64 | 0.20 |
| Overview | Overview text only (no title, no tagline) | fastembed ONNX `all-MiniLM-L6-v2`, normalized | 384 | 0.20 |
| Era and language | Release decade + original language | One-hot over fixed lists (languages: top N + "other") | small | 0.05 |

The encodings, model, starting weights, 64-dimension SVD size and top 5 cast are all Confirmed; the weights are starting values to be tuned with evaluation (Section 14). Popularity and rating are deliberately **not** in the vector; quality is applied at scoring time (Section 9).

### Combining the blocks

Each block is L2-normalized, multiplied by the square root of its weight, and the blocks are concatenated. With weights summing to 1:

```latex
\cos(a, b) = \sum_i w_i \cdot \cos(a_i, b_i)
```

So each weight reads directly as "genres count 30%". **Empty block** (no keywords, no overview, and so on): the block is all zeros and the remaining blocks' weights are scaled up so they sum to 1 (for example genres 0.30 / 0.80 = 0.375 when the overview is missing).

### Feature build job (weekly, right after ingestion)

1. Load every active movie with its genres, keywords, people, overview, release date and language.
2. **Fit** the catalog-wide pieces: genre list, language and decade lists, keyword TF-IDF + SVD, people TF-IDF + SVD.
3. **Overview embeddings:** reuse the cached embedding from `overview_embeddings` when the overview hash and model are unchanged; embed only new or edited overviews.
4. **Build** every movie's vector with the shared `build_vector` function.
5. **Write** all vectors under a new version with status `building`; create the index; then mark it `active` and the previous version `retired`. The app never mixes vectors from two versions. Keep the previous version's vectors for rollback; delete older ones.
6. **Upload the fitted-pieces bundle** for the new version (below) and record its path in `feature_versions`.
7. Then the weekly evaluation runs on the new version (Section 14).

Why refit everything weekly: IDF weights and SVD components are learned from the whole catalog, so any catalog change shifts them slightly for every movie.

### Fitted-pieces bundle (Confirmed)

The reference data both the weekly job and the API need so that a vector built anywhere lines up with the catalog:

| File | Contents |
| --- | --- |
| `meta.json` | version, dims, block weights, genre list, language list, decade list, embedding model name, keyword vocabulary with IDF weights, people vocabulary with IDF weights |
| `tables.npz` | keyword SVD components (one row per keyword × 64), people SVD components (one row per person × 64) |

- About 10 to 15 MB per version.
- **Plain data (JSON + numpy), not pickle**: pickle can run code when loaded and breaks across scikit-learn versions.
- Stored in **Supabase Storage** at `features/v{N}/`, accessed only through a small storage interface (`upload(version, files)`, `download(version)`), so it can be swapped for Google Cloud Storage later in an hour or two with no other changes. Tests use an in-memory fake.
- Keep the last 3 to 4 versions; the weekly job deletes older bundles (stays well under Supabase's 1 GB storage limit).
- **The API loads the bundle lazily**: only on the first out-of-catalog like an instance handles, never at startup. Downloading at every cold start would exceed Supabase's 5 GB/month egress limit. The API keeps the loaded version in memory and reloads only when `feature_versions` shows a newer active version.

### One recipe: the shared `build_vector` function (Confirmed)

- `build_vector(movie_data, pieces) -> vector` lives in the shared recommender package and is the **only** code that builds vectors, used by both the weekly job and the API.
- The API refuses to build with pieces whose version does not match the active version.
- **Parity test (CI):** build sample movies' vectors through the job path and the API path and assert they are identical.
- Keywords or people the fitted vocabulary has never seen are ignored until the next weekly refit; the movie relies on its other blocks meanwhile.

### Settings

Block weights, SVD size, cast count and an **ablation switch** (set the overview weight to 0 for one evaluation run) are configuration, not code changes. Measure the API's real memory use with the ONNX model loaded early (Milestone 6) to confirm the 1 GB Cloud Run setting.

## 9. Recommender

A user's taste profile is the plain average of their liked movies' vectors; the 200 nearest catalog movies are scored on similarity plus a small quality term, the top 10 are returned with a reason each. It is computed on request (tens of milliseconds) and has no recency weighting of any kind.

### Pipeline for `GET /me/recommendations`

1. **Usable likes** = the user's liked movies that are active and have a vector in the active version.
2. **Cold start:** fewer than 3 usable likes → return popular and top-rated catalog movies with a reason like "Popular right now", plus a flag the UI uses to show "Like a few more movies to personalize this".
3. **Profile** = the plain, equal-weight average of the usable liked vectors, then L2-normalized. **Every like counts equally. Like timestamps are never used** (Confirmed): unliking and re-liking a movie produces exactly the same list.
4. **Retrieve** the 200 nearest active catalog movies by cosine similarity via pgvector, excluding all liked movies (including liked movies that have no vector yet).
5. **Score** each candidate:

```latex
\text{score} = 0.85 \cdot \text{similarity} + 0.15 \cdot \text{quality}
```

Quality is a vote-adjusted (Bayesian average) rating scaled to 0 to 1, so a 9.5 from 12 votes cannot outrank an 8.4 from 30,000:

```latex
\text{quality} = \frac{v \cdot R + m \cdot C}{v + m} \Big/ 10
```

R = the movie's `vote_average`, v = its `vote_count`, C = the catalog's mean rating, m = the votes needed before a movie's own rating dominates (start at 500). "How good" means TMDB users' ratings; it is a light nudge, not an objective measure.

6. **Pick 10.** By default, take the top 10 by score. The diversity step below exists but is **off by default** (Deferred).
7. **Reasons:** for each pick, the liked movie with the highest cosine similarity to it → "Because you liked Heat".
8. **Log impressions** (`rec_impressions`: movie, rank, model version) and respond with the movies, scores, reasons and `model_version`.

### Diversity (MMR): built, off by default, decision Deferred

Maximal marginal relevance picks one movie at a time, rewarding relevance and penalizing similarity to movies already picked, to avoid lists of near-duplicates (for example five Marvel films):

```latex
\text{value} = (1 - \lambda) \cdot \text{score} - \lambda \cdot \max_{p \in \text{picked}} \cos(\text{candidate}, p)
```

- λ = 0 means off (pure top 10 by score). λ = 0.3 was the original suggestion.
- λ is a setting. The owner will choose after using the **dev-only side-by-side view** (Section 12) and seeing evaluation numbers for several λ values (Section 14).
- Similarity to the user's taste is still rewarded; only redundancy within the list is penalized.

### "More like this" (`GET /movies/{id}/similar`)

Item-to-item: the nearest active catalog movies to that movie's vector, excluding itself, with the same quality term. No login required.

### Designed for a future hybrid (collaborative filtering)

The pipeline is a set of small interfaces so collaborative filtering is an addition, not a rewrite:

| Interface | Role | Today |
| --- | --- | --- |
| `CandidateGenerator` | user → (movie\_id, score) candidates | `ContentGenerator` (pgvector) |
| `Blender` | combines candidates from several generators with weights | Single source |
| `Reranker` | final ordering (quality term, optional MMR) | Score + optional MMR |
| `Explainer` | reason per item | Most similar liked movie |

Later: an `ItemItemCFGenerator` (co-like similarity from `likes`) or an implicit-ALS generator is added to the blender, with a weight that can grow with the user's like count. `rec_impressions` provides exposure data for it.

### Known limitations (to discuss honestly)

- One averaged profile blurs users with several distinct tastes. Possible fix later: 2 or 3 centroids per user.
- Content features cannot capture taste signals that only show up in behavior; that is what collaborative filtering adds.
- No deep learning model is trained: the overview model is pre-trained and used for inference only. Training needs interaction data the app does not have; simple, well-tuned methods are competitive.

## 10. Likes and the out-of-catalog like flow

Likes are idempotent and feel instant; a like on a movie outside the catalog is fully processed, including a complete vector, before the API responds, so it counts toward recommendations within about a second.

### Normal like and unlike

- `PUT /api/v1/me/likes/{movie_id}` → insert or no-op → 204. `DELETE` → delete or no-op → 204.
- Idempotent by design: retries and double-clicks cannot corrupt state. No "toggle" endpoint.
- The browser updates immediately and never waits on these calls (Section 12).

### Out-of-catalog like (Confirmed)

Search and Trending reach all of TMDB, but the catalog is about 10k movies, so a user can like a movie that is not stored. Steps, all inside the request:

1. Upsert check on `movies` by primary key (effectively free).
2. Not stored → fetch from TMDB with the shared fetch-and-store function (one call with keywords and credits). Store it with `source = 'user_like'`.
3. **Build its complete vector** with `build_vector`, including the overview block via the ONNX model in the API. Load the fitted-pieces bundle first if this instance has not loaded it yet (lazy load, about 1 second, once per instance).
4. Save the vector to `movie_features` under the active version (`built_by = 'instant'`).
5. Save the like.
6. Respond 204.

**Why before responding:** with Cloud Run's request-based billing, CPU is throttled after the response is sent, so background work after responding is unreliable. Building the vector adds only tens of milliseconds; the TMDB fetch takes longer. The user never notices because the heart already filled.

From then on the movie counts in the user's profile immediately and can be recommended to other users. The next weekly run rebuilds it with the newly fitted pieces like every other movie.

### Failure cases

| Failure | Behavior |
| --- | --- |
| TMDB unavailable during the fetch | Like fails; the browser rolls the heart back and shows a quiet message; user can retry |
| Vector build fails | Like is still saved; the movie is skipped in the profile until it has a vector; the weekly job fills it in |
| Bundle download fails | Same as above; retry on the next out-of-catalog like |
| Keyword or person unseen by the current vocabulary | Ignored in that block until the next weekly refit |

### Tracking

Every `user_like` movie feeds the weekly out-of-catalog report (Section 14): the share of likes outside the top list and the most-liked such movies.

## 11. API design

A versioned REST API under `/api/v1` with typed Pydantic schemas, one consistent error shape, and the user always taken from the token, never from the URL.

| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| GET | `/health` | No | Liveness + database check |
| GET | `/movies/search?query=&page=` | No | Live TMDB search, short in-memory cache |
| GET | `/movies/trending` | No | Home row (TMDB trending, cached) |
| GET | `/movies/top-rated` | No | Home row (catalog by quality score) |
| GET | `/movies/{id}` | No | Details from the catalog; falls back to TMDB if not stored |
| GET | `/movies/{id}/similar` | No | "More like this" (Section 9) |
| GET | `/me/likes?cursor=` | Yes | Liked movies with details, newest first, cursor-paginated |
| GET | `/me/likes/ids` | Yes | All liked movie ids (feeds the browser's liked set) |
| PUT | `/me/likes/{movie_id}` | Yes | Like (idempotent, 204); handles out-of-catalog movies (Section 10) |
| DELETE | `/me/likes/{movie_id}` | Yes | Unlike (idempotent, 204) |
| GET | `/me/recommendations?limit=10` | Yes | For You list with reasons, cold-start flag, `model_version` |
| DELETE | `/me` | Yes | Delete account (refused for the demo account) |
| POST | `/demo/session` | No | Reset the demo account and return a session for it (Section 13) |

### Conventions

- **Auth:** a FastAPI dependency verifies the Supabase JWT (signature against the project's signing keys/JWKS, expiry, audience) and yields `user_id`.
- **Errors:** always `{"code": "...", "message": "..."}` with the right HTTP status.
- **Dev-only diversity parameter:** `GET /me/recommendations?diversity=0.3` is honored only when the app runs in development mode; production ignores it and uses the configured value.
- **TMDB caching:** search and trending results cached in memory per instance for a few minutes. Cloud Run instances are short-lived, so this is best-effort only.
- **CORS:** only the Vercel frontend origin (and localhost in development). Preview deployment URLs may also need allowing; decide when setting up Vercel.
- **Types:** the frontend's TypeScript API types are generated from FastAPI's OpenAPI schema (openapi-typescript), regenerated in CI.
- **Layering:** routers (HTTP only) → services (business logic) → repositories (database queries). Each layer is testable alone.

## 12. Frontend

A React + TypeScript SPA on Vercel with a distinctive movie-app look built from Tailwind and shadcn/ui. TanStack Query makes likes instant and caches recommendations by the exact set of liked movies.

### Routes

`/` Home · `/movie/:id` Movie detail · `/liked` Liked · `/for-you` For You · `/login`, `/signup`, `/reset-password` · `/settings` · `/about` · `/dev/compare` (development builds only).

### Instant likes (Confirmed)

- On sign-in, load `GET /me/likes/ids` into one cached Set of liked ids. Every heart on every card reads from it; no card waits on a request.
- The like mutation uses TanStack Query's optimistic pattern: `onMutate` cancels in-flight likes queries, snapshots the Set, and flips the movie immediately; `onError` restores the snapshot and shows a quiet toast; `onSettled` refreshes the Liked page list.
- Rapid clicks on one movie: mutations share a per-movie `mutationKey` and only the final intended state needs to reach the server. Idempotent endpoints make last-write-wins correct.
- **Required test:** a failed like request rolls the heart back.

### Recommendations cache: memoized by liked-set hash (Confirmed)

- The For You query key is `["recommendations", likedSetHash]`, where `likedSetHash` is a short hash of the sorted liked ids (for example `155,550,27205` → hash). A fast non-cryptographic hash or the sorted id string is enough.
- Same set → cache hit, shown instantly, no request. New set → new key → fetch and cache. Click order does not matter; unlike-then-re-like returns to the same key.
- `staleTime: Infinity`; unused entries dropped after about 30 minutes (`gcTime`). No "mark stale on like" step is needed.
- Limits: per tab and session (a reload starts fresh); server-side changes such as a new weekly vector version are not seen until reload. Both are acceptable.
- Scaling story for interviews: move this memoization server-side (for example Redis keyed by user + liked-set hash + model version), then to event-driven precomputation at very large scale.

### Pages and UI behavior

- **Home:** search bar (debounced, live TMDB), rows (Trending, Top Rated, For You once there are likes).
- **Movie detail:** poster, overview, genres, cast, like button, "More like this".
- **Liked:** grid, newest first, unlike in place (optimistic).
- **For You:** 10 cards, each with its "Because you liked X" reason; under 3 usable likes, show popular picks plus a prompt to like a few more.
- **Auth:** Google sign-in as the primary button; email/password as the secondary option; after email sign-up, a notice to confirm the address that also says to check the spam folder; "Try the demo" button.
- **Settings:** delete account with a confirmation step (hidden for the demo account).
- **Footer and About:** TMDB logo and notice (Section 5).

### Dev-only diversity comparison view (Confirmed)

At `/dev/compare`, for the current user's likes: the left list is the plain top 10 (diversity 0); the right list is driven by a slider from 0 to 0.5. Moving the slider refetches the right list with the dev-only `diversity` parameter. Not included in production builds.

### Visual design

- **Direction (Confirmed):** a movie-app style, poster-forward and cinematic, not a generic SaaS dashboard.
- **Styling (Confirmed):** Tailwind CSS + shadcn/ui; icons via lucide-react.
- **Design skills (Confirmed):** the owner will use the Claude skills **frontend-design** and **UI/UX Pro Max**. The split (Confirmed), to avoid conflicting direction: frontend-design owns the visual direction (colors, fonts, layout); UI/UX Pro Max is used only for its UX guidelines, accessibility checklist and React/Tailwind/shadcn stack guidance, not its catalog style picks. Apply this split from the start of frontend work.
- **Quality floor:** responsive from phone to desktop, visible keyboard focus, sufficient contrast, `prefers-reduced-motion` respected, real buttons and links with accessible names.

## 13. Authentication and accounts

Supabase Auth handles every auth flow; the only extra piece is an outside email provider for delivery, because Supabase's built-in sender is for testing only. All items in this section are Confirmed unless marked.

### Sign-in methods

| Method | Details |
| --- | --- |
| Google sign-in (primary) | Supabase's Google provider with an OAuth client created in the owner's Google Cloud project. No emails involved. |
| Email + password (secondary) | `supabase.auth.signUp`; **email confirmation on**: the user confirms their address once, at first sign-up. |
| Forgot / reset password | `supabase.auth.resetPasswordForEmail` sends the link; the reset page calls `supabase.auth.updateUser` with the new password. |
| Demo account | "Try the demo" button for interviewers (below). |

The browser holds the Supabase session (supabase-js refreshes it) and sends the access token as `Authorization: Bearer` on API calls. The API verifies it and never sees passwords.

### Email delivery

- Supabase's built-in sender is rate-limited to a few emails per hour and may only deliver to project team members, so real users need a custom SMTP provider. It is a settings change in the Supabase dashboard, not code.
- **No custom domain for now** (Confirmed). Use a provider that allows verifying a **single sender address** instead of a domain. Which provider is **Deferred**; Brevo was mentioned as one that likely supports this, to be verified. Some providers (for example Resend) need a domain to email anyone but yourself.
- Expect more spam-folder delivery without a domain; the sign-up notice tells users to check spam. Google sign-in is unaffected.
- **Local development:** the local Supabase stack includes a test inbox that catches all auth emails, so confirmation and reset flows are built and tested with no real email sent.

### Demo account

- One shared account with a preset list of liked movies, so recommendations are visible in one click.
- **Reset on every "Try the demo" click** to the preset likes, so every interviewer starts from the same state.
- Protected: cannot be deleted; email and password changes are hidden in the UI and refused by the API.
- Accepted edge case: two people using the demo at the same moment can reset each other's likes.
- **Implementation:** `POST /api/v1/demo/session` (server-side, service key): reset the demo user's likes to the preset, set the demo password from a secret, sign in server-side and return the session; the browser calls `supabase.auth.setSession`. This keeps the demo password out of the frontend.

### Account deletion

- "Delete account" in Settings with a confirmation step.
- `DELETE /api/v1/me` → the API deletes the user via Supabase Auth's admin function (service key) → `likes` and `rec_impressions` rows are removed by `ON DELETE CASCADE` → the browser signs out.
- Refused for the demo account.

### Settings that change if a domain is added later (no code changes)

Vercel domain; Supabase Auth site URL and redirect URLs; Google OAuth authorized domains; API CORS allowlist; email provider domain verification and sender address.

## 14. Evaluation and quality tracking

Recommendation quality is measured offline by hiding some of real MovieLens users' likes and checking whether the recommender's top 10 finds them; results are saved per run and guarded in CI. This tests recommendation quality, not code correctness.

### Data preparation

- Download MovieLens 32M during evaluation (cached in GitHub Actions); never commit it.
- A rating of 4 or more = a like. Map MovieLens ids to TMDB ids via `links.csv`.
- **Evaluate on the catalog overlap only** (Confirmed, option 1): held-out likes for movies outside the catalog are excluded from scoring, because the recommender can never recommend them. The catalog is not seeded with MovieLens movies.

### Protocol

1. Per user, sort likes by time. Hold out the most recent 5 in-catalog likes; give the rest to the recommender as that user's likes. Require a minimum number of likes per user (for example 10).
2. Run the same recommender code as the app (equal-weight profile, scoring, optional MMR) on the given likes.
3. Compare the top 10 against the held-out likes.
4. Average over users.

### Metrics and baselines

| Measure | What it tells you |
| --- | --- |
| Recall@10 | Share of held-out likes found in the top 10 |
| NDCG@10 | Same, rewarding hits ranked higher |
| Catalog coverage | Share of the catalog that ever gets recommended |
| Intra-list diversity | How different the 10 items are from each other |
| Mean popularity of recommendations | Whether it just recommends blockbusters |
| Random baseline | Proves likes follow patterns (random hits about 0.1% at 10 of 10k) |
| Most-popular baseline | Hard to beat on recall; beating it shows real taste modeling |

### Sample sizes (Confirmed)

| Run | Users | When |
| --- | --- | --- |
| CI check | Fixed sample of 2,000, fixed random seed | On code changes to recommender or feature code |
| Weekly | Fixed sample of 20,000 | After each weekly feature build |
| Full | All eligible users | Manual runs only |

### Experiments the harness must support

- **Ablation:** run once with the overview weight set to 0 to measure what plot summaries add (replaces the old v1/v2 comparison).
- **Diversity sweep:** report accuracy and diversity for λ in {0, 0.1, 0.2, 0.3, 0.5} to inform the deferred diversity decision.
- **Weight tuning:** grid-search block weights, the 0.85/0.15 split and m on a validation split; report on a separate test split.

### Quality gate in CI

No threshold exists until the first measured score. After Milestone 8 records a baseline on the fixed 2,000-user sample, CI fails a change if Recall@10 or NDCG@10 drops more than about 5% relative to the baseline, or if the recommender stops beating the most-popular baseline. Intentional trade-offs (for example turning on diversity) update the baseline deliberately. Adjust the 5% once run-to-run variation is known.

### Tracking reports (Confirmed)

| Report | Stored in | Contents |
| --- | --- | --- |
| Every evaluation run | `eval_runs` | Date, git SHA, feature version, catalog size, sample, settings, metrics |
| MovieLens overlap | `eval_runs` | Overlap rate (share of held-out likes in the catalog) and the most-missed movies ranked by how many users they were held out for |
| Real users' out-of-catalog likes | `catalog_reports` (weekly) | Share of likes on `source = 'user_like'` movies and the most-liked such movies |

These guide catalog improvements: frequently missed movies are evidence for adding them to the catalog later (for example MovieLens movies with 50+ ratings), backed by data. All reports are cheap: counts during runs plus one weekly SQL query.

### Honest caveats

MovieLens is a proxy: its users are self-selected movie enthusiasts, likes partly reflect popularity and exposure, and its tastes may differ from this app's users. The baselines separate real taste modeling from popularity.

## 15. Testing strategy

Every subtask ships with tests that prove it works; integration tests run against a real local Postgres with pgvector, because mocking the database would hide the queries that matter most. The parity test and the like-rollback test are required.

| Level | Tools | What it covers |
| --- | --- | --- |
| Unit (backend) | pytest | Feature blocks and `build_vector` with tiny hand-built fixtures and known answers (for example two movies sharing only a genre have cosine = genre weight); scoring, Bayesian quality, MMR, cold start, reasons, liked-set handling |
| API (backend) | pytest + httpx, auth dependency overridden | Every endpoint: status codes, error shape, idempotent likes, `/me` scoping, demo protections, account deletion |
| Integration | pytest against local Supabase (`supabase start`) | Migrations apply cleanly; pgvector retrieval; cascade deletes; ingestion upserts |
| External APIs | respx | TMDB mocked, including 404, 429 and thin-data movies (no keywords, no director, no overview) |
| Storage | In-memory fake behind the storage interface | Bundle upload/download, lazy load, version checks |
| Parity (required) | pytest in CI | Same movies' vectors via the weekly-job path and the API path are identical |
| Frontend | Vitest + React Testing Library + MSW | Components and hooks; **required:** a failed like request rolls the heart back; liked-set hash is order-independent |
| End to end | Playwright | Sign up (local test inbox), search, like, see it on Liked, get recommendations; demo button; delete account |
| Recommendation quality | Evaluation harness (Section 14) | CI gate on the 2,000-user sample |

### Rules

- A subtask is not done until its tests pass locally and in CI.
- Use deterministic fixtures and fixed random seeds; no live network calls in tests.
- Add a regression test for every bug fixed.

## 16. Scheduled jobs and CI/CD

All jobs run on GitHub Actions (free and unlimited on standard runners for a public repo). Only two run on a schedule in production: the weekly pipeline and the keep-alive.

| Workflow | Trigger | Steps |
| --- | --- | --- |
| Weekly pipeline | Weekly cron (for example Sunday night) + manual "run now" | 1. Catalog ingestion (Section 7) → 2. Feature build + bundle upload + version switch (Section 8) → 3. Weekly evaluation, 20k users (Section 14) → 4. Cleanup: delete bundles beyond the last 3 to 4 versions and vectors older than the previous version → 5. Out-of-catalog report (`catalog_reports`) |
| Keep-alive | Every 2 to 3 days | One simple database query so the Supabase free project does not pause after 7 days of low activity |
| CI | Every pull request and push | ruff, mypy, eslint, tsc; backend and frontend tests; parity test; OpenAPI type generation check; evaluation gate on the 2k sample when recommender or feature code changes |
| Migrations | Merge to `main` | Apply new SQL migrations to the hosted database |
| Deploy backend | Merge to `main` | Build the container, push to Google's registry, deploy to Cloud Run (Workload Identity Federation, no stored Google key) |
| Deploy frontend | Vercel's GitHub integration | Merges to `main` deploy production; every pull request gets a preview link |
| Manual evaluation | Manual | Full dataset, ablation, diversity sweep, tuning runs |

### Notes

- Each scheduled job records itself in `job_runs` (start, finish, status, counts).
- The weekly pipeline is a sequence: if ingestion fails, features and evaluation do not run; the app keeps serving the last active version.
- Weekly runtime estimate: a few minutes for ingestion and feature build, plus a few minutes of evaluation (estimates).
- Rollback: Cloud Run keeps every deployed revision and can switch traffic back instantly; Vercel has instant rollback; vectors keep the previous version.

## 17. Local development

The owner develops with Claude Code in VS Code on their own machine; the full stack runs locally with Docker, and local development needs no TMDB key, no real email and no cloud services.

### Prerequisites

Git, Docker Desktop, Python 3.12+, uv, Node.js (LTS), the Supabase CLI. Later, for deployment: the Google Cloud CLI.

### Commands

| Command | Does |
| --- | --- |
| `supabase start` | Local Postgres + pgvector, Auth, Storage and the test inbox for auth emails |
| `make migrate` | Apply migrations to the local database |
| `make seed` | Load the \~500-movie fixture, build its vectors and bundle, create the demo account with preset likes |
| `make api` | Run FastAPI with reload |
| `make web` | Run the Vite dev server |
| `make test` | Backend and frontend tests |
| `make eval-ci` | Evaluation on the 2,000-user sample (downloads MovieLens once) |

### Environment

- A `.env` file on the owner's machine, ignored by Git, holds local secrets.
- A committed `.env.example` lists every variable name with no values.
- Local Google sign-in needs a local OAuth configuration; email/password with the local test inbox works out of the box.

### Working with Claude Code

- This document lives in the repo (for example `docs/PROJECT_PLAN.md`).
- A short `CLAUDE.md` at the repo root summarizes the stack, the rules (never commit secrets or MovieLens, one milestone at a time, tests with every subtask, follow Confirmed decisions, keep Deferred items configurable) and points to this document.
- The two design skills (frontend-design, UI/UX Pro Max) are installed in Claude Code before frontend work begins.

## 18. Deployment, infrastructure, secrets and observability

Backend on Google Cloud Run's free tier, frontend on Vercel, database/auth/storage on Supabase's free plan, jobs on GitHub Actions; total running cost is $0 at portfolio traffic.

### Google Cloud Run (backend)

| Setting | Value | Why |
| --- | --- | --- |
| Billing mode | Request-based (default) | Pay only while handling requests; free allowance covers a portfolio app |
| Minimum instances | 0 (scale to zero) | Free; cold starts of roughly 2 to 6 seconds after idle (estimate, measure after deploy) |
| Memory | 1 GB | Comfortable for FastAPI + numpy + scikit-learn + ONNX model; confirm with the Milestone 6 measurement |
| Startup CPU boost | On | Shorter cold starts |
| Region | Same region as the Supabase project | Low database latency; choose both together at setup |
| Budget alert | About $1 | Email if usage ever leaves the free allowance |
| Image | ONNX model file baked in; no PyTorch | No model download at startup |

Keep the image small (no unused libraries) to shorten cold starts. With request-based billing, CPU is throttled after a response, so never rely on background work after responding (Section 10). The API keeps its default Cloud Run address; users never see it.

### Vercel (frontend)

- Connected to the GitHub repo: `main` deploys production; pull requests get preview links.
- Environment variables: API base URL, Supabase project URL, Supabase anon key (public by design; database rules protect data, not the key).
- SPA rewrite: every path serves `index.html`, so `/liked` and `/movie/949` work when opened directly.
- Address: the default `*.vercel.app` URL (no custom domain for now).

### Supabase (hosted, free plan)

| Limit | Free plan | This app |
| --- | --- | --- |
| Database | 500 MB | Well under 100 MB expected |
| File storage | 1 GB | 10 to 15 MB per bundle; keep last 3 to 4 |
| Egress | 5 GB/month | Bundle loaded lazily, never at every cold start |
| Monthly active users | 50,000 | Tiny |
| Inactivity pause | After 7 days of low activity | Keep-alive job every 2 to 3 days |
| Backups | Not downloadable on Free | No backup (accepted): movie data can be rebuilt from TMDB; user accounts, likes and history cannot |

### Secrets (Confirmed)

| Secret | Used by | Lives in |
| --- | --- | --- |
| TMDB API key | API, weekly job | Google Secret Manager (Cloud Run); GitHub Actions secrets (jobs) |
| Database connection string | API, weekly job, keep-alive | Same |
| Supabase service key | API (storage, account deletion, demo), weekly job | Same; never in the frontend |
| Demo account password | API | Same |
| Google Cloud deploy identity | CI | Workload Identity Federation: GitHub proves its identity to Google; no long-lived key stored |
| Email provider SMTP password | Supabase Auth | Supabase dashboard |
| Google sign-in client secret | Supabase Auth | Supabase dashboard |

Not secret: Supabase URL, anon key, API URL (Vercel environment). Locally: `.env` (ignored) + `.env.example` (committed). The repo is **public** (Confirmed); enable GitHub secret scanning and push protection.

### Observability (Confirmed: no telemetry platform)

- Structured JSON logs per request: request id, user id, endpoint, status, duration. Viewed in Cloud Run's log viewer.
- `/health` endpoint.
- Job and evaluation reports in the database (`job_runs`, `eval_runs`, `catalog_reports`).
- Worth watching: recommendation latency (should stay in tens of milliseconds), failed like requests (each one is a rolled-back heart), TMDB errors and 429s, weekly pipeline success.
- Sentry and an uptime monitor are left out (Confirmed).

## 19. Project structure and coding standards

One monorepo with the backend, frontend, database migrations and evaluation side by side; the recommender is a shared package used by both the API and the jobs.

```text
movie-recs/
├─ backend/
│  ├─ app/
│  │  ├─ api/v1/         # routers only: HTTP <-> service calls
│  │  ├─ core/           # settings, auth (JWT verification), errors, logging
│  │  ├─ db/             # session, models, repository functions
│  │  ├─ schemas/        # Pydantic request/response models
│  │  ├─ services/       # movies, likes, recommendations, accounts, demo
│  │  ├─ recommender/    # build_vector, feature blocks, generators, blender, reranker, explainer
│  │  ├─ storage/        # storage interface + Supabase implementation + in-memory fake
│  │  └─ tmdb/           # typed TMDB client, fetch-and-store, rate limiting, cache
│  ├─ jobs/              # ingest.py, build_features.py, cleanup.py, keepalive.py, reports.py
│  ├─ tests/
│  ├─ Dockerfile
│  └─ pyproject.toml
├─ frontend/
│  ├─ src/
│  │  ├─ api/            # generated types + fetch client
│  │  ├─ auth/           # Supabase session provider, protected routes
│  │  ├─ features/       # movies, likes, recommendations, account (hooks + components)
│  │  ├─ components/ui/  # shadcn/ui components
│  │  └─ pages/
│  └─ package.json
├─ supabase/
│  ├─ migrations/        # SQL: tables, RLS, pgvector
│  └─ seed/              # ~500-movie fixture, demo preset likes
├─ eval/                 # MovieLens preparation, harness, metrics, baselines
├─ docs/PROJECT_PLAN.md  # this document
├─ .github/workflows/    # ci, weekly, keepalive, deploy
├─ CLAUDE.md
├─ Makefile
└─ .env.example
```

### Coding standards

- Python: type hints everywhere, ruff for lint and format, mypy clean. TypeScript: strict mode, eslint, no `any` without a reason.
- Layering: routers → services → repositories; no SQL in routers, no HTTP concerns in services.
- All tunable numbers (block weights, 0.85/0.15, m, λ, candidate count, cold-start threshold, cast count, sample sizes) live in settings, never as literals in code.
- Small, focused commits with descriptive messages; one subtask per commit where practical.
- Comments explain why, not what. Non-obvious decisions reference the section of this document.
- No secrets, keys or MovieLens data in the repo; `.env` is ignored.

## 20. Build order

Eleven milestones, each building on the previous ones. Break every milestone into small subtasks, complete and verify one at a time, and get the owner's confirmation before moving on.

### Milestone 1: Foundation

- Public GitHub repo with the structure in Section 19; `CLAUDE.md`; this document in `docs/`.
- Python (uv, ruff, mypy, pytest) and frontend (Vite, React, TypeScript, Tailwind, shadcn/ui, Vitest) tooling.
- Local Supabase running; first migration (enable pgvector); `.env.example`; Makefile.
- CI: lint, type-check and test on every push; secret scanning and push protection on.
- **Done when:** `make api`, `make web` and `make test` work on a clean checkout; CI is green; `/health` returns OK against the local database.

### Milestone 2: Accounts

- Google sign-in, email + password with confirmation (local test inbox), forgot/reset password.
- API JWT verification dependency; a protected `/me` test route.
- Demo account: preset likes seed, `POST /demo/session`, reset on each click, protections.
- Delete account (`DELETE /me`, cascade, confirmation UI).
- **Done when:** each flow works end to end locally; API tests cover auth failures, demo protections and deletion cascade.

### Milestone 3: Movie catalog

- Catalog tables and `job_runs` migrations; RLS on all tables.
- Typed TMDB client with throttling and backoff; shared fetch-and-store function; `source` field.
- Ingestion job (Section 7) including changes endpoint, 14-day fallback, inactive marking.
- The \~500-movie seed fixture and `make seed`.
- **Done when:** a local ingestion run of a few hundred movies stores correct, complete records (spot-checked against TMDB); tests cover thin data, 404s and 429s.

### Milestone 4: Browsing

- Install the design skills and apply their split (Section 12); establish the movie-app visual direction.
- Home (live search + rows), movie detail, TMDB attribution in footer and About.
- Movie endpoints (search, trending, top-rated, details).
- **Done when:** search and browsing work against the local catalog and live TMDB; responsive and keyboard-accessible.

### Milestone 5: Likes

- `likes` table; like/unlike endpoints; `GET /me/likes` and `/me/likes/ids`.
- Optimistic liked-set, per-movie mutation keys, rollback toast; Liked page.
- Out-of-catalog like: fetch and store with `source = 'user_like'` (vector added in Milestone 6).
- **Done when:** likes feel instant; the rollback test passes; out-of-catalog movies appear on the Liked page.

### Milestone 6: Movie vectors

- Feature build job: all five blocks, overview via fastembed ONNX, embedding cache, versioned vectors, version switch, index.
- Fitted-pieces bundle (JSON + npz) behind the storage interface; Supabase Storage implementation; cleanup.
- Shared `build_vector`; parity test.
- **Measure the API's real memory use with the ONNX model loaded** and record it.
- **Done when:** every seeded movie has a vector; parity test passes; similar-movie spot checks look sensible (for example Cars 2 near Cars); memory measured.

### Milestone 7: Recommendations

- Recommender pipeline (Section 9) behind the hybrid-ready interfaces; `rec_impressions`; `model_version`.
- For You page with reasons and cold start; liked-set-hash caching; "More like this".
- Instant vectors for out-of-catalog likes, built before responding; lazy bundle loading.
- Dev-only diversity comparison view with slider.
- **Done when:** recommendations return in tens of milliseconds; unlike/re-like returns an identical cached list with no request; an out-of-catalog like changes recommendations immediately; tests cover scoring, cold start and MMR.

### Milestone 8: Evaluation

- MovieLens 32M preparation (never committed), protocol, metrics, baselines (Section 14).
- `eval_runs`, overlap rate and most-missed movies; `catalog_reports`.
- Ablation (overview off) and diversity sweep; weight tuning.
- Record the baseline and enable the CI quality gate.
- **Done when:** the 2k CI run is deterministic and beats both baselines; reports are stored; the gate blocks an intentionally degraded change.

### Milestone 9: Deployment

- Hosted Supabase project (region chosen with Cloud Run); migrations via CI.
- Cloud Run deploy via Workload Identity Federation and Secret Manager, with the settings in Section 18; budget alert.
- Vercel production and previews; CORS.
- Email provider (single verified sender) in Supabase; Google OAuth for production.
- Weekly pipeline and keep-alive workflows on schedule.
- **Done when:** the deployed app passes the end-to-end happy path; the first scheduled weekly run succeeds; cold-start time measured.

### Milestone 10: Finishing

- Playwright end-to-end tests in CI.
- README with architecture diagram, setup steps, evaluation results and trade-offs.
- The owner decides on diversity using the comparison view and evaluation numbers; set the configured λ.
- **Done when:** a new developer can run the project from the README; the owner can explain every part.

### Milestone 11 (optional): Collaborative filtering

- Add an item-item CF generator (co-likes) or implicit ALS to the blender; evaluate the hybrid against content-only.
- **Done when:** the hybrid is measured and its weight chosen with evaluation evidence.

## 21. Open items, decision log and glossary

The owner approved every proposed default on 2026-11-13, so they are all Confirmed; only the Deferred items below remain open, and they stay configurable until decided.

### Approved defaults (Confirmed on 2026-11-13)

- Backend libraries: FastAPI, Pydantic, SQLAlchemy 2.0 async + asyncpg, Supabase CLI migrations, uv/ruff/mypy
- Frontend libraries: Vite, React Router, supabase-js (auth only), openapi-typescript, lucide-react, Recharts
- Backend-mediated data access with RLS as defense in depth; recommendations computed on request
- API endpoint list and conventions (Section 11), including the demo session implementation
- Exact table columns (Section 6); HNSW index vs exact scan
- Catalog size of about 10k and the discover-list mix (vote count, popularity, rating)
- Block weights 0.30/0.25/0.20/0.20/0.05, 64-dimension SVD, top 5 cast
- Scoring 0.85/0.15, m = 500, 200 candidates, cold-start threshold of 3 usable likes
- Evaluation protocol details (hold out last 5, minimum likes per user) and the \~5% CI gate
- Testing tools (Section 15)
- Pages and routes (Sections 2 and 12); project structure (Section 19)
- Design-skill split: frontend-design for visual direction, UI/UX Pro Max for UX rules and stack guidance only
- Sentry and uptime monitor left out along with the telemetry platform
- Build order (Section 20)

### Deferred (build configurable; do not decide)

- [ ] Diversity on or off and the λ value: decided by the owner after the dev comparison view and the evaluation sweep (Milestone 10)
- [ ] Email provider (must support a single verified sender without a domain)
- [ ] Custom domain: none for now; adding one later is settings only (Section 13)
- [ ] Region for Supabase and Cloud Run (choose together at setup)
- [ ] Whether Vercel preview URLs are allowed by CORS
- [ ] Collaborative filtering (optional Milestone 11)

### Decision log (Confirmed)

| Decision | Reason |
| --- | --- |
| Fresh project; ignore any earlier movie app | Independent design |
| Python + React + Supabase + TMDB | Owner's requirements |
| Weekly catalog refresh (not nightly) | Freshness need is low; simpler |
| Single recommender version with the overview block from the start; no v1/v2 | Owner's choice; ablation replaces the v1/v2 comparison |
| Overview text only; no title, no tagline | Titles create false matches; taglines are mostly marketing |
| ONNX (fastembed) in both job and API; no PyTorch anywhere | Complete instant vectors on a small server |
| Instant vectors for out-of-catalog likes, built before responding | A week's delay was too long; Cloud Run throttles CPU after responding |
| One shared `build_vector` + versioned pieces + parity test | Prevent silent drift between the two vector paths |
| Fitted-pieces bundle in Supabase Storage behind a swappable interface; JSON + npz | Simplest; switchable to GCS; safe format |
| Bundle loaded lazily, not at startup | Stay within 5 GB/month egress |
| No recency weighting; equal-weight profile | Owner does not want recent likes favored in any way |
| Browser memoization by liked-set hash | Identical liked sets never recompute |
| Evaluate on MovieLens overlap only; catalog not seeded from MovieLens | Scores reflect the recommender; catalog stays TMDB-defined |
| Track MovieLens overlap and real-user out-of-catalog likes | Evidence for future catalog changes |
| MovieLens 32M; 2k CI, 20k weekly, full manual | Enough users and recent ratings; fast enough |
| Diversity built as a setting, dev side-by-side view with slider 0 to 0.5 | Owner wants to see both before deciding |
| No telemetry platform | Not needed at this scale |
| Backend on Cloud Run free tier; frontend on Vercel | $0; owner's choice |
| Keep-alive job every 2 to 3 days; no database backup | Prevent Supabase pause; data-loss risk accepted |
| Google sign-in primary; email/password secondary with confirmation; forgot/reset password | Easy for users; Supabase handles flows |
| Demo account for interviewers, reset on every click | Same clean start for everyone |
| In-app account deletion | User control over their data |
| Secrets in Secret Manager and GitHub secrets; Workload Identity Federation for deploys | No long-lived keys |
| Public repo | Visible portfolio; free Actions minutes |
| TMDB setup and attribution as in Section 5 | Terms compliance |
| No custom domain for now | Can be added later with settings only |
| Movie-app visual style with Tailwind + shadcn/ui; frontend-design and UI/UX Pro Max skills | Distinctive look the owner wants |

### Glossary

| Term | Meaning |
| --- | --- |
| Catalog | The \~10k movies stored in the database; the only movies that can be recommended |
| Out-of-catalog movie | A movie found via search or Trending that is not yet stored |
| Vector / embedding | A list of numbers representing a movie's content; similar movies have close vectors |
| Cosine similarity | Closeness of two vectors, from 0 (unrelated) to 1 (identical direction) |
| TF-IDF / IDF | Weighting where rare keywords or people count more than common ones |
| SVD | Compression of thousands of keyword or people columns into 64 numbers |
| Fitted pieces | Reference data learned from the catalog (vocabularies, IDF weights, SVD tables) needed to build vectors consistently |
| ONNX / fastembed | A lightweight engine and library that run the pre-trained text model without PyTorch |
| pgvector / HNSW | Postgres vector extension and its approximate nearest-neighbor index |
| Bayesian average | A rating pulled toward the catalog mean when a movie has few votes |
| MMR | Maximal marginal relevance: re-ranking that trades relevance for variety |
| Cold start (users) | Too few likes to personalize; popular picks shown instead |
| Cold start (server) | Delay while a sleeping Cloud Run instance starts |
| Held-out like | A like hidden from the recommender during evaluation to check its predictions |
| Recall@10 / NDCG@10 | Share of held-out likes found in the top 10 / same, rewarding higher ranks |
| Ablation | Switching one part off to measure its contribution |
| Memoization | Reusing a stored result for the same inputs (here: the same liked set) |
| Optimistic update | Updating the UI before the server confirms, rolling back on failure |
| Idempotent | Repeating the request has the same effect as doing it once |
| RLS | Postgres row-level security |
| Egress | Data transferred out of a service; Supabase Free allows 5 GB/month |
| Workload Identity Federation | GitHub proving its identity to Google without a stored key |
