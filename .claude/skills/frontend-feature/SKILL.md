---
name: frontend-feature
description: Build ZunZun ToDo frontend work in Next.js (App Router) — pages, components, the calendar, progress charts, API clients, and TanStack Query hooks. Contains the goldenrod-and-white design system that gives the app its bright, warm look. Use whenever the user says "build a screen", "make the UI", "add a component", "implement the frontend", "style this", or mentions the calendar, charts, feed, profile, or About page, and whenever starting client-side work on a feature.
---

# Frontend Implementation Guide

## Design Direction (Read First)

ZunZun ToDo exists to make effort visible and to keep people encouraged. The interface is built on
**goldenrod (yamabuki) and white** and must feel **bright, warm, and welcoming** — never cold, clinical, or corporate.

### Color Tokens

Tailwind CSS v4 defines theme tokens in CSS. Put these in `src/app/globals.css` and use only these tokens —
never hard-code other colors.

```css
@import "tailwindcss";

@theme {
  --color-yamabuki-50:  #FFFBEB;  /* faint backgrounds, hover fills */
  --color-yamabuki-100: #FEF3C7;  /* soft accent surfaces */
  --color-yamabuki-200: #FDE68A;  /* borders, gentle emphasis */
  --color-yamabuki-300: #FCD34D;  /* secondary fills, chart hover */
  --color-yamabuki-400: #FBBF24;  /* light goldenrod */
  --color-yamabuki-500: #F5B301;  /* ★ brand: primary actions, today, progress */
  --color-yamabuki-600: #D99A00;  /* hover */
  --color-yamabuki-700: #B37D00;  /* pressed, strong text on light fills */
  --color-cream: #FFFDF7;         /* page background — warmer than white */
  --color-ink:   #3D3528;         /* body text — warm dark brown, not black */
  --color-muted: #8B8073;         /* secondary text */
  --color-line:  #EDE6D8;         /* borders and dividers */
}
```

These generate utilities such as `bg-yamabuki-500`, `text-ink`, `border-line`, `bg-cream`.

**Rules**
- Page background `bg-cream`; cards `bg-white` so they lift off the page
- Never pure black text: body `text-ink`, secondary `text-muted`
- Goldenrod only for **primary actions, today's date, and progress**. Overusing it looks cheap
- Dividers `border-line`; avoid heavy gray rules
- White text on `yamabuki-500` fails contrast for small text — use **bold** white text at ≥ 14 px, or `text-ink` on yamabuki

### Shape and Texture

- Rounding: cards `rounded-2xl`, buttons and inputs `rounded-xl`, chips `rounded-full`
- Soft shadows only: `shadow-sm` / `shadow-md`
- Roomy spacing: `p-5`–`p-6` in cards, `gap-6` / `space-y-6` between sections
- Icons: `lucide-react` with `strokeWidth={1.75}`

### Typography

- Font: **M PLUS Rounded 1c** via `next/font/google` (rounded and warm; covers the Japanese logo text)
- Headings `text-2xl font-bold text-ink`; body `text-base text-ink`; details `text-sm text-muted`
- **Make the numbers big.** Totals and completion rates use `text-3xl font-bold`

### Buttons and Cards

```tsx
// Primary
"rounded-xl bg-yamabuki-500 px-5 py-2.5 font-bold text-white shadow-sm transition hover:bg-yamabuki-600 active:bg-yamabuki-700 disabled:opacity-50"
// Secondary
"rounded-xl border border-line bg-white px-5 py-2.5 font-medium text-ink transition hover:bg-yamabuki-50"
// Card
"rounded-2xl border border-line bg-white p-6 shadow-sm"
```

### Voice and Microcopy

- Encouraging, never scolding: "New week, fresh start" rather than "You did nothing last week"
- Celebrate effort: "You recorded 2 h 15 min today", "One more step today — zun-zun!"
- Empty states teach: "No records yet — pick a day on the calendar and add what you did"
- English only. Dates `9 Oct 2026` (`date-fns` format `d MMM yyyy`); 24-hour time; weeks start on Monday

### Zun-zun Vocabulary (UI copy only — never rename code, types, or API fields)

| Code / API | UI wording | Example |
|---|---|---|
| Project | Goal / destination | "Heading for: Pass the B2 French exam" |
| Task | Step | "Next step: Mock exam #2" |
| Completing a task | Zun! | "Zun! 3 of 12 steps done" |
| Completed public project | Achievement | "Mina reached a goal 🎉" |

**Motion:** completing a task shows a short forward "bounce" (≈250 ms translate-x with ease-out).
Wrap it in `motion-safe:` so it is skipped when the user prefers reduced motion.

Brand: show **ずんずんToDo** in the logo; use **ZunZun ToDo** in text. Tagline: **"Keep moving forward, zun-zun."**

---

## Navigation

| Tab | Route |
|---|---|
| Home | `/home` |
| Calendar | `/calendar` |
| Projects | `/projects`, `/projects/[id]` |
| Progress | `/progress` |
| People | `/people` |
| About | `/about` (public) |

Desktop: top bar with all six tabs and an avatar menu (Profile, Settings, Sign out).
Mobile: bottom bar with Home, Calendar, Projects, Progress, People; About is an ⓘ icon button in the header.
Public routes: `/about`, `/u/[handle]`, `/login`, `/register`.

---

## Calendar (`/calendar`)

Build the month grid yourself with `date-fns` (`startOfMonth`, `startOfWeek(…, { weekStartsOn: 1 })`, …) — no calendar library.

- Opens on the current month with **today selected**
- Header: month label, ‹ › buttons, and a **Today** button
- **Today** — always highlighted, even when another date is selected:
  `bg-yamabuki-500 text-white font-bold rounded-full` on the date number, plus `aria-current="date"` and a visually hidden "Today"
- **Selected date** — `ring-2 ring-yamabuki-500` around the cell
- **Days with records** — a small `bg-yamabuki-400` dot under the number; a check icon when a task was completed that day
- **Future dates** — `text-muted/50`, not selectable for new records
- Day cells are buttons with full date labels for screen readers ("Thursday 9 October 2026, 2 records")
- Day panel (beside the grid on desktop, bottom sheet on mobile): records grouped by project, total time, edit/delete per record, and **Add record**:
  Project → Task → "Completed" / "Worked on" (segmented control) → time (hours + minutes, optional) → note (optional, 280 chars)
- Fetch one month per request: `GET /logs?from=<first day of grid>&to=<last day of grid>`; derive dots from the result

## Progress Charts (`/progress`)

Use **Recharts**.

- Controls: Scope (All projects / project select) · Metric (Time / Completed tasks) · Period (Week / Month) with ‹ › and a label (`5–11 Oct 2026`, `October 2026`)
- All projects → `BarChart`; one project → `LineChart`
- Series color `#F5B301`; grid lines `#EDE6D8`; axis text `#8B8073`; rounded bar tops (`radius={[6, 6, 0, 0]}`)
- Time: the API sends minutes — display hours with one decimal (`(m / 60).toFixed(1)`), tooltip shows `2 h 15 min`
- Completed tasks: integer axis (`allowDecimals={false}`)
- Period total above the chart in `text-3xl font-bold` ("6.9 h this week")
- Empty period: friendly message with a link to the calendar instead of an empty chart
- Wrap charts in a client component; give the chart an `aria-label` summarizing the total

## About Page (`/about`)

Same content as the README section *Why "ZunZun"?*: what ずんずん means, why it fits big goals, the mapping
(Project = where you are heading, Task = each stride, records = the distance behind you, followers = people walking
beside you), and the tagline. Server Component, no data fetching, reachable signed out.

## Feed, Achievements, Profiles

- Achievement card: avatar, name and @handle, project title, number of steps, completion date, reaction bar, comment count
- Reactions: 👏 🔥 💪 🎉 ❤️ as toggle chips with counts; my reactions use `bg-yamabuki-100 border-yamabuki-300`
- Comments: author can edit/delete; achievement owner can delete any; 500-character limit with a counter
- Public profile `/u/[handle]` is a **Server Component** (SEO and link previews) with client islands for Follow, reactions, and comments
- Avatar fallback: initials on `bg-yamabuki-100 text-yamabuki-700`
- Every project shows 🌐 Public or 🔒 Private

---

## Implementation Conventions

### Directory Layout

```
frontend/src/
├── app/
│   ├── (auth)/          # login, register, auth/callback, choose-handle
│   ├── (app)/           # signed-in shell; layout.tsx holds navigation
│   ├── u/[handle]/      # public profile
│   └── about/           # public About page
├── components/
│   ├── ui/              # Button, Card, Input, Chip, Avatar … (build these first)
│   ├── calendar/        # MonthGrid, DayPanel, RecordForm
│   ├── charts/          # ProgressBarChart, ProgressLineChart
│   └── social/          # AchievementCard, ReactionBar, CommentThread
├── lib/api/             # typed API clients, one per domain
├── hooks/               # TanStack Query hooks
└── types/               # types mirroring backend DTOs
```

### Data and Auth

- Server state in TanStack Query, UI state in `useState`; never mix them
- Components never call `fetch` directly — always through `lib/api/`
- The API client keeps the **access token in memory**, sends it as `Authorization: Bearer`, and on `401` calls
  `POST /auth/refresh` (with `credentials: "include"`) once, then retries. Never store tokens in `localStorage`
- Mutations use optimistic updates (`onMutate`, roll back in `onError`, invalidate in `onSettled`)
- After a record is added or removed, invalidate the month's logs, the project's tasks, and the stats queries

### Always Implement Three States

- **Loading** — skeletons (`animate-pulse bg-yamabuki-50`), not spinners everywhere
- **Error** — what happened plus a retry button; never raw technical messages
- **Empty** — a short hint about what to do next

### Accessibility and Responsiveness

- Never rely on color alone (today has a label; charts state totals in text)
- Everything keyboard-operable; icon-only buttons have `aria-label`
- Mobile-first, works from 360 px wide

---

## Checklist

- [ ] Only the theme tokens were used; background cream, cards white, text ink
- [ ] Rounded, soft, roomy — matches the warm direction
- [ ] Calendar: today highlighted, selected ring, record dots, future dates disabled
- [ ] Charts: bar for all projects, line for one project; correct units; total shown
- [ ] Loading, error, and empty states exist
- [ ] Microcopy is encouraging; dates formatted `d MMM yyyy`; weeks start Monday
- [ ] `'use client'` only where needed; API calls via `lib/api/`; no tokens in `localStorage`
- [ ] Works at 360 px wide
- [ ] `npm run typecheck && npm run lint` passes
