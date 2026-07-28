# TestEffortCardio — Concept

A single-file web app that guides and records a **submaximal incremental cycle
ergometer test** on an indoor bicycle, so the numbers can be compared against
the cardiologist's stress test done every 2 years.

Draft v0.1 — open questions marked **[?]**.

---

## 1. What this is, and what it is not

**Is:** a stopwatch, a load coach, a structured data logger, and a report
generator for a self-administered bike test. It tells you what load to ride,
when to take a measurement, records what you enter, and afterwards computes the
same derived metrics a cardiologist's report shows, so a home test in year 1 can
be laid next to a clinical test in year 0 and year 2.

**Is not:** a medical device, and not a stress test in the clinical sense. It is
a self-tracking log. The app should say so once, plainly, on first run — and
carry a short **stop-now card** (§7) rather than pretending the question doesn't
arise.

**Where exactly the gap is.** The clinical report this app is modelled on has
eight per-palier columns, and we can fill seven of them. The one we cannot is the
**ECG** — 6 modified standard leads plus 6 precordial. The report's findings on
myocardial ischaemia and on arrhythmia rest *entirely* on those leads. That is
where the diagnostic value of a clinical test lives, and no amount of app design
reaches it.

What a home test *can* track is the rest of the report: blood-pressure behaviour
under load and in recovery, and physical capacity versus the previous test. That
is worth having between clinical visits — it is just not a substitute for one,
and the app must never imply it screens for ischaemia.

---

## 2. Test structure

```
 SETUP  →  REST  →  WARM-UP  →  STAGE 1 → STAGE 2 → … → STAGE n  →  RECOVERY  →  REPORT
           (opt)     (opt)      ├─ 2:00 at target load                  ├─ 1 min
                                └─ measurement window                   ├─ 3 min
                                                    ↑ rider says STOP   └─ 5 min
```

### Rest
Seated on the bike, not pedalling. One HR + BP reading. This is the baseline
every later delta is measured from, and the cardiologist's report always has it.

### Warm-up (optional, default off)
E.g. 3 min at the start load. Not counted as stage 1.

### Stages — *paliers*
Each palier is a fixed duration (**default 2:00**, Settings) at a fixed
resistance level, ridden at a constant target cadence. At the end of the palier
the resistance steps up and the next one begins, with no break in the clock.

### Recovery
The part that is easy to skip and clinically the most informative. After STOP,
keep the clock running and prompt for HR + BP at **1, 3 and 5 min** (list
configurable). Heart-rate recovery at 1 min is a standalone prognostic number —
worth having.

### Report
Table of every stage, derived metrics (§6), charts (§8), export.

---

## 3. Defining the load — NordicTrack G LE Recumbent

The bike is a **NordicTrack G LE Recumbent** (model `NTEX99025`): **26
electronically controlled resistance levels**, **Silent Magnetic Resistance**,
and a **5" LCD** console. ("4C16" was probably a batch or date code on the
serial decal — Icon model numbers look like `NTEX99025`.)

Two things this settles, one it doesn't.

**Settled, and good news: the resistance is magnetic.** Magnetic braking is
stable over years in a way a friction pad is not, so "level 9" should mean the
same thing at your next test in 2028 as it does today. That is the foundation the
whole year-over-year comparison rests on, and it holds.

**Settled, and important: SmartAdjust must be off.** The G LE auto-scales
resistance during iFIT workouts. If iFIT is driving the brake, the protocol is
not reproducible and the test is worthless for comparison. Manual mode only —
the app's pre-test checklist must say so explicitly.

**Mostly settled: what the console displays.** Your recollection is **effort
level + speed in km/h + probably watts** — to be confirmed at home. No spec sheet
mentions RPM, so the working assumption is **level, km/h and watts, but no
cadence in rpm**. That's a perfectly workable set (see §3.1), and if watts turn
out to be absent the app still works (§3.3).

### 3.1 The app asks the console what it can show

Rather than hard-coding a spec I can't verify, **first-run setup asks you to tick
which fields your console displays**: Watts / RPM / Speed / Resistance level.
Everything downstream adapts. Ten seconds with the bike in front of you, and it
removes the dependency on my research being right.

Given your recollection, the app's **default assumption** is level + km/h +
watts, with **km/h as the cadence variable** — which is fine, because on a
stationary bike speed is cadence times a constant (§3.2), so holding a target
km/h *is* holding a target cadence.

It also settles your original question — "increment Watts = speed in km/h,
rotations per minute". Those are **not three independent things**:

- **Resistance level** is what you set. Always available.
- **Cadence** is what you hold. Shown as RPM if the console has it, otherwise as
  **km/h — which on a stationary bike is just cadence times a constant.** So if
  there's no RPM field, holding a target speed *is* holding a target cadence, and
  km/h becomes the cadence proxy with no loss.
- **Watts**, if shown at all, is *computed by the console* from those two. Icon
  manuals that do document it call it "the **approximate** number of watts" — no
  strain gauge, no calibration.

So a stage's target is a **pair**, and the app displays whichever units your
console speaks:

```
Palier   Level   Cadence        → Watts (console estimate, if available)
   1       3     70 rpm / 24 km/h        ~50
   2       5     70 rpm / 24 km/h        ~75
   3       7     70 rpm / 24 km/h       ~100
   …
```

Note the cadence column doesn't change — see §3.2.

### 3.2 This is not a constant-power brake — so cadence *is* the load

This is the most important practical point in the whole document. A clinical
ergometer holds watts **constant**: pedal slower and the brake bites harder to
compensate, so the load you were prescribed is the load you get. Your bike does
not do that. At a fixed resistance level, **watts fall when your cadence falls.**

Which means: late in the test, exactly when it's hardest, a sagging cadence
silently reduces the load — and the stage you record as 150 W was really 130 W.
That corrupts the HR-vs-watts relationship the whole report rests on.

So the app must:

- **Coach cadence as the primary number on screen** — bigger than watts, bigger
  than anything except the clock. Target cadence with a clear in-range /
  drifting indication, and an audible nudge if it sags more than ~5 RPM (or
  ~2 km/h) below target for more than a few seconds.
- **Record actual cadence at every measurement**, not just the target.
- **Flag paliers where cadence wasn't held**, so a palier ridden at 62 RPM
  instead of 70 is visibly not comparable to the same palier last year.

**One fixed target cadence for the whole test** — 70 RPM (or its km/h
equivalent) is a reasonable default — is much better than varying it per palier.
It keeps one variable constant, so resistance level is the only thing that
changes, which is exactly what makes the protocol reproducible.

### 3.3 The protocol axis is (level, cadence); watts is a derived label

Here is the property that rescues the accuracy problem entirely. For comparing
*your own* tests across years, we do not need accurate watts — we need
**reproducible load**. "Level 9 at 70 RPM" is exactly reproducible in two years
on the same magnetic brake, whether or not the console's watt estimate is right,
and whether or not the console shows watts at all.

So the stored protocol is a **load table keyed on (resistance level, target
cadence)**, and watts is a *label* — recorded as the console shows it if it
shows it, and marked in every report as an estimate rather than a calibrated
figure. The cardiologist should be able to see that distinction on the printout.

**The consequence worth stating outright: every metric can be computed on the
level axis instead of the watt axis.** PWC170 doesn't need watts — "resistance
level at HR 170" is a perfectly good number, exactly reproducible, and it is the
*better* year-over-year comparison because it has no estimation error in it at
all. So the report carries both:

- **PWC170 = level 8.4** — exact, for comparing your 2026 test to your 2028 test
- **PWC170 ≈ 165 W** — estimated, for the conversation with the cardiologist

If the console turns out to show no watts, the first line still works perfectly
and the app is fully useful. That's the design being robust to §3's open
question rather than blocked on it.

### 3.4 Recumbent vs upright — a caveat that must be visible

Your bike is **recumbent**; a clinical stress test is essentially always
**upright** (or a treadmill). These are not interchangeable. Semi-recumbent
cycling recruits muscle differently, is back-supported, and typically yields a
somewhat **lower peak power and peak HR** for the same subjective effort. Posture
also shifts blood pressure readings.

So there is a *systematic modality offset* sitting on top of the watt-estimation
error. Two rules follow:

- Every stored test records its **modality** (home recumbent / clinic upright),
  and the trend charts label it. Home tests trend against home tests. A clinical
  point plotted on the same axes must be visually distinguishable, not silently
  merged into the same line.
- The report should never claim your home peak is comparable in absolute terms
  to the clinic's. The comparison that *is* valid is **shape and change**: is the
  HR-vs-load slope steeper than last time, is recovery slower, has PWC170 moved.

### 3.5 Later: cross-calibrate against the clinical test using HR

Once there's a clinical test in the history, the two can be bridged. Clinical
PWC170 is in **true** watts; home PWC170 is in level-units at known cadence. HR
is the common currency, so the pair gives a rough watts-per-level figure for your
bike in your riding position.

Rough — fitness change between tests and the modality offset in §3.4 both
confound it. So this is a "level 9 is worth roughly 150 W for you" hint, not a
calibration certificate, and it should be labelled that way. Still worth having:
it costs nothing but arithmetic once two tests are stored, and it's the only
route to a watt number at all if the console doesn't show one.

---

## 4. Measurement timing — decided: pedal through it

**The clock never stops and the load steps up on time.** Measurements are taken
during the last 30 s of each stage while still pedalling, as the cardiologist
does it, because stopping lets HR fall and destroys the HR-vs-load relationship
the whole test is built on.

You ride **alone**, which makes this the hard case: nobody else can hold the
phone or the cuff. Three consequences drive the whole UI.

### 4.1 The clock is never blocked by data entry

Entry is a **queue, not a modal step**. When a stage's measurement window opens,
the app creates an open entry card for *that stage* and cues you. The card can
be filled in during the last 30 s, or during the next stage, or three stages
later, or after the test is over. The value is tagged to the stage it was
**measured in**, never the stage it happened to be typed in.

This matters because the alternative — a sheet that must be dismissed before the
test continues — either stops the clock (which we've ruled out) or silently
loses the reading when you're too busy breathing to deal with it. A visible
"2 cards open" badge is a much better failure mode than a lost stage.

### 4.2 Entry is prefilled from the trend, so it's a few taps

The recumbent position helps here too: back supported, hands not needed for
balance, and the console has a **device shelf** to hold the phone at a readable
distance. You can genuinely use two hands for a moment, which an upright bike
wouldn't allow. So this is less desperate than it would otherwise be — but the
design should still assume you're breathing hard and don't want to aim carefully.

No keyboard. Big ± steppers, thumb-sized. The trick that makes it workable: each
field opens **already holding its most likely value**, extrapolated from the
paliers already recorded.

- HR rises roughly 10–15 bpm per 25 W step → prefill = last HR + observed slope
- Systolic rises roughly 10 mmHg per 25 W → prefill = last systolic + slope
- Diastolic barely moves under load → prefill = last diastolic

Typical entry then becomes one to three taps rather than nine digits. Steppers
move ±1 on tap and ±5 on hold. Every field shows whether it's still the
prefilled guess or has been confirmed — an unconfirmed prefill must never be
recorded as if it were a measurement, so the card tracks "touched" per field and
the report marks untouched values as estimated or drops them.

### 4.3 Blood pressure — the recumbent bike is a real advantage here

I had this filed as the app's biggest practical risk. On an **upright** bike,
alone, an automatic oscillometric cuff usually fails while pedalling: the arm is
loaded holding the handlebars, it moves, and the cuff reads noisily or errors
out. A clinical test avoids this by having a second person with a manual
sphygmomanometer.

**Your recumbent bike largely removes the problem.** You're seated in a
supported chair position, your back is braced, and — the decisive part — **your
arm can rest completely still on the armrest while you pedal.** That is close to
the posture an automatic cuff is designed for. Pedal-through BP alone is
genuinely plausible here, which it would not have been on an upright.

Two things still to respect:

- **Posture affects the reading.** A semi-recumbent BP is not identical to a
  clinical upright one, which is part of the modality offset in §3.4. Record the
  posture with the test; don't pretend the numbers are interchangeable.
- **Cuff timing needs a head start.** An automatic cuff takes 30–40 s to
  inflate, measure and settle. So the *measure* cue should fire at **T−45 s**,
  not T−30 s, so the reading lands inside the last 30 s of the palier rather
  than after the load has already stepped up. Worth making this a Setting, since
  it depends on your cuff.

HR and BP are still treated as separate streams:

- **HR — every palier.** A watch or chest strap reads continuously; you glance
  and confirm.
- **BP — scheduled, best-effort.** Settings picks which paliers ask: every one,
  every second one, or rest + peak + recovery only. A failed or skipped reading
  is a blank cell; a palier with no BP is still a complete palier.

If it turns out not to work in practice after all, the fallback remains BP at
**rest, peak and recovery** — enough for peak systolic and the double product,
losing only the per-palier BP curve. Either way the report shows **which**
paliers have measured BP rather than interpolating over the gaps.

**Later idea, not v1:** spoken entry ("cent trente-cinq sur quatre-vingt-dix")
would solve entry-while-pedalling neatly, but `SpeechRecognition` is absent or
unreliable on iOS Safari and misheard numbers are worse than no numbers here.
Revisit once the rest works.

### 4.4 Cues — eyes off the phone

Riding alone at the limit, you should not have to look at the screen to know
where you are. Ported straight from Atmen's Web Audio synthesis, three distinct
cues plus optional French speech:

| Moment | Cue | Spoken (FR) |
|---|---|---|
| T−45 s (cuff start, §4.3) | rising double tone | « tension, maintenant » |
| Last 5 s of a palier | soft ticks | — |
| Palier end / load step | firm single tone | « palier 4, niveau 9 » |
| Cadence sagging >5 rpm | soft repeating nudge | « cadence » |
| Test end → recovery | descending three-tone | « récupération » |
| Recovery measurement due | rising double tone | « mesurez » |

The new resistance level is **announced out loud** at each step, so the target
reaches you without a glance — which matters because on this bike you have to
reach over and press the level buttons yourself. Screen stays awake via the wake
lock throughout.

The cadence nudge is the one genuinely novel cue versus Atmen, and per §3.2 it's
the one doing the most work to keep the test valid.

### 4.5 Which values, per palier

| Field | Source | Notes |
|---|---|---|
| Heart rate | watch / chest strap | every stage; glance-and-confirm |
| BP systolic / diastolic | cuff | best-effort, per §4.3 |
| RPE (Borg 6–20) | rider | one tap on a scale; cheap and genuinely useful |
| **Actual cadence** | console (rpm or km/h) | **required** — on this bike cadence is the load (§3.2) |
| Resistance level | you set it | recorded per palier; the exact, reproducible axis |
| Actual watts | console, if it has the field | the console's estimate, recorded as shown |
| Symptom flags | rider | chest, breath, legs, dizzy |

Nothing is mandatory to advance a stage. A missed BP is a blank cell, not a
blocked test.

---

## 5. Stopping

The rider taps **STOP** at their comfortable limit. Immediately after, capture:

- **Reason** — leg fatigue / breathlessness / chest discomfort / target reached
  / other. This is the single most important qualitative field; a peak of 200 W
  stopped by tired legs and one stopped by chest discomfort are not the same
  test.
- **Peak RPE.**
- Time into the final stage — a stage held 0:40 is not a completed stage, and
  peak workload should be reported honestly (e.g. "175 W, 0:40 of 2:00").

---

## 6. Derived metrics

**These are no longer my guesses — they are read off the reference clinical
report and reverse-engineered to the exact formula.** Derivation and
verification against the printed values live in `REFERENCE-*.md`, which is
gitignored: it holds personal medical data and must never be committed. Every
formula below reproduces the report's printed value exactly.

I was wrong about two things earlier and they're worth correcting plainly:

- **The report doesn't use PWC or PMT at all.** It leads with Watts/kg, METs
  against a theoretical METs, and % of FMT. So those become the headline numbers.
- **FMT is `220 − age`, not Tanaka.** The app must use 220 − age to match the
  report, and must compute age from date of birth *at test date* — every
  reference value below moves each year.

**The report's own metrics — reproduce exactly**

| Quantity | Formula |
|---|---|
| Watts/kg | `peak W / weight` |
| METs | `4 × W/kg` |
| METs théorique | `18 − 0.15 × age` (Morris, men) |
| FMT | `220 − age` |
| Sous-max. | `0.85 × FMT` |
| % FMT atteint | `peak FC / FMT` |
| **Double produit** | `TAs in cmHg × FC` |

**Blood pressure is in cmHg, not mmHg.** The report prints `13,5/8` for 135/80,
and the double product is computed on the cmHg figure — which is why it reads
1040 rather than 10395. The app stores mmHg internally and **displays cmHg with a
decimal comma**, so the printout matches the report line for line.

**Ours to add, as secondary**
- Total exercise time, total work (kJ)
- HR slope — bpm per watt, and bpm per resistance level (exact axis, §3.3)
- **PWC 130 / 150 / 170** — load interpolated at those heart rates. Not on the
  clinical report, but kept because a home test is **symptom-limited and will
  likely stop well short of the 85–95 % FMT a supervised test reaches**. Every
  peak-based metric (Watts/kg, METs,
  % FMT) then understates you. A submaximal index doesn't — it compares how your
  heart responded to a load you can reproduce exactly, regardless of how hard you
  chose to push. This is the number that makes a cautious home test still
  informative.
- Systolic rise per 25 W, and any systolic *fall* under rising load (flagged)
- **SpO₂** — the report's `Tc PO₂` column, 98 % throughout. A fingertip pulse
  oximeter costs very little, gives SpO₂ *and* pulse, and works well on a
  recumbent where the hand can rest still. Cheapest way to add a real column.

**Recovery**
- HRR at 1 / 3 / 5 min = peak HR − HR at that time (the report's exact schedule)
- TA return toward baseline

**Context fields the report carries, so we must too**
- Weight and height at test date — METs and W/kg are most sensitive to weight
- **Traitement** (current medication). The report has this field for a reason: a
  rate-limiting drug blunts HR and makes % FMT incomparable across a change. The
  app should warn when comparing two tests whose medication differs.
- Indication, risk factors, motif d'arrêt

**Test quality** — specific to this bike, and what makes the numbers trustworthy
- Cadence held: mean and minimum actual cadence per palier vs target, and a
  per-palier flag when it drifted (§3.2)
- Which paliers have measured BP vs blank (§4.3)
- Which values were confirmed vs left on their prefilled guess (§4.2)
- Modality recorded (home recumbent vs clinic upright, §3.4)
- Confirmation that SmartAdjust was off (§3) — a checklist tick stored with the
  test, because a test ridden with iFIT driving the brake is not comparable to
  anything and there is no way to detect that after the fact.

All flags are descriptive with the threshold stated, never a diagnosis:
"systolic 232 mmHg at peak (commonly flagged above 210)".

---

## 7. Stop-now card

One always-reachable screen, phrased as the reasons a supervised test would be
terminated, so the decision isn't being made from scratch mid-effort: chest
pain or pressure, marked breathlessness, dizziness or feeling faint, a systolic
reading that *drops* under rising load, a very high systolic reading, or simply
having had enough. Plus: this is not medical advice, and anything worrying goes
to a doctor, not to the app.

---

## 8. Report & history

- **Session table** — one row per palier: level, target and actual cadence,
  estimated watts, HR, BP, RPE, flags.
- **Charts** (inline SVG, no library). X axis is **resistance level** with an
  estimated-watts axis alongside, per §3.3:
  - HR vs load, with PWC 130/150/170 markers and the regression line
  - Systolic + diastolic vs load, showing gaps where BP wasn't measured rather
    than interpolating across them
  - HR vs time across the whole test *including recovery* — the shape of that
    curve is the most legible single picture of the test
- **Trend across tests** — PWC170, peak load, HRR1, peak double product over the
  years. Home recumbent tests form the primary series; clinical upright tests are
  plotted as visually distinct points (§3.4), never merged into the same line.
  This is the real payoff of the whole app.
- **Export** — JSON (complete, re-importable), CSV (spreadsheet), and a print
  stylesheet so a one-page A4 summary can be handed to the cardiologist.

Manually entering the clinical tests means the data model must accept a test
with **no live timing** — imported stages only. Worth designing in from the
start rather than bolting on.

---

## 9. Technical approach

Keep Atmen's shape. It is the right shape for this.

- **Single `index.html`**, no build step, no dependencies. Works offline, goes
  on the iPhone home screen, deploys to GitHub Pages by pushing one file.
- Reuse directly from Atmen: the CSS-variable theme blocks, the settings sheet,
  the `requestAnimationFrame` clock, Web Audio cue synthesis (§4.4's six cues),
  the screen wake lock, and the `APP_VERSION` self-update check that works around
  iOS Safari's cache.
- **Storage.** Atmen keeps one settings blob. This app also keeps a *test
  history*, which is unbounded — so: settings under one key, each test under its
  own key, and an index. localStorage is fine at this data volume (a test is a
  few kB), but export must be easy and obvious, because localStorage can be
  wiped by the browser without warning. **A test that only exists in
  localStorage is not saved.** Offer the JSON download at the end of every test.
- Voice cues via `speechSynthesis`, as in Atmen — **French voice** (`fr-FR`),
  and it matters more here than in Atmen since eyes are off the phone.
- **i18n: French primary**, English and German selectable — same structure as
  Atmen but with `fr` as the default and the fallback. Terminology in §10.
- **Mounted-phone layout.** The G LE has a **device shelf** above the console, so
  the phone sits there at arm's length, read by someone breathing hard. Bigger
  type than Atmen throughout, and the three numbers that matter during a palier —
  **cadence**, time remaining, and the resistance level you should be on —
  legible without leaning in.

---

## 10. French terminology

French cardiology has settled terms for all of this, and using them makes the
printout readable by the cardiologist without translation. To be confirmed
against the report:

| English | French |
|---|---|
| Effort / stress test | épreuve d'effort, test d'effort |
| Stage / increment | **palier** |
| Load, power | charge, puissance (W) |
| Resistance level | niveau de résistance |
| Cadence | cadence (tr/min) |
| Heart rate | fréquence cardiaque (**FC**) |
| Predicted max HR | fréquence maximale théorique (**FMT**) |
| Blood pressure | tension artérielle (**TA**) — systolique / diastolique |
| Rest | repos |
| Warm-up | échauffement |
| Recovery | récupération |
| Reason for stopping | motif d'arrêt |
| Perceived exertion (Borg) | échelle de Borg, pénibilité perçue |
| Leg fatigue | fatigue des jambes |
| Breathlessness | essoufflement, dyspnée |
| Chest discomfort | douleur thoracique, gêne thoracique |
| Dizziness | vertiges |
| Double product | double produit (FC × TA systolique) |
| Max tolerated power | puissance maximale tolérée (**PMT**) |

**Correction to my earlier guess:** your report uses **neither** PMT nor PWC. Its
headline block is `Watts / kg`, `METs`, `METs théorique`, `Fréquence cardiaque
max. théorique`, `Sous-max.` — and per-palier `T.A.` (in cmHg), `Fréq. card.`,
`Double produit (TAs × FC)`, `Tc PO₂ %`, `Symptômes`, `ECG`. The app should use
exactly those labels. Also confirmed from the report: *palier*, *Motif d'arrêt*,
*Fatigue des membres inférieurs*, *Repos*, *Après l'effort*.

---

## 11. Open questions, gathered

Two are now closed: the resistance is **magnetic** (good for reproducibility) and
there are **26 levels**. Remaining:

1. **What does the 5" console actually display?** (§3.1) — tick which of Watts /
   RPM / Speed / Level appear. Every spec sheet lists only time, distance, speed
   and calories, so I'd guess **no watts and possibly no RPM** — but you can
   settle it in ten seconds, and §3.1/§3.3 mean the app works either way.
2. **Does your BP cuff cope while you pedal?** (§4.3) — worth one trial ride
   before we finalise the measurement schedule. The recumbent position should
   make this work; better to know than to assume.
3. **Reference values** (§6) — which normal-value set your cardiologist uses;
   I'd rather read it off the report than pick one and be wrong.
4. **PMT vs PWC** (§10) — which your report leads with.

### The protocol — now taken straight from the clinical test

My earlier "+2 levels per palier" draft is superseded. The report tells us
exactly what to do, and matching it is far more valuable than inventing a ladder:

```
Repos (seated, not pedalling) — TA + FC          ← no warm-up, the report has none
Palier 1   50 W    2:00        Palier 5  150 W   2:00
Palier 2   75 W    2:00        Palier 6  175 W   2:00
Palier 3  100 W    2:00        Palier 7  200 W   2:00
Palier 4  125 W    2:00        …+25 W while able
→ STOP  →  Après l'effort: measurements at 1′, 3′, 5′
```

**50 W start, +25 W every 2 min, no warm-up palier.**

The point of matching the watt ladder exactly is that **heart rate at 100 W this
year is directly comparable to heart rate at 100 W in the clinical test** —
palier for palier, against a real supervised measurement. That is worth far more
than any self-invented scale, and it is why the watts question (§3.1) matters: we
need to know which resistance level gives ~50 W, ~75 W and so on at the chosen
cadence. If the console shows watts, that's one calibration ride. Once found, the
level↔watt table is **frozen** and reused every year.

---

## 12. Suggested build order

1. **Runnable test.** Data model, settings, the rest → warm-up → palier →
   recovery state machine, the clock, the audio/French-voice cues, the
   cadence-first load display, and the non-blocking entry queue with prefilled
   steppers.
2. **Storage and export.** History index, JSON download offered at the end of
   every test, CSV.
3. **Analysis.** Derived metrics (§6) and the single-session report with charts.
4. **Trend view** across tests, plus manual entry of the clinical tests so they
   land on the same axes. Cross-calibration (§3.4) falls out of this.
5. **Print stylesheet** for the one-page A4 hand-out, in French.

Step 1 is usable on its own — it can run a real test and export raw numbers
before any of the analysis exists. Given that a test happens twice a year at
most, getting step 1 right before your next ride matters more than the report.

---

## 13. Things deliberately not in v1

- **Bluetooth heart-rate strap.** Web Bluetooth would let the app read HR
  directly and remove the main reason to touch the screen — genuinely the single
  biggest usability win available. But it does not work in Safari on iOS at all,
  so it can't be the primary path on a phone. Revisit as an enhancement for
  desktop/Android, not as the design centre.
- **Spoken number entry** (§4.2) — unreliable on iOS, and misheard numbers are
  worse than none.
- **ECG of any kind.** Out of scope, and the absence is the honest reason this
  is a log and not a stress test.
- **Cloud sync / accounts.** Export is the sync story.
