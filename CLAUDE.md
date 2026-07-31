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

Three files, all logic lives in `game.js` (~300 lines):

- `index.html` — DOM shell: `<canvas id="board" width="300" height="600">` for the board, a `<canvas id="next-canvas">` for the piece preview, HUD spans (`#score`, `#lines`, `#level`), a hidden `#overlay` used only for the Game Over state, and a separate hidden `#pause-menu` overlay for pausing.
- `style.css` — dark/retro arcade visuals.
- `game.js` — game state and loop, all in module-level `let` variables (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `dropInterval`, etc.), no classes/modules.

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
- **Pause menu** (`#pause-menu`, `togglePause()`): `KeyP` and `Escape` both call `togglePause()`, which toggles `paused` and shows/hides `#pause-menu` (game-over still uses the separate `#overlay`). While `paused` is true the `keydown` listener returns early (after `preventDefault()`-ing arrows/Space so focused buttons don't get clicked and the page doesn't scroll) so game inputs are blocked. The menu has: Reanudar (`resumeBtn` → `togglePause()`), Reiniciar (`pauseRestartBtn` → `init()`), Ver controles (`toggleControlsBtn` toggles the `.hidden` class on `#pause-controls-list`, which reuses the existing `.controls`/`kbd` CSS), and a Nivel inicial `<select>` (`startLevelSelect`, options 1–15) whose choice is persisted to `localStorage['tetris-start-level']` and read back by `getStartLevel()` — `init()` calls `getStartLevel()` to set `level` and derives `dropInterval` from it instead of hardcoding `1`/`1000`.

Tunable constants at the top of `game.js`: `COLS`, `ROWS`, `BLOCK`, `COLORS`, `PIECES`, `LINE_SCORES`. If `COLS`/`ROWS`/`BLOCK` change, update the `#board` canvas `width`/`height` in `index.html` to match (`COLS × BLOCK`, `ROWS × BLOCK`).
