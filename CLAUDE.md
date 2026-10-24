# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Single-file browser simulations of reinforcement learning control, each one self-contained (inline CSS + JS, no dependencies, no build step). Open a file directly in a browser (double-click) to run it.

- `pendulo.html` — inverted pendulum (cart-pole). Port of `Pendulo4.java` from the sibling repo `../Pendulo` (same physics, colors and behavior; keep them in sync when porting changes).
- `semaforo.html` — traffic-light intersection; fuzzy actor-critic learner (75 Takagi-Sugeno rules), expert and fixed-time modes.
- `semaforo2.html` — same intersection controlled by a PPO neural actor-critic; keeps fuzzy, expert and fixed-time as comparison modes.
- `semaforo2_teoria.pdf` — Portuguese theory notes for `semaforo2.html`. It was printed from an HTML source that is not in the repo (headless Chrome `--print-to-pdf`), so updating it means regenerating that HTML.

## Running and testing

There is no build, lint or test suite. Verification is done by running the pages:

- Open: `open -a "Google Chrome" semaforo2.html`. Force a language with `?lang=pt|es|en` (otherwise it follows `navigator.languages`, falling back to English).
- Headless screenshots: `"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --allow-file-access-from-files --virtual-time-budget=3000 --screenshot=out.png file:///path/harness.html`. Headless Chrome cannot make windows narrower than ~500px, so test phone widths by loading the page in an `<iframe>` of the desired width inside a harness page; drive it via `iframe.contentWindow.eval(...)` (e.g. `model.step(0.1)` in a loop to fast-forward, `startSimulation()`, `applyTheme(false)`), and dispatch synthetic `PointerEvent`s on `#scene` to test clicks. Print results with `console.log` and read them with `--enable-logging=stderr --v=0`.
- Learning-quality checks are done in Node: copy the model/learning classes (everything between the "Traffic model"/PPO section and "Application state and simulation loop") into a `.js` file with `module.exports`, then run many simulated hours and compare average delay against the fixed-time baseline across several random seeds.

## Architecture (shared by all three pages)

Each file is one `<script>` organized in the same order; the traffic pages were derived from `pendulo.html`, so most UI code is duplicated across files — a UI fix in one page usually needs to be mirrored in the others.

1. **Constants and i18n**: `STRINGS = {pt, es, en}`, `L = STRINGS[LANG]`, `applyLanguage()` fills elements with `data-i18n`. Every user-visible string must exist in all three languages; numbers go through `num()` (decimal comma for pt/es).
2. **Theme**: `applyPalette(dark)` fills the global `T` palette (canvas colors as `[r,g,b]`, drawn via `rgba()`); `applyTheme()` also pushes CSS variables and repaints sidebar widgets. Choice is persisted in `localStorage` (wrapped in try/catch).
3. **Model** (pure logic, no DOM): physics/traffic simulation plus controllers.
   - Pendulum: `CartPoleModel` (RK4, 5 ms steps, controller every 20 ms), `FuzzyRules` (225 rules) and `FuzzyActorCritic`.
   - Traffic: `Intersection` (4 one-lane approaches, cars follow the Intelligent Driver Model, red light = stationary obstacle at the stop line), `TrafficModel` (Poisson arrivals with a rush-hour sine, a hidden fixed-time "baseline" intersection fed the exact same cars for fair comparison, 2-minute chart periods, semi-Markov reward = −waiting cars, discounted by elapsed time). `semaforo2.html` adds `Mlp`, `PpoAgent` (per-environment trajectories, GAE, clipped objective) and `TrainingWorld` (hidden parallel intersections that feed the PPO buffer).
4. **Simulation loop**: `setTimeout` tick that advances fixed time steps by wall-clock × `speed` (capped by `MAX_CATCH_UP`); rendering is separate on `requestAnimationFrame`.
5. **Sidebar**: every input is a `ValueSlider` (canvas-drawn; double-click resets to default; mouse wheel intentionally ignored). Mode switch is the `.segmented` control mapped to `MODES`.
6. **Canvas scene**: drawn in logical pixels scaled by `view.scale`; `layout()` handles three cases — wide (sidebar on the right), stacked (`max-width: 999px`, scene above sidebar, header moved to the top by `placeHeader()`, speed slider moved below the scene by `placeSpeedSlider()`), and narrow (canvas < 600 CSS px: cards stacked, two-column telemetry, bigger floor buttons). Font sizes go through `setFont()`/`ui()` with `FONT_SCALE`. Static scenery is cached in an offscreen `backdrop` canvas keyed by size/theme.
7. **HUD**: telemetry card, learning card (bar chart + moving average + rule/policy map), floor buttons (start/pause, restart, push/burst), toast, footer with keyboard shortcuts. Click targets are tracked as `*Bounds` rectangles set during painting and hit-tested in the pointer handlers.

## Conventions

- Code, identifiers and comments in English; UI text in pt/es/en via `STRINGS`.
- Perturbation controls (push buttons, bursts, manual light switch) only act while the simulation is running (`canPush()` / `canInteract()`), and are drawn dimmed otherwise.
