# Recreating Word Worm with Claude

This document is a self-contained project brief. Paste the entire contents (everything below the line) into [claude.ai](https://claude.ai) — or any capable LLM — and you should get back a working, single-file implementation of Word Worm that matches the version in this repository.

Tested with Claude Sonnet 4.5+ and Opus 4.7. For best results, request the file in one shot rather than in parts; the model will keep the HTML, CSS, and JavaScript coherent across the whole document.

If you want to bundle your own dictionary, build a fresh embedded payload after the model produces the file — see the **Dictionary** section at the end.

---

## Project: Word Worm — a single-file word-puzzle PWA

Build a complete, production-quality word-finding game in a **single `index.html` file** with no build step, no server, no external dependencies, and no network calls after page load. The game must look and feel like PopCap's classic *Bookworm*: a honeycomb grid of letter tiles where the player traces words across hex-adjacent tiles, earns gem tiles for longer words, and watches out for burning tiles that can end the game.

The entire game — HTML markup, CSS, and JavaScript — must live in one file. The 172,000-word ENABLE2K dictionary must be **embedded inline as gzipped base64** in a `<script id="dict-data" type="application/octet-stream">` block, and decoded at load time using the browser's native `DecompressionStream` API. No `fetch`, no XHR, no service workers required for gameplay.

### Target platforms

- iPhone Safari 16.4+ (primary). Portrait orientation. Add-to-Home-Screen as a PWA.
- Android Chrome (secondary). Same PWA install flow.
- Desktop Chrome / Edge / Firefox (tertiary, mostly for playtesting).

The layout is mobile-first portrait. Do not build a sidebar — there is no room for it on a 390 px-wide viewport.

---

## Visual design

### Color palette

| Role | Value |
| --- | --- |
| Background base | `#2c1611` (dark walnut) |
| UI panel bg | `rgba(20, 10, 5, 0.85)` |
| Body text | `#ffffff` |
| Accent (gold) | `#f1c40f` |
| Valid word | `#2ecc71` |
| Parchment top | `#f3e0b3` |
| Parchment mid | `#e8d4a2` |
| Parchment bottom | `#c9b07a` |
| Parchment border | `#8b6f3a` |
| Invalid (orange) | `#e67e22` / `#fce18a` |

Use two radial-gradient background images layered on the body to suggest aged parchment / wood: an `ellipse at top` light vignette plus two 40 px-tiled dotted patterns.

### Typography

- Body, buttons, headings: `Georgia, serif`. Do not use Comic Sans except as a deep fallback.
- Tile letters: `bold ${tileSize * 0.45}px Georgia, 'Times New Roman', serif`, centered, with a subtle dark shadow.
- Tile per-letter score: corner-pinned at `${tileSize * 0.16}px`.

### Tile rendering

All tiles are drawn on a single full-window `<canvas>` (devicePixelRatio-aware).

**Plain parchment tile**: rounded-rect with a vertical bevel gradient (3 stops, top→mid→bottom of the parchment palette), a soft drop shadow offset by `(2, 3)`, a translucent top highlight gradient, and a `#8b6f3a` border. When selected and the word is valid, swap the fill to a green gradient `#a8e6c1` → `#27ae60`; when selected and the word is invalid, swap to a warm orange `#fce18a` → `#e67e22`.

**Gem tile**: same shape, but the fill is a diagonal 3-stop gradient using gem-specific colors, plus:
- An animated white **shimmer band** that sweeps horizontally across the tile, clipped to the rounded rect: `(Math.sin(time * 0.003 + px * 0.02) + 1) / 2` parameterizes its position.
- A **pulsing outer glow** via `ctx.shadowColor / shadowBlur` driven by `(Math.sin(time * 0.005) + 1) / 2`.
- A top highlight gradient (white → transparent).

Gem palettes (top, mid, bottom):

| Gem | Top | Mid | Bottom | Multiplier |
| --- | --- | --- | --- | --- |
| Gold | `#fff4b3` | `#f1c40f` | `#b8860b` | ×1.5 |
| Emerald | `#b9f6ca` | `#2ecc71` | `#1e8449` | ×2 |
| Sapphire | `#b3d9ff` | `#3498db` | `#1f4e79` | ×2.5 |
| Ruby | `#ffb3b3` | `#e74c3c` | `#922b21` | ×3 |
| Diamond | `#ffffff` | `#e0f7fa` | `#80deea` | ×4 |
| Fire (hazard) | `#ffeb99` | `#ff6b1a` | `#b71c00` | — |

**Fire tile**: in addition to the gem shimmer/glow, overlay 5 animated radial-gradient flame blobs at the bottom of the tile, oscillating with `Math.sin(time * 0.008 + i)`. Yellow core → orange middle → transparent red edge.

### Arrows between selected tiles

Between each pair of consecutive selected tiles, draw a small **green chevron** with these properties:

- Size: `tileSize * 0.22`
- Shape: 6-point chevron (left rectangle with a triangular notch on the left, point on the right). Coordinates relative to size `s`: `(-s*0.7, -s*0.55) → (s*0.2, -s*0.55) → (s*0.7, 0) → (s*0.2, s*0.55) → (-s*0.7, s*0.55) → (-s*0.2, 0)`.
- Fill: vertical gradient `#a8f5b8` → `#2ecc71` → `#1e8449`.
- Inner highlight: smaller white-alpha chevron on the top edge.
- Border: 2 px `#0a3d1a`.
- Drop shadow: `rgba(0,0,0,0.6)` blur 4, offset `(0, 2)`.
- Gentle bob: translate by `Math.sin(time * 0.012) * 2` along the local x axis after rotating to the tile-to-tile angle.

Under the arrows, draw a **faint connection line** through the selected tile centers: stroke width `tileSize * 0.25`, color `rgba(46,204,113,0.35)` for valid words / `rgba(243,156,18,0.35)` for invalid, `lineJoin: round`, `lineCap: round`.

### Floating particles

When a word is submitted, push a particle at the centroid of the selected tiles:

```
{ x, y, text: `+${pts}`, life: 60, vy: -1.5 }
```

Each frame: y += vy, life--; alpha = `min(1, life/40)`. Render `bold ${tileSize * 0.5}px Georgia, serif`, black 4 px stroke, then a vertical gold gradient fill (`#fff4b3` → `#f1c40f`).

When the player scrambles, push a similar particle from the board centroid with `text: "-50"` and a red gradient fill (`#ff8a80` → `#c0392b`). Use an optional `color` field on the particle object to switch palettes.

---

## Game mechanics

### Grid

- **7 columns × 8 rows.** Even-indexed columns (0, 2, 4, 6) have 8 tiles; odd columns (1, 3, 5) have 7 tiles. Odd columns are staggered **down** by half a tile, producing a honeycomb.
- Tile center: `gridOffsetX + c*tileSize + tileSize/2` horizontally; `gridOffsetY + r*tileSize + stagger + tileSize/2 + animY + shake` vertically, where `stagger = (c % 2 !== 0) ? tileSize/2 : 0`.
- Tile visual size: `tileSize * 0.92`, corner radius `tileSize * 0.13`.
- Compute `tileSize` from `Math.min((width - 20)/(COLS + 0.5), (height - 200)/(ROWS + 0.5))`.

### Letter distribution

Independent weighted random per tile (no bag, no vowel guarantee). Letter weights, derived from PopCap's Bookworm Adventures via the [franklindyer/BWA](https://github.com/franklindyer/BWA) reverse-engineering project, sum to ~100:

```
A 8.50  B 2.07  C 4.54  D 3.38  E 11.16  F 1.81  G 2.47  H 3.00
I 7.58  J 0.20  K 1.10  L 5.49  M 3.01  N 6.65  O 7.16  P 3.17
Qu 0.20 R 7.58  S 5.74  T 6.95  U 3.63  V 1.01  W 1.29  X 0.29
Y 1.78  Z 0.27
```

Per-tile scores (used for the corner number and the base of the word score):

```
A=1  B=4  C=4  D=3  E=1  F=4  G=3  H=4  I=1  J=8
K=5  L=2  M=3  N=2  O=1  P=4  Qu=10 R=2  S=2  T=2
U=2  V=5  W=4  X=8  Y=4  Z=10
```

`Qu` is a single tile rendered as the two-character string `"Qu"`.

### Adjacency (hex)

For tiles `t1` at `(c1, r1)` and `t2` at `(c2, r2)`:

- Same column (`dc = 0`): adjacent iff `|dr| === 1`.
- Adjacent columns (`|dc| === 1`):
  - If `t1.c` is **even**: adjacent iff `dr ∈ {0, +1}` (where `dr = t1.r - t2.r`).
  - If `t1.c` is **odd**: adjacent iff `dr ∈ {0, -1}`.

The sign flip is because odd columns are staggered downward. Get this wrong and the player will only be able to chain diagonally in one direction across every other column.

### Word selection

The player can **tap** individual tiles or **drag** across them. Build the selection like this:

- **handleInputStart(x, y)** (mousedown/touchstart):
  - If no tile is selected yet, start with this tile.
  - If this tile is already the last selected, pop it (tap-to-deselect).
  - If this tile is earlier in the selection, rewind back to it (truncate selection at this index).
  - If this tile is hex-adjacent to the last selected, append it.
  - Otherwise, replace the entire selection with this tile.
  - Set `isDragging = true`.
- **handleInputMove(x, y)** (mousemove/touchmove): while `isDragging`, if the tile under the cursor is adjacent to the current last tile and not already in the selection, append it. If it's the second-to-last selected, pop the last (backwards-drag undo).
- **handleInputEnd**: set `isDragging = false`. **Do not auto-submit.** The Submit button is the only way to commit a word; this prevents accidental submissions and matches Bookworm Adventures Deluxe.

`Math.hypot(px - tile.x, py - tile.y) < tileSize * 0.5` determines hit-testing.

### Word validation and scoring

A selection is valid iff it has ≥3 letters and exists in the loaded dictionary (uppercase). Show the current state in the word display:

- **Empty selection**: blank.
- **Invalid**: render the letters in orange (`#f39c12`) without a breakdown.
- **Valid**: render the word in green with a smaller breakdown line below it.

The breakdown line:

```
{letterScores joined by '+'}    ×{lengthBonus} len    ×{mult} {gem emoji}    =  +{total}
```

The `×... len` chip is shown only when `lengthBonus > 1`. Each gem multiplier in the selection becomes its own chip with the matching emoji: 🟡 Gold, 🟢 Emerald, 🔵 Sapphire, 🔴 Ruby, 💎 Diamond. (Fire tiles don't contribute a multiplier.) Use these CSS hooks:

```html
<span class="mult">×2 len</span>  ...  <span class="total">+140</span>
```

Score formula:

```js
base       = sum of per-tile scores
lengthBonus = Math.max(1, tiles.length - 2)   // 3-letter = 1, 4-letter = 2, …
gemMult    = product of multipliers on selected non-fire gem tiles
total      = Math.round(base * lengthBonus * gemMult * 10)
```

### Submitting a word

When the player presses Submit:

1. Add `total` to the running score; update the score display.
2. Spawn a `+{total}` particle at the centroid of the selected tiles.
3. If the new score beats the saved high score, persist to `localStorage` under the key `wordworm_highscore` and update both the in-game and menu displays.
4. Determine the **gem reward** for the next refill based on word length: 4→gold, 5→emerald, 6→sapphire, 7→ruby, 8+→diamond. (3-letter words give no gem.) Only spawn the gem if there isn't already one of that color on the board.
5. **Remove** every selected tile from its column array (this safely clears any fire tile that was included).
6. **Snapshot** all surviving fire tiles by reference (these are the ones that will descend this turn — newly-dropped fires get one free turn at the top).
7. Increment `wordsSubmitted`. Every 7 words, also drop a fire tile at the top of a random column. Pick separate random columns for the gem and the fire so they don't collide.
8. **Refill** each column by unshifting new `Tile` instances to the front until full capacity is restored, then reindex each tile's `r` property. The first new tile in the chosen column gets the gem; the first new tile in the chosen fire column gets `gem: 'fire'`.
9. **Descend** the snapshotted fires (see Fire tiles).
10. Open the **definition popup** (see below).
11. Clear the selection.

### Fire tiles

- Spawned every 7 submitted words (constant `FIRE_SPAWN_EVERY = 7`).
- Each turn, every snapshotted (pre-existing) fire tile **swaps positions with the tile directly below it in its column**, processing bottom-up so vertically stacked fires cascade correctly. The descend runs *after* refill so the column is at full capacity when the bottom-row check happens.
- If a fire is at `col.length - 1` (the bottom row) when descend runs, trigger **game over**: hide the game layer, set `isPlaying = false`, show a full-screen `BURNED OUT` overlay with the final score and a `PLAY AGAIN` button.
- Include a fire tile in any submitted word to clear it safely (it gets spliced out before the snapshot, so it's not in the descend list).
- Gravity interaction is intentional: when the player removes tiles below a fire, the column shrinks and the next refill drops new tiles on top, effectively pushing the fire toward the bottom. This is faithful to the original.

### Scramble

A third footer button labeled **Scramble** (golden-amber style, distinct from the red Clear and green Submit):

- Re-rolls every tile's `char` and `score` using `getRandomLetter()`.
- Preserves `gem` type, fire state, and grid positions.
- Sets each tile's `animY = -tileSize * 1.5` for a subtle drop-in animation.
- Costs **50 points** (`SCRAMBLE_COST = 50`), clamped so the score never drops below zero (`Math.min(SCRAMBLE_COST, score)`).
- Spawns a red `-{cost}` particle from the centroid of the board to make the cost visible.

### Definition popup

After submit, fetch `https://api.dictionaryapi.dev/api/v2/entries/en/{word.toLowerCase()}` and display the first definition in a popup that fades away after 2.2 seconds. Show the word + `+{pts} points` + a part-of-speech tag like `(n.) `. On 404 or network error, gracefully hide the definition line. This is the only network call in the entire game and the game must remain fully playable when it fails.

---

## Layout & UI shell

```
<body>
  <div id="loading">…</div>
  <div id="menu-screen" style="display:none;">      <!-- shown after dict loads -->
    <h1>WORD WORM</h1>
    <svg>… stylized worm + book … </svg>
    <div class="menu-high-score">High Score: <span id="menu-high-score-val">0</span></div>
    <button onclick="startGame()">PLAY GAME</button>
    <p id="install-tip">…</p>
  </div>
  <div id="game-over-screen" style="display:none;">  <!-- shown on fire bottoming out -->
    <h1 class="burn-title">BURNED OUT</h1>
    <div class="final-score">Final Score: <span id="game-over-final-score">0</span></div>
    <p class="reason">A burning tile reached the bottom row.</p>
    <button onclick="startGame()">PLAY AGAIN</button>
  </div>
  <div id="game-layer" style="display:none;">       <!-- visible during gameplay -->
    <canvas id="gameCanvas"></canvas>
    <div id="ui-layer">                              <!-- pointer-events: none -->
      <div class="header">
        <div class="score-container">
          <div id="score-display">Score: 0</div>
          <div id="high-score-display">High Score: 0</div>
        </div>
        <div id="word-display"></div>
      </div>
      <div class="footer">                          <!-- pointer-events: auto -->
        <button id="btn-clear" onclick="clearSelection()">Clear</button>
        <button id="btn-scramble" onclick="scrambleBoard()">Scramble</button>
        <button id="btn-submit" onclick="submitWord()" style="display:none">Submit</button>
      </div>
    </div>
    <div id="definition-popup">
      <div class="word"></div>
      <div class="points"></div>
      <div class="definition"></div>
    </div>
  </div>
  <script id="dict-data" type="application/octet-stream">…base64…</script>
  <script>… all game code …</script>
</body>
```

### Critical CSS gotchas

- **Do not** give the canvas `z-index: -1`. Because both `html` and `body` have an opaque `background-color`, a negative-z-index canvas paints behind the body's background and disappears. The canvas comes first in the `#game-layer` DOM, so the `#ui-layer` is naturally above it without needing z-index.
- `#ui-layer` must have `pointer-events: none` so canvas clicks/touches pass through; only `.footer` (which contains buttons) and `#definition-popup` should re-enable `pointer-events: auto`.
- Apply `env(safe-area-inset-*)` padding to `#ui-layer` for iPhone notch and home-indicator clearance.
- `touch-action: none` on body, `user-select: none`, and `-webkit-tap-highlight-color: transparent` for a native-feeling mobile interaction.

### Platform-aware install tip

Detect platform at script start and add one of `is-ios`, `is-android`, `is-desktop`, `is-standalone` to `<body>`. iPadOS 13+ identifies as macOS in the UA string — detect it via `navigator.platform === 'MacIntel' && navigator.maxTouchPoints > 1`. Use `window.matchMedia('(display-mode: standalone)').matches` (or `navigator.standalone` on iOS) to detect already-installed PWAs and hide the tip entirely.

Three different tip texts:

- **iOS**: `Tip: On iPhone, tap Share ↑ → Add to Home Screen to install`
- **Android**: `Tip: On Android, tap ⋮ menu → Add to Home screen (or Install app) to install`
- **Desktop**: `Tip: In Chrome or Edge, click the install icon in the address bar (or ⋮ menu → Install Word Worm…) to add it as a desktop app`

---

## PWA manifest & icons

Include all the standard PWA meta tags:

```html
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="WordWorm">
<meta name="theme-color" content="#2c1611">
<link rel="apple-touch-icon" sizes="180x180" href="apple-touch-icon.png">
<link rel="apple-touch-icon" sizes="167x167" href="apple-touch-icon-167x167.png">
<link rel="apple-touch-icon" sizes="152x152" href="apple-touch-icon-152x152.png">
<link rel="icon" type="image/png" sizes="32x32" href="favicon-32x32.png">
<link rel="icon" type="image/png" sizes="16x16" href="favicon-16x16.png">
<link rel="shortcut icon" href="favicon.ico">
<link rel="manifest" href="site.webmanifest">
```

The viewport meta must include `maximum-scale=1.0, user-scalable=no, viewport-fit=cover` to disable pinch-zoom and respect the safe-area insets.

---

## Dictionary

The game accepts any word of 3+ letters that is in the embedded dictionary. Use **ENABLE2K** (172,819 words, public domain). Other lists you might try: TWL06, SOWPODS, Collins Scrabble Words — all give a similar feel. **Do not** use `words_alpha` from dwyl/english-words — it contains far too many archaic/dialectal entries (e.g., `pur`, `vod`) that look like nonsense to casual players.

### Embed flow

To produce the inline `<script id="dict-data">` block, gzip the wordlist at level 9 and base64-encode the result. In Python:

```python
import gzip, base64, pathlib
raw = pathlib.Path("enable2k.txt").read_bytes()
b64 = base64.b64encode(gzip.compress(raw, compresslevel=9)).decode("ascii")
print(f'<script id="dict-data" type="application/octet-stream">{b64}</script>')
```

For ENABLE2K this produces ~614 KB of base64. The full `index.html` ends up around 660 KB on disk — small enough to load and decode in under a second on a phone.

### Decode flow (in-browser)

At script start, *before* any code that touches `Dictionary`:

```js
(async () => {
    try {
        const b64 = document.getElementById('dict-data').textContent.trim();
        const binStr = atob(b64);
        const bytes = new Uint8Array(binStr.length);
        for (let i = 0; i < binStr.length; i++) bytes[i] = binStr.charCodeAt(i);
        const stream = new Blob([bytes]).stream().pipeThrough(new DecompressionStream('gzip'));
        const text = await new Response(stream).text();
        text.split(/\r?\n/).forEach(word => {
            const w = word.trim().toUpperCase();
            if (w.length >= 3) Dictionary.add(w);
        });
        document.getElementById('loading').style.display = 'none';
        document.getElementById('menu-screen').style.display = 'flex';
    } catch (err) {
        document.getElementById('loading').innerHTML =
            'Failed to load dictionary.<br><small>' + err.message + '</small>';
    }
})();
```

---

## Deliverable checklist

- [ ] Single `index.html` file, valid HTML5, no external CSS/JS files.
- [ ] PWA manifest, theme color, apple-touch-icons referenced.
- [ ] Embedded gzipped+base64 ENABLE2K dictionary, decoded via DecompressionStream at startup.
- [ ] Loading screen → menu screen → game layer → game-over screen transitions.
- [ ] 7×8 hex grid, odd columns staggered down half a tile.
- [ ] BWA-derived letter weights, independent weighted random per tile.
- [ ] Tap-to-add / drag-to-chain selection with tap-deselect, tap-rewind, replace-on-non-adjacent.
- [ ] No auto-submit on input end; Submit button only.
- [ ] Live score breakdown beneath the word during selection.
- [ ] 5 gem types with shimmer + glow + length-based rewards + stacking multipliers.
- [ ] Fire tile descend hazard, game-over when a fire reaches the bottom row.
- [ ] Scramble button: -50 pts, re-rolls letters only.
- [ ] Floating gold `+N` and red `-N` particles.
- [ ] Definition popup via `dictionaryapi.dev`, with graceful network failure.
- [ ] High score persisted in `localStorage` under `wordworm_highscore`.
- [ ] Platform-aware install tip on the menu (iOS / Android / desktop / hidden when standalone).
- [ ] No `z-index: -1` on the canvas (it disappears behind the body's opaque background).
- [ ] DevicePixelRatio-aware canvas scaling in `resize()`.

When the file is done, open it in a browser and verify all of the above. The whole thing should be playable offline from `file://`.
