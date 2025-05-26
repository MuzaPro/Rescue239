# Project Requirements Document (PRD)

**Product Name:** Rescue 239 – Rapid Situation Reporting Suite  
**Version:** 0.1  
**Last updated:** {{TODAY}}

---

## 1 ‒ Executive Summary

Reserve‑based Search‑and‑Rescue teams (IDF Home‑Front Command, Unit 239) need a *zero‑training* digital toolset that lets them:

- **Field Teams** report casualty tallies from collapsed‑building worksites in ≤ 3 s on their own smartphones.

- **Commanders** visualise live numbers, geospatial distribution, and event history for up to 5 concurrent teams.

The suite ships as two web apps (same codebase, different routes):

| App            | Primary user  | Device                               | Key action time‑budget                         |
| -------------- | ------------- | ------------------------------------ | ---------------------------------------------- |
| `/#/responder` | Field rescuer | Any modern Android/iOS browser (PWA) | Tap → Send ≤ 3 s                               |
| `/#/commander` | Command post  | Laptop + projector                   | 1‑glance situational awareness (≤ 2 s refresh) |

No installs, works offline, encrypts data, minimal UI chrome.

---

## 2 ‒ Goals & Non‑Goals

### 2.1 Goals

1. **Sub‑3‑second input** for adding / updating casualty counts.

2. **Live aggregation** of casualties across ≤ 5 teams and ≤ 12 worksites.

3. **Offline resilience** – queue & sync when connectivity returns.

4. **Dark‑mode ready** for night ops.

5. **Training playback** (timeline scrub) – *out‑of‑scope MVP* but schema planned.

### 2.2 Explicit Non‑Goals (MVP)

- Medical triage detail beyond *Dead | Immediate | Delayed | Evac*.

- Resource logistics (stretchers, cranes, etc.).

- Push notifications / SMS fallback.

---

## 3 ‒ Personas & User Stories

### 3.1 Personas

- **Ori (Team Leader, 34)** – holds a stretcher in one hand, phone in the other; hates typing.

- **Major Levi (Commander, 47)** – juggles radios; needs macro view, alarms on stale data.

### 3.2 Top User Stories

| ID  | As a        | I want                                                       | So that                                  |
| --- | ----------- | ------------------------------------------------------------ | ---------------------------------------- |
| U1  | Team Leader | to bump counts with +/− buttons and tap *Update*             | I spend <3 s on the phone                |
| U2  | Commander   | to see a heatmap of worksites with totals and last heartbeat | I can allocate ambulances quickly        |
| U3  | Commander   | to switch to dark mode                                       | the display doesn’t kill my night vision |
| U4  | Ops Officer | to export CSV after the incident                             | we can run an after‑action review        |

---

## 4 ‒ Functional Requirements

### 4.1 Responder App

| Ref   | Requirement                                                                | Priority |
| ----- | -------------------------------------------------------------------------- | -------- |
| R‑F‑1 | Autodetect GPS ±accuracy; banner turns amber (>50 m) red (>100 m)          | Must     |
| R‑F‑2 | Four tiles (⚫ Dead, 🟥 Immediate, 🟨 Delayed, 🟩 Evac) each with − count + | Must     |
| R‑F‑3 | *Update* button disabled until state changes; optimistic UI & local queue  | Must     |
| R‑F‑4 | Vibrate 150 ms on successful queue/send                                    | Should   |
| R‑F‑5 | Works offline via Service‑Worker cache & IndexedDB queue                   | Must     |

### 4.2 Commander Dashboard

| Ref   | Requirement                                                              | Priority |
| ----- | ------------------------------------------------------------------------ | -------- |
| C‑F‑1 | Top bar shows global totals & ‘Last network heartbeat’                   | Must     |
| C‑F‑2 | Left sidebar lists teams with mini tallies                               | Must     |
| C‑F‑3 | Central map shows site circles; radius ∝ total, stroke colour by urgency | Must     |
| C‑F‑4 | Clicking a team spawns an info card under the map                        | Should   |
| C‑F‑5 | Event log pane streams all updates chronologically                       | Must     |
| C‑F‑6 | Dark‑mode toggle flips palette and map overlay                           | Must     |

---

## 5 ‒ Non‑Functional Requirements

| Category      | Requirement                                                               |
| ------------- | ------------------------------------------------------------------------- |
| Performance   | ≤ 300 KB gzipped bundle per app; <100 ms render after Service‑Worker load |
| Security      | JWT auth; HTTPS only; data contains no PII; AES‑256 at rest               |
| Resilience    | Queue holds 1,000 packets min; automatic back‑off retry every 15 s        |
| Accessibility | WCAG 2.1 AA contrast; all icons have aria‑labels                          |
| Browsers      | Chrome 95+, Safari 15+, Firefox 93+, Edge 95+                             |

---

## 6 ‒ Data Model & API

### 6.1 Packet Schema (`POST /api/report`)

```jsonc
{
  "deviceId"  : "string",     // hashed UUID
  "teamId"    : 2,            // 1‑5
  "siteCode"  : "A",          // A‑Z
  "counts"    : {
      "dead": 8,
      "imm" : 12,
      "del" : 0,
      "evac": 20
  },
  "loc"       : {
      "lat": 32.0785,
      "lon": 34.7749,
      "acc": 7        // metres
  },
  "ts"        : 1713891234567  // ms epoch
}
```

### 6.2 WebSocket Broadcast (`wss://…/stream`)

*Server fan‑out of each accepted packet*

---

## 7 ‒ Information Architecture

```
/                ──▶ index.html (landing)
/responder       ──▶ PWA shell  (field UI)
/commander       ──▶ dashboard  (map UI)
/api             ──▶ REST endpoints
  /reportPOST    (packets)
  /auth          (login)
/stream          ──▶ WebSocket live feed
```

---

## 8 ‒ Tech Stack

| Layer      | Choice                                 | Rationale                                 |
| ---------- | -------------------------------------- | ----------------------------------------- |
| Front‑end  | Vanilla JS + Lit HTML                  | tiny, fast to hydrate                     |
| Styling    | CSS variables + Tailwind utility layer | easy dark‑mode switch                     |
| PWA        | Workbox                                | offline cache + queue                     |
| Backend    | FastAPI (Python)                       | async + type hints, easy JWT              |
| DB         | PostgreSQL 15 (tiny table)             | geospatial ext. if future heatmaps needed |
| Deployment | Docker → Render / Fly.io               | free tier for demo                        |

---

## 9 ‒ Milestones & Timeline

| Week | Deliverable                                           |
| ---- | ----------------------------------------------------- |
| 1    | Hi‑fi Figma screens + clickable prototype             |
| 2    | Responder PWA shell w/ cache + local queue            |
| 3    | API + Postgres + WebSocket fan‑out                    |
| 4    | Commander dashboard (static data)                     |
| 5    | Real‑time integration; offline tests in airplane mode |
| 6    | Security hardening, README, user training deck        |

---

## 10 ‒ KPIs

- Median field input → dashboard display **< 5 s** under 3G.

- Successful offline queue flush rate **> 99 %** within 10 min after connectivity.

- Field‑tester Net Promoter Score **≥ +40** at end of pilot.

---

## 11 ‒ Open Questions

1. Do we need Hebrew localisation for on‑device UI?

2. Should the event log redact counts when sharing screenshots?

3. Is IDF VPN required or is commercial HTTPS acceptable for MVP?

---

## 12 ‒ Appendices

### A. Colour Tokens

| Token      | Light     | Dark      |
| ---------- | --------- | --------- |
| `--bg`     | `#f4f6f9` | `#0e151c` |
| `--orange` | `#f48b1e` | same      |
| `--blue`   | `#0068b9` | same      |

### B. Glossary

| Term         | Meaning                               |
| ------------ | ------------------------------------- |
| *Immediate*  | Red‑tag victims requiring urgent evac |
| *Delayed*    | Yellow‑tag – can wait                 |
| *Evac‑Ready* | Green‑tag – ambulatory victims        |

---

© 2025 Muza Productions / Rescue 239
