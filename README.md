# AttendWise

**Smart Attendance. Smarter Decisions.**

A student-facing dashboard that turns attendance numbers and the real semester timetable into an action plan.

## Overview

**Problem.** College portals show your attendance percentage, but not what to do next: how many classes you can miss, how many you must attend to recover, whether 75% or 90% is still reachable, or when recovery has become mathematically impossible.

**Solution.** AttendWise combines your attendance, your section's timetable, today's date and a planning date, then calculates the answer for every subject. "Your attendance is 72%" becomes "You must attend 16 of the remaining 18 classes, and you can miss only 2."

## Features

- 10 sections, timetables transcribed from the department PDFs (semester 29 Aug – 29 Nov 2026)
- Attended/conducted or percentage-only input, with validation
- Per subject: 75% and 90% requirements, safe misses, maximum possible attendance, status, irreversible-detention alert
- **Attendance Health** analytics (Chart.js): subject bars with 75%/90% markers, health doughnut, current vs projected; switchable to "after leave" or "after what-if"
- Plan Ahead (date picker + slider), Upcoming Classes, What If? simulator
- **Leave Simulator**: Medical, OD or Custom leave, using actual timetable periods
- **Attendance Advisor**: floating chat that answers from the same engine
- Demo data that shows all four states, register-number login (local), light/dark theme, responsive layout

Screenshots: _add images to `docs/` and link them here._

## Tech Stack

HTML, CSS, vanilla JavaScript (ES modules), [Vite](https://vitejs.dev) for dev/build, [Chart.js](https://www.chartjs.org) 4 (loaded from a CDN), Node's built-in test runner. No framework.

## Attendance Intelligence

With `A` attended, `C` conducted, `R` remaining scheduled classes (all whole numbers):

| Quantity | Formula |
|---|---|
| Current % | `A / C × 100` |
| Must attend for target `t` (75 or 90) | `max(0, ceil(t/100 × (C + R) − A))` |
| Safe misses | `R − required` (none if `required > R`) |
| Maximum possible % | `(A + R) / (C + R) × 100` |
| Irreversible detention | maximum possible `< 75%` (`required75 > R`) |

Status per subject: `SAFE` (≥ 90%), `CAUTION` (75–90%), `DETENTION RISK` (< 75%, recoverable), `IRREVERSIBLE` (75% unreachable). The 90% target means "at least 90%". Everything lives in `src/engine/attendance.js`; charts, leave simulator and advisor all call the same `calc()`.

## Leave Simulator

Dates are expanded into real calendar days, then matched to the section timetable, so only periods that actually exist are counted, per subject (weekends and configured holidays are skipped). With `q` affected classes:

- **Counts as absent:** `A' = A`, `C' = C + q`
- **Counts as attended:** `A' = A + q`, `C' = C + q`
- **Excluded from denominator:** `A' = A`, `C' = C`

`R' = R − q` in all cases, then the normal engine recomputes status. Defaults: Medical/Custom → absent, OD → attended, but you can change it. College policy varies; treat results as estimates.

## AI Attendance Advisor

- **Local deterministic advisor (default, no key needed).** `advisor()` in `src/components/chatbot.js` recognises intents (skip tomorrow, leave over dates, next N classes, how many can I miss, reach 75/90, riskiest subject, "recover from 68%", what to attend this week) and answers using the engine. It never invents numbers; unknown questions get a helpful fallback.
- **Optional external AI.** `buildContext()` creates a structured snapshot (section, dates, subjects, requirements, upcoming classes, what-if and leave results). If you set `VITE_ADVISOR_ENDPOINT`, the app POSTs `{ question, context }` to **your backend** and expects `{ reply }`; on any failure it falls back to the local advisor. API keys must stay in server-side environment variables, never in this repo or frontend code. No backend is included.

## Round 2 — Free Class Locator

Timetable-based free classroom discovery, built on the same timetable data as the attendance predictor.

- **What it does:** shows which rooms have no class now or at a chosen date/time, on which floor, for how long, and whether they stay free for the whole time you need. Rooms are grouped by floor, shown on a floor map, and each result explains why it matched.
- **AI Room Finder:** type "AC room on the ground floor for 2 hours". A local deterministic parser (`src/engine/naturalLanguageRoomParser.js`) works offline and shows what it understood. An optional external interpreter can be enabled with `VITE_ROOM_AI_ENDPOINT` (your backend; no keys in the frontend); any failure falls back to the local parser. The AI only produces filters; the deterministic engine decides availability.
- **Data:** the nine 2026-27 timetables (II ECE DS A/B, II BME, III ECE A/B/DS, III BME, IV ECE A/B). Rooms come from each PDF's printed venue (e.g. `IST 518/FN`) and explicit lab/slot rooms (e.g. `LAB-108/309`). The I Year PDF is the 2024-25 timetable and is excluded from availability.
- **Room metadata:** only what the PDFs support. Floor is derived from the first digit of a room number and labelled as derived. **AC and capacity are not in the timetables, so they show "Unknown"** and filters on them return unverified candidates instead of claims.
- **Conflicts:** two sections claiming one room at once show "Data conflict" (currently Monday 10:50, room 107: II ECE DS B vs III BME).
- **Important limitation:** availability is **inferred from these timetables**. It is not physical occupancy detection, and classes of other departments are not included.
- **Docs:** [`docs/ROUND2_ARCHITECTURE.md`](docs/ROUND2_ARCHITECTURE.md), [`docs/ROUND2_FEATURES.md`](docs/ROUND2_FEATURES.md).

## Project Structure

```
index.html            page shell + Chart.js CDN
src/main.js           entry point, session restore
src/app.js            navigation, header, theme, render()
src/state.js          shared state St, save(), attendance input validation
src/data/             semester.js (dates, holidays), timetable.js (all timetables), rooms.js (venues, slot rooms, periods)
src/engine/           attendance.js, timetable.js, planner.js, leaveSimulator.js (pure logic),
                      roomAvailability.js, roomSearch.js, naturalLanguageRoomParser.js, roomAI.js (Round 2)
src/components/       freeClassLocator (Round 2), dashboard, attendance, planner, upcoming, whatIf, leaveSimulator, analytics, chatbot, landing, common
src/auth/auth.js      local login
src/utils/            dates.js, storage.js (all localStorage access)
src/styles/main.css   styles, light/dark tokens
tests/                engine tests + app smoke test
```

## Installation

```bash
npm install
npm run dev      # http://localhost:5173
```

## Build

```bash
npm run build    # output in dist/
npm run preview
npm test         # 86 tests (Round 1 + Round 2)
```

## Usage / Demo

1. Create an account with a register number (e.g. `RA2411030050054`) and any password.
2. Select your section, then **My Attendance → Load Demo Attendance**.
3. Dashboard: overall %, Attendance Health charts, subject cards (all four states), irreversible banner.
4. **Leave Simulator**: try a 3-day Medical Leave. **What If?** and **Plan Ahead** update the charts too.
5. Open **✨** and ask "Can I skip tomorrow?" or "How many can I miss?"

## Deployment

`npm run build`, then upload `dist/` to any static host (Vercel, Netlify, GitHub Pages, Cloudflare Pages). Build command `npm run build`, output directory `dist`.

## Limitations

- Login is a browser-only prototype (passwords hashed with PBKDF2 in localStorage). It is **not** secure server-side authentication and cannot verify identity or recover passwords.
- Attendance data stays in the browser on that device unless a backend is added.
- Actual OD/medical-leave rules vary by college; results are mathematical estimates.
- **I Year timetable is older 2024–25 data and is not verified for 2026–27**, so the section shows "Timetable data requires verification" instead of guessed data.
- `HOLIDAYS` in `src/data/semester.js` is empty. Add official holidays before real use.
- Lab blocks not listed in some subject tables (III BME, II BME) and "B-Proj" periods appear as printed and are counted as class periods.
- Percentage-only input estimates classes conducted from the timetable.
- Chart.js loads from a CDN, so charts need an internet connection.

## Future Scope

College portal integration, real student accounts, cloud database, faculty/admin dashboard, automated timetable import, official holiday calendar, notifications, mobile app.
