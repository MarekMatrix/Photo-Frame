# Brief: make the iPad home-screen web app fill the entire screen

## Context

`index.html` in this repo is a single-file photo-frame web app, served via GitHub Pages
at `https://marekmatrix.github.io/Photo-Frame/`. It is added to the iPad home screen and
run as a standalone web app. It shows my photos full-bleed with no motion and no crop,
one every N minutes.

Everything about it works except one thing: **32 points of the screen are unreachable.**

## Hardware / environment

- iPad Pro 11" (M1), iPadOS, Display Zoom apparently set to More Space
- Screen in landscape: **1389 x 970 points**, devicePixelRatio 2
- The app has a built-in Diagnostics panel (Settings -> Diagnostics) that prints
  `window`, `document`, `visualvp`, `screen`, `avail screen`, `shortfall`, `safe area`,
  `fullscreen`, and measured values of `100vh / 100lvh / 100svh / 100dvh /
  -webkit-fill-available`. **Use it. Do not guess at these values.**

## What we have established on-device

Launched from the home-screen icon (`navigator.standalone === true`):

```
window       : 1389 x 938      <- 32 points short
avail screen : 1389 x 970
shortfall    : 32 pt
safe area    : t32 r0 b25 l0
fullscreen   : unsupported     <- Fullscreen API not offered in standalone
100vh        : 970
100lvh       : 970
100svh       : 938
100dvh       : 938
fill-avail   : 938
```

The missing 32 points is exactly the status bar height.

*(These numbers are all correct. The reading of them was not: `window` and `100svh/dvh`
describe the **layout viewport**, not the web view. `100vh / 100lvh` reporting 970 was the
clue — the view really was 970 all along. See findings 7–10.)*

### Things already tried and their outcomes

1. **`apple-mobile-web-app-status-bar-style: black-translucent`** + `viewport-fit=cover`.
   Web view is positioned at y=0, content draws under the status bar at the top, but the
   viewport is only 938 tall -> **32pt black strip at the bottom**. Confirmed visually with
   a full-document magenta flash: the document genuinely stops 32pt short of the bottom.

2. **`apple-mobile-web-app-status-bar-style: black`.** Web view positioned at y=32,
   938 tall -> reaches the bottom, but there is now an **opaque status bar strip at the top**.
   No gap at the bottom. This is the current `index.html`.

3. **Sizing the stage with `100vh` / `100lvh`** (both of which *measure* 970).
   Does not work. The stage element becomes 970 tall but `#stage` is `position: fixed`
   and fixed elements are clipped to the visual viewport, so the extra 32 points are never
   painted. The measurement is real; the pixels are not. **Do not retry this.**

4. **Fullscreen API from the home-screen icon.** `requestFullscreen` /
   `webkitRequestFullscreen` are not present. Reports `unsupported`.

5. **Fullscreen API in a normal Safari tab.** *Works.* Status bar hides, full 1389 x 970.
   But it needs a user tap per launch and gives up the home-screen icon.

6. **Guided Access.** Hides the status bar, inset drops to 0, full screen. Works, but ends
   on reboot and has to be re-armed by hand.

---

*Findings 7–15 added 2026-08-11. Device: iPadOS, WebKit/Safari 26.6. All measured from the
home-screen icon with `navigator.standalone === true`. The screenshots behind these findings
are in `Testing/`, which is deliberately gitignored — they live on Marek's Mac only.*

7. **Findings 1 and 3 were drawn from an unsound test — disregard their conclusions.** The
   "Flash the page bounds" button painted `#stage`, which is `position: fixed` and sized to
   `innerHeight`. That leaves 32pt unpainted whether the web view frame is short or not, so
   it could never distinguish "the pixels don't exist" from "fixed elements can't reach
   them". It has been replaced with a test that paints the **root element** instead.

8. **The pixels were always there.** A root `background-color` propagates across the entire
   canvas however tall the view frame is, while a root `background-image` is confined to the
   root box. Painting a labelled ruler as the image over a red background-color showed red
   *outside the root box* in every variant: under the status bar with `black`/`default`/
   manifest, and below the page with `black-translucent`. The canvas is the full 1389 x 970.

9. **The layout viewport is 938 tall, and where it sits depends on the status bar style.**
   - `black`, `default`, manifest: screen y = 32…970, `safe-area-inset-top: 0` — spare strip
     at the **top**, under the status bar, where in-flow content cannot reach it.
   - `black-translucent`: screen y = 0…938, `safe-area-inset-top: 32` — spare strip at the
     **bottom**, which in-flow content flows into naturally. This is why the fix uses it.

10. **`position: fixed` is clipped to the layout viewport; in-flow and absolute content is
    not.** A `position:fixed; top:0; height:100lvh` element still stopped at 938 and left red
    canvas below it, *even while `innerHeight` reported 970*. An ordinary in-flow
    `height:100lvh` div painted edge to edge and pushed `innerHeight` itself to 970,
    `shortfall: 0 pt`. **This is the fix.** No element that must reach the screen edge may be
    `position: fixed` — see the comment at the top of the stylesheet in `index.html`.

11. **Portrait behaves identically.** `window 970 x 1389`, `shortfall 0 pt`, edge to edge.

12. **Manifest `display: fullscreen` is half-honoured and useless.**
    `matchMedia('(display-mode: fullscreen)')` reports true, but nothing about the geometry
    changes — status bar stays, shortfall stays 32. Worth knowing: *without* a manifest,
    `display-mode` reports `browser` even though `navigator.standalone === true`.

13. **`black` no longer means an opaque status bar.** On 26.6 the canvas showed through the
    status bar in every style. Only the layout viewport position differs between them. There
    is therefore no meta value that hides or blanks the status bar — it is always a
    translucent overlay over the photo.

14. **`black-translucent` is deprecated** by WebKit, with removal announced, and the fix
    depends on it. If it is ever removed, the fallback is `black` plus shifting the body up
    by `safe-area-inset-top` so it spans 0…970 — **untested**, and worth testing before it
    becomes urgent rather than after.

15. **Not pursued, because they turned out to be unnecessary:** video-element fullscreen
    (`canvas.captureStream()` → `video.webkitEnterFullscreen()`), Apple Configurator
    supervision / Single App Mode, and any per-launch tap. There is no `hidden` or
    `light-content` value for `apple-mobile-web-app-status-bar-style`; that claim appears in
    secondary sources and is false — the only values are `default`, `black`,
    `black-translucent`.

## The goal

Full 1389 x 970 from the **home-screen icon**, with **no per-launch tap** and **no Guided
Access**. If that is impossible, I want to know that clearly rather than have a workaround
dressed up as a fix.

## STATUS: solved (2026-08-11), with one caveat

The 32 points were never missing. The web view canvas is the full 1389 x 970 in every
variant. What is 938 is the **layout viewport**, and `position: fixed` elements are clipped
to it — ordinary in-flow and absolutely-positioned content is not. `#stage` was
`position: fixed`, so it could never reach the last 32 points no matter what height it was
given. That is the entire bug. See findings 7–11 below for the evidence.

The fix is a layout change, not a trick: `index.html` is now one `100lvh`-tall
`position: relative` body with everything absolutely positioned inside it, on
`status-bar-style: black-translucent`. Measured on device: `window 1389 x 970`,
`shortfall 0 pt`, content edge to edge, in both orientations. No tap, no Guided Access,
no manifest, no video hack. **Confirmed working on the device from a re-added home-screen
icon, 2026-08-11.**

**The caveat:** the status bar itself cannot be hidden or blanked. On iPadOS 26.6 it is a
translucent overlay in *every* status-bar style — `black` no longer paints an opaque strip
(finding 13). So the clock, wifi and battery now sit on top of the photo in the top-left and
top-right corners. The screen is fully used; the status bar is still drawn over it. If that
is not acceptable, Guided Access remains the only way to remove it.

## Avenues worth investigating (none verified — treat as hypotheses)

- **Video-element fullscreen.** iOS grants `video.webkitEnterFullscreen()` in places where
  element fullscreen is refused, including possibly standalone mode. Feasibility test:
  render the slideshow to a `<canvas>`, `canvas.captureStream()` into a `<video>`, and put
  the video fullscreen. Check whether it still needs a user gesture, whether it survives
  photo changes, and whether image quality holds up at 2778x1940 device pixels.
- **Web app manifest with `"display": "fullscreen"`.** Recent iOS versions honour
  `manifest.webmanifest` for home-screen apps rather than only the legacy `apple-*` meta
  tags. Worth testing whether `display: fullscreen` (as opposed to `standalone`) removes
  the status bar. Note the manifest is read at add-to-home-screen time.
- **Whether the shortfall differs in portrait**, or after a rotation, or with Display Zoom
  set to Default rather than More Space.
- **Whether the current iPadOS version has changed any of this.** Check the actual OS
  version on the device and look for recent WebKit changelog entries about standalone
  viewport sizing or `viewport-fit`. Some of the behaviour above is long-standing WebKit
  bugginess and may have a bug report with a known workaround.

## Hard constraints

- Single self-contained `index.html`. No build step, no framework, no npm dependencies.
  It is served as a static file and I want to keep being able to read the whole thing.
- No external network calls at runtime. Photos live in IndexedDB on the device and must
  never be uploaded anywhere.
- Do not break existing stored photos. They are keyed to the **origin**
  (`https://marekmatrix.github.io`), so paths and query strings are safe to change,
  but anything that changes the origin loses them.
- Keep the existing Diagnostics panel and the `BUILD` constant + "Check for a newer
  version" mechanism. They are how I verify what is actually running on the device.

## How to test

The GitHub Pages push-and-wait loop is slow (Pages caches for ~10 minutes, and iOS caches
on top of that). Serve locally from the Mac instead and hit it from the iPad over the LAN:

```
python3 -m http.server 8000
# then on the iPad: http://<mac-lan-ip>:8000/
```

Caveat: a plain-http LAN origin is not a secure context, so the Wake Lock API will report
unsupported there. Layout, viewport sizing, and standalone behaviour all test fine.
If you need a secure context, set up a local cert (e.g. mkcert) rather than pushing to Pages
for every iteration.

**Anything that changes `apple-mobile-web-app-*` meta tags or the manifest requires
deleting the home-screen icon and re-adding it** — iOS caches those at install time.
Bake a distinguishing string into the page (the `BUILD` constant already does this) so we
can always tell which variant is actually running.

I have to run every on-device test by hand, so batch your hypotheses: put several variants
behind switchable settings or separate files in one go rather than asking me to re-add an
icon once per idea.

## Also please clean up — all done

- ~~`edge.html` is a duplicate of `index.html`.~~ Deleted; `black-translucent` won and is
  folded into `index.html`.
- ~~The "Stage height source" selector in Diagnostics is dead.~~ Removed, along with the
  `hUnit` setting and the whole `--vh` / `fitViewport()` mechanism it fed. The measured
  `100vh / 100lvh / 100svh / 100dvh / fill-avail` readouts are kept, and a new
  `stage painted` line reports the height the stage actually occupies.
- ~~The settings panel and bottom control bar are positioned against the 938-point
  viewport.~~ Fixed as a side effect of the layout rework: they are absolute inside the
  `100lvh` body, so `env(safe-area-inset-bottom)` now measures from the real screen edge.
- ~~Delete the stray `.DS_Store` and add a `.gitignore`.~~ Done.
- Also removed: the "Try true fullscreen" button and the "Go fullscreen on first tap"
  toggle. Both were workarounds for a problem that no longer exists. The `Fullscreen API`
  readout is kept in Diagnostics.
- ~~The probe harness (`probe.js`, `probe-*.html`, `fs.webmanifest`) is throwaway
  scaffolding.~~ Deleted once the fix was confirmed. The screenshots that settled the
  question are the evidence for findings 7–13 and are worth keeping, but they are 5 MB of
  PNGs and don't belong in the repo, so `Testing/` is gitignored and stays local.

## Ground rules

Verify on the device before claiming anything works — read the Diagnostics `shortfall`
line, don't infer it. If an avenue turns out to be a dead end, say so plainly and add it to
the "already tried" list above with the reason, so we stop circling back to it. **"This is
not achievable in a standalone iOS web app" is a completely acceptable answer**, and a more
useful one than a fix that only appears to work.
