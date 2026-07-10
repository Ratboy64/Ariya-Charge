# Ariya Charge Timer

A single-file Progressive Web App that estimates charging time for a **2024 Nissan Ariya (87 kWh usable pack)**. Enter your current state of charge, pick a target (80 / 85 / 90 / 100%, or fine-tune to any value), select the charger you're plugged into, and get time-to-target plus a "done around" clock time.

Built mobile-first for iPhone Safari. No build step, no dependencies, no API keys. Deployed via GitHub Pages.

## How the estimate works

The app never does a naive `kWh ÷ kW` division. It integrates in 0.5% steps (0.435 kWh each), asking a power model how many kW the pack accepts *at each state of charge*:

| Charger | Model |
|---|---|
| **Level 1** (120V) | 1.44 kW at the wall × 82% efficiency ≈ 1.2 kW to pack, flat |
| **Level 2** (240V) | min(station kW, 7.2 kW onboard limit) × 90% efficiency, slight taper above 96% |
| **DC Fast** (CCS) | 13-point piecewise taper curve: ~130 kW peak near 20% SoC → ~42 kW at 80% → single digits near 100%, capped by station max (50 / 62.5 / 150 / 350 kW) |

An **ambient temperature slider** (-10°F to 110°F) applies a piecewise power multiplier: DC charging is heavily curtailed in the cold (~55% power at freezing, ~30% at -10°F — the BMS limits current into cold cells to prevent lithium plating) and mildly throttled above ~95°F; AC is barely rate-limited by temperature, carrying only a small cold penalty for battery-heater overhead.

A **cost estimator** takes a $/kWh rate (remembered separately for home AC vs. public DC, since they differ 3–5×) and computes session cost on the *wall side* — pack energy divided by charger efficiency — because that's what the meter or station actually bills. Costs appear in the result line and per-target in the comparison table.

When DC Fast is selected, a live taper-curve chart appears with your session shaded on it — a visual explanation of why 80→100% takes nearly as long as 20→80% on a fast charger.

Validated against Nissan's published figures: ~10.5 h for a full Level 2 charge, ~40 min for 20→80% DC on a 130 kW-capable station.

## Files

| File | Purpose |
|---|---|
| `index.html` | The entire app — markup, styles, logic |
| `manifest.json` | PWA manifest for home-screen install |
| `icon-192.png` / `icon-512.png` | App icons |
| `.nojekyll` | Tells GitHub Pages to serve files as-is (skip Jekyll processing) |
| `.gitattributes` | Normalizes line endings; marks PNGs binary |
| `README.md` | This file |
| `DEV_OUTLINE.md` | Feature roadmap and technical notes |

## Deploy to GitHub Pages

1. Create a new public repository (e.g. `ariya-charge`)
2. Upload all files in this package to the repo root
3. Settings → Pages → Source: **Deploy from a branch** → Branch: `main` / `(root)` → Save
4. Wait ~1 minute, then visit `https://<username>.github.io/ariya-charge/`

## Install to iPhone home screen

1. Open the GitHub Pages URL in **Safari**
2. Tap the Share button → **Add to Home Screen**
3. The app launches fullscreen with its own icon, like a native app

## Storage

Your last-used inputs (current %, target, charger, cold toggle) persist in `localStorage` under the key `ariya_charge_state`. No data leaves the device.

## Disclaimer

Estimates assume 87 kWh usable capacity and typical Ariya charge behavior. Real-world times vary with battery temperature, pack age, station power sharing, and preconditioning. Treat estimates above 90% SoC as ±15%.
