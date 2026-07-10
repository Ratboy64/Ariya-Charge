# Ariya Charge Timer — Development Outline
**Version 1.0 | Single-file PWA (index.html)**

> Companion project to DriverGuard AI. Same architecture principles: single file, localStorage persistence, no build step, GitHub Pages deployment, iPhone Safari first.

---

## Architecture

- **All logic in `index.html`** — vanilla JS, no frameworks
- **State object** persisted to `localStorage` (`ariya_charge_state`)
- **Power models** return kW delivered *to the pack* at a given SoC:
  - `powerL1(soc)` — flat 1.44 kW × 0.82
  - `powerL2(soc)` — min(station, 7.2) × 0.90, taper >96%
  - `powerDC(soc)` — piecewise-linear `DC_CURVE` interpolation, capped at station × 0.97
- **`hoursToTarget(from, to)`** — midpoint-sampled numerical integration in 0.5% steps
- **SVG taper chart** rendered from the same `DC_CURVE` used for math (single source of truth)

## Key constants

```
CAP = 87 kWh usable  →  KWH_PER_PCT = 0.87
Onboard AC limit: 7.2 kW
DC_CURVE peak: 130 kW @ 20% SoC
Cold multipliers: DC ×0.60, AC ×0.95
```

---

## v1.0 — SHIPPED

- [x] Current SoC slider (1–99%) with live battery visual
- [x] Target chips (80/85/90/100) + ±1% fine-tune
- [x] Battery bar with animated copper "target span" overlay
- [x] Three charger cards with sub-options (L2: 3.3/6.6/7.2 kW; DC: 50/62.5/150/350 kW)
- [x] Cold battery toggle
- [x] Result card: duration, done-at clock time, kWh added, avg kW
- [x] All-targets comparison table
- [x] DC taper curve SVG with session band shading
- [x] localStorage persistence
- [x] PWA install (manifest + icons + apple-touch meta)

---

## v1.1 — Candidate features

### Charging cost estimate
- Input: $/kWh (home rate) or $/min / $/kWh (DC station rate)
- Output: session cost next to duration
- Storage: `ariya_cost_config`
- Note: DC stations bill on *wall-side* energy — divide pack kWh by efficiency

### Range added
- Convert kWh to estimated miles: Ariya e-4ORCE ≈ 2.9 mi/kWh, FWD ≈ 3.2 mi/kWh
- Toggle in a small settings row; show "+87 mi" next to kWh added
- Optional: seasonal efficiency adjustment tied to cold toggle

### Departure planner (reverse mode)
- Input: "I need to leave at 7:30 AM at 80%"
- Output: "Plug in by 11:58 PM tonight" (inverse of current calculation)
- UI: mode switch at top of result card

### Charge scheduling reminder
- Local notification via `Notification` API when plug-in time approaches
- **iOS caveat:** Web Push on iOS requires the PWA to be installed to home
  screen (iOS 16.4+) — same class of Safari restriction as the Web Speech
  gesture-gating solved in DriverGuard Sprint 1

### Session log
- "Start charging" button → logs actual start; "Done" → compares actual vs. predicted
- Over time, auto-tune the efficiency constants from real sessions
  (same philosophy as DriverGuard's drive-baseline calibration, with
  safety clamps so one bad data point can't skew the model)
- Storage: `ariya_sessions` (max 50, FIFO — same pattern as `dg_trip_summaries`)

### Multi-vehicle support
- Vehicle profiles: capacity, onboard AC limit, DC curve points
- Preset library (Ariya 63/87, Leaf 40/62, common others)
- Storage: `ariya_vehicles`

---

## Known model limitations

1. **DC curve is an approximation** — real Ariya sessions vary with pack temperature, initial SoC when plugged in, and station power sharing
2. **No preconditioning model** — the Ariya lacks native DC preconditioning; cold toggle is a blunt instrument
3. **Top-end taper (>95%) varies pack to pack** — cell balancing time is unpredictable; treat >90% estimates as ±15%
4. **L1 amperage assumed 12A** — some EVSEs allow 8A/10A selection; could expose as a sub-option

---

*Last updated: v1.0 release*
