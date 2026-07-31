# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Vanilla JavaScript Tetris — HTML5 Canvas + CSS, no dependencies, no build step, no `package.json`. Just static files served or opened directly.

## Running

No install/build required.

```bash
open index.html        # macOS
xdg-open index.html    # Linux
start index.html       # Windows
```

Or serve locally (needed if testing anything that would be blocked by `file://` restrictions):

```bash
python3 -m http.server 8000
npx serve .
php -S localhost:8000
```

There is no test suite, linter, or build/lint/typecheck command in this repo.

## Architecture

Three files, all logic lives in `game.js`:

- `index.html` — DOM shell: `<canvas id="board" width="300" height="600">` for the board, a `<canvas id="next-canvas">` for the piece preview, HUD spans (`#score`, `#lines`, `#level`), a hidden `#overlay` reused for both Pause and Game Over states (with a `#gameover-extra` sub-block for the highscore table/form, hidden during Pause), and an initial `#start-screen` overlay shown before the first game.
- `style.css` — dark/retro arcade visuals.
- `game.js` — game state and loop, all in module-level `let` variables (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `dropInterval`, `combo`, `bestCombo`, `started`, etc.), no classes/modules.

Key mechanics:

- **Board model**: `ROWS × COLS` matrix, each cell is `0` (empty) or a piece color index `1–7`.
- **Pieces**: square matrices in `PIECES`; rotation is done via `rotateCW` (transpose + reverse rows), not precomputed rotation states.
- **Collision** (`collide`): checks board bounds and existing locked cells.
- **Wall kicks** (`tryRotate`): after rotating, tries offsets `[0, -1, 1, -2, 2]` until one doesn't collide.
- **Game loop** (`loop`): driven by `requestAnimationFrame`; accumulates `dt` in `dropAccum` and advances the piece one row once `dropInterval` is exceeded.
- **Line clear** (`clearLines`): scans bottom-up, splices full rows out and unshifts empty rows in; re-checks the same index after a splice.
- **Scoring**: `LINE_SCORES = [0, 100, 300, 500, 800]` × current level; hard drop adds 2 pts/row dropped, soft drop adds 1 pt/row.
- **Leveling**: level = `floor(lines / 10) + 1`; `dropInterval = max(100, 1000 - (level - 1) * 90)` ms.
- **Ghost piece** (`ghostY`): projects the current piece straight down to its landing row, drawn at `globalAlpha = 0.2`.
- **Game over**: triggered in `spawn()` when the newly spawned piece already collides at its start position.
- **Combo tracking**: `clearLines()` returns the number of rows cleared. `lockPiece()` increments module-level `combo` when that count is `> 0` (reset to `0` on a lock that clears nothing) and tracks `bestCombo` (the run's high point) whenever `combo` exceeds it. Both reset in `init()`.
- **Start screen**: `#start-screen` overlay is shown by default (no `hidden` class) and the game loop does not start until "Jugar" (`#play-btn`) is clicked, which hides it and calls `init()`. A `started` flag guards the `keydown` listener so key presses before the first "Jugar" click are no-ops. `renderStartScreen()` populates its highscore table and all-time best combo/max lines on load and after a reset.
- **Highscore table & stats persistence**: on `endGame()`, all-time stats (`tetris-stats`: `{bestCombo, maxLines}`) are updated if the current run beat them, then the top-5 table (`tetris-highscores`: array of `{name, score, lines, level, combo, date}`, sorted by `score` descending, capped at 5) is rendered into `#gameover-extra`. If the current score would place in the top 5, a name `<input>` + save button (`#save-score-form`) is shown; saving inserts the entry, re-sorts/trims/persists, and re-renders the table with the new row highlighted (`.highscore-new`). All `localStorage` reads/writes for both keys are wrapped in try/catch with defensive JSON-shape validation, falling back to `[]` / `{bestCombo: 0, maxLines: 0}` so a corrupt value or disabled storage (e.g. `file://`, private browsing) never throws.
- **Reset records**: "Resetear records" buttons on both the start screen and game-over screen call `resetRecords()`, which `confirm()`s, clears both `localStorage` keys, and re-renders whichever table(s) are visible.

Tunable constants at the top of `game.js`: `COLS`, `ROWS`, `BLOCK`, `COLORS`, `PIECES`, `LINE_SCORES`. If `COLS`/`ROWS`/`BLOCK` change, update the `#board` canvas `width`/`height` in `index.html` to match (`COLS × BLOCK`, `ROWS × BLOCK`).
