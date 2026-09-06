---
name: testing-bgsn-overlay
description: How to run and test the BGSN scoreboard controller/overlay locally, including Firebase production-data safety, ticker animation instrumentation, and browser/window setup on the Devin box.
---

# Testing the BGSN scoreboard (controller + overlay)

Static site, no build step. Serve the repo root and open the pages:

```bash
cd <repo> && python3 -m http.server 8000 &
# controller: http://localhost:8000/index.html
# overlay:    http://localhost:8000/bgsn-overlay.html
```

Both pages talk to the **production** Firebase Firestore project `bgsn-scoreboard`
(doc `scoreboard/live`) with the API key embedded in the HTML — no credentials needed,
and no staging environment exists.

## Production data safety (mandatory)

Snapshot the doc before touching anything and restore it afterwards. Firestore REST works
without auth (rules are open) and is the most reliable way to restore exact values,
including long unicode ticker strings:

```bash
KEY=$(grep -o 'apiKey: "[^"]*"' index.html | head -1 | cut -d'"' -f2)
BASE="https://firestore.googleapis.com/v1/projects/bgsn-scoreboard/databases/(default)/documents/scoreboard/live"
curl -s "$BASE?key=$KEY" -o /tmp/live_initial.json          # snapshot
# restore: PATCH the same JSON field objects with updateMask.fieldPaths=<field> per field
```
Diff the final doc against `/tmp/live_initial.json` and report "NONE" before finishing.
Beware: stray clicks in the controller (its +/- buttons write immediately) silently change
production values — always diff every field, not just the ones you intended to change.

## Controller UI map (index.html)

- Home/Away score `+1` quick buttons and `+`/`−`; Name Position `+`/`−` per team.
- Upper Ticker card: Messages textarea (`#ticker`), Scroll Speed ±2 px/s, Bar Size ±2 px.
- Lower Ticker card: same controls for `#ticker2`.
- Text edits only publish when you click **PUSH TO OVERLAY**; +/- buttons publish instantly.
- The controller page has no inner scroll container: click the page body first, then use
  `End` / `Page Down`; mouse-wheel scroll sometimes does nothing until the body is focused.

## Overlay ticker internals worth knowing

`fillTicker()` duplicates the item list and animates `.ticker-scroll` with
`@keyframes scroll-left` (0 → −50%); duration = `scrollWidth/2 / speed`. Therefore:
- Long production ticker text ⇒ duration of ~10 minutes; you cannot wait for a wrap.
  To observe the loop/seam, temporarily set a short 2-item text (duration drops to ~1 min).
- Any change of the animation shorthand (i.e. duration) causes a *position discontinuity*,
  because a CSS animation's `currentTime` is preserved while progress = t/duration.
  Bar Size and Scroll Speed changes legitimately do this; treat it as expected.

## Measuring ticker smoothness objectively

Reaction time and video are not enough. Inject an additive monitor before `</body>` into a
copy of the overlay (`_test_fix.html`) and, for a before/after contrast, into
`git show origin/main:bgsn-overlay.html` (`_test_main.html`). Per ticker, sample each frame:
- `el.getAnimations()[0].currentTime` (an infinite animation's currentTime grows monotonically,
  so a decrease == the animation was re-created),
- `effect.getTiming().duration` (count changes — the real cause of visible snaps),
- `new DOMMatrixReadOnly(getComputedStyle(el).transform).m41` and compare the per-frame delta
  against the expected `-(halfW/dur)*dt`; flag deviations > ~25px, excluding the frame where
  `dx ≈ +halfW` (that is the legitimate seamless wrap),
- frame `dt` spikes.
Add a "RESET COUNTERS" button and reset on `visibilitychange`: background tabs are rAF-throttled,
so counters collected while a tab was hidden are garbage. Remember to delete the `_test_*.html`
copies from the repo when done.

Notes/pitfalls:
- Chrome on this box has **no GPU** ("Exiting GPU process due to errors during initialization"),
  so GPU-compositing improvements (`will-change`, `translate3d`) cannot be measured as an FPS win;
  idle software-rendered frames already show occasional 40–100 ms hitches. Say so in the report.
- A small (~50 px) lower-ticker position jump a few seconds after overlay load exists on `main`
  as well — it comes from `applyTickerSpeed()` recomputing `scrollWidth` after fonts settle.
  Don't attribute it to the change under test without comparing against `origin/main`.

## Browser/window setup on the box

- Launch with CDP so console output is readable: `google-chrome --no-first-run
  --remote-debugging-port=9222 --window-size=1600,1000 <url> &`. Passing several URLs may still
  open only the first; open the rest with `ctrl+t` then type the URL.
- `browser_console` may report "Could not connect to Chrome via CDP" for a self-launched Chrome;
  fall back to an on-page monitor div and the DevTools Console panel (`ctrl+shift+j`, then reload
  the page with DevTools open so load-time errors are captured).
- Split screen for controller-vs-overlay work (screen is 1600x1200):
  `wmctrl -r "BGSN Control Panel" -e 0,0,0,760,1150` and `wmctrl -r "BGSN Overlay" -e 0,780,0,820,1150`.
  `ctrl+w` with a single tab closes the whole window — check the tab count first.

## Devin Secrets Needed

None — the Firebase config and API key are embedded in the pages.
