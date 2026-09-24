# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Vanilla-JS Tetris: three files (`index.html`, `style.css`, `game.js`), no dependencies, no build step, no package.json, no test suite.

## Running

Open `index.html` directly, or serve statically for a cleaner reload story:

```bash
python3 -m http.server 8000   # then http://localhost:8000
```

There is nothing to build, lint, or test — verify changes by playing in the browser.

## Architecture (`game.js`)

Everything lives in one file at module top-level scope: constants, canvas/DOM handles pulled once at load, mutable game state in a single `let` declaration list, then plain functions. `'use strict'` at the top; the script tag sits at the end of `<body>`, so DOM lookups are safe without a load listener.

State model:

- `board` is a `ROWS × COLS` matrix of ints: `0` = empty, `1..7` = piece type, which doubles as the index into `COLORS` and `PIECES`. Piece identity, board occupancy, and render color are all the same number — keep that invariant when adding features.
- `current` / `next` are `{ type, shape, x, y }`. `shape` is a *copy* of the `PIECES` template (mutated in place by rotation), so never assign a `PIECES` entry directly to a piece.
- Rotation is computed geometrically (`rotateCW` = transpose + row reverse) rather than stored as a rotation index, and `tryRotate` applies naive horizontal-only wall kicks (`[0,-1,1,-2,2]`) — this is not SRS.

Loop and control flow:

- `init()` resets all state and starts the `requestAnimationFrame` loop; `restartBtn` calls it directly. It cancels any in-flight `animId` first — new loops must preserve that or frames double up.
- `loop(ts)` accumulates `dt` into `dropAccum` and steps the piece down once `dropInterval` is exceeded, then draws every frame. Pause/game over work by cancelling the rAF rather than by an in-loop guard, so `togglePause` must reset `lastTime` before restarting (otherwise a huge `dt` fires an immediate drop).
- `lockPiece()` → `merge()` → `clearLines()` → `spawn()` is the single funnel for a piece landing; both gravity and drops route through it. Game over is detected in `spawn()` when the freshly promoted piece already collides.
- `clearLines` splices full rows and unshifts empty ones, decrementing with `r++` inside the descending loop to re-check the shifted row.

Scoring/speed: `LINE_SCORES[n] * level` for clears, +2/cell hard drop, +1/row soft drop; `level = floor(lines/10)+1` and `dropInterval = max(100, 1000 - (level-1)*90)`.

Rendering: two canvases (`#board`, `#next-canvas`) share `drawBlock`, which takes an explicit context and cell size. The ghost piece is the same draw call with `alpha = 0.2`. The full board is cleared and repainted each frame — no dirty-rect logic.

## Gotchas

- Canvas pixel dimensions are hard-coded in `index.html` (`300 × 600` board, `120 × 120` next). Changing `COLS`, `ROWS`, or `BLOCK` in `game.js` requires updating those attributes to match `COLS*BLOCK × ROWS*BLOCK`.
- `drawNext` centers within a fixed 4×4 grid at `NB = 30`, tied to the 120px preview canvas.
- User-facing strings (overlay text, README, HTML) are in Spanish; keep new UI text in Spanish.
