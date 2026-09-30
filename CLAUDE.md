# MMM-SymmetricIcons — context for Claude sessions

Charles's MagicMirror² module: symmetric chaotic icons after Field & Golubitsky's *Symmetry in
Chaos*, one point hopping millions of times until a symmetric picture develops, for his hallway
mirror, as one page in a rotation of pages. Split out of MMM-ChaosTheory on 2026-09-28 with its
history (it was the `icons` simulation of that module's chaos page).

## Files

- `MMM-SymmetricIcons.js` — module shell: one canvas plus an HTML caption (equations + live
  readout, updated 2×/s). New sim every `cycleSeconds` (60) and on each `resume()`. Loop:
  `setTimeout` until a frame is due, then one `requestAnimationFrame`. `suspend()` stops it; a
  sim with `resting = true` is polled only every 500 ms; while MagicMirror fades the module out
  (`hidden` is set at the start, `suspend()` comes after), frames draw nothing.
  `turns: { of, at }`: only every nth showing; otherwise the wrapper gets `display: none` and
  nothing starts.
- `simulations/icons.js` — the `icons` sim on `window.IconsSimulations`. The map
  z → (λ + α z z̄ + β Re(zⁿ) + iω) z + γ z̄ⁿ⁻¹; five parameter sets, four colour ramps, both
  picked at random per showing (not a deck). Each frame hops for `budgetMs` (9 ms), counting
  visits per pixel in a `Uint32Array`; twice a second the counts are coloured (log scale) and
  put back with one full-canvas `putImageData`. After `developSeconds` (45) a last colouring,
  then rests. `icon`, `budgetMs` and `developSeconds` are read from the module config (the sim
  gets the whole config), but aren't in `defaults` or the README; `?icon=1` in the preview.
- UMD-style, so the maths runs in Node: `tests/icons.test.js` (`node --test`, no
  dependencies): equivariance under rotation by 2π/n, bounded and chaotic, n-fold symmetric
  sector counts (within 3% over 10⁶ hops), inlined hop = `hop()`.
- `node_helper.js` — the stats panel (`statsPanel: true`): CPU of Electron and cage, per core,
  temperature, from `/proc`, only while shown.
- `dev/preview.html` — runs the module in a desktop browser (`python3 -m http.server` in the
  repo, then `/dev/preview.html?icon=1`).

Note: the header comment in `icons.js` and the caption say the map commutes with reflection /
has "the symmetry of a regular n-gon"; that holds only for ω = 0. With ω ≠ 0 (pentagram,
swirl, square vortex) it commutes with the rotations only. The tests check rotation only.

MMM-ChaosTheory has the same simulation (`icons`): fixes to the simulation belong in both. The
shell (`MMM-SymmetricIcons.js`, `node_helper.js`'s stats panel, `dev/preview.html`) is shared in
spirit with the sibling modules (MMM-ChaosTheory, MMM-LorenzAttractor, MMM-DoublePendulum,
MMM-FractalBasins, MMM-LogisticMap, MMM-ThreeBody, MMM-ChaoticBilliards, MMM-Rule30, and the
non-chaos pages MMM-Atom, MMM-FractalZoom, MMM-Chladni, MMM-SacredGeometry, MMM-Tilings,
MMM-PlanetsDance, MMM-SnowCrystal, MMM-NightSky, MMM-PhotoDeck, MMM-StandardMap, MMM-ChaoticWaterwheel, MMM-DoubleSlit, MMM-Sandpile, MMM-Harmonograph): a fix there probably belongs
in the siblings too.

On the mirror this simulation runs as part of MMM-ChaosTheory's chaos page (60 s, one
simulation per showing), not as this module.

## Measured cost on the Pi

900², 20 fps, Electron + cage: 66% of a core while developing (~45 s, at 17 fps), ~7% once
developed. Hidden: 0.3% (baseline 0.2%). The hopping is JS under a fixed 9 ms budget per frame;
the canvas cost is the twice-a-second full-canvas colouring.

## Performance findings on the Pi (measured)

- A frame that changes the canvas costs ~2%/fps fixed; beyond that, cost scales with the
  **bounding box of everything changed in the frame**. Full redraws of a 900² canvas at 20 fps
  saturate the pipeline (~150%). JS is never the bottleneck (<3 ms/frame; here the hopping
  deliberately takes 9 ms).
- So: keep each frame's changes spatially compact (or, as here, rare), and rest when the
  picture is static. Line width, opacity, `rAF` vs timer made no difference.
- MagicMirror applies `electronSwitches` after app ready, so `remote-debugging-port` can't be set
  that way; use `debugStats: true` and a `grim` screenshot to see fps on the Pi.

## Hard constraints: the target device

- **Raspberry Pi 3 B+, 905 MB RAM, 64-bit Debian 13.** Mirror runs Electron 42 in a cage
  Wayland kiosk.
- **No GPU acceleration, and it can't be enabled**: the Pi 3's VideoCore IV only does GLES 2.0,
  Chromium needs ES 3.0 (tested). All canvas drawing is CPU. **No WebGL / three.js.**
- Screen will be **portrait 1200×1920** once mounted (Dell U2413, rotated). Design for portrait.
- Electron baseline is ~0.5% of one core. **Measure, don't guess**: on the Pi,
  `~/.cache/mm-sample.sh 60` prints Electron CPU% and RSS over 60 s. Record before/after numbers
  in the README.
- The mirror rotates pages every 15-30 s (MMM-pages, which hides/shows modules). `suspend()` and
  `resume()` must fire on page changes, or the loop burns CPU 24/7.

## Deploying and testing

- This repo is public so the Pi can `git clone`/`git pull` without credentials.
- Pi access: `ssh fatherson@raspberrypi.local` (key auth). Module path:
  `~/MagicMirror/modules/MMM-SymmetricIcons`. Restart: `pm2 restart MagicMirror`
  (pm2 is in `~/.npm-global/bin`). Logs: `pm2 logs MagicMirror`.
- The mirror's **config.js lives in a separate private repo**, `charleswest775/magicmirror-setup`
  (cloned at `~/dev/magicmirror-setup`). Add the module's config block there, then
  `./deploy.sh diff` and `./deploy.sh push` (push validates config before restarting).
  Don't hand-edit config.js on the Pi without `./deploy.sh pull` afterwards.
- Faster iteration: run it in a desktop browser (`dev/preview.html`), then confirm performance
  on the Pi.
- Commit as Charles's GitHub noreply address (set in this repo's git config).
