---
project: RAISE
status: CURRENT
last_verified: 2026-09-22
verification_method: code + browser
related:
  - knowledge/user-profile.md
  - knowledge/composio-capabilities.md
  - knowledge/hermes-tooling.md
  - project-notes/Raise-Video-Toolkit/README.md
scope: |
  RAISE architecture, workflow, API, browser usage guide, and browser-vs-code
  verification taxonomy. Verified from Kabirconnects/raise-app source code and
  project docs. Does NOT cover: live deployment status, API key configuration,
  current DB state, or live UI behavior (those require browser verification).
---

# RAISE — Operating Guide for Hermes

**Status:** RAISE is one of three core projects alongside MoneyPrinterTurbo and Nekonee.

**Do not assume this is current.** Verify before relying on it in a future session.

---

## 1. What RAISE Does

RAISE is an AI content intelligence platform that studies successful YouTube content and generates original, multi-platform content packages on the same topic in the user's own voice. [Verified from docs/introduction.md and docs/product-overview.md]

Core loop: **Research → Understand → Create**

- **Research:** Add YouTube channels, rank their videos by Outlier Score (how much a video's views-per-day outperformed the channel's historical baseline).
- **Understand:** When generating, RAISE fetches the source video's metadata + transcript and produces a 14-field Source Strategy Analysis deconstructing the video's hook style, storytelling structure, retention techniques, curiosity loops, CTA strategy, teaching framework, etc. [Verified from docs/analysis/strategy-analysis.md — 14 fields listed]
- **Create:** Generates a full content package including YouTube script (teleprompter format), YouTube package (hooks, titles, description, tags, chapters, thumbnail directions), short-form scripts (YouTube Shorts, TikTok, Instagram Reels, Facebook Reels, LinkedIn Video), social assets (Twitter/X thread, LinkedIn post, Instagram caption, Facebook post, community post), and written content (newsletter blurb, email version, blog outline). [Verified from docs/outputs/]

RAISE is **not** a chatbot, paraphrasing tool, transcript clipboard, video editor, or replacement for original research. It preserves the source's **topic** but rebuilds the communication through the user's own perspective. [Verified from docs/introduction.md]

---

## 2. Repository Architecture

**Owner/repo:** `Kabirconnects/raise-app` (private)
**Deployment:** Render.com — backend at `raise-backend-pymy.onrender.com`, frontend at `raise-7i17.onrender.com` [Verified from repo metadata + render.yaml]

### Directory structure (299 entries, 4 top-level dirs)

```
raise-app/
├── backend/
│   ├── aiModels/          # Gemini provider + fallback chain
│   ├── config/            # envVars.js, db.js
│   ├── controllers/       # API handlers (contentEditorController.js, etc.)
│   ├── middlewares/       # Auth, rate limiting, capacity controls
│   ├── models/            # Mongoose models (GenerationJob, Channel, User, etc.)
│   ├── routes/            # 18 Express route modules
│   ├── services/          # Generation queue/worker, memberships, PayPal
│   ├── utils/             # Content generation, YouTube API, transcript, SSE, etc.
│   └── server.js          # API startup
├── frontend/              # React + Vite SPA
│   ├── src/pages/         # 10+ pages (ContentEditor, GenerationHistory, Channel, etc.)
│   ├── src/api/           # Axios client
│   ├── src/components/    # UI components
│   └── src/hooks/         # TanStack Query hooks
├── docs/                  # Product docs, analysis docs, output specs, API reference
└── .github/               # CI workflow
```

### Key backend modules (verified from code)

| Module | Purpose | Verified details |
|---|---|---|
| `server.js` | Express API entry point | 18 route groups mounted under `/api`. Security headers (HSTS in production, X-Content-Type-Options, X-Frame-Options, Referrer-Policy, Permissions-Policy). CORS restricted to `FRONTEND_URL`. JSON body limit 1MB. General API rate limit: 300 req/15min. Cookie-based JWT auth. [Verified from server.js] |
| `aiModels/index.js` | AI model registry | Additive pattern: `providers` array. Currently only Gemini registered. All content pipeline calls go through `completeText()`, `completeRaw()`, `completeWithInfo()` which try configured providers in priority order with fallback. [Verified from index.js] |
| `aiModels/gemini.js` | Gemini provider | Uses `@google/genai` SDK. Primary model: `gemini-3.6-flash`. Fallback models: `gemini-3.5-flash`, `gemini-3.5-flash-lite`. Supports per-model fallback (429/503/timeout → secondary Gemini → registry fallback). JSON mode via `responseMimeType: "application/json"`. Lazy client creation. [Verified from gemini.js] |
| `aiModels/fallback.js` | Generic fallback chain | Tries candidates in priority order, skips unconfigured, returns aggregate error when all fail. [Verified from fallback.js] |
| `utils/contentGenerator.js` | Core content generation engine (111KB) | Generates: content strategy package, YouTube package, teleprompter script, social assets, scripts. Handles retries with exponential backoff + jitter, JSON mode, verification/correction passes, token budget scaling with video duration (140 wpm baseline). [Verified from contentGenerator.js — imports, function names, configuration constants] |
| `models/GenerationJob.js` | MongoDB job model | Fields: user, channel, videoId, payload, idempotencyKey, status (queued/processing/completed/failed), stage, result, error, startedAt, lastHeartbeatAt, workerId, queuedAt, completedAt. Indexed for queue processing. Results expire after 24h. [Verified from GenerationJob.js] |
| `services/generationQueue.js` | Queue management (12KB) | `enqueueGeneration()` with idempotency, membership plan checks, usage tracking. `drainGenerationQueue()` processes jobs up to `MAX_CONCURRENT_GENERATIONS` (default 3, clamp 1-10). Per-user limits: max 2 active, max 8 queued. Capacity bounded by LLM RPM quota, DB pool, server RAM. Process-local queue (not distributed across instances). [Verified from generationQueue.js] |
| `services/generationWorker.js` | Job execution (4.4KB) | `runGenerationJob()`: fetches channel, fetches source video details + transcript, optionally fetches up to 5 comparison videos, calls `generateContentStrategyPackage()`, builds language-aware brand voice, produces full result. [Verified from generationWorker.js] |
| `utils/youtubeApi.js` | YouTube Data API v3 client (11KB) | `fetchTopVideosList()` — velocity-based outlier scoring, 7-day recent window, filters Shorts (<180s). `fetchFullVideoDetails()`, `fetchChannelAnalytics()`, `fetchChannelInsights()` (upload cadence, engagement rate). Normalizes `@handle` / channel URL / channel ID. [Verified from youtubeApi.js] |
| `utils/transcriptApi.js` | Transcript fetching (409 bytes) | Thin wrapper around `youtube-transcript` package. Returns null if no transcript available. [Verified from transcriptApi.js] |
| `routes/contentEditorRoutes.js` | Content editor API routes | POST `/sessions`, GET `/sessions/:id`, POST `/sessions/:id/messages`, POST `/sessions/:id/messages/stream` (SSE), POST `/sessions/:id/apply`, POST `/sessions/:id/reset`. Auth + rate limiting (20 msg/10min) + MongoDB capacity middleware. [Verified from contentEditorRoutes.js] |
| `controllers/contentEditorController.js` | Content editor logic (22KB) | Edit sessions with revision-safe updates, apply/reset, brand context extraction, related content sync (when hook text changes, propagates to teleprompter script, short form, social posts). Monthly membership allowances enforced server-side. [Verified from contentEditorController.js — function names, imports, constants] |

### Frontend (inferred from structure)

React + Vite SPA. Pages include: Home, ContentEditor (`/content-editor/:generationId`), GenerationHistory, Channel, VideoDetail (`/channels/:channelId/videos/:videoId`), ChannelAnalytics, and others. Real-time SSE streaming from backend for generation progress. Built-in teleprompter view for scripts (auto-scroll, mirror mode, speed control). [Inferred from frontend/ directory structure + docs — individual component code not inspected]

### Deployment (verified from render.yaml)

Two Render services:
- **raise-backend:** Node.js, `npm start`, port 10000, free plan
- **raise-frontend:** Static, Vite build, `dist/` publish, free plan. `/api/*` requests rewritten to backend.

Secrets (MONGO_URI, JWT_SECRET, API keys, PayPal, Resend, Gemini keys) are dashboard-managed with `sync: false` — NOT in the repo.

CI: `.github/workflows/ci.yml` — Node 20.19.0, runs `npm ci && npm test && npm audit` for backend, `npm ci && npm run lint && npm run build && npm audit` for frontend.

---

## 3. RAISE Workflow (verified from code)

### End-to-end flow

```
Add Channels (POST /api/channels with @handle or channel URL)
    ↓
Rank Videos (GET /api/channels/:id/videos — Outlier Score ranking)
    ↓
Open Video + Set Brand Inputs (expertise, offer, positioning, business goals,
    brand voice, content pillars, ICP, target duration)
    ↓
Generate (POST /api/channels/:id/videos/:videoId/generate)
    ↓
Create MongoDB GenerationJob (queued)
    ↓
Worker picks up job → drainGenerationQueue()
    ↓
Fetch source: fetchFullVideoDetails(videoId) + fetchVideoTranscript(videoId)
    ↓
Optional: fetch up to 5 comparison videos
    ↓
generateContentStrategyPackage() → Gemini provider (with fallback chain)
    ↓
Validation: schema checks + semantic verification + bounded correction passes
    ↓
Save result to GenerationJob.result
    ↓
Client polls GET /api/generation-history/:id until completed
    ↓
Review outputs in UI / Content Editor
```

### Generation job lifecycle (verified from generationQueue.js + generationWorker.js)

1. **Enqueue:** `enqueueGeneration()` validates payload (≤256KB), derives idempotency key (SHA-256 of user+channel+videoId+payload), checks membership plan limits and per-user queue limits, creates `GenerationJob` with status `queued`.
2. **Drain:** `drainGenerationQueue()` finds oldest queued job, sets status to `processing`, assigns `workerId`, calls `processJob()`.
3. **Process:** `processJob()` runs `runGenerationJob()` with heartbeat timer (30s interval). On success: status `completed`, result saved. On failure: status `failed`, error saved.
4. **Stale recovery:** `reclaimStaleProcessingJobs()` requeues jobs whose worker missed heartbeat (3min timeout). `cleanupStaleQueuedJobs()` fails jobs queued >24h.

### Content Editor flow (verified from contentEditorRoutes.js + contentEditorController.js)

1. `POST /api/content-editor/sessions` — create/edit session for a generation
2. `POST /api/content-editor/sessions/:id/messages` — send edit instruction (rate limited: 20/10min, MongoDB capacity enforced)
3. `POST /api/content-editor/sessions/:id/messages/stream` — SSE variant for real-time editing
4. `POST /api/content-editor/sessions/:id/apply` — apply accumulated edits
5. `POST /api/content-editor/sessions/:id/reset` — reset to original

---

## 4. Important UI Workflows (inferred from frontend structure + docs)

### Channel management
- Add channel by `@handle` or YouTube channel URL
- View channel's ranked videos (Outlier Score)
- Declare one channel as "own" for benchmarking
- Compare own channel against one competitor at a time

### Video research
- Open a video to see details, stats, transcript status
- Set brand inputs on the video page before generating
- Brand inputs: expertise, offer, positioning, business goals, brand voice, content pillars, ICP, target duration

### Generation
- Click **Generate** → async job starts → UI polls until complete
- Results panel shows: Source Strategy Analysis (14 fields), Knowledge Extraction, YouTube package, teleprompter script, short-form scripts, social assets, written content
- Open teleprompter view for script reading (auto-scroll, mirror mode, speed control)

### Content Editor
- Access at `/content-editor/:generationId`
- Send edit instructions in natural language
- Apply or reset edits
- Revision-safe: conflicts detected if edited in another tab

---

## 5. Important Backend/API Workflows (verified from code)

### Authentication
Passwordless OTP flow:
1. `POST /api/users/send-otp` — sends 6-digit OTP (5 req/IP/15min)
2. `POST /api/users/verify-otp` — verifies, sets JWT cookie (10 req/IP/15min)
3. `GET /api/users/current-user` — returns profile
4. `POST /api/users/logout` — clears cookie

### Key API endpoints (all under `/api`)

| Route group | Notable endpoints | Notes |
|---|---|---|
| `/api/health` | `GET` → `{status: "ok"}` | Skipped by rate limiter |
| `/api/users` | send-otp, verify-otp, current-user, logout | OTP auth |
| `/api/channels` | CRUD, bulk add, list videos, generate | Channel allowance checked. Generation requires plan allowance + 10/hr rate limit. |
| `/api/channels/extras` | analytics, declare-own, compare | One own channel + one competitor at a time |
| `/api/generation-history` | list, get, delete | Scoped to current user |
| `/api/content-editor` | sessions CRUD, messages, apply, reset, SSE | Auth + rate limit + MongoDB capacity |
| `/api/paypal` | webhook, plans, subscribe, my-plan-access, cancel | Unauthenticated webhook; rest authenticated |
| `/api/admin` | overview, membership-plans, users, issues, support tickets | Requires `requireOwner` + auth |
| `/api/growth` | `GET /stats` | Public aggregate stats |

### Generation request/response

**Request:** `POST /api/channels/:id/videos/:videoId/generate` with body:
```json
{
  "targetDurationMinutes": 10,
  "compareVideoIds": ["...", "..."],  // optional, max 5
  // brand inputs from video page
}
```

**Response:** Returns the `GenerationJob` object. Client polls `GET /api/generation-history/:id` until `status: "completed"`.

**Result structure** (from generationWorker.js + docs):
- `contentArchetype` — type of content
- `sourceStrategyAnalysis` — 14-field analysis
- `knowledgeExtraction` — tools, platforms, concepts extracted
- `youtube` — hooks (7 options + selected), hookAnalysis, seoTitles (3), description, chapters, tags, thumbnailIdeas, visualDirections
- `teleprompterScript` — full long-form script in timestamped format
- `shortForm` — YouTube Shorts, TikTok, Instagram Reels, Facebook Reels, LinkedIn Video hook
- `social` — Twitter/X thread, LinkedIn post, Instagram caption, Facebook post, community post
- `written` — newsletter blurb, email version, blog outline
- `sourcesUsed` — list of source videos used

---

## 6. Browser Operating Guide

### 6.1 Pre-flight checklist

Before operating RAISE in the browser, confirm:

1. **Frontend reachable:** `https://raise-7i17.onrender.com/` loads and shows the RAISE homepage.
2. **Backend reachable:** `https://raise-backend-pymy.onrender.com/api/health` returns `{status: "ok"}` (or the frontend loads without API errors).
3. **Authenticated:** You are signed in. If not, complete OTP sign-in before any channel operations.
4. **On the correct page:** Navigate to `/channel` (Studio) for channel operations.
5. **Browser session stable:** No console errors that would interfere with UI interaction.
6. **Plan allowance confirmed (if doing generations):** Check the "GENERATIONS LEFT" / "EDITS LEFT" counters on the Studio page.

If any pre-flight check fails, do not proceed. Report the failure.

### 6.2 Browser procedures

This section gives a brief overview. For the full step-by-step browser procedures —
navigating pages, filling forms, submitting generations, handling the Generation
History SPA, interaction methods, and failure recovery — see
`project-notes/RAISE/browser-how-to.md`.

Brief summary of what is covered in the separate file:

- **Channel management** (§6 of browser-how-to.md): adding channels, reliable click
  methods, verification, failure recovery. The "+ ADD A CHANNEL" button requires CDP
  mouse events; the "ADD CHANNEL" submit button requires querying current coordinates.
- **Video research** (§7): opening video detail pages from the Studio list.
- **Generation input fields** (§9): all 10 fields with labels, types, placeholders,
  widths, and the verified `fill_input()` interaction method.
- **Submitting a generation** (§10): scroll requirement, JS `.click()` works reliably
  for both generate buttons, button-disable is the start signal.
- **Generation status** (§12): poll the API `GET /api/generation-history/:id` — do not
  wait for UI feedback. Status flow: queued → processing → completed.
- **Accessing results** (§13): API returns the full result object; `teleprompterScript`
  is null when the source has no transcript (expected, not an error).
- **Generation History SPA** (§8): React SPA may show only the shell on load; API is
  more reliable.
- **Interaction methods summary** (§17): which buttons work with JS `.click()`, which
  require CDP, when to scroll, when to use the API.
- **Common failures and fixes** (§15): table of observed failures and their known fixes.
- **Verification steps** (§16): what to check after each action.

`VERIFIED` — channel-add workflow browser-verified 2026-09-22; generation workflow
browser-verified 2026-09-22.

---

## 7. Browser vs. Code Verification — Operating Distinction

This README draws on **two distinct verification methods**. They are complementary, not interchangeable, and Hermes must know which method produced any given fact before relying on it.

### 7.1 Code verification (READ-ONLY inspection of `Kabirconnects/raise-app`)

**What it is:** Reading the source files in the `Kabirconnects/raise-app` repository — via `gh api`, `gh repo clone`, or comparable READ-ONLY access — and extracting verified facts from the actual code, config, and docs in that repo.

**What it verifies:**
- Backend architecture, route structure, AI model registry, generation queue/worker flow, content editor API, YouTube API client, transcript fetching, deployment config (`render.yaml`), CI workflow, output format specs, doc files.
- Module purpose, function names, configuration constants, field definitions, rate limits, auth flow structure.

**What it does NOT verify:**
- Whether the deployed application is currently live or functional.
- Whether API keys (Gemini, YouTube, etc.) are currently configured and working.
- Whether the database is currently connected.
- Actual UI layout, button labels, form behavior, page URLs as seen by a user.
- Real-world generation latency or current system load.

**Constraints:**
- `Kabirconnects/raise-app` is **READ-ONLY** during code verification. Hermes inspects files but does not modify, commit, push, branch, or change deployment/config/secrets in that repository.
- Code verification can be performed regardless of whether the app is deployed or reachable.

**Tagging convention in this README:** Facts verified by reading raise-app source are tagged `[Verified from <file>]` with the specific file(s) inspected. Facts inferred from directory structure without reading the individual file are tagged `[Inferred from <dir> structure + docs]`.

### 7.2 Browser verification (operating the deployed application)

**What it is:** Opening the deployed RAISE application in a browser — currently `https://raise-7i17.onrender.com/` for the frontend and `https://raise-backend-pymy.onrender.com/api` for the backend — and operating it as a normal user would: signing in, adding channels, browsing videos, setting brand inputs, starting generations, reviewing outputs, using the Content Editor.

**What it verifies:**
- That the deployed URLs are reachable and currently serving the expected application.
- The current UI: page layouts, button labels, form fields, navigation, available controls and options.
- The live auth flow (does OTP sign-in work with the current configuration?).
- Real-world generation speed and latency.
- Whether the Content Editor real-time editing experience works as expected.
- Whether the live UI matches (or has diverged from) what the code and docs describe.

**What it does NOT verify:**
- Source code details that are not surfaced in the UI (internal module structure, queue internals, model fallback chains, middleware behavior beyond what error messages reveal).
- Deployment config, CI workflow, or repo metadata.
- Anything that requires reading raise-app source files.

**Constraints:**
- Browser verification operates the **live application as a user**. It does **not** grant permission to modify the `Kabirconnects/raise-app` repository, its source code, deployment, GitHub settings, or configuration.
- Consequential actions in the browser (starting a generation that consumes quota, editing content) are subject to explicit approval.
- Browser verification requires the app to be deployed and reachable. If the app is down, browser verification cannot proceed and code verification remains the available method.

### 7.3 Verification methods — clear distinctions

This README draws on multiple verification methods. They are NOT interchangeable. An agent must know which method produced a fact before relying on it.

#### 7.3.1 Source-code verification (READ-ONLY inspection of `Kabirconnects/raise-app`)

**What it is:** Reading source files in the `Kabirconnects/raise-app` repository — via `gh api`, `gh repo clone`, or comparable READ-ONLY access — and extracting verified facts from the actual code, config, and docs.

**Verifies:** Backend architecture, route structure, AI model registry, generation queue/worker flow, content editor API, YouTube API client, transcript fetching, deployment config (`render.yaml`), CI workflow, output format specs, module purpose, function names, configuration constants, field definitions, rate limits, auth flow structure.

**Does NOT verify:** Whether the deployed app is live/functional. Whether API keys are configured. Whether the database is connected. Actual UI layout, button labels, form behavior, page URLs as seen by a user. Real-world generation latency.

**Tagging:** `[Verified from <file>]` — fact read directly from a raise-app source file.

#### 7.3.2 Browser/UI verification (operating the deployed application)

**What it is:** Opening the deployed RAISE application in a browser and operating it as a normal user would: signing in, adding channels, browsing videos, setting brand inputs, starting generations, reviewing outputs, using the Content Editor.

**Verifies:** That deployed URLs are reachable and serving the expected application. Current UI: page layouts, button labels, form fields, navigation, available controls. Live auth flow. Real-world generation speed and latency. Whether the live UI matches (or has diverged from) what code and docs describe.

**Does NOT verify:** Source code details not surfaced in the UI (internal module structure, queue internals, model fallback chains, middleware behavior beyond error messages). Deployment config, CI workflow, or repo metadata. Anything requiring raise-app source files.

**Tagging:** "browser-verified" with the date — e.g. "browser-verified 2026-09-22".

#### 7.3.3 API verification (observing network calls from the browser)

**What it is:** Observing the actual HTTP requests and responses the frontend makes to the backend during UI interaction. This can be done by inspecting network activity in the browser's developer tools or by monitoring the CDP network events.

**Verifies:** That specific API endpoints are reachable, accept the expected request shapes, and return the expected response shapes during live operation. It confirms the API behaves as the frontend expects. It does NOT require reading raise-app source code.

**Does NOT verify:** Source code implementation details. Correctness of the backend logic beyond what the response shape reveals. Deployment config or CI.

**When to use:** When you need to confirm an API endpoint works end-to-end without reading source code, or when browser UI behavior is ambiguous and you want to see whether the request actually reached the server and what it returned.

**Limitations:** API verification confirms the request/response cycle works; it does not confirm the source code is correct, only that the observed behavior matches expectations.

#### 7.3.4 Unverifiable assumptions

Some things cannot be verified without the live app AND additional access that may not be available:

- Whether the Gemini API key is currently configured and working — requires either a generation attempt or source-code/config inspection.
- Whether the YouTube API key is currently configured and working — requires either a channel add/video fetch attempt or source-code/config inspection.
- Current database connection status — requires server access or a live health check that exposes DB status.
- Any recent changes to the codebase not reflected in the inspected commit.
- Exact behavior of features not yet exercised in the browser (e.g. comparison video feature, content editor real-time editing).

**Rule:** Do not assert these as facts. Flag them as "unknown" or "not yet verified" and confirm via the appropriate method when needed.

### 7.4 How to read the tags in this README

| Tag | Meaning |
|---|---|
| `[Verified from <file>]` | Fact read directly from a raise-app source file via code verification. |
| `[Verified from repo metadata + <file>]` | Fact from repo-level metadata (description, homepage) plus a specific file. |
| `[Inferred from <dir> structure + docs]` | Fact deduced from directory layout and doc files without reading the individual source file. Lower confidence. |
| `[Verified from render.yaml]` | Fact read from the deployment config file. |
| "browser-verified" + date | Fact confirmed by operating the live application in the browser. |
| "needs browser confirmation" | Fact plausible from code/docs but not checked against the live UI. |

### 7.5 When the two methods disagree

If browser verification reveals a meaningful difference from what code verification or this README says:
- Prefer the **current live UI** for any action being taken now.
- Report the difference.
- When appropriate, update this README with the newly browser-verified information, clearly tagging it as browser-verified rather than code-verified.

### 7.6 Reusable task template — browser channel add

Use this template when adding YouTube channels to RAISE via browser automation. Follow every step in order. Do not skip verification.

```
TASK: Add N YouTube channels to RAISE
Channels: [@handle1, @handle2, ...]
Session: <date>

PRE-FLIGHT:
1. Confirm frontend reachable: https://raise-7i17.onrender.com/ loads
2. Confirm authenticated (or complete OTP sign-in)
3. Navigate to /channel
4. Note existing channels in the list (to detect duplicates)

FOR EACH channel handle H:
1. Check if H (uppercase) is already in the channel list. If yes, skip and note "already present".
2. Click "+ ADD A CHANNEL" button:
   - Use CDP Input.dispatchMouseEvent at the button's current coordinates
   - This button's React event handling did NOT respond reliably to click_at_xy during browser verification (see Section 6.3)
   - Query coordinates from the live page every time
3. Confirm modal opened ("TRACK A CREATOR" / "ADD YOUTUBER" visible)
4. Fill input (id: channel-handle) with H using fill_input()
5. Click "ADD CHANNEL" button (type="submit"):
   - Query current button coordinates first
   - Use click_at_xy or CDP mouse click at those coordinates
6. Wait for modal to close and page to settle
7. VERIFY: search page text for H in uppercase
   - If found as a channel entry (with "REMOVE" and video list or status): SUCCESS
   - If "Couldn't load videos" shown: SUCCESS (video fetch failed, not add)
   - If NOT found: FAILED — do not count. Investigate and retry.
8. Record result: SUCCESS or FAILED with reason

POST-TASK:
- Count only verified SUCCESS channels toward the target
- Report: channels added, channels skipped (already present), channels failed
- Update the verification log below if this is a new verified workflow
```

### 7.7 Rule: only document as "known-good" after verification

A workflow, interaction method, or UI behavior should only be documented in this README as **verified** or **known-good** after it has actually been exercised successfully in the browser (or confirmed via source-code inspection, as appropriate to the claim).

- **"Tested and worked"** — you ran the workflow and it succeeded.
- **"Verified from source"** — you read the source code and confirmed the behavior.
- **"Inferred"** — you deduced it from structure without direct confirmation. Label it as lower confidence.
- **"Not yet tested"** — you have not confirmed it. Do not present it as known-good.

Do not promote an inferred or untested observation to "verified" status without actual confirmation. When in doubt, label it as "needs verification" and verify before relying on it.

---

## 8. Integration Potential with Raise-Video-Toolkit

RAISE produces structured outputs that could feed into Raise-Video-Toolkit for automated video production. **Do not implement this integration yet — document only verified potential.**

### Verified RAISE outputs that could pass to RVT

| RAISE output | RVT use case | Status |
|---|---|---|
| `teleprompterScript` | Script for video production/narration | Verified format: timestamped sections with visual cues + spoken dialogue |
| `youtube.seoTitles` | Video title options | Verified: 3 titles |
| `youtube.description` | Video description for upload | Verified |
| `youtube.tags` | Video tags | Verified |
| `youtube.chapters` | Video chapters | Verified: timestamped |
| `youtube.thumbnailIdeas` | Thumbnail design brief | Verified: text concepts only — need design step |
| `youtube.visualDirections` | B-roll/visual plan | Verified: text guidance |
| `shortForm` (Shorts, TikTok, Reels scripts) | Short-form video scripts | Verified: platform-specific scripts |
| `social` (Twitter thread, LinkedIn, etc.) | Social media publishing | Verified: platform-native posts |
| `sourceStrategyAnalysis` | Content strategy context for RVT decision-making | Verified: 14-field breakdown |
| `knowledgeExtraction` | Research/context for content planning | Verified: tools, platforms, concepts |

### What RAISE does NOT produce (would need additional work for RVT)
- Actual thumbnail images (text concepts only)
- Video editing instructions beyond visual directions text
- Audio/production specs
- Scheduled publishing commands

---

## 9. Hermes Operating Rules

- **RAISE repository = READ-ONLY** unless explicitly authorized. No modifying code, creating files, committing, pushing, branching, or changing deployment.
- **Hermes workspace** (`Kabirconnects/hermes/project-notes/RAISE/`) is where reusable RAISE documentation belongs.
- **Browser use** = operate the deployed application as a user. Subject to explicit approval for consequential actions.
- **Never expose secrets** — no API keys, tokens, passwords, cookies, credentials, or account identifiers.
- **Verify before acting** — check this document for the verified baseline, then inspect raise-app directly if needed.
- **Update when outdated** — when new verified facts are discovered, update this file rather than creating scattered notes.
- **Know which verification method applies** — before relying on any fact in this README, check whether it was code-verified, browser-verified, inferred, or unverified, and choose the appropriate live check for the task at hand.

---

## 10. Known Limitations / Unknowns

### Verified from code
- Backend architecture, route structure, AI model registry, generation queue/worker flow, content editor API, YouTube API client, transcript fetching, deployment config, output formats
- All 18 backend route groups and their purposes
- Generation job lifecycle (enqueue → queue → worker → result)
- Content Editor SSE flow and revision-safe editing
- Membership plans (Free/Creator/Power) and their limits

### Inferred from structure (not individually inspected)
- Exact React component implementations in `frontend/src/components/`
- Individual hook implementations in `frontend/src/hooks/`
- Frontend API client details in `frontend/src/api/api.js`
- Specific UI behavior of each page beyond what docs describe
- Middleware implementations (auth, rate limiting, capacity) beyond what routes reference
- Email/notification utilities, caching layer

### Not yet browser-tested
- Deployed application URLs — **now browser-verified**: `raise-7i17.onrender.com` is live and functional
- Actual UI behavior, page layouts, form interactions — **partially browser-verified**: channel add workflow tested; generation workflow fully tested (see browser-how-to.md §10–§13); other workflows (sign-in, video ranking, content editor) not yet tested
- Generation speed and real-world latency — **browser-verified 2026-09-22**: ~2.5 min from queue to complete for a single-video generation with no transcript (see browser-how-to.md §12.3). Single observed data point; actual times will vary.
- Content Editor real-time editing experience — not yet tested
- Whether OTP auth works with current phone/email configuration — not yet tested (session was already authenticated)

### Unknown
- Current deployment status — **now verified**: the app is live and functional at `raise-7i17.onrender.com`
- Whether Gemini API key is currently configured and working
- Whether YouTube API key is currently configured and working
- Current database connection status
- Any recent changes to the codebase not reflected in the inspected commit
- Exact behavior of comparison video feature (fetched but not individually traced)

---

## How to Use This File

Before working on RAISE tasks, check this document for the verified baseline. Note which verification method (code, browser, or API) produced each fact you plan to rely on, and perform the matching live check when the task calls for it.

**Section guide:**
- Sections 1-5: Architecture, workflow, and API — verified from source code. Use for understanding the system.
- Section 6: Browser operating guide — the practical reference for any browser automation task. Read this first when operating RAISE in the browser.
- Section 7: Verification taxonomy, task template, and documentation rules — use to understand what is verified and how to document new findings.
- Sections 8-10: Integration potential, operating rules, and known limitations.
- Section 11: Verification log — record of browser-verified workflows and lessons learned.

When you discover new verified facts about RAISE's architecture or workflow, update this file rather than creating scattered notes. When something here is no longer accurate, correct it.

**Documentation rule:** Only document a workflow or interaction method as "known-good" or "verified" after you have actually confirmed it works (via browser or source inspection, as appropriate). Do not promote inferred observations to verified status without confirmation.

---

## 11. Browser Verification Log

### 2026-09-22: Channel-add workflow tested

**Task:** Add 15 English-language YouTube channels to RAISE via browser UI (batch 1)

**Result:** SUCCESS — all 15 channels added and verified

**Channels added:**

|| # | @handle | Channel name | Notes |
||---|---|---|---|
|| 1 | @sandyleeai | Sandy Lee AI | Reference channel; top 6.7x |
|| 2 | @AIJason | AI Jason | No videos in last 7 days |
|| 3 | @mattwolfe | Matt Wolfe | Could not fetch videos |
|| 4 | @matthewberman | Matthew Berman | top 1.3x |
|| 5 | @upgradeduser | Upgraded User | Could not fetch videos |
|| 6 | @aiadvantage | The AI Advantage | top 3.2x |
|| 7 | @AllAboutAI | All About AI | top 9.7x |
|| 8 | @worldofai | World of AI | Could not fetch videos |
|| 9 | @ben_aime | Ben Afriat | Could not fetch videos |
|| 10 | @TwoMinutePapers | Two Minute Papers | top 13.4x |
|| 11 | @promptengineering | Prompt Engineering | top 2.5x (older content) |
|| 12 | @FutureTools | FutureTools | Could not fetch videos |
|| 13 | @aisupremacy | AI Supremacy | No videos in last 7 days |
|| 14 | @DavidSh0ck | David Shock | Could not fetch videos |
|| 15 | @ANgrypug | Angrypug | top 4.9x (heavy content) |

**Pre-existing channel:** @HIMANSHUARODIA (the user's own channel)

**Total after batch 1:** 16 channels (15 new + 1 pre-existing)

---

### 2026-09-22: Channel-add workflow tested (batch 2)

**Task:** Add 7 additional English-language YouTube channels to RAISE via browser UI

**Result:** SUCCESS — all 7 channels added and verified

**Channels added:**

|| # | @handle | Channel name | Notes |
||---|---|---|---|
|| 16 | @LiamOttley | Liam Ottley | AI automation agencies; top 14.3x |
|| 17 | @nicksaraev | Nick Saraev | Practical AI automation / n8n; top 7.4x |
|| 18 | @jonocatliff | Jono Catliff | AI automation and implementation; top 4.7x |
|| 19 | @GregIsenberg | Greg Isenberg | AI startups, business ideas; top 17.9x |
|| 20 | @SkillLeapAI | SkillLeap AI | Practical AI productivity; top 6.4x |
|| 21 | @AIJasonZ | AI Jason Z | Building AI apps, agents; top 41.4x |
|| 22 | @WesRoth | Wes Roth | AI developments, models; top 13.7x |

**Duplicate/skipped candidates (already in RAISE):**
- @aiadvantage — already added in batch 1 (#6)
- @matthew_berman — already added as @matthewberman in batch 1 (#4)
- @mreflow — described as "Matt Wolfe"; already added as @mattwolfe in batch 1 (#3)

**Total after batch 2:** 23 channels (22 new + 1 pre-existing @HIMANSHUARODIA)

---

### 2026-09-22: Channel-add workflow tested (batch 3)

**Task:** Add 6 additional English-language YouTube channels to RAISE via browser UI

**Result:** SUCCESS — all 6 channels added and verified

**Channels added:**

|| # | @handle | Channel name | Notes |
||---|---|---|---|
|| 23 | @nateherk | Nate Herk | AI automation, business systems |
|| 24 | @sabrina_ramonov | Sabrina Ramonov | AI workflows, tutorials, tech |
|| 25 | @TechWithTim | Tech With Tim | Python/AI coding tutorials |
|| 26 | @TinaHuang1 | Tina Huang | AI tools, tech, productivity |
|| 27 | @patrickdang | Patrick Dang | AI, tech, business content |
|| 28 | @AlexHormozi | Alex Hormozi | AI business, entrepreneurship, scaling |

**Total after batch 3:** 29 channels (28 new + 1 pre-existing @HIMANSHUARODIA)

---

### 2026-09-22: Channel-add workflow tested (batch 4 — test only)

**Task:** Add 10 additional AI business-related YouTube channels to RAISE via browser UI (test batch)

**Result:** SUCCESS — all 10 channels added and verified

**Channels added:**

|| # | @handle | Channel name | Notes |
||---|---|---|---|
|| 29 | @TechWithAI | Tech With AI | AI news, tools, tutorials |
|| 30 | @Murtaza_School | Murtaza's School | AI tutorials, tech education |
|| 31 | @keepcoding | Keep Coding | AI/software development tutorials |
|| 32 | @aiperspectives | AI Perspectives | AI insights, tech commentary |
|| 33 | @BoxOfA.I. | Box of A.I. | AI tools, tips, practical guides |
|| 34 | @MakeMoneyWithAI | Make Money With AI | AI monetization, business |
|| 35 | @AIwithZ | AI With Z | AI tutorials, tools, workflows |
|| 36 | @DavidSharpe | David Sharpe | AI business, marketing, coaching |
|| 37 | @simonsmauger | Simon Smauger | AI, tech, business |
|| 38 | @TheAIEdge | The AI Edge | AI tools, business advantage |

**Total after batch 4:** 39 channels (38 new + 1 pre-existing @HIMANSHUARODIA)

**Note:** This batch was added for testing purposes only. All channels verified in the live UI.

**Browser-automation lessons learned:**

1. **Use CDP for reliable button clicks.** The "+ ADD A CHANNEL" button requires clicking via CDP `Input.dispatchMouseEvent` at exact coordinates. JavaScript `click()` on the DOM element did not work reliably. The button's position is at viewport coordinates approximately (56, 633) with size 233x56px.

2. **Use `fill_input()` for form input.** The channel handle input field (id: `channel-handle`) accepts text via the `fill_input()` helper. Direct CDP keyboard events also work but `fill_input()` is simpler.

3. **The form closes after each addition.** After clicking "ADD CHANNEL", the modal closes. To add another channel, click "+ ADD A CHANNEL" again to reopen the form.

4. **Visual verification is essential.** After each add, verify the channel appears in the list. The handle is displayed in UPPERCASE. Some channels show "ADDING..." or "SCORING VIDEOS AGAINST CHANNEL AVERAGE..." during processing.

5. **"Couldn't load videos" ≠ failed add.** Some channels show "Couldn't load videos: Could not fetch videos for this channel right now" with a "TRY AGAIN" button. This is a video-fetching issue, NOT a channel-add failure. The channel IS in the list.

6. **Channel handles are case-insensitive but displayed uppercase.** Entering `@sandyleeai` results in `@SANDYLEEAI` appearing in the list.

7. **The ADD CHANNEL button position varies.** The submit button's coordinates change slightly between form openings. Always query the current button position via CDP before clicking rather than using hardcoded coordinates.

8. **Interaction method hierarchy.** Use `fill_input()` for text inputs. Use CDP mouse click for the "+ ADD A CHANNEL" button (JavaScript `.click()` on this DOM element did not work reliably). For the "ADD CHANNEL" submit button, query its current coordinates and click at those coordinates. See Section 6.3 for the full reliable interaction methods and Section 6.4 for anti-patterns to avoid.

**Verification method:** Browser verification (operating the live application at `https://raise-7i17.onrender.com/channel`)

**Status:** Verified 2026-09-22
