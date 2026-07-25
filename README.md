# Trail Brake Trainer

A single-file, offline brake-modulation trainer for sim racing. It reads your
brake pedal directly through the browser's Gamepad API, plays a "ghost"
target trace (a stab-and-release brake profile), and scores how closely your
live pedal input follows it — with a specific focus on **release smoothness**,
since that's the skill this tool was built to train.

No install, no server, no account. All your round history lives in your
browser's local storage and can be exported to CSV at any time.

## Development notes

This project was built collaboratively with Claude (Anthropic, Sonnet 5) —
iterating through feature design, debugging real hardware issues (including
a pedal-detection bug where calibration was locking onto a connected
wheelbase instead of the actual pedals), and working through licensing and
IP tradeoffs. It's a working example of directed, iterative AI-assisted
development rather than one-shot generation — a longer write-up of that
process is planned as a blog post.

---

## Getting started

1. Open `trail-brake-trainer.html` directly (double-click the file, or drag it
   into Chrome/Edge). It will not work correctly inside an embedded preview
   frame — it needs to be a normal top-level page for the Gamepad API to work.
2. Click **CALIBRATE**.
3. Press your brake pedal once. Browsers hide gamepads from the page until
   they receive an input, so this "wakes up" the device.
4. Click **BEGIN**. Press the brake to 100% and hold — the app watches
   *every* connected device (wheelbase, pedals, buttons, etc.) and locks onto
   whichever axis moves the most, so it should correctly find your pedals
   even if a wheelbase is also connected.
5. Release the pedal fully when prompted. Calibration is done.

Recalibrate any time you change your pedal's brake force / load-cell
sensitivity setting, since that changes the raw signal range.

If you don't have pedals handy, **USE MOUSE INSTEAD** on the calibration
screen lets you test the app by dragging vertically on the trace area.

---

## Modes

**Challenge** — a timed round (30s / 60s / 2min) made of several braking
"corners" separated by rest periods, with a 3-2-1 countdown before it starts.
Ends with a scored results card.

**Free Practice** — a single corner shape looped continuously, unscored.
Use this to drill one specific release shape or pressure level without the
pressure of a timer.

---

## Ghost configuration

| Setting | What it does |
|---|---|
| **Round length** | 30s / 60s / 2min challenge duration |
| **Release shape** | `Exponential` (steep-then-flat, the classic trail-braking release) · `Linear` · `Late release` (holds, then drops) · `Steps` (pressure-hold drill, see below) · `Mixed` (randomizes per corner) |
| **Peak hold** | How long the ghost holds at peak pressure before releasing: 0.5 / 1 / 2 / 3s, or `Variable` (randomized 0.5–3s per corner). Default 2s. |
| **Release time** | How long the release phase lasts: 1.5 / 2.5 / 3.5s, or `Variable` (randomized 1.8–3.5s per corner). Default 2.5s. Not used by the Steps shape, which derives its own length from Peak Hold. |
| **Peak cap** | Ceiling on how hard the ghost ever brakes, e.g. cap at 85% to train under an ABS threshold |
| **Window** | Seconds of trace visible on screen at once (4–30s, default 10s) |
| **Bars / Live % / Audio cue** | Optional display toggles — see below |

**Steps shape**, in detail: after the peak hold, the ghost drops to 75%,
50%, and 25% of peak pressure, holding each level for the same duration as
your Peak Hold setting, before releasing to zero. With an 80% peak cap and a
2s hold, that's roughly 80% → 60% → 40% → 20%, 2 seconds each — a drill for
holding the pedal at specific partial pressures rather than tracing a curve.

---

## Display options

- **Brake bars** — vertical fill bars for ghost and you, alongside the trace. On by default.
- **Live %** — numeric readout of ghost %, your %, and the delta between them. Off by default, since watching numbers can pull your eyes off the trace shape — turn it on if you want a precise real-time read instead.
- **Audio cue** — a tone that rises in pitch with the ghost's brake pressure, for eyes-up practice.

Your trace flashes red on any point where you deviate from the ghost by more
than 12 percentage points, so errors are visible without reading numbers.

---

## Metrics — what they mean and what to aim for

All metrics compare your recorded pedal trace against the ghost at 60
samples/second.

### RMSE — overall error (↓ lower is better)
Root Mean Squared Error between your trace and the ghost across the whole
round, in % of full pedal travel. This is the single best "how far off was
I overall" number — it punishes big misses more than small ones, since
errors are squared before averaging.
- Under ~8% is solid.
- Under ~5% is excellent.

### Release smoothness — SD of release rate (↓ lower is better)
**This is the core metric for the original goal of this tool.** It's the
standard deviation of your pedal's rate of change (% per second), measured
*only* during the ghost's release phases (not the ramp-up, not the hold).

A perfectly smooth release has a nearly constant rate of change — your foot
is moving at a steady speed off the pedal — so its standard deviation is
low. A jagged, stop-start release has rate-of-change spikes every time you
hesitate or correct, which drives this number up. Track this across weeks;
a downward trend is the app doing its job.

### Reversals per corner (target: 0)
Counts how many times per corner your pedal *changed direction* during a
release — i.e., you were releasing pressure, then pressed back down (or
vice versa). Each reversal is one visible "jag" in your trace.

- **0** — a clean, monotonic release. This is the goal.
- **1–2** — minor correction, not unusual while learning a new shape.
- **4+** — you're chasing the ghost reactively rather than executing one
  planned motion. Usually fixed by slowing practice reps down deliberately
  (see the original coaching advice: exaggerate smoothness first, then
  bring it back to speed).

### Timing — lag/lead (target: within ±100ms)
Found via cross-correlation: the app shifts your trace forward and backward
in time to find the offset that best matches the ghost, then reports that
offset.

- **Positive / "reacting late"** — you consistently start braking after the
  ghost does. Fix: look further ahead on the trace (the window shows
  upcoming corners for exactly this reason).
- **Negative / "anticipating"** — you're starting early, which is unusual
  but can happen if you're anxious about missing a corner.
- Under ±100ms is very good — note that this is a *timing* issue, separate
  from shape. You can have perfect timing and a jagged release, or great
  shape but consistently late starts. The corner table below the headline
  metrics helps you tell which one is happening on which corners.

Hover any metric tile in the app for this same explanation in-app.

---

## Data & export

Round history (every corner of every completed challenge) is stored in
the browser's local storage automatically — nothing is sent anywhere.

Click **EXPORT CSV** on the results screen to download the full history as
a flat CSV, one row per corner, with round-level and corner-level metrics
together. Built for dropping straight into a pandas notebook.

**Note:** local storage is tied to the specific browser profile on the
specific machine you're using. It won't follow you to a different browser
or PC, and clearing browser data will erase it. Export to CSV periodically
if you want a durable copy.

---

## Wish list / planned for v2

Parked ideas, roughly in the order we're likely to tackle them:

1. **Basic insight engine (no LLM yet).** Surface patterns automatically
   from the round history that are currently only visible if you dig
   through the CSV yourself — e.g. "your release-smoothness SD is 40% worse
   on corners above 80% peak pressure than below it" or "your reversals
   cluster in the first second of the release, not the last." Since all the
   raw data already exists, this is mostly threshold/correlation logic
   against the stored history, no model required. A natural home for this
   is a new "Insights" tab next to results, refreshed after every round.

2. **Split results into Peak/Hold phase vs. Release phase.** Right now
   every metric (RMSE, smoothness, reversals) is either whole-round or
   release-only. Add a parallel set scoped to the ramp+hold phase — e.g.
   "hold stability" (SD of your pressure *while* the ghost is flat, since
   ideally that should be ~0) and "peak accuracy" (how close your actual
   peak got to the ghost's peak, and how long it took you to get there).
   Display as two labeled rows in the results grid: **PEAK & HOLD** /
   **RELEASE**.

3. **Results history browser.** Add a "PAST ROUNDS" button (top bar or
   drawer) that reopens the results overlay for any previous round, not
   just the one you just ran — backed by the same local-storage history
   already used for export and the trend chart.

4. **Bigger / relocated smoothness trend chart.** Increase the y-axis
   range/scaling so small changes in release SD are visually readable
   instead of looking flat, and consider moving it to a side column in the
   results card so it doesn't compete for width with the corner table.

**A few additions I'd suggest adding to this list:**

- **Per-shape breakdown in Insights** — since you now have five release
  shapes, tracking your smoothness/reversals *per shape* would show
  whether "steps" (pressure holding) is a genuinely different skill from
  your curve-following ability, or improves together with it.
- **Session summary, not just per-round** — a lightweight "today's session"
  rollup (rounds played, average smoothness, best round) shown when you
  stop practicing, so you don't have to mentally track it across rounds.
- **.ibt real-lap ghost import** — still on the table as discussed; the
  ghost format is already a generic list of (time, pressure) points
  internally, so this plugs in without restructuring anything.
- **Configurable deviation-flash threshold** — the red-flash cutoff is
  hardcoded at 12%; as your accuracy improves that threshold will start
  triggering constantly, so making it a slider will keep the visual signal
  useful as you get better.
- **Tolerance rails.** Instead of (or in addition to) a single ghost line,
  render a shaded band around it and score "inside the band" rather than
  exact-match distance. A thin line is unrealistically precise to chase;
  a band trains a more realistic skill and reads more calmly on screen.
  Keep exact-line RMSE available underneath for players who want the
  harder mode — band width becomes another difficulty knob.
- **Named combo sequences.** Alongside the randomized/parametric corner
  generator, add a small library of fixed, named sequences (e.g. "Hairpin →
  90° → Sweeper") that always play back identically. These give you a
  repeatable benchmark to re-run and compare against your own history,
  rather than only ever facing new randomized corners.
- **Throttle and steering-angle traces.** Extend beyond brake-only to a
  second (and eventually third) input channel, each with its own ghost and
  live trace on the same rolling window — throttle especially, since a
  "sawtooth" throttle trace (chasing wheelspin) is as diagnostic a flaw as
  a jagged brake release. Steering angle is the natural third channel for
  a fuller "pedal-and-wheel" practice session that transfers more directly
  into an actual sim lap. This is a bigger architectural change than the
  others — worth designing the multi-channel data model deliberately
  rather than bolting it on, since round history/export/metrics all
  currently assume a single brake channel.

---

## Competitive landscape

A few other tools cover similar ground, worth knowing about both as
feature inspiration and honest positioning:

- **[Braking Lab](https://www.brakinglab.com/en)** — the most feature-rich
  free alternative. Browser-based like this tool, but adds a curve editor,
  a training dashboard with trends, and real-lap import (drag in a
  Garage61 CSV and it auto-extracts actual braking zones from real laps —
  effectively the .ibt-import idea on this wish list, already shipped
  there). Built with input from real drivers, including Suellio Almeida.
  Worth a close look for UX ideas.
- **[Trail Braking Trainer](https://trailbraking.uk/)** — closest in scope
  and philosophy to this project: single-file-feeling, browser + Gamepad
  API, moving ghost, countdown start, RMSE scoring, run history. Notably
  uses tolerance rails instead of a single line to match, and offers named
  fixed combos plus an endless "survival" mode — both referenced above.
- **TrackPro (Sim Coaches)** — a full paid telemetry/haptics/motion-control
  suite where a brake trainer is one small feature inside a much larger
  commercial product. Different category entirely, but evidence this
  feature is valuable enough to be worth commercializing at scale.

None of these make the core "ghost trace + live pedal + scoring" concept
patentable (see the license discussion above) — but they're good reading
before designing v2, and good company to be building alongside.


---

## Known limitations

- Requires Chrome or Edge (or another browser with a working Gamepad API);
  the app does not work inside embedded/iframe previews.
- History is per-browser-profile, per-machine, and not automatically backed
  up — use CSV export for anything you want to keep long-term.
- Sampling is fixed at 60Hz regardless of display refresh rate, which is
  well above the resolution needed for this kind of motor-control training.
