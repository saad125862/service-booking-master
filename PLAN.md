# Seba (সেবা): Service Booking & Management App: Implementation Plan

**App name:** Seba (সেবা, "service"). Logo: orange rounded square with a white circle above a cupped bowl. Repo names stay descriptive (`service-booking-*`).

**Stack (decided):** Flutter + Riverpod · Node.js/Express · PostgreSQL · Gemini
**Deadline:** 72 hours · **Status:** plan only, no code written yet

---

## 0. Environment check (this machine)

| Tool | Status |
|---|---|
| Flutter 3.47.9 / Dart 3.13.7 | OK, `flutter doctor` reports no issues |
| Android SDK 37 | OK (Flutter's toolchain check passes) |
| Node 24 / npm 11 | OK |
| Git 2.56 | OK |
| Docker 29 | OK, use it to run local PostgreSQL (no local `psql`) |
| Working folder `E:\FB International Task` | Empty, not a git repo |

**Needs you (account or credential steps I should not do for you):**
1. Create two empty GitHub repos: `service-booking-backend`, `service-booking-app`.
2. Get a Gemini API key from Google AI Studio.
3. Create free accounts on **Neon** (PostgreSQL) and **Render** (Node hosting).

---

## 1. Key technical decisions (with interview justifications)

| # | Decision | Why |
|---|---|---|
| 1 | **PostgreSQL + Prisma 7 (client, `@prisma/adapter-pg`, Prisma Migrate)** *(changed from raw SQL at the user's request)* | Models and queries go through Prisma. Prisma's schema language can't express the *no overlapping bookings* GiST exclusion constraints or CHECKs, so they are hand-written SQL appended to the init migration. The database still enforces conflicts, and the app maps the violation to `409`. |
| 1b | **Swagger UI at `/api/docs`** from a hand-written `docs/openapi.yaml` | Interactive API docs with a Bearer "Authorize" button; doubles as the API documentation deliverable. |
| 1c | **Multilingual AI assistant: English, Bangla (বাংলা) and Banglish** | The assistant detects the language of each user message and answers in the same one (Banglish = Bangla in Latin letters). It understands all three for intent and recommendations. |
| 2 | **Exclusion constraint for conflicts** (`btree_gist`) | Race-condition-proof. Two simultaneous requests can't both succeed, because the DB rejects the second. App-level checks alone would be racy. |
| 3 | **Single JWT access token (7-day expiry)**, stored in secure storage | Simple and sufficient. Refresh tokens are a documented stretch goal, with the trade-off stated in the README. |
| 4 | **Riverpod without code generation** (`Notifier` / `AsyncNotifier`) | Avoids the `build_runner` build step and its failure modes in a 72-hour window, and the code stays easy to explain. |
| 5 | **Feature-first Flutter structure with a repository layer** | Separation of concerns: UI → providers → repositories → API client. |
| 6 | **`go_router` + `dio` + `flutter_secure_storage`** | Declarative auth-aware redirects. Interceptors for the token and 401 handling. Encrypted token storage. |
| 7 | **Gemini called only from the backend**, structured JSON output | The key never reaches the app. The app gets `reply` + `recommendedServiceIds` and can render tappable service cards. |
| 8 | **Zod validation, helmet, CORS, rate limiting, bcrypt** | Input validation and standard hardening. The rate limit is stricter on `/ai/chat`, which costs money. |
| 9 | **Docker locally; Render (API) + Neon (DB) live** | One `Dockerfile` serves both, with local Postgres in Compose and managed Postgres on Neon. Free tier, GitHub deploys, HTTPS. Neon supports `btree_gist`, **to be verified in phase 3**. See section 1b. |
| 10 | **OpenAPI (Swagger UI at `/api/docs`) + README tables** | Covers the "API documentation" deliverable and gives the reviewer something to try. |

---

## 1b. Environments: local (Docker) vs live (Render + Neon)

| | **Local (Docker)** | **Live** |
|---|---|---|
| API | `api` container built from the repo's `Dockerfile` | **Render** web service, built from the **same `Dockerfile`** |
| Database | `postgres:16` container with a named volume | **Neon** PostgreSQL |
| Config | `.env` (git-ignored), loaded by `docker-compose` | Render environment variables |
| Migrations | Run on API container start (`npm run migrate`) | Run as a Render pre-deploy command, against Neon's **direct** (non-pooled) connection string |
| Seed | `npm run seed`, auto on first local start | Run once manually against Neon |
| DB SSL | Off | **On** (`sslmode=require`), handled by `DATABASE_SSL=true` in config |
| Flutter `API_BASE_URL` | Emulator: `http://10.0.2.2:3000/api/v1`; real device: `adb reverse tcp:3000 tcp:3000` then `http://localhost:3000/api/v1` | `https://<service>.onrender.com/api/v1` (used for the release APK) |

**Why the same Dockerfile for both:** local and live run the identical image, so "works on my machine" bugs are minimized, and it's a strong, explainable decision.

**Backend repo additions**
- `Dockerfile` (multi-stage, Node 24-slim, non-root user, production deps only)
- `docker-compose.yml` with services `db` (healthcheck) and `api` (`depends_on: db: condition: service_healthy`, live-reload mount for dev)
- `.dockerignore`, `.env.example` (local values) and `.env.production.example` (Render and Neon variable names, no values)
- `render.yaml` (optional blueprint) so the deploy config is versioned

**Env vars:** `NODE_ENV`, `PORT`, `DATABASE_URL`, `DATABASE_SSL`, `JWT_SECRET`, `JWT_EXPIRES_IN`, `GEMINI_API_KEY`, `GEMINI_MODEL`, `ADMIN_API_KEY`, `BUSINESS_TZ`, `CORS_ORIGINS`. All are validated by zod at boot, so the app refuses to start with a bad config.

**Deployment gotchas to handle**
- Neon provides pooled and direct connection strings. Use **direct for migrations** and **pooled for the app**.
- Neon autosuspends idle compute, and Render's free tier sleeps. The first request after idle can be slow, so the app needs a loading state and the README needs a note. I'll warm both up before recording the demo.
- Render injects `PORT`, so the server must listen on `process.env.PORT`.
- Verify `CREATE EXTENSION btree_gist` and `citext` succeed on Neon in phase 3.

---

## 2. Repositories and structure

Two repos, created as sibling folders inside `E:\FB International Task\`.

### Backend: `service-booking-backend`
```
src/
  config/        env.js (zod-validated env), constants.js (working hours, slot rules)
  db/            pool.js, migrations/, seeds/
  middleware/    auth.js, validate.js, errorHandler.js, rateLimit.js, notFound.js
  modules/
    auth/        routes, controller, service, schema
    users/
    services/
    appointments/   (includes slot generation + conflict handling)
    ai/             (gemini client, prompt builder)
    admin/          (status transitions, API-key protected)
  utils/         ApiError.js, asyncHandler.js, response.js, time.js
  app.js  server.js
docs/            openapi.yaml, er-diagram.png (+ .mmd source), screenshots/
tests/
Dockerfile  .dockerignore  docker-compose.yml  render.yaml
.env.example  .env.production.example  README.md
```

### Flutter: `service-booking-app`
```
lib/
  core/
    network/   api_client.dart (dio), auth_interceptor.dart, api_exception.dart
    storage/   token_storage.dart
    router/    app_router.dart (go_router, auth redirect, bottom-nav shell)
    theme/     app_theme.dart (Material 3, tokens)
    widgets/   app_button, app_text_field, loading_view, empty_view,
               error_view (retry), status_chip, service_card
    utils/     validators.dart, formatters.dart
  features/
    auth/          data/ domain/ presentation/   (login, register, splash)
    services/      (home, search/filter, details)
    booking/       (date + slot picker, confirmation)
    bookings/      (my bookings, details, cancel)
    profile/       (view, edit, change password)
    ai_assistant/  (chat UI, recommendation cards)
  main.dart
```
Each feature has `data/` (repository + models), `domain/` where needed, and `presentation/` (screens, widgets, providers).
The API base URL comes from `--dart-define=API_BASE_URL=...`, never hard-coded.

---

## 3. Database design

```
users 1 ──< appointments >── 1 services
```

**users:** `id uuid PK`, `name`, `email` (unique, case-insensitive via `citext`), `password_hash`, `phone` (nullable), `created_at`, `updated_at`

**services:** `id uuid PK`, `name`, `description`, `price numeric(10,2)`, `duration_minutes int`, `image_url`, `category`, `is_featured bool`, `is_active bool`, `created_at`

**appointments:** `id uuid PK`, `user_id → users`, `service_id → services`, `start_time timestamptz`, `end_time timestamptz`, `status` (enum: `pending | confirmed | cancelled | completed`), `notes` (nullable), `created_at`, `updated_at`

**Integrity rules enforced in the DB:**
- `EXCLUDE USING gist (service_id WITH =, tstzrange(start_time, end_time) WITH &&) WHERE (status IN ('pending','confirmed'))`
  → no overlapping active bookings for the same service. Cancelled bookings free the slot.
- `CHECK (end_time > start_time)`, `CHECK (price >= 0)`, `CHECK (duration_minutes > 0)`
- Optional second exclusion on `user_id`, so one user can't hold two overlapping bookings.
- Indexes: `appointments(user_id, start_time)`, `services(category)`, `services(is_featured)`, and a trigram or `ILIKE` index for search if time allows.

**Seed:** about 10–12 services across 4–5 categories, with real image URLs, 3–4 marked featured, plus a demo user (documented in the README).

---

## 4. Booking model (the central logic)

Assumptions, all configurable in `constants.js`:
- Each service has **one provider (capacity 1)**, so a service can't be double-booked.
- Working hours **09:00–17:00**, every day. Slot grid is every **30 minutes** in the `BUSINESS_TZ` zone (env). The DB stores UTC.
- A slot is valid if `start + duration` ends within working hours.
- A slot is bookable if it is in the future (≥ 60 min lead time), within 60 days, and not overlapping an active appointment.

**Flow**
1. `GET /services/:id/availability?date=YYYY-MM-DD` returns slots `[{ start, end, label, available }]`. This is **advisory only**, for UI display.
2. `POST /appointments` runs inside a transaction and does a plain `INSERT`.
3. If PostgreSQL raises `23P01` (exclusion violation), the API returns `409 SLOT_UNAVAILABLE`. **The DB is the source of truth.**
4. The Flutter UI disables taken slots, and if a 409 comes back anyway it shows "slot just taken" and refreshes the slots.

**Status lifecycle:** `pending → confirmed → completed`, or `pending → cancelled`.
- Users can cancel **only `pending`** bookings (ownership and status enforced server-side).
- Nothing in the spec moves bookings forward, so I'll add a small **admin-key-protected** `PATCH /admin/appointments/:id/status` endpoint for confirming and completing. A lightweight job marks confirmed, past appointments as `completed`. Both are documented with curl examples so the demo can show every status.

---

## 5. API specification (all under `/api/v1`)

**Response envelope**
```json
{ "success": true,  "data": { }, "meta": { "page": 1, "limit": 20, "total": 42 } }
{ "success": false, "error": { "code": "SLOT_UNAVAILABLE", "message": "...", "details": [ ] } }
```

| Method | Route | Auth | Notes |
|---|---|---|---|
| POST | `/auth/register` | No | name, email, password → user + token. 409 on duplicate email. |
| POST | `/auth/login` | No | email, password → user + token. Generic error on failure. |
| GET | `/users/me` | Yes | Current user |
| PATCH | `/users/me` | Yes | name, phone |
| PATCH | `/users/me/password` | Yes | current + new password (optional feature, cheap to add) |
| GET | `/services` | No | `search`, `category`, `featured`, `minPrice`, `maxPrice`, `sort`, `page`, `limit` |
| GET | `/services/categories` | No | For filter chips |
| GET | `/services/:id` | No | Full details |
| GET | `/services/:id/availability` | Yes | `date` query param |
| POST | `/appointments` | Yes | serviceId, startTime, notes |
| GET | `/appointments` | Yes | `scope=upcoming\|past`, `status` |
| GET | `/appointments/:id` | Yes | Owner only (404 otherwise) |
| PATCH | `/appointments/:id/cancel` | Yes | Pending only |
| POST | `/ai/chat` | Yes | message + short history → reply + recommended services |
| PATCH | `/admin/appointments/:id/status` | Admin key | Demo and ops |
| GET | `/health` | No | For Render |

**Security checklist:** bcrypt (cost 12), JWT secret from env (min length enforced at boot), helmet, restricted CORS, global + per-route rate limits, zod validation on every body/query/param, uniform error handler (no stack traces in production), no secrets in git, ownership checks on every appointment query.

---

## 6. AI assistant design

- **Endpoint:** `POST /ai/chat` with `{ message, history[] }`. Limits: message ≤ 500 chars, history ≤ 10 turns, rate limit about 10 requests/min/user.
- **Grounding:** on each call, load active services (id, name, category, price, duration, short description) and inject them into the system prompt, so answers come from real data.
- **System prompt:** scope it to this business's services. Politely decline off-topic or prompt-injection attempts. Never invent services or prices.
- **Output:** use Gemini's structured JSON output: `{ "reply": string, "recommendedServiceIds": string[] }`. The backend **validates the IDs against the DB** before returning.
- **Model:** set via `GEMINI_MODEL` env, using a current Flash-tier model. I'll confirm the exact model name against the live docs when we build it, since names change.
- **Resilience:** 15s timeout, and on failure a friendly `503 AI_UNAVAILABLE`. The Flutter UI shows a retry.
- **Flutter UI:** chat bubbles, a typing indicator, suggested prompt chips ("Something relaxing under $50"), and tappable recommended-service cards that open the service details.

---

## 7. Flutter screens and state

**Screens:** Splash → Login / Register → Home → Service Details → Booking (date strip + slot grid + notes) → Confirmation → My Bookings (Upcoming/Past tabs) → Booking Details → Profile (+ edit, change password) → AI Assistant.
**Bottom navigation:** Home · Bookings · Assistant · Profile.

**Riverpod providers (examples)**
- `authProvider` (`AsyncNotifier`): restore the session on launch (read token → `GET /users/me`), login, register, logout.
- `servicesProvider(filters)`: list with debounced search, category chips, featured section.
- `serviceDetailProvider(id)`, `availabilityProvider(serviceId, date)`
- `bookingControllerProvider`: submit with loading, success and error states
- `myBookingsProvider(scope)`, `cancelBookingProvider`
- `profileProvider`, `chatProvider`

**Cross-cutting UI rules:** every async screen goes through shared Loading / Empty / Error(retry) widgets. A snackbar or banner shows success and error feedback. Form validation lives in a shared `validators.dart`. The layout is responsive via `LayoutBuilder` (grid columns 1/2/3 by width). A 401 from the interceptor signs the user out and redirects to login.

---

## 8. Timeline (72 hours, vertical slices)

| Phase | Hours | Work | Exit criteria |
|---|---|---|---|
| **0 Setup** | 0–4 | Init both repos and `.gitignore`, `Dockerfile` + `docker-compose` (api + Postgres), Express skeleton, Flutter skeleton, env handling, first commits | `docker compose up` serves `/health`; Flutter app runs |
| **1 Database + backend core** | 4–14 | **Database first:** schema, migrations (incl. `btree_gist` + exclusion), seeds. Then auth, users, services + search/filter | Schema applies cleanly; auth and services tested via curl/Postman |
| **2 Booking logic + AI endpoint** | 14–24 | Availability, create/list/get/cancel, 409 handling, admin status route, **Gemini `/ai/chat` endpoint**, backend tests, OpenAPI | Backend feature-complete; conflict test passes under concurrency |
| **3 Early deploy** | 24–28 | Neon DB + Render (Docker deploy from the same `Dockerfile`), migrations and seed on Neon, **confirm `btree_gist` works there**, health check | Public HTTPS API serves seeded services |
| **4 Flutter foundation** | 28–36 | Theme, dio client, secure storage, router, shared widgets, auth screens and session persistence | Register, login, restart app, still logged in |
| **5 Flutter booking flow** | 36–48 | Home (search/filter/featured), details, slot picker, confirmation, My Bookings, cancel, profile | Full booking journey against the live API |
| **6 Flutter AI chat UI** | 48–52 | Chat screen + recommendation cards against the already-built endpoint | Assistant answers from real data |
| **7 Hardening** | 52–62 | Edge cases, state polish, backend tests, a few Flutter tests, `flutter analyze` clean, release APK against the prod API on a real device | APK works end to end |
| **8 Docs and delivery** | 62–72 | README ×2, OpenAPI, ER diagram, screenshots, demo video, final APK, **GitHub Release (tag `v1.0.0`) with the APK attached**, form submission | All 7 deliverables ready, buffer for retakes |

**Rule:** if a phase slips, cut polish (extra animations, optional change-password UI), never the core flow or docs.

---

## 9. Testing plan

**Backend (vitest + supertest, against a test DB)**
- register/login validation and duplicate email
- protected route rejects a missing or invalid token
- **double-booking race:** two parallel `POST /appointments` for the same slot → exactly one 201 and one 409
- cancel allowed only for `pending` and only by the owner
- another user's appointment returns 404
- availability excludes booked and past slots

**Flutter:** unit tests for validators and repositories (fake client), 1–2 widget tests (login form, empty/error states), `flutter analyze` clean.

---

## 10. Git conventions

- Conventional commits (`feat(auth): ...`, `fix(booking): ...`, `docs: ...`), small and frequent, one concern each.
- Work on `main` with short-lived feature branches if useful. The history should read as a story.
- Before any commit: confirm `.env` is ignored and secrets aren't staged. Commit `.env.example` only.
- I won't push until you've created the repos and told me to.

---

## 11. Deliverables checklist

- [ ] Flutter repo (clean history, README with screenshots)
- [ ] Backend repo (README, `.env.example`, `docker-compose.yml`)
- [ ] DB schema + ER diagram (Mermaid source and PNG) + setup instructions
- [ ] API docs (OpenAPI + README endpoint tables with example responses)
- [ ] Demo video (3–5 min)
- [ ] Release APK (built with the production API URL, tested on a device), **published as a GitHub Release asset** on the Flutter repo (see section 11b)
- [ ] Screenshots of main screens
- [ ] Form submission at https://forms.gle/HdGXLww44KRSDsEh7

### 11b. APK distribution via GitHub Releases

1. Build: `flutter build apk --release --dart-define=API_BASE_URL=https://<service>.onrender.com/api/v1` → `build/app/outputs/flutter-apk/app-release.apk`. **Build with `kotlin.incremental=false`** (set `kotlin.incremental=false` in the app's `android/gradle.properties`) for the final release build.
2. Rename the file to something clear, e.g. `service-booking-v1.0.0.apk`. **Never commit the APK to git**; keep `*.apk` in `.gitignore`.
3. Tag the commit (`v1.0.0`) and create a GitHub Release on `service-booking-app` with the APK as the release asset, e.g. `gh release create v1.0.0 <apk> --title "v1.0.0" --notes "..."` (needs the `gh` CLI authenticated, or upload through the GitHub web UI).
4. Release notes: features, the API URL it points to, the demo login, the cold-start warning, and install steps (allow "install unknown apps").
5. Put the Release link and a download badge at the top of the Flutter README, and use the same link in the Google Form.
6. **Signing:** the release build is signed with the debug keystore unless we create a real one. If we create one, it stays **out of git** (`key.properties` and the `.jks` are git-ignored). Back it up if you ever want to ship updates.
7. Optional stretch: a GitHub Actions workflow that builds the APK on a `v*` tag and attaches it to the Release. Only worth it if there is spare time in phase 8.
8. Before publishing: install the downloaded Release APK on a clean device and verify the full flow. Creating the Release is outward-facing, so I'll do it only after you say so.

**Demo script (about 4 min):** register → browse, search and filter → open a service → book a slot → confirmation → My Bookings → try to rebook the same slot (blocked) → cancel a pending booking → show Confirmed/Completed via the admin endpoint → AI assistant recommendation → profile edit → logout and relogin (persistence) → 20 seconds on architecture and the DB constraint.

---

## 12. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Render free tier cold starts (30–60 s) and Neon autosuspend | Hit `/health` and one DB-backed route before the demo, add a loading state, note it in the README |
| Local and live drift (SSL, pooling, env differences) | Same Dockerfile, zod-validated env, `DATABASE_SSL` flag, and a deploy smoke test in phase 3 |
| `btree_gist` unavailable on the host | Verify in phase 3 (before Flutter work). Fallback: serializable transaction + advisory lock on `service_id` |
| Timezone bugs in slots | Store UTC, generate slots in `BUSINESS_TZ`, and have the API return a display `label` |
| Gemini quota, errors or model rename | Env-configurable model, timeouts, friendly fallback, key kept server-side |
| APK can't reach the API | HTTPS prod URL, `INTERNET` permission, test the release build on a real device |
| Scope creep | Out of scope: payments, push notifications, admin UI, social login, multi-provider scheduling |
| Not understanding generated code | Keep a running `DECISIONS.md` and re-read each module after it lands |

---

## 13. Assumptions to confirm before starting

1. Working hours 09:00–17:00, 30-minute slot grid, 60-minute minimum lead time, 60-day horizon.
2. One provider per service (capacity 1).
3. Admin-key endpoint for confirming and completing bookings (plus a completion job).
4. Business timezone: **which one should `BUSINESS_TZ` be?** (It defaults to UTC if you don't say.)
5. Single JWT with 7-day expiry (no refresh tokens).
6. Debug-keystore-signed release APK is acceptable (a store-signed build isn't required).

---

## 14. Added scope: admin accounts (requested during the build)

Two kinds of accounts: **customer** and **admin**. Admins add services ("products") and track sales.

- **Roles:** `role` on users (`user` | `admin`). Public registration always creates a customer. The demo admin is seeded. Admin routes need an admin-role JWT (the `X-Admin-Key` ops route stays).
- **Dynamic add-service form:** the form is schema-driven. Admins can define extra fields (text, long text, number, yes/no, dropdown, required or not) that then appear on the form for every service. Field definitions live in `service_fields`; per-service values live in a JSONB `attributes` column, validated against the definitions. Customers see the filled values as "Extra details" on the service page.
- **Dashboard:** sales today and this month, bookings by status, top services, recent bookings, bookings per day.
- **Reports:** date range plus grouping by day, service or category; revenue (confirmed and completed bookings) and booking counts; CSV export.
- **Bookings management:** admins list all bookings and confirm, complete or cancel them (replaces the demo-only curl step).
- **Flutter:** admin accounts get their own shell and tabs (Dashboard, Services, Bookings, Reports, Profile), guarded by role.
- **Order of work:** finish the customer flow first (booking, my bookings, profile, AI chat), then the admin backend, then the admin screens.
