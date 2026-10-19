# ZunZun ToDo — Project Memory

**ずんずんToDo (ZunZun ToDo) — Keep moving forward, zun-zun.**

A web app for reaching big goals step by step: users break a **Project** into **Tasks**, record each day's
progress on a calendar, see their effort in charts, and cheer each other on when public projects are completed.

*Zun-zun* (ずんずん) is Japanese onomatopoeia for moving forward steadily and energetically.
Keep domain/code names plain (`Project`, `Task`, `ProgressLog`); the zun-zun vocabulary belongs only in UI copy.
In code, identifiers, and URLs the name is written `zunzuntodo` / `ZunZunToDo`.

- Design: `docs/design-specification.md` (SDS) — the source of truth for scope, schema, and API
- Plan: `docs/implementation-plan.md` (14 days)
- Progress notes: `docs/dev-log.md` — read the latest "Next" entry when starting work
- Decisions: `docs/adr/`

**Out of scope for v1.0 — do not add:** AI features, timers or stopwatches, push notifications,
multiple languages, Redis. If a task seems to need one of these, stop and ask.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Java 25 · Spring Boot 4.1.x (Web MVC, Data JPA, Security, OAuth2 Client, OAuth2 Resource Server, Validation, Actuator) |
| Build | Maven via the wrapper (`./mvnw`) |
| Database | PostgreSQL 17 · Flyway |
| Libraries | MapStruct · springdoc-openapi · AWS SDK v2 (S3) · Bucket4j |
| Frontend | Next.js (App Router) · TypeScript · Tailwind CSS · TanStack Query · Recharts · date-fns |
| Testing | JUnit Jupiter · Mockito · Testcontainers 2 (PostgreSQL) · Spring Security Test |
| Hosting | Vercel (cdg1, Paris) · AWS eu-west-3 (Paris): ECS Fargate, RDS, S3 |

## Directory Layout

```
zunzun-todo/
├── backend/                                     # Maven project
│   ├── pom.xml
│   └── src/main/java/com/zunzuntodo/            # feature packages (below)
│   └── src/main/resources/db/migration/         # Flyway
├── frontend/src/app/                            # Next.js App Router
├── docs/                                        # SDS, plan, deployment, dev log, ADRs
└── docker-compose.yml                           # local PostgreSQL
```

The backend is organized **package-by-feature**. Never create layer-wide packages such as `controllers/` or `services/`.

```
com.zunzuntodo/
├── config/    common/    auth/     user/
├── project/   task/      progress/ stats/   social/
```

Each feature package contains its own controller, service, repository, entities, DTOs, and mapper.

---

## Commands

```bash
docker compose up -d                       # local PostgreSQL

cd backend && ./mvnw spring-boot:run       # API on http://localhost:8080
cd backend && ./mvnw test                  # unit + integration tests (starts Testcontainers)
cd backend && ./mvnw verify                # full build, as CI runs it

cd frontend && npm run dev                 # app on http://localhost:3000
cd frontend && npm run typecheck
cd frontend && npm run lint
```

---

## Non-Negotiable Rules

1. **Never edit an applied Flyway migration.** Always add a new version.
2. **Entities never leave the service layer.** Controllers accept and return DTOs only.
3. **Never commit secrets.** `.env` and `.env.local` are gitignored; placeholders go in `.env.example`.
4. **Never skip authorization.** Every read or write checks ownership; every path returning another user's data
   checks visibility (only `PUBLIC` projects; achievements only when `PUBLIC` and `COMPLETED`).
   Progress records are never visible to anyone but their owner. Each such rule has an integration test.
5. **Progress records are the single source of truth** for task completion and for chart data.
   Completing or un-completing a task always goes through the progress service.
6. **Return 404, not 403, for other users' private resources**, so the API never reveals that they exist.

## Coding Conventions

**Backend**
- Spring Boot 4: use the modular starters (`spring-boot-starter-webmvc`, `spring-boot-starter-flyway`, …) and
  Jackson 3 (`tools.jackson.*`; annotations stay in `com.fasterxml.jackson.annotation`). Don't copy Boot 3 snippets as-is
- Constructor injection; DTOs are Java records
- Validation annotations on request DTOs; business rules in services
- Throw domain exceptions; `GlobalExceptionHandler` turns them into RFC 7807 responses. No try/catch in controllers
- `@Transactional` on any service method that writes more than once; `@Transactional(readOnly = true)` for reads
- Timestamps are `TIMESTAMPTZ` / `Instant` in UTC. Calendar dates are `DATE` / `LocalDate` and are never converted
- The user's `time_zone` is used only to decide what "today" is
- Durations are whole **minutes** (`minutes`); converting to hours is a presentation concern
- Weeks run Monday to Sunday
- Inject `Clock` wherever "now" or "today" matters; never call `LocalDate.now()` directly
- Images go through the `ImageStorage` interface (local files in development, S3 in production)

**Frontend**
- Server Components by default; `'use client'` only for state, events, charts, and TanStack Query
- Server state in TanStack Query, UI state in `useState`; never mix them
- All API calls go through `lib/api/`; components never call `fetch` directly
- Access token in memory only (never `localStorage`); the refresh token is an HttpOnly cookie handled by the API
- English UI; dates as `9 Oct 2026`; 24-hour time; weeks start on Monday

---

## Design Direction

The UI is built on **goldenrod (yamabuki) and white** for a bright, warm, welcoming feel.
Always read `.claude/skills/frontend-feature/SKILL.md` before writing UI code.

---

## Workflow

- One feature at a time, vertically: backend → frontend → verify locally
- One branch per issue (`feature/xxx`) → PR with `Closes #n` → CI green → merge
- Small commits, one logical change each; add a dev-log entry at the end of every work block
