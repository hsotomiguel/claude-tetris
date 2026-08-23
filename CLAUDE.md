# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A classic Tetris implementation in vanilla JavaScript with HTML5 Canvas — no dependencies, no build process, no `package.json`. The entire game lives in three files:

- `index.html` — DOM structure: main `<canvas id="board">` (300×600, i.e. `COLS×BLOCK` × `ROWS×BLOCK`), the `<canvas id="next-canvas">` preview, HUD panel, and pause/game-over overlay.
- `style.css` — dark/retro arcade visual theme.
- `game.js` — all game logic (~300 lines, single file, no modules).

## Running the game

No install/build step. Open `index.html` directly, or serve it statically:

```bash
python3 -m http.server 8000
# or
npx serve .
```

There is no test suite, linter, or build tooling in this repo — verify changes by opening the page in a browser and playing.

## Architecture (game.js)

Everything is global state and top-level functions operating on it — no classes, no modules.

- **Board model**: `board` is a `ROWS × COLS` matrix; each cell is `0` (empty) or a piece color index `1–7`.
- **Pieces**: `PIECES` are square matrices keyed by color index (see `COLORS`). Rotation is done via `rotateCW` (transpose + reverse), not by storing rotation states.
- **Collision**: `collide(shape, ox, oy)` is the single source of truth for both boundary and stack collisions; used for movement, rotation, ghost projection, and spawn-blocking (game over) checks.
- **Wall kicks**: `tryRotate()` rotates then retries the placement at offsets `[0, -1, 1, -2, 2]` before giving up on the rotation.
- **Game loop**: `loop(ts)` runs via `requestAnimationFrame`, accumulates elapsed time in `dropAccum`, and advances the piece one row once `dropAccum >= dropInterval`. Pausing/resuming works by cancelling/restarting this rAF chain (`togglePause`), not by gating logic inside the loop.
- **Locking/spawning**: `lockPiece()` → `merge()` (bake piece into `board`) → `clearLines()` → `spawn()`. `spawn()` promotes `next` to `current` and generates a new `next`; if the new `current` immediately collides, `endGame()` fires.
- **Line clearing**: `clearLines()` scans bottom-up, splices out full rows and unshifts empty ones at the top; re-checks the same row index after a splice (`r++` inside the loop) since rows shift down.
- **Scoring/leveling**: `LINE_SCORES = [0,100,300,500,800]` × `level`; hard drop adds `2` × cells dropped, soft drop adds `1` per row. `level = floor(lines/10) + 1`; `dropInterval = max(100, 1000 - (level-1)*90)`.
- **Ghost piece**: `ghostY()` projects `current` straight down until collision; drawn at `globalAlpha = 0.2` in `draw()`.
- **Rendering**: `draw()` (main board + grid + ghost + current piece) and `drawNext()` (preview canvas) both go through `drawBlock()`, the shared per-cell renderer.

Tunable constants live at the top of `game.js`: `COLS`, `ROWS`, `BLOCK`, `COLORS`, `LINE_SCORES`, initial `dropInterval`. If `COLS`/`ROWS`/`BLOCK` change, update the `#board` canvas `width`/`height` in `index.html` to match (`COLS×BLOCK`, `ROWS×BLOCK`).
