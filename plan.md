# Project Specification & Implementation Plan: Creator Hub (Personal Multi-Platform Studio)

## 1. System Overview & Scope Boundaries
A centralized personal web application for a content creator to view analytics/KPIs, manage and reply to comments in a unified inbox, and cross-post videos across **YouTube, Facebook Page, and Instagram (Creator/Business)**.

### Strict Scope Boundaries (DO NOT EXCEED)
* **Supported Platforms ONLY:** YouTube, Facebook Page, Instagram. (Do NOT implement TikTok, X/Twitter, or any other platform).
* **Single-User Personal Mode:** This app runs locally for a single creator. **Do NOT implement** multi-tenant user registration, login screens, password hashing, or JWT user authentication.
* **No Heavy Queue Brokers:** Use Python's built-in `asyncio` / FastAPI `BackgroundTasks` and `APScheduler`. **Do NOT** add Redis, Celery, or RabbitMQ.

### Architecture Pattern (Backend-Centric 3-Tier)
* **Frontend:** Next.js (App Router, TypeScript, Tailwind CSS, Shadcn UI). Communicates **strictly** with the Python FastAPI Backend via REST API. It has zero direct knowledge of Supabase.
* **Backend (Local Execution):** Python FastAPI. Acts as the single source of truth holding all business logic, OAuth tokens, APScheduler cronjobs, Platform Adapters, and database connections.
* **Database (Cloud):** Supabase (PostgreSQL) accessed exclusively by the FastAPI Backend using SQLAlchemy ORM.
* **Temporary Object Storage (Cloud):** Cloudflare R2 (S3-compatible) used by the Backend to temporarily stage video files for Meta (Facebook/Instagram) Graph API ingestion.

---

## 2. Strict Execution Protocol for AI Agent (Claude Code / Codex)

> **CRITICAL RULES — YOU MUST FOLLOW THIS WORKFLOW FOR EVERY SINGLE TASK:**
> 1. **One Task at a Time:** Execute strictly one Task at a time in the exact sequence listed below. Do NOT jump ahead or combine multiple tasks.
> 2. **Implement:** Write clean, modular, type-annotated code strictly within the task's deliverables.
> 3. **Write Automated Tests:** Immediately after implementing the task, write comprehensive unit/integration tests (`pytest` for Backend, `vitest` for Frontend) covering:
>    - Normal execution (Happy Path)
>    - Edge cases (empty data, null fields, rate limits, expired tokens)
>    - Error handling (API failures, invalid inputs, DB constraint violations)
> 4. **Run Tests & Self-Heal:** Execute the test suite in the terminal. If ANY test fails or throws an error, analyze the traceback, fix the implementation code, and re-run the tests until **100% of tests pass**.
> 5. **Checkpoint:** Once all tests for the current task pass, report the test summary and wait for user confirmation before proceeding to the next task.

---

## 3. Directory Structure (Monorepo)

```text
creator-hub/
├── backend/
│   ├── app/
│   │   ├── main.py                 # FastAPI app initialization, CORS, lifespan scheduler
│   │   ├── config.py               # Pydantic BaseSettings (.env loader)
│   │   ├── database.py             # SQLAlchemy engine, SessionLocal, get_db dependency
│   │   ├── models.py               # SQLAlchemy ORM Models (7 core tables)
│   │   ├── schemas/                # Pydantic DTOs (Normalized DTOs & API Request/Response)
│   │   ├── repositories/           # Data Access Layer (CRUD & Upsert operations)
│   │   ├── adapters/               # External API Integrations (YouTube, FB, IG, R2)
│   │   ├── services/               # Business Logic (TokenManager, Sync, Analytics, Publish)
│   │   └── routers/                # FastAPI Endpoints (accounts, dashboard, comments, publish)
│   ├── tests/                      # Pytest suite mirroring app/ structure
│   ├── temp_videos/                # Local staging directory for video uploads
│   ├── requirements.txt
│   └── .env.example
└── frontend/
    ├── src/
    │   ├── app/                    # Next.js App Router pages (Dashboard, Inbox, Publish, Accounts)
    │   ├── components/             # UI Components (KPI cards, Charts, Inbox Thread, Upload Form)
    │   ├── lib/api.ts              # Typed HTTP client talking to FastAPI
    │   └── types/                  # TypeScript interfaces matching FastAPI Pydantic schemas
    ├── __tests__/                  # Frontend unit/component tests (Vitest)
    └── .env.local.example
```

---

## 4. Database Architecture & Data Flow

### 4.1 Core Tables (7 Tables)
1. **`connected_accounts`**: `id (UUID, PK)`, `platform (ENUM: YOUTUBE, FACEBOOK, INSTAGRAM)`, `external_account_id (VARCHAR)`, `account_name`, `avatar_url`, `access_token`, `refresh_token`, `token_expires_at`, `followers_count (BIGINT)`, `total_views_count (BIGINT)`, `last_synced_at`, `created_at`. *(Unique constraint on `platform + external_account_id`)*.
2. **`account_metrics_snapshots`**: `id (BIGSERIAL, PK)`, `account_id (FK)`, `snapshot_date (DATE)`, `followers_count`, `total_views`, `platform_extra (JSONB)`. *(Unique constraint on `account_id + snapshot_date`)*.
3. **`media_posts`**: `id (UUID, PK)`, `account_id (FK)`, `campaign_id (FK, Nullable)`, `platform (ENUM)`, `post_type (ENUM: VIDEO, SHORT, REEL)`, `external_post_id (VARCHAR)`, `title`, `thumbnail_url`, `permalink`, `published_at`, `views_count`, `likes_count`, `comments_count`, `shares_count`, `updated_at`. *(Unique constraint on `platform + external_post_id`)*.
4. **`post_metrics_snapshots`**: `id (BIGSERIAL, PK)`, `post_id (FK)`, `recorded_at`, `views_count`, `likes_count`, `comments_count`, `shares_count`, `watch_time_minutes`, `avg_view_duration_sec`, `raw_metrics (JSONB)`. *(Index on `post_id, recorded_at DESC`)*.
5. **`comments`**: `id (UUID, PK)`, `post_id (FK)`, `parent_comment_id (FK self-reference, Nullable)`, `platform (ENUM)`, `external_comment_id (VARCHAR)`, `author_name`, `author_avatar_url`, `is_from_owner (BOOL)`, `content`, `likes_count`, `is_replied (BOOL)`, `published_at`. *(Unique constraint on `platform + external_comment_id`, Index on `is_replied, published_at DESC`)*.
6. **`publish_campaigns`**: `id (UUID, PK)`, `master_title`, `master_caption`, `local_file_path`, `r2_temp_url`, `created_at`.
7. **`publish_tasks`**: `id (UUID, PK)`, `campaign_id (FK -> publish_campaigns.id)`, `account_id (FK -> connected_accounts.id)`, `status (ENUM: PENDING, UPLOADING, PROCESSING, PUBLISHED, FAILED)`, `error_message`, `resulting_post_id (FK -> media_posts.id, Nullable)`, `updated_at`.

### 4.2 Backend 4-Layer Data Flow
1. **Layer 1 (Routers):** Receives HTTP requests from Next.js, validates input via Pydantic Schemas, delegates to Services, and returns formatted JSON responses.
2. **Layer 2 (Services / Orchestrator):** Executes core business logic (`SyncService`, `AnalyticsService`, `CommentService`, `PublishService`, `TokenManagerService`).
3. **Layer 3 (Platform Adapters):** Communicates with external APIs (YouTube Data API v3, Meta Graph API, Cloudflare R2) and normalizes raw external JSON payloads into standard internal Pydantic DTOs (`NormalizedAccountDTO`, `NormalizedPostDTO`, `NormalizedCommentDTO`).
4. **Layer 4 (Repositories):** Executes SQLAlchemy CRUD and idempotent `UPSERT` operations against Supabase PostgreSQL.

---

## 5. Step-by-Step Implementation Tasks

### Phase 1: Backend Foundation & Database Layer

#### [ ] Task 1: Project Scaffolding & Configuration
* **Goal:** Initialize the monorepo folder structure, Python virtual environment dependencies (`fastapi`, `uvicorn`, `sqlalchemy`, `psycopg2-binary`, `pydantic-settings`, `httpx`, `apscheduler`, `boto3`, `python-multipart`, `pytest`, `pytest-asyncio`), and `app/config.py`.
* **Deliverables:**
  - `backend/requirements.txt`, `backend/.env.example`
  - `backend/app/config.py` loading environment variables with validation.
  - `backend/app/main.py` with a basic `/health` endpoint and CORS middleware.
* **Verification & Testing (`tests/test_config_and_health.py`):**
  - Test environment variable loading and default fallbacks.
  - Test `GET /health` returns status 200 and `{"status": "ok"}`.
  - Run `pytest` and fix any import or configuration errors until 100% pass.

#### [ ] Task 2: Database Connection & SQLAlchemy ORM Models
* **Goal:** Implement `app/database.py` and `app/models.py` containing all 7 tables, enums, foreign keys, cascades, unique constraints, and indexes.
* **Deliverables:**
  - `backend/app/database.py` (Engine setup supporting both PostgreSQL URL and in-memory SQLite for isolated fast unit testing).
  - `backend/app/models.py` (All 7 models + Enums: `PlatformType`, `PostType`, `PublishStatus`).
* **Verification & Testing (`tests/test_models.py`):**
  - Test table creation (`Base.metadata.create_all`).
  - Test creating records for all 7 tables and verifying relationships (`Account -> Posts -> Comments`, self-referencing `Comment -> Replies`, `Campaign -> Tasks`).
  - Test `UniqueConstraint` (verify that inserting duplicate `platform + external_post_id` raises `IntegrityError`).
  - Test Cascade deletes (deleting an account deletes its posts and snapshots).
  - Run `pytest` and resolve all ORM mapping or constraint bugs until 100% pass.

#### [ ] Task 3: Pydantic Schemas & Normalized Internal DTOs
* **Goal:** Create strict data schemas in `app/schemas/` for internal data normalization (Adapter -> Service) and external REST API contracts (Router -> Next.js).
* **Deliverables:**
  - `NormalizedAccountDTO`, `NormalizedPostDTO`, `NormalizedCommentDTO`.
  - Request/Response schemas for Dashboard KPIs, Post lists, Comment Inbox, Reply payload, Account creation, and Video Upload status.
* **Verification & Testing (`tests/test_schemas.py`):**
  - Test valid data serialization and type coercion (e.g., string numbers `"1500"` from YouTube converting cleanly to `int`).
  - Test validation errors on missing required fields or invalid Enum values.
  - Run `pytest` and fix any schema validation issues until 100% pass.

#### [ ] Task 4: Data Access Repositories (CRUD & Upsert Logic)
* **Goal:** Implement `app/repositories/` (`account_repo.py`, `post_repo.py`, `comment_repo.py`, `publish_repo.py`) to encapsulate all SQLAlchemy queries.
* **Deliverables:**
  - Idempotent **Upsert** functions for Accounts, Posts, Account Snapshots, and Comments (if `external_id` exists, update metrics; if not, insert new row).
  - Comment reply state updater (inserting an owner reply automatically sets the parent comment's `is_replied = True`).
  - Dashboard aggregation queries (sum of followers/views, top performing posts, time-series history lookup).
* **Verification & Testing (`tests/test_repositories.py`):**
  - Test Upserting the same post twice updates `views_count` without creating duplicate rows.
  - Test Account Snapshot upsert on the same `snapshot_date` updates the existing row, while a new date creates a new row.
  - Test Comment Inbox filtering (`is_replied = False`, ordered by `published_at DESC`).
  - Test marking a parent comment as `is_replied = True` when a child reply is inserted.
  - Run `pytest` and fix any query or transaction bugs until 100% pass.

---

### Phase 2: External Platform Adapters & Token Management

#### [ ] Task 5: Base Adapter Interface & Token Lifecycle Manager
* **Goal:** Create the abstract `BasePlatformAdapter` interface and `TokenManagerService` that automatically checks `token_expires_at` and refreshes expired OAuth tokens before any API call.
* **Deliverables:**
  - `backend/app/adapters/base.py` defining abstract methods: `fetch_account_stats`, `fetch_recent_posts`, `fetch_comments`, `reply_to_comment`, `upload_video`.
  - `backend/app/services/token_manager.py` handling Google OAuth refresh token flow and Meta long-lived token exchange/validation.
* **Verification & Testing (`tests/test_token_manager.py`):**
  - Mock HTTP responses (`unittest.mock`).
  - Test valid token returns immediately without calling external OAuth endpoints.
  - Test expired YouTube token triggers refresh endpoint, updates `access_token` and `token_expires_at` in DB, and returns the new token.
  - Test failed token refresh raises a custom `TokenRefreshError` and logs the issue cleanly.
  - Run `pytest` and fix any async/mocking errors until 100% pass.

#### [ ] Task 6: YouTube Data API v3 Adapter
* **Goal:** Implement `YouTubeAdapter` translating Google API responses into Normalized DTOs, plus comment replies and resumable video stream uploads.
* **Deliverables:**
  - `backend/app/adapters/youtube_adapter.py`
* **Verification & Testing (`tests/test_youtube_adapter.py`):**
  - Mock YouTube API JSON payloads (`channels.list`, `playlistItems.list`, `videos.list`, `commentThreads.list`, `comments.insert`, video upload init/put).
  - Verify `YouTubeAdapter` accurately parses nested statistics into `NormalizedPostDTO` and `NormalizedCommentDTO`.
  - Test quota exceeded (HTTP 403) and network timeout error handling.
  - Run `pytest` and fix any parsing edge cases (e.g., missing thumbnails or disabled comments) until 100% pass.

#### [ ] Task 7: Cloudflare R2 Storage Service & Meta Adapters (Facebook & Instagram)
* **Goal:** Implement `R2StorageService` (upload local video -> get public URL -> delete file) and `FacebookAdapter` / `InstagramAdapter` using Meta Graph API.
* **Deliverables:**
  - `backend/app/adapters/r2_storage.py` (using `boto3` S3 client).
  - `backend/app/adapters/facebook_adapter.py` & `instagram_adapter.py`.
* **Verification & Testing (`tests/test_meta_and_r2.py`):**
  - Mock `boto3` S3 calls to verify upload, URL generation, and cleanup deletion.
  - Mock Meta Graph API responses for Page/IG insights, Reels/Video metrics, comment fetching, comment replies, and container-based video publishing (passing the R2 URL).
  - Test handling of missing insights data on newly published posts.
  - Run `pytest` and fix any errors until 100% pass.

---

### Phase 3: Core Backend Services & Orchestration

#### [ ] Task 8: Background Sync Engine (Analytics & Comments Orchestrator)
* **Goal:** Implement `SyncService` that iterates through all `connected_accounts`, obtains valid tokens via `TokenManager`, calls the respective Platform Adapter, and persists normalized data via Repositories.
* **Deliverables:**
  - `backend/app/services/sync_service.py`
  - `backend/app/scheduler.py` (APScheduler configuration for Fast Comment Sync every 15 mins & Slow Analytics Sync every 4 hours).
* **Verification & Testing (`tests/test_sync_service.py`):**
  - Test full end-to-end sync flow using mocked adapters and test DB: verify `connected_accounts`, `media_posts`, `post_metrics_snapshots`, and `comments` tables are populated accurately.
  - **Fault Isolation Test:** Simulate Facebook API throwing a 500 error during sync, and verify that YouTube and Instagram accounts STILL finish syncing without crashing the whole job.
  - Run `pytest` and fix any transaction rollback or isolation issues until 100% pass.

#### [ ] Task 9: Unified Comment Reply & Analytics Aggregation Services
* **Goal:** Implement `CommentService` (for replying to comments across platforms) and `AnalyticsService` (for computing cross-platform KPI summaries, growth rates, and chart datasets).
* **Deliverables:**
  - `backend/app/services/comment_service.py`
  - `backend/app/services/analytics_service.py`
* **Verification & Testing (`tests/test_comment_and_analytics_services.py`):**
  - Test replying to a comment calls the correct platform adapter based on the comment's `platform` field, saves the reply in DB, and sets `is_replied = True`.
  - Test that if the external API reply fails, the DB remains `is_replied = False` (atomic consistency).
  - Test `AnalyticsService` KPI math (summing across YT/FB/IG, handling division-by-zero when calculating growth percentage on empty history).
  - Run `pytest` and fix any calculation or state bugs until 100% pass.

#### [ ] Task 10: Cross-Platform Video Publish Pipeline Service
* **Goal:** Implement `PublishService` to handle multipart video saving, creating `PublishCampaign` & `PublishTask` records, uploading to R2 once (if FB or IG is selected), executing parallel uploads (`asyncio.gather`), linking completed uploads to `media_posts`, and cleaning up temp files.
* **Deliverables:**
  - `backend/app/services/publish_service.py`
* **Verification & Testing (`tests/test_publish_service.py`):**
  - Test campaign creation with 3 target platforms creates 1 Campaign and 3 `PENDING` Tasks.
  - Test background execution transitions tasks through `UPLOADING` -> `PUBLISHED`, creates 3 `media_posts` linked via the same `campaign_id`, and deletes the local temp file + R2 object.
  - **Partial Failure Test:** Simulate YouTube succeeding while Instagram fails. Verify YouTube task becomes `PUBLISHED`, Instagram task becomes `FAILED` with `error_message` recorded, and retry logic works for failed tasks only.
  - Run `pytest` and fix any async concurrency or file cleanup bugs until 100% pass.

---

### Phase 4: FastAPI REST Routers (API Layer)

#### [ ] Task 11: FastAPI Endpoints & Integration Tests
* **Goal:** Expose all services via clean REST endpoints in `app/routers/`:
  - `GET /api/dashboard/summary`, `GET /api/dashboard/posts`, `GET /api/dashboard/chart`
  - `GET /api/comments`, `POST /api/comments/{comment_id}/reply`
  - `POST /api/publish/upload`, `GET /api/publish/campaigns/{campaign_id}`, `POST /api/publish/tasks/{task_id}/retry`
  - `GET /api/accounts`, `POST /api/accounts/connect` (register/update personal OAuth tokens), `DELETE /api/accounts/{account_id}`, `POST /api/sync/trigger`
* **Deliverables:**
  - Router files wired into `app/main.py`.
* **Verification & Testing (`tests/test_api_endpoints.py`):**
  - Use FastAPI `TestClient` / `AsyncClient` to test every single endpoint.
  - Test HTTP status codes (200, 201, 202, 400, 404, 422).
  - Test multipart file upload endpoint with a dummy `.mp4` binary payload.
  - Run the entire backend `pytest` suite (`pytest -v`) and ensure **100% pass rate** across all backend tasks.

---

### Phase 5: Next.js Frontend Implementation

#### [ ] Task 12: Frontend Setup, Typed API Client & Layout Shell
* **Goal:** Initialize Next.js project in `frontend/`, configure Tailwind CSS + UI components, create TypeScript interfaces matching FastAPI schemas, and build `src/lib/api.ts` with error handling.
* **Deliverables:**
  - `frontend/src/types/api.ts`
  - `frontend/src/lib/api.ts`
  - Sidebar navigation layout (Dashboard, Unified Inbox, Cross-Post Studio, Connected Accounts) + Global "Sync Now" button.
* **Verification & Testing:**
  - Write unit tests for `api.ts` using `vitest` (mocking `fetch` responses and error states).
  - Run `npm run test` and `npm run build` to verify zero TypeScript compilation or linting errors.

#### [ ] Task 13: Unified Analytics Dashboard Page
* **Goal:** Build the main Dashboard (`src/app/page.tsx`) displaying:
  - Cross-platform KPI Summary Cards (Total Views, Followers, Likes).
  - Time-series Growth Chart (filterable by platform and date range).
  - Top Performing Videos Table + Cross-Posted Campaign Comparison view (comparing YT vs FB vs IG performance for the same video).
* **Verification & Testing:**
  - Write component tests (`vitest`) verifying KPI cards, platform filters, and empty-state rendering when no accounts are connected.
  - Run test suite and `npm run build` to fix any rendering or prop-type bugs.

#### [ ] Task 14: Unified Comment Inbox Page
* **Goal:** Build the Comment Inbox (`src/app/inbox/page.tsx`) featuring:
  - Filter tabs: "Unreplied (`is_replied=false`)" vs "All Comments", plus Platform filter (All, YT, FB, IG).
  - Comment list showing video thumbnail context, platform badge, author avatar, timestamp, and nested replies.
  - Inline Reply Box that calls `POST /api/comments/{id}/reply`, shows loading state, and updates the comment to "Replied".
* **Verification & Testing:**
  - Test filtering toggles, submitting an inline reply, and displaying error notifications if the backend reply fails.
  - Run `npm run test` and `npm run build` and fix any state management issues.

#### [ ] Task 15: Cross-Platform Video Publishing Studio & Account Settings
* **Goal:** Build the Video Upload Studio (`src/app/publish/page.tsx`) and Account Connection page (`src/app/accounts/page.tsx`):
  - Video file selector, Master Title/Caption inputs, and checkboxes to select target accounts (YouTube, FB, IG).
  - Real-time polling tracker displaying individual status badges (`PENDING`, `UPLOADING`, `PUBLISHED`, `FAILED` with retry button) for each platform.
  - Simple Account Management form to add/update connected accounts and view token health.
* **Verification & Testing:**
  - Test form validation (preventing submission without a video file or selected platform).
  - Test status polling UI transitions from `UPLOADING` to `PUBLISHED` or `FAILED`.
  - Run full Frontend (`npm run test` & `npm run build`) and Backend (`pytest`) test suites to verify the entire application is 100% green.