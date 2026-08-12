# Mamdani Dance Floor — handoff

A single self-contained HTML page: a cartoon Mamdani with his real photographed
face, dancing on the beat to whatever audio the machine can hear, in front of
beat-reactive backgrounds themed on real things he did in office. Built to run
on a projector at a party.

Live at <https://sthouvenot.github.io/dance-floor/>.
Domain `mamdanidancefloor.com` is registered and its `CNAME` is committed;
GitHub Pages custom domain still needs finishing (see **Deploy**).

---

## The one rule

**`index.html` is the entire application.** ~225 KB, no build step, no
dependencies, no external requests. HTML, CSS, JS, and four base64 WebP head
photos all inline. Open it with `file://` and it works.

Do not split it into modules. Do not add a bundler. The whole point is that it
can be emailed, dropped on a USB stick, or served from any static host.

---

## Layout of the file

Roughly in order:

| Region | What's there |
| --- | --- |
| `<style>` | All CSS. Poster landing page, HUD chips, fact box, share dialog, debug panel. Mobile overrides are in a `@media (max-width:700px)` block **at the very end** — it must stay last, see Gotchas. |
| `<body>` markup | `#stage` canvas, `#start` poster overlay, `#fact`, `#corner` (full-screen button + status chip), `#share` dialog, `#dbg` panel. |
| Canvas setup | `resize()`, `cache()`. |
| Audio | `attach()`, `readAudio()`, `onset()`, `pulse()`, `hearing()`, `pickSource()`. |
| Draw helpers | `grad`, `rr`, `skyline`, `lightCones`, `icicle`, `grocery`, `pistol`, `trophy`, etc. |
| `SCENES[]` | Eleven scene objects. The bulk of the file. |
| `drawSpectrum()` | The top-hanging analyser bars. |
| Dancer | `getPose()`, `limb()`, `drawHeadPhoto()`, `drawDancer()`, `drawCrowd()`. |
| Loop | `frame()`, `draw()`. |
| UI wiring | Start options, share dialog, status chip, full screen, debug panel. |

---

## State

Everything lives on one object, `S`, exposed as `window.__party` for tests.

```js
S = { started, mic, listen, sens, t, dt, bpm, phase, beat,
      energy, bass, air, kick, flash, scene, factScene, auto, sceneAt,
      sceneFade, busX, lastBeat, intervals, parts, head, tint,
      sync, src, still, offset, lastSignal, hearSince, wasHearing,
      srcLabel, rate, fps, specRef, specVals,
      dbg:{ flux, thresh, onsets, lastOnset, scope, level } }
```

The ones that matter when writing scene art:

- **`S.kick`** — 0..1, spikes on each detected onset and decays. This is what
  you multiply by for anything that should punch on the beat.
- **`S.phase`** — 0..1 position within the current beat. Use
  `Math.pow(1 - S.phase, k)` for a decaying per-beat envelope.
- **`S.beat`** — integer beat counter. Use `S.beat % n` to pick "which thing
  lights up this beat".
- **`S.bass` / `S.air`** — smoothed low and high band energy.
- **`S.still`** — true when there's no real beat. The dancer breathes instead of
  dancing. Scenes keep animating but shouldn't imply rhythm.

---

## How the beat detection works

Spectral flux onset detection feeding a phase-locked oscillator.

1. `readAudio()` runs `getByteFrequencyData` on a 4096-point FFT, computes
   positive spectral flux against the previous frame, and compares it to a
   rolling adaptive threshold.
2. An onset nudges the PLL rather than driving animation directly, so a missed
   or spurious onset doesn't produce a visible stutter. Tempo is trimmed toward
   the observed inter-onset interval, clamped to 70–180 BPM.
3. `S.still` gates the dancer. It requires 0.7 s of continuously audible sound
   before he starts moving, and drops him back to breathing when audio stops.

This design was arrived at after several failed attempts, most of which are
recorded in Gotchas. Be careful changing it.

---

## Audio sources

Two, chosen on the landing page. One per page load; reload to switch.

- **This computer's sound** — `getDisplayMedia` with the picker shaped by
  `displaySurface:'browser'`, `systemAudio:'include'`,
  `selfBrowserSurface:'exclude'`, `surfaceSwitching:'include'`. The share dialog
  spells out the audio toggle in numbered steps because that switch is off by
  default and is the single most common failure.
- **Microphone** — plain `getUserMedia`.

There is **no** Spotify integration, no BPM lookup API, no tap tempo. All three
were built and removed. Spotify locks `audio-analysis` and `audio-features` for
apps created after late 2024; AcousticBrainz is dead; MusicBrainz has no BPM;
Deezer's coverage is about two thirds and poor on modern pop. Don't rebuild
these. Listening to audio is the only approach that works.

On iPhone, system audio capture is impossible — Safari has no `getDisplayMedia`
on iOS and there's no loopback device. The landing page detects this and offers
microphone only.

---

## The scenes

`SCENES[]`, eleven entries, ten seconds each, advancing automatically. Each has
`name`, `fact`, `kind` (particle type), optional `thrown` / `throwY`, a
four-colour `pal[]`, and a `draw()`.

| # | Name | Subject |
| --- | --- | --- |
| 0 | WORLD CUP WATER | FIFA reversed its water-bottle rule, June 2026 |
| 1 | RENT FROZEN | RGB voted 0% on ~1M stabilized apartments, June 2026 |
| 2 | FREE THE ICE CREAM | Ice cream permit scrapped, OPEN for Small Business, July 2026 |
| 3 | KNICKS | Beat the Spurs in five, first title since 1973, June 2026 |
| 4 | ELECTION NIGHT | Beat Cuomo twice, primary June 2025 and general Nov 2025 |
| 5 | FREE CHILDCARE | $1.7B program, first 2,000 two-year-olds |
| 6 | CITY GROCERIES | La Marqueta named first city-owned site, April 2026 |
| 7 | FAIR FARES | Eligibility 960k to 1.3M, June 2026 |
| 8 | SAFEST ON RECORD | Fewest shootings on record, murders down 25.6% |
| 9 | POTHOLE BLITZ | 100,000th pothole on day 100, three Saturday blitzes |
| 10 | BEAT THE HEAT | 500+ cooling centers, 7 misting stations, 15-van COOL fleet |

**Every fact was verified by web search against real 2026 reporting.** Don't
invent new ones. If you add a scene, research it first.

The fact text appears in the `#fact` box bottom-left, sized to its content, max
two thirds of the viewport width.

### Scene conventions

- Draw in viewport units: `W`, `H`, `Math.min(W,H)`. Never hard-code pixels.
- Expensive static art goes through `cache(key, w, h, fn)`, which keys on
  viewport size and rebuilds on resize. Change the key when you change the art
  or you'll get a stale cache during a session.
- The spectrum hangs from the **top** of the stage, so leave sky or open space
  up there. This is why the rent-freeze building has a roofline.
- The fact box covers the bottom-left. Don't put anything load-bearing there.
- No em dashes in any user-facing string. Code comments are fine.
- American spelling in user-facing strings.

---

## The dancer

- Skeletal 2D, drawn every frame. Twelve moves in `getPose(b, m, e)`, switching
  every 8 beats: two-step, hands-up sway, disco point, spin, running man,
  shimmy, sprinkler, lawnmower, robot, Charleston, side slide, jump with
  fist-pump.
- Both arms draw **in front of** the torso. An earlier version put one behind
  and it disappeared during several moves.
- The head is one of four photo cutouts in `HEADS[]`, cycling on scene change.
  Each entry has `ax`/`ay` (the neck anchor as a fraction of the image) and `s`
  (scale). The images are base64 WebP in `HEAD_SRC`.
- Source photos and the background-removal script are in `heads/` and
  `heads.py` (PIL + scipy, flood-fill from the border so teeth and pupils
  survive). If you re-cut a head you must recompute `ax`/`ay` — rotating or
  re-cropping moves the anchor. `heads.py` has the transform maths.
- The crowd is thirteen baked silhouettes with hand-picked irregular spacing.
  **Do not re-randomise them.** This went through about six rounds of feedback.

---

## The spectrum

Log-spaced bands, edges walked forward so no two bands share an FFT bin, drawn
as straight vertical bars hanging from the top edge.

The normalisation is the subtle part. All bars share **one slow-ceiling
reference** (`S.specRef`, attack 0.035, release 0.99955). Per-bar auto-gain
pegs everything at maximum by definition, and chasing the current frame's peak
normalises away the difference between a drop and quiet playing. Both were
tried. Don't.

On screens under 720 px wide it drops to 52 bars a side at more than double the
height, because 244 hairlines are invisible on a phone.

---

## Mobile

The scenes are composed wide. Rather than squash them into a phone's portrait
viewport, the stage is **letterboxed**: `resize()` caps the drawing height at
`W * MAXR` (0.72) and centres that band vertically, filling the rest with
`#05070f`. On a 390×844 phone the stage is about a third of the screen. `draw()`
clips everything to the band so particles don't escape.

`W`/`H` are the **stage** dimensions. `VW`/`VH` are the viewport. `offY` is the
letterbox offset, already baked into the canvas transform, so scene code needs
no changes.

Other mobile handling: the HUD chips move to the top bar, the fact box spans the
bottom bar, the poster stacks with his face above the title and offers one
button, and the full-screen button hides itself when the browser has no
Fullscreen API (iPhone Safari).

---

## Testing

Playwright with headless Chromium and `--use-fake-device-for-media-stream`.

```bash
node shoot.js          # main regression: console/page errors, beat advances
```

`shoot.js` is the one that must always pass. It loads the page, fakes an audio
device, and asserts the beat counter advances with no errors.

Ad-hoc probes live in `/tmp/*.js` and get rewritten constantly. Useful patterns:

```js
// force a scene
await p.evaluate(i => { const S = window.__party;
  S.auto = false; S.scene = i; S.factScene = i; S.sceneFade = 1; }, 6);

// attach fake audio so the beat clock actually runs
await p.evaluate(async () => {
  const s = await navigator.mediaDevices.getUserMedia({ audio: true });
  window.__attach(s, 'fake');
});
```

**Screenshot everything you change.** Canvas bugs are invisible in the source
and obvious in a PNG. Half the bugs in this project's history were caught by
looking at a render, not by reading code.

Press `D` in the browser for the live debug panel: input, level, flux, onsets,
tempo, clock, and a scope.

---

## Gotchas, all learned the hard way

**Batch patch scripts that write at the end.** The single most damaging class of
bug here. A Python helper that did all its replacements then wrote the file
once, with asserts inside, would abort on the first failed assert and silently
discard every earlier successful edit. This lost an FFT upgrade, an odometer, a
spawn feature, and several tunings before it was noticed. **Validate every
target exists before writing anything.**

**The beat clock is pinned when there's no audio.** `S.phase = 0` every frame
when nothing is audible. Any test of per-beat animation that doesn't attach a
fake stream will show a frozen frame and look like the animation is broken. This
wasted a round on the ballot-into-the-slot bug.

**Draw order determines occlusion, and it bit us twice.** The ballot was drawn
before the ballot box, so the lid painted over it and it vanished behind the rim
instead of entering the slot. Same class of bug hid the ice cream cart's wheels
behind the pavement. When something "goes behind" instead of "goes into",
it's draw order, not geometry.

**FFT size and band edges must agree.** Bands tuned for 4096 running on a 2048
analyser started at ~140 Hz, silently excluding the entire kick drum from the
visualiser.

**`cache()` keys on viewport size only.** Change the art without changing the
key and you'll see the old version until the window resizes.

**CSS source order beats media queries at equal specificity.** The mobile
`#corner` override sat before the base `#corner` rule, so `bottom:16px` won over
`bottom:auto` and the chips stretched into the middle of the screen. The mobile
block must stay at the end of the stylesheet.

**Global element selectors leak.** A bare `canvas { }` rule stretched the debug
scope to full screen. Scope selectors to their container.

---

## Deploy

Repo: <https://github.com/sthouvenot/dance-floor>, GitHub Pages from `main` at
root. `index.html` and `CNAME` (`mamdanidancefloor.com`) are the only files that
matter.

In Claude Code on the user's machine, `git push` just works.

Remaining setup, once:

1. Upload the current `index.html` (the live one predates the mobile pass).
2. Cloudflare DNS for `mamdanidancefloor.com`: four A records on `@` pointing at
   `185.199.108.153`, `.109.153`, `.110.153`, `.111.153`, plus a CNAME on `www`
   to `sthouvenot.github.io`. **All set to DNS only, grey cloud** — the orange
   proxy breaks GitHub's certificate issuance.
3. Repo Settings → Pages → Custom domain → `mamdanidancefloor.com` → Save, then
   tick **Enforce HTTPS** once the certificate provisions (minutes to an hour).

Note for whoever picks this up in a cloud sandbox: pushes there fail with 403
because the git proxy only injects credentials for repos attached to the
session, and repos are attached at task start. That constraint doesn't exist in
local Claude Code.

---

## Open items

Nothing outstanding. Last round delivered the mobile pass: letterboxed stage,
his face on the landing page, bigger bars, the fact box on phones, and the
full-screen fix.
