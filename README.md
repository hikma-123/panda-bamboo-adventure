# Panda Bamboo Adventure

A three-world panda platformer powered by the **University Banana API**
(CIS046-3 Software for Enterprise, Assignment 1 artefact).

Upgraded from *Panda Puzzle Adventure – Enterprise Edition*. The original was a single
`index.html` (2,520 lines) with a falling-items catcher game.

---

## 1. How to run

| Option | Steps |
|---|---|
| **Recommended (VS Code Live Server)** | Open the folder in VS Code → right-click `index.html` → *Open with Live Server*. |
| Python | In the folder: `python -m http.server 8000` → open <http://localhost:8000>. |
| Double-click | Open `index.html` directly. Works because the scripts are plain (non-module) files. If your browser blocks the API request from a `file://` page, use one of the options above. |
| Single file | `panda-bamboo-adventure-single-file.html` has all the CSS and JS inlined. Copy that one file anywhere and open it. |

You need an internet connection, because every puzzle comes live from the API.

**Demo login:** `panda` / `1234`. You can also create your own account.

**Controls:** ← → or A D to move · Space / ↑ / W to jump (double jump in World 3) · P or Esc to pause ·
M for music · N for sound effects · digits 0–9 + Enter in the puzzle window. Touch buttons appear on phones.

---

## 2. Project structure

```
panda-bamboo-adventure/
├── index.html                 Page structure: canvas, HUD, menus, overlays, challenge modal
├── css/style.css              All styling (gaming UI, mascot moods, responsive layout)
└── js/
    ├── core/
    │   ├── eventBus.js        Publish/subscribe EventBus (event-driven backbone)
    │   └── storage.js         Cookies, localStorage, per-player ProgressStore, UNLOCK_RULES
    ├── services/
    │   ├── bananaApi.js       University Banana API client (no visual code)
    │   ├── apiLog.js          Request log & analytics (observer of api:* events)
    │   ├── authService.js     Virtual identity: register/login, salted SHA-256, sessions
    │   ├── telemetry.js       API & gameplay telemetry (observer)
    │   └── audio.js           Procedural music + SFX (Web Audio API, no audio files)
    ├── game/
    │   ├── levels.js          3 level designs, challenge stages, achievements, level generator
    │   ├── pandaRenderer.js   Full-body animated panda (canvas)
    │   ├── environment.js     3 themed worlds: parallax, lighting, terrain, props, particles
    │   ├── input.js           Keyboard & touch input
    │   └── engine.js          Game loop, physics, collisions, state machine
    ├── ui/
    │   ├── icons.js           SVG icon set (replaces the emoji icons)
    │   ├── mascot.js          Full-body SVG panda mascot with moods
    │   ├── pandaSkin.js       Bananas → pandas on the API puzzle image (visual layer)
    │   ├── challenge.js       Banana API Challenge interface & flow
    │   ├── screens.js         Title, locations, HUD, overlays, unlock animation, auth UI
    │   ├── lab.js             Banana API Lab: architecture, playground, pipeline, log, charts
    │   └── diagnostics.js     In-browser automated tests
    └── main.js                Bootstrap & orchestration (navigation, rewards, unlocks)
```

The scripts load in this order: core → services → game → ui → main.

---

## 3. Banana API integration (top priority)

| Requirement | Where / how |
|---|---|
| Real endpoint preserved | `BananaApiClient.BASE_URL = "https://marcconrad.com/uob/banana/api.php"`, requested with `?out=json` (`js/services/bananaApi.js`). |
| No fake puzzles | The original **offline fallback that generated fake puzzles has been removed.** If the API fails, the player sees an error panel with **Fetch New Puzzle** (retry). The game never invents a question. |
| No hard-coded answers | The answer is checked with `BananaApiClient.checkAnswer(puzzle, answer)` against the API's own `solution` field. |
| Fresh puzzle every time | Every challenge, every round and every retry makes a new GET request. Any request still in flight is cancelled with `AbortController`. |
| Loading / error / success | A 9 s timeout and typed errors (`timeout`, `network`, `http`, `parse`, `schema`, `image`). The puzzle image is preloaded, and the UI shows a loading, ready or error state. |
| Data shown unchanged | The `question` image URL from the response goes straight into `<img src>`. Only the frame around it is styled. |
| Clearly highlighted | A gold **LIVE · University Banana API** banner in the challenge window shows the exact request URL, HTTP status, round-trip time and request number. The HUD has an API status lamp (Ready / Requesting / Live · ms / Unreachable). Gate signs read **BANANA API SEAL**. The title screen has a **Banana API Live Demo** button. |
| Demonstrable | **View API JSON** shows the real request and response. The `solution` is masked until you answer, so it can't be used as a cheat. The **System & Diagnostics** panel has a live API test, RTT and accuracy stats, and a JSON export. |
| Kept separate from visuals | `bananaApi.js` has no DOM or panda code. `ui/challenge.js` only presents what the API returns. |
| No secrets | The API needs no key, and none is stored. |

### Challenge progression (implemented by the game – the API has no difficulty levels)

| Stage | Trigger | Rule | Result |
|---|---|---|---|
| **API Challenge Level 1 – Bomb Defuse** | First bomb collision (later bombs reuse it) | 1 puzzle. A wrong answer costs 1 life. | +50 pts, bomb defused |
| **API Challenge Level 2 – Valley Gate Seal** | Reaching the Bamboo Forest gate with enough bamboo | 1 puzzle. A wrong answer loads a fresh puzzle. | +150 pts, **unlocks Mystic Bamboo Valley** |
| **API Challenge Level 3 – Secret World Trial** | Reaching the Valley's Crystal Arch | **2 correct in a row**. A wrong answer restarts the trial with fresh puzzles. | +300 pts, **unlocks Panda's Secret World** |
| **Final API Challenge – Golden Shrine Seal** | Reaching the Golden Shrine in World 3 | 1 puzzle | +500 pts, final completion screen |

**Unlock from the location screen:** tap a locked world to play its Banana API challenge right there
(Valley → Challenge Level 2, Secret World → Challenge Level 3, which needs 2 puzzles in a row). Solving it plays the unlock animation and
opens the world. Reaching the gate at the end of a world does the same. Worlds unlock in order: the Secret World asks you to unlock
the Valley first.

**Colourful puzzle themes:** each challenge has its own palette. Bomb Defuse is *ember* (red/orange), Valley Gate Seal is *mystic*
(purple/teal), Secret World Trial is *blossom* (pink/gold), Golden Shrine Seal is *golden*, and the Live Demo is *rainbow*. Every
theme has a rainbow keypad, floating leaves and stars, a colour burst on correct answers, and a panda peeking over the puzzle board.

**Panda symbol for the Banana API:** wherever the game shows the Banana API (HUD, challenge banner, title, map, gate seals),
the symbol is a panda badge with a live status light.

**Panda skin for the puzzle image** (`js/ui/pandaSkin.js`):
1. The game requests `api.php?out=json&base64=yes`, an option listed in the API documentation, so the image arrives *as data*
   inside the JSON instead of as a link to another website. Because it is same-origin data, the browser allows the code to read its pixels.
   If the base64 form is ever not understood, the game automatically falls back to the normal link form and shows the original image.
2. It finds the yellow banana shapes (connected regions of banana-coloured pixels) and paints a panda face over each one,
   filling the area with the cell's own background colour first.
3. Digits, operators, grid and the answer are unchanged. The answer is still checked against the API's `solution`.
4. **Full restyle:** the puzzle sheet is rebuilt in the game's look. The light-green grid lines are detected and every cell becomes a
   coloured tile on a dark board in the challenge's theme colours. The API's own digits and operators are copied pixel-for-pixel
   (only recoloured: digits glow white, operators turn gold), so every equation reads exactly as the API sent it. Bananas become
   pandas on gold "mystery" tiles. If the grid can't be found, a simpler colour tint is used instead.
5. A badge shows "Panda skin · N bananas → pandas", and **Show original API image** switches to the untouched image at any time.
   This is a visual layer on top of the real API data. It's a good interoperability point for the video.

The game pauses while a puzzle is open. `ChallengeController.isOpen` and the engine's
`challengeLock` together make sure **only one puzzle window can ever be open**.

---

## 3b. Banana API Lab (final-year showcase)

Open it from the title screen (**Banana API Lab**, gold button) or the location screen (**API Lab**).

| Panel | What it shows | Why it matters |
|---|---|---|
| **Live architecture** | Diagram of the system: Player → game client (EventBus, Challenge UI, BananaApiClient, Panda Style, storage) → the University Banana API on another server. A yellow packet flies to the API on every request and a green one comes back, labelled with status, latency and size. Failures turn the API node red. Below it is a strip of interoperability facts (HTTPS, GET, JSON, base64 PNG, no auth, JavaScript ↔ PHP, JSON-contract coupling). | Shows **distributed architecture** and **interoperability**. It's event-driven, so it also animates when a request happens during a game. |
| **API Playground** | Build the request yourself (`out=json`, `base64=yes/no`), see the exact URL, **Send live request**, then view the panda-style image, the original image or the raw JSON (solution hidden). Type an answer to check it against the API's `solution`. | Shows the protocol and the JSON contract, and compares the link and data modes. |
| **Resilience test** | `simulate` = network failure / timeout / HTTP 500. The request is deliberately failed so you can show the error handling: no fake puzzle, a clear message, and a log entry. It's clearly labelled SIMULATED and switches off when you leave the Lab. | Robust handling of third-party failures. |
| **Request pipeline** | Milliseconds spent in each stage: HTTP round trip → JSON parse & validate → image decode → panda style, plus the player's solve time. | Performance thinking: the network dominates and local processing is cheap. |
| **Request log** | Every API call from the game and the Lab: source (e.g. Valley Gate L2), mode, HTTP status, latency, payload size, pandas, answer ✓/✗. Click a row to inspect it. | Traceability and observer pattern (`ApiLog` only listens to events). |
| **Analytics** | Response-time chart, answer-accuracy donut, and how often each solution digit 0–9 came back from the API. | Data from a real third-party service. |
| **KPIs** | Requests, success rate, average and p95 latency, average payload, accuracy. | Service-quality metrics. |

**Suggested demo path (about 2 minutes):** Lab → Send live request (watch the packet) → JSON tab → answer it →
switch `base64` to **no** and send again (the image comes as a link, so the panda style can't apply; explain why) →
simulate **network failure** → show the red API node and the log row → Back → play a level to show requests from the game
appearing in the log.

## 3c. Reset progress (play from the beginning)

**Reset Progress** on the title screen (or **Reset** on the location screen) opens a confirmation dialog. Confirming
locks Mystic Bamboo Valley and Panda's Secret World again, marks every Banana API challenge as unsolved, and clears
scores, stars, bamboo totals, achievements and the best score for the current player (Guest or logged-in account).
The account itself, the sound settings and the API log are kept. The reset is saved, so it survives a refresh.

## 4. Changes made to each file

The original single file was split into modules. Every original feature was either kept or replaced
with something equivalent or better:

| Original (index.html section) | New location | What changed |
|---|---|---|
| CSS block | `css/style.css` | Complete redesign: full-screen canvas with an overlaid HUD (no card-in-card layout or empty areas). Display font Baloo 2 plus Nunito. 3D-style buttons, glowing animated challenge border, mascot mood animations, unlock animation, responsive breakpoints for laptop and phone, and reduced-motion support. |
| HTML markup | `index.html` | New screens: title, location select, HUD, pause, level complete, game over, unlock animation, completion, challenge modal. The banana/emoji icons were replaced with SVG icons and the SVG panda mascot. |
| 1. EventBus | `js/core/eventBus.js` | Same `on`/`emit` API, plus `once()`/`off()`. History is kept for the diagnostics log. |
| 2. CookieStorageRepository | `js/core/storage.js` | Kept unchanged. Added `LocalStore` and `ProgressStore` (per-player save), and declarative `UNLOCK_RULES`. |
| 3. AuthIdentityService | `js/services/authService.js` | Kept salted SHA-256. Accounts moved from a cookie to localStorage (with automatic migration). The session token is now checked on restore, so a forged cookie is rejected. Usernames are validated. |
| 4. BananaApiClient | `js/services/bananaApi.js` | Same endpoint and JSON contract. **Fake offline fallback removed.** Added: typed errors, timeout, request cancellation, image preloading, schema validation and a pure `checkAnswer()`. |
| 5. TelemetryService | `js/services/telemetry.js` | Kept (observer of EventBus). Now also counts API errors and level results. |
| 6. DiagnosticsSuite | `js/ui/diagnostics.js` | Kept the original 4 tests and added 6: API contract validation, answer validation, unlock rules, persistence, duplicate-window guard and level-generator determinism. |
| 7. BambooGameEngine | `js/game/engine.js` + `levels.js` + `environment.js` + `pandaRenderer.js` + `input.js` | The falling-items catcher became a side-scrolling platformer. It keeps the rAF loop, delta time, particles and circle/box bomb collision, and adds: gravity, jumping (coyote time, jump buffer, variable height), double jump, one-way and moving platforms, rocks, ravines, checkpoints, 3 bomb types (static, rolling, falling), camera, gates and a state machine. |
| Canvas panda | `js/game/pandaRenderer.js` | Full-body panda with idle (breathing and blinking), walk, jump, fall, dance, munch, hurt and shock poses. Its scarf colour changes per world. |
| SVG welcome panda | `js/ui/mascot.js` | Rebuilt as a reusable component with moods (idle, thinking, happy, sad, wow, dance) and pupils that follow the mouse. Used beside the puzzle, on the title screen and on the completion screen. |
| 8. UI controller | `js/ui/screens.js`, `js/ui/challenge.js`, `js/main.js` | Split into the presentation (screens), the API challenge flow (challenge) and orchestration (main). |
| – (new) | `js/services/audio.js` | Procedural background music for each world and the challenge, plus about 20 sound effects. Music and SFX toggles are saved. |

---

## 5. Features

1. **Panda reactions:** munching and speech bubbles when collecting bamboo, a startled "Eek!" on bombs, a happy dance on solved puzzles, and a sad face on wrong answers. The mascot changes mood in the challenge window too.
2. **Bamboo counter:** HUD shows `collected / needed`. It glows when the gate can open.
3. **Lives:** three hearts with a lose animation. You lose one for a ravine fall, a wrong bomb defuse or taking the hit.
4. **API challenge indicator:** HUD lamp, challenge chips 1-2-3-★, the challenge track on the map and gate signs.
5. **Location unlock animation:** the lock shakes and shatters, a light burst plays, the world is revealed and a chime sounds.
6. **Level progression:** level tag, a progress bar to the gate, an objective banner and 1–3 stars per world.
7. **Rewards:** bamboo 10, crystal 25, golden bamboo 50. API challenges give +50, +150, +300 and +500. Lives left give a bonus, reward points are totalled, and there are 9 achievements.
8. **Sound controls:** separate music and SFX toggles in the HUD, title and pause menu (M / N keys).
9. **Pause & resume:** P or Esc, or the HUD button. The game auto-pauses when the tab is hidden.
10. **Completion screen:** final score, total bamboo, API puzzles solved, achievement grid, confetti and a dancing panda.

Progress is saved in `localStorage` under `pba_progress_<player>`, so unlocked worlds survive a refresh
and every virtual identity has its own save.

---

## 6. Testing evidence

**Automated browser test:** 73 of 73 checks passed (Chrome, headless, 1440×860, 1280×680 and 390×844 touch).
The university API isn't reachable from the test machine, so **only in the test harness** were requests to
`marcconrad.com/uob/banana/api.php` answered with the same JSON shape and a stand-in image. The game code
itself has no mock and always calls the real endpoint.

| # | Requirement | Result |
|---|---|---|
| 1 | Game starts correctly | Pass |
| 2 | Panda character appears correctly (mascot + in-game) | Pass |
| 3 | Bamboo collection works | Pass |
| 4 | Bomb collision works | Pass |
| 5 | First bomb collision opens API Challenge Level 1 | Pass |
| 6 | API puzzle image loads | Pass |
| 7 | Answers validated against the API solution (correct and wrong) | Pass |
| 8 | API Challenge Level 2 can be completed | Pass |
| 9 | Mystic Bamboo Valley unlocks only after Challenge 2 (stays locked after a wrong answer) | Pass |
| 10 | Level 3 progression (2-round trial, then Secret World, then final seal) | Pass |
| 11 | Locations and buttons work | Pass |
| 12 | Progress persists after refresh | Pass |
| 13 | No duplicate puzzle windows | Pass |
| 14 | No critical console errors | Pass |
| + | API failure shows an error and retry (no fake puzzle) | Pass |
| + | Register/login, salted hash only, session restore, forged cookie rejected | Pass |
| + | Pause/resume, double jump, touch controls | Pass |
| + | Locked worlds unlock by playing their API challenge from the location screen, in order | Pass |
| + | Panda skin: base64 request, 3 bananas replaced, original-image toggle | Pass |
| + | Banana API Lab: live request, panda image, 4-stage pipeline, answer check, JSON view, simulated failure, log, charts | Pass |
| + | Reset progress: worlds lock again, persists after refresh | Pass |
| + | In-app diagnostics suite: 13/13 | Pass |

**Check it yourself in the browser:** *System & Diagnostics → Run Automated Tests*.
The last test makes a real request to the live API.

**Quick demo path for the assessment (about 2 minutes):**
Title → **Banana API Live Demo** (shows the live request, open *View API JSON*) → Play → walk into a bomb
(Challenge Level 1) → collect bamboo → reach the Forest Gate (Challenge Level 2) → watch the Valley unlock
→ refresh the page to show progress persisted.

---

## 7. Where each assessment theme lives in the code

| Theme | Look at |
|---|---|
| **Version control** | The modular structure (one responsibility per file) makes small, focused commits and feature branches practical, e.g. `feature/banana-api-client`, `feature/level-unlocks`, `fix/duplicate-modal`. |
| **Event-driven programming** | `core/eventBus.js`. Engine events in `game/engine.js` (`game:bombHit`, `game:gateReached`, `level:completed`). Handlers in `ui/challenge.js` and `main.js` (`bindGameFlow`). DOM events in `game/input.js`. SFX react to events in `services/audio.js` (`bindEvents`). The live event stream is in System & Diagnostics. |
| **Interoperability** | The **Banana API Lab** (`ui/lab.js`, `services/apiLog.js`) is the live demonstration. `services/bananaApi.js`: HTTPS GET with the Fetch API, a JSON contract, schema validation, timeout/abort, and an image from a remote server. The JSON inspector in the challenge window shows the exchange. |
| **Virtual identity** | `services/authService.js` (salted SHA-256, session token check), `core/storage.js` (cookies vs localStorage, per-identity `ProgressStore`), the cookie consent banner and the identity panel in System & Diagnostics. A known limitation to discuss: it's client-side only. A production system would verify on a server with HttpOnly, Secure cookies. |

---

## 8. References and acknowledgements

- **Banana API:** Marc Conrad, University of Bedfordshire. <https://marcconrad.com/uob/banana/doc.php>
  (provided for the unit; puzzle images and solutions are the API's).
- **Fonts:** *Baloo 2* and *Nunito* via Google Fonts (SIL Open Font License).
- **Web platform APIs:** Fetch, AbortController, Web Crypto (SHA-256), Web Audio, Canvas 2D, localStorage, cookies (MDN Web Docs).
- **Generative AI:** the upgrade (structure, rendering, UI and test code) was produced with the help of an LLM
  (Anthropic Claude), as allowed by the brief for code. All code was reviewed, run and tested before submission.
  This is also noted in the header comment of `index.html`.
