# ZunZun ToDo — Implementation Plan (14 Days)

**Goal:** version 1.0 as specified in the [Design Specification](design-specification.md), running locally with tests by Day 12 and deployed to Vercel and AWS (Paris) by Day 14.
**Assumed effort:** about **6 focused hours per day**. With less time per day, keep the order and stretch the calendar — the day numbers are a sequence, not dates.
**Approach:** vertical slices — each feature goes backend → frontend → verified locally before the next one starts.

---

## Working Rules

- **One day, one milestone-sized slice.** Each day ends with something that works end to end.
- **One GitHub issue per day** (or per slice), one branch, one pull request with `Closes #n`. Merge only on green CI.
- **End every work block** with a commit, a push, and two lines in [`dev-log.md`](dev-log.md): what was done, what is next.
- **Behind schedule? Cut scope, not quality.** Use the cut list at the end; never skip tests for authorization.
- **With Claude Code:** start by asking it to read the latest dev-log entry and today's section of this plan; request one deliverable at a time and read everything it generates before committing.

---

## Overview

| Day | Focus | Tag |
|---|---|---|
| 1 | Project setup, local environment, CI | `m1-setup` |
| 2 | Password authentication and tokens (backend) | |
| 3 | Google sign-in, sign-in UI, app shell, About page | `m2-auth` |
| 4 | Projects and tasks — backend | |
| 5 | Projects and tasks — frontend | |
| 6 | Progress records — backend | |
| 7 | Calendar — frontend | `m3-calendar` |
| 8 | Progress charts | **`m4-core`** |
| 9 | Profiles, images, hashtags, people search | |
| 10 | Follow, public profiles, feed | |
| 11 | Reactions and comments | `m5-social` |
| 12 | Hardening, account deletion, README polish | |
| 13 | AWS infrastructure (Paris) | |
| 14 | CI/CD, domain, Vercel production, release | **`v1.0.0`** |

---

## Day 1 — Setup → `m1-setup`

**Backend**
- Create the repository; commit the documentation, `CLAUDE.md`, and skills first
- Create the Maven backend with Spring Initializr in `backend/`: Java 25, Spring Boot **4.1.x** (latest stable patch; 4.1.1 at the time of writing), group and package `com.zunzuntodo`, Jar, YAML configuration ([ADR 0001](adr/0001-spring-boot-4.md))
  — dependencies: Spring Web, Spring Data JPA, Spring Security, OAuth2 Client, OAuth2 Resource Server, Validation, Actuator, PostgreSQL Driver, Flyway Migration, Testcontainers
  — Boot 4 uses modular starters (`spring-boot-starter-webmvc`, `spring-boot-starter-flyway`, `spring-boot-starter-security-oauth2-client`, …) and matching `-test` starters; keep the names Initializr generates. Check that `flyway-database-postgresql` and `testcontainers-postgresql` are present. The Maven Wrapper is included
- Add MapStruct and springdoc-openapi **3.x** (the line built for Boot 4) to `pom.xml`
- `docker-compose.yml` with PostgreSQL 17; `application.yml` + `application-local.yml`; `.env.example`
- Flyway `V1__users_and_auth.sql` (users, oauth_accounts, refresh_tokens)
- `/actuator/health` public; CORS for `http://localhost:3000`
- `AbstractIntegrationTest` (Testcontainers, `@ServiceConnection`) with one passing test
- GitHub Actions: backend CI (`./mvnw -B verify` with Temurin 25) on pull requests and on `main`

**Frontend**
- `create-next-app` (App Router, TypeScript, Tailwind, ESLint) in `frontend/`
- Theme tokens in `globals.css`, M PLUS Rounded 1c font, `lib/api/` client skeleton
- Home page calls `/actuator/health` and shows the status
- GitHub Actions: frontend CI (typecheck, lint, build)

**Done when:** `docker compose up -d`, `./mvnw spring-boot:run`, and `npm run dev` work; the browser shows the API as UP; both CI workflows pass on a pull request.

---

## Day 2 — Password Authentication (backend)

- `GlobalExceptionHandler` (RFC 7807) and domain exceptions
- `POST /auth/register` (bcrypt cost 12, handle and password rules), `POST /auth/login`
- JWT access tokens (HS256, 15 min) via `NimbusJwtEncoder`; resource-server configuration validates them
- Refresh tokens: opaque, SHA-256 hashed, rotated on every use, family revoked on reuse; `HttpOnly; Secure; SameSite=Lax` cookie on `/api/v1/auth`
- `POST /auth/refresh` with `Origin` allowlist check; `POST /auth/logout`
- `GET /users/me`
- Bucket4j rate limit on register and login
- Integration tests: register → login → `/users/me`; rotation; reuse detection; 401 without token; rate limit

**Done when:** the full flow works with `curl`, and `./mvnw verify` is green.

---

## Day 3 — Google Sign-in, UI Shell, About → `m2-auth`

**Backend**
- `oauth2Login` with Google; success handler finds or creates the user by `(GOOGLE, sub)`, refuses emails that already have a password account, sets the refresh cookie, and redirects to `/auth/callback`
- `PATCH /users/me` for choosing a handle

**Frontend**
- UI components: Button, Card, Input, Chip, Avatar
- Sign-in and sign-up pages with "Continue with Google"
- `/auth/callback` (calls refresh) and choose-handle page
- In-memory access token, refresh-on-401 in the API client, auth context, protected `(app)` routes
- App shell: desktop top bar and mobile bottom bar (§8.2 of the SDS)
- **About page** with the zun-zun explanation

**Done when:** a new user can sign up with a password **and** with Google (choosing a handle), sign out, and sign back in; About is reachable signed out.

---

## Day 4 — Projects and Tasks (backend)

- Flyway `V2__projects_and_tasks.sql`
- Project CRUD, `complete`, `reopen`; completion rate in responses
- Task CRUD and reordering
- Ownership checks (404 for other users' projects); `@Version` optimistic locking
- Tests: ownership, completion rate, complete/reopen rules

**Done when:** all project and task endpoints work and are covered by integration tests.

---

## Day 5 — Projects and Tasks (frontend)

- Projects page: Active / Completed tabs, cards with 🌐/🔒 and completion bar, create dialog
- Project detail: edit, visibility toggle, task list with add/rename/delete/reorder, "Mark project complete" / "Reopen"
- Task checkbox wired to a placeholder (connected to completion records on Day 7)
- Loading, error, and empty states

**Done when:** projects and tasks can be managed entirely through the UI.

---

## Day 6 — Progress Records (backend)

- Flyway `V3__progress_logs.sql` (including `uq_logs_one_completion_per_task`)
- `ProgressLogService`: create, edit, delete; completion records set the task to Done and back
- `PUT` / `DELETE /tasks/{id}/completion` go through the same service
- `GET /logs?from=&to=` (max 62 days); future dates rejected using the user's time zone and an injected `Clock`
- Tests: completion consistency, 409 on second completion, 422 on future dates and invalid minutes, owner-only access

**Done when:** records and task status always agree, proven by tests.

---

## Day 7 — Calendar (frontend) → `m3-calendar`

- Month grid with `date-fns`, Monday first, ‹ › and **Today**
- Today highlighted (filled goldenrod circle), selected date ring, record dots and completion checks, future dates disabled
- Day panel / bottom sheet: records grouped by project, total time, edit and delete
- Add-record form: project → task → Completed / Worked on → optional time → optional note
- Project-page checkbox now records a completion for today; "Zun!" bounce on completion

**Done when:** a day's progress can be recorded, edited, and deleted from the calendar, and the project page reflects completions immediately.

---

## Day 8 — Progress Charts → `m4-core`

**Backend**
- `GET /stats/progress` — metric `MINUTES` / `COMPLETED_TASKS`, period `WEEK` (Monday–Sunday) / `MONTH`, optional `projectId`, zero-filled days, total
- Tests: week and month boundaries (including leap-year February), zero days, project scope, completion-only counting

**Frontend**
- Progress page: scope, metric, and period controls with ‹ › navigation and period label
- Bar chart for all projects, line chart for one project; hours with one decimal or integer counts; total above the chart; empty state

**Done when:** a new user can sign up, create a project, record progress on the calendar, and see it in both charts. **The core of the product is complete.**

---

## Day 9 — Profiles, Images, Hashtags, Search

- Flyway `V4__hashtags.sql`
- `PATCH /users/me` (name, bio, time zone), `PUT/DELETE /users/me/avatar` with content validation, `PUT /users/me/hashtags`
- `ImageStorage` interface with a local-file implementation (S3 comes on Day 13); presigned or local URLs in responses
- `GET /users/search?q=&hashtag=`
- Settings page: profile fields, image upload with preview, hashtag editor
- People page: search box, hashtag chips, result cards
- Tests: upload validation (non-image, over 2 MB), hashtag normalization and limit

**Done when:** a user can set an image and hashtags and find others by hashtag.

---

## Day 10 — Follow, Public Profiles, Feed

- Flyway `V5__social.sql` (follows, reactions, comments)
- Follow / unfollow, followers and following lists
- `GET /users/{handle}` and `/users/{handle}/projects` (public active projects and achievements)
- `GET /feed?cursor=` — followed users' public completed projects, newest first
- Public profile page `/u/[handle]` as a Server Component with a Follow button
- Home page: "Today" summary and the feed
- Visibility tests: private and reopened projects never appear on profiles or in feeds

**Done when:** with two accounts, completing a public project shows it on the other account's home screen and on the owner's profile.

---

## Day 11 — Reactions and Comments → `m5-social`

- Reactions `PUT/DELETE /projects/{id}/reactions/{emoji}`; counts and "mine" in achievement responses
- Comments: create, edit (author), delete (author or achievement owner); 404 for non-achievements
- Reaction bar and comment thread on achievement cards (home and profile)
- Tests: permissions for edit/delete, 404 for private or reopened projects

**Done when:** two accounts can react to and discuss an achievement, and the owner can remove a comment.

---

## Day 12 — Hardening

- `DELETE /users/me` (cascades, removes the image) and the delete-account flow in Settings
- Fill test gaps; run a full manual walkthrough on desktop and at 360 px wide
- Loading, error, and empty states everywhere; keyboard navigation; contrast check
- Demo seed script (`scripts/`) with a few users, projects, records, and an achievement
- README: screenshots of the calendar, charts, and feed

**Done when:** no known defects; `./mvnw verify` and frontend CI are green.

---

## Day 13 — AWS Infrastructure (Paris)

Details: [`aws-deploy-plan.md`](aws-deploy-plan.md).

- Budget alert first; production Dockerfile
- ECR, VPC and security groups, RDS PostgreSQL, S3 bucket for images, Secrets Manager
- `S3ImageStorage` and `application-prod.yml`
- ECS Fargate service behind an ALB; Flyway migrates the production database on start-up

**Done when:** `https://<alb-dns-name>/actuator/health` returns UP.

---

## Day 14 — Release → `v1.0.0`

- Domain: `zunzuntodo.app` → Vercel, `api.zunzuntodo.app` → ALB with an ACM certificate
- Production CORS and `Origin` allowlist; Google OAuth production redirect URI
- GitHub Actions deployment to ECS (OIDC)
- Vercel production (function region `cdg1`), `NEXT_PUBLIC_API_URL`
- Full walkthrough in production; demo data; production URL in the README and the repository's About box

**Done when:** the production site works from sign-up to an achievement with comments.

---

## Cut List

If the schedule slips, cut in this order — never cut authorization or its tests.

| Order | Cut | Effect |
|---|---|---|
| 1 | Task reordering | Tasks keep creation order |
| 2 | Comment editing | Authors delete and re-post instead |
| 3 | People search by name | Search by hashtag only |
| 4 | Custom domain (Day 14) | Use the default URLs; sign-in persistence needs the same-site domain, so plan it soon after |
| 5 | Automated ECS deployment | Deploy with a documented manual command |

---

## Milestones

| Tag | Day | Meaning |
|---|---|---|
| `m1-setup` | 1 | Backend and frontend run locally and talk to each other |
| `m2-auth` | 3 | Password and Google sign-in work end to end |
| `m3-calendar` | 7 | Projects, tasks, and calendar records work |
| `m4-core` | 8 | **Charts complete — the core loop is done** |
| `m5-social` | 11 | Profiles, follow, feed, reactions, comments |
| `v1.0.0` | 14 | **Production release** |
