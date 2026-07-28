# TestEffortCardio

A single-file web app for running a **submaximal incremental cycle-ergometer
test** on an indoor bike, and recording it in the same shape as a cardiologist's
ergometry report — so a home test can be laid next to a clinical one.

Open `index.html` in any modern browser. No build step, no dependencies.

## Not a medical device

There is **no ECG**. A clinical stress test has a continuous 12-lead ECG and a
physician watching it, and that is where its diagnostic value lives — this app
cannot screen for myocardial ischaemia or for arrhythmia, and does not try to.

What it does is keep the rest of the record: load, heart rate, blood pressure,
SpO₂, perceived exertion, reason for stopping, and the recovery curve. That is
useful between clinical visits. It is not a substitute for one. Talk to your
doctor before doing an effort test alone, and stop at the first sign of chest
discomfort, marked breathlessness, dizziness or feeling unwell.

## The test

```
Repos  →  Palier 1 … n  →  STOP  →  Après l'effort  →  Rapport
          2 min each,            measurements at
          load steps up          1, 3 and 5 min
```

Default protocol, matching a standard Swiss ergometry test: rest, then **50 W and
+25 W every 2 minutes**, no warm-up stage, symptom-limited stop, recovery
measurements at 1/3/5 min. All adjustable — but keeping it fixed year to year is
what makes tests comparable.

## Design notes

- **The clock never stops for data entry.** A measurement creates an open card in
  a queue and is tagged to the palier it was *measured in*, not the one it was
  typed in. Stopping to type would let heart rate fall and destroy the
  HR-vs-load relationship the whole report rests on.
- **Fields open prefilled** from a regression over the paliers already recorded,
  so a typical entry is a couple of taps on large ± steppers rather than nine
  digits. An untouched field stays visibly flagged as an estimate and is not
  recorded as a measurement.
- **Cadence is the largest number on screen.** A consumer trainer is not a
  constant-power ergometer: at a fixed resistance level, watts fall when cadence
  falls. So cadence is coached, recorded, and paliers ridden below target are
  flagged in the report.
- **The resistance level is the exact axis; watts is a label.** "Level 9 at
  24 km/h" is reproducible in two years on the same magnetic brake whether or not
  the console's watt estimate is accurate. Metrics are computed on both.
- **Blood pressure in cmHg**, as Swiss reports print it — `13,5/8` for 135/80 —
  and the double product is computed on the cmHg figure.

## Metrics

Formulas taken from a real ergometry report and verified to reproduce its printed
values exactly:

| Quantity | Formula |
|---|---|
| Watts/kg | peak W / weight |
| METs | 4 × W/kg |
| METs théorique | 18 − 0.15 × age (Morris, men) |
| FMT (max HR) | 220 − age — *not* Tanaka |
| Sous-maximale | 0.85 × FMT |
| Double produit | systolic in **cmHg** × HR |

Plus, beyond the clinical report: total work, HR/load slope, **PWC 130/150/170**
(load interpolated at those heart rates — a submaximal index, which matters
because a home test usually stops well short of the FMT a supervised test
reaches), systolic slope per 25 W, and heart-rate recovery at 1/3/5 min.

## Features

- Rest → paliers → recovery state machine with a wake lock and a progress bar
- Audio cues (Web Audio, synthesised) plus French speech: measure now, stage
  change, last five seconds, cadence sagging, recovery
- Report: full palier table in the clinical column order, three charts
  (HR vs load with previous tests overlaid, BP vs load, HR over time including
  recovery), descriptive flags with the threshold always stated
- History, comparison table across tests, JSON export/import, CSV, print
  stylesheet for a one-page hand-out
- Three themes, French / English / German
- Checks once per launch for a newer deployed version and reloads past the cache
  (iOS Safari otherwise serves a stale copy for days)

## Privacy

Everything stays in the browser's `localStorage`. Nothing is uploaded anywhere.

Two consequences:

1. **A test that only exists in localStorage is not saved.** A browser can clear
   its own storage without warning. The app offers a JSON download at the end of
   every test — take it.
2. **This repository is public, so no personal medical data belongs in it.**
   `.gitignore` excludes `REFERENCE-*.md`, `*.doc`, `*.pdf`, `reports/` and
   `data/` for that reason. Keep report transcriptions and exported tests out of
   the repo.

## Credits

Structure, theming, audio synthesis, wake lock and the cache-busting update check
are carried over from [Atmen](https://github.com/fservais911/Atmen).
