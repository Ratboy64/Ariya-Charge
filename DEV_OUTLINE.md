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

## v1.1 — SHIPPED

- [x] "Plug in at" start-time slider (now → +24h, 15-min steps, copper track)
  - Shifts "done around" clock and all-targets table finish times
  - Session-only: intentionally NOT persisted (a saved offset is relative to
    a "now" that no longer exists — always reopens at "Now")
  - Minute-tick re-render keeps clock times honest while the app stays open

## v1.2 — SHIPPED

- [x] Ambient temperature slider (-10°F to 110°F) replacing cold toggle
  - `TEMP_DC` / `TEMP_AC` piecewise multiplier curves via shared `interp()`
  - Old boolean `cold` in saved state migrates to 30°F / 70°F on load
  - DC taper chart reflects temperature in real time
- [x] Cost estimator ($/kWh)
  - Per-charger rates persisted: `state.rates = {L1, L2, DC}` (home vs. DC differ 3-5x)
  - Billed on wall-side energy: `packKwh / wallEff()` (L1 0.82, L2 0.90, DC 0.97)
  - Cost shown in result meta line + 4th column of all-targets table

## v1.3 — SHIPPED

- [x] Range estimator with adjustable efficiency (`state.miPerKwh`, default 3.0)
  - `rangeAt(pct) = pct/100 x 87 kWh x miPerKwh`
  - Result card: range at target + miles added; "Already there" state shows range at current %
  - 5th column (green) in all-targets table; padding tightened for 5 columns on ~380px
  - Persisted with the rest of state; input clamped 1.0-6.0 mi/kWh

## v1.4 — SHIPPED

- [x] Calibration & session log (forecast vs. actual)
  - `logSession(from, to, actualHours)`: measured = rawModelHours / actualHours
    computed against the UNCALIBRATED model so factors never compound
  - Per-charger factor `state.cal = {L1, L2, DC}`, EWMA alpha 0.30
  - Safety clamps: factor [0.50, 1.25], single measurement [0.40, 1.60]
  - `hoursToTarget(from, to, useCal)` — useCal=false for raw model
  - Sessions in `ariya_sessions` (max 50 FIFO): date, charger, from/to %,
    actual vs. model minutes, measured factor, factor-after, tempF
  - UI: per-charger factor badges, log form (uses selected charger + temp),
    last-8 history list, reset button
  - Aging signal: with 5+ sessions per charger, first-half vs. recent-half
    average measured factor; >5% sustained drop flagged as possible pack
    aging / charger derating
  - Result meta line shows "calibrated" tag when a factor is active

---

## v1.1 — Candidate features

### Per-minute / idle-fee DC billing
- v1.2 shipped $/kWh billing; some stations bill $/min or add idle fees
- Input: optional $/min rate + idle fee; blend with kWh rate

### Seasonal efficiency coupling
- v1.3 shipped the range estimator with manual mi/kWh
- Candidate: auto-derate range with the ambient temperature slider
  (cold-weather driving efficiency drops ~20-30% below freezing —
  cabin heat + denser air + cold pack)

### Departure planner (reverse mode)
- Forward direction shipped in v1.1 (start-time slider). Reverse remains:
- Input: "I need to leave at 7:30 AM at 80%"
- Output: "Plug in by 11:58 PM tonight" (deadline − duration)
- UI: mode switch at top of result card; reuses `hoursToTarget()` unchanged

### Charge scheduling reminder
- Local notification via `Notification` API when plug-in time approaches
- **iOS caveat:** Web Push on iOS requires the PWA to be installed to home
  screen (iOS 16.4+) — same class of Safari restriction as the Web Speech
  gesture-gating solved in DriverGuard Sprint 1

### Live session timer
- v1.4 shipped manual session logging; candidate: "Start charging" button
  that timestamps start automatically, then one tap + end % on completion
- Would remove the two most error-prone manual fields (start %, duration)

### Multi-vehicle support
- Vehicle profiles: capacity, onboard AC limit, DC curve points
- Preset library (Ariya 63/87, Leaf 40/62, common others)
- Storage: `ariya_vehicles`

---

## Known model limitations

1. **DC curve is an approximation** — real Ariya sessions vary with pack temperature, initial SoC when plugged in, and station power sharing
2. **No preconditioning model** — the Ariya lacks native DC preconditioning; the ambient slider models *ambient*, not actual pack temperature (a pack warmed by highway driving charges faster than ambient suggests)
3. **Top-end taper (>95%) varies pack to pack** — cell balancing time is unpredictable; treat >90% estimates as ±15%
4. **L1 amperage assumed 12A** — some EVSEs allow 8A/10A selection; could expose as a sub-option

---

*Last updated: v1.0 release*
