# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Classic Tetris in vanilla JavaScript + HTML5 Canvas + CSS. No dependencies, no `package.json`, no bundler/transpiler, no tests, no linter. The README and in-game UI text are in Spanish.

## Running

There is nothing to install or build. Either open `index.html` directly in a browser, or serve the folder statically (recommended):

```bash
python3 -m http.server 8000   # then open http://localhost:8000
npx serve .                   # alternative
```

Verification is manual: reload the page and play.

## Architecture

Three files: `index.html` (DOM: `#board` canvas, side panel with `#score`/`#lines`/`#level`, `#next-canvas`, and a shared `#overlay` used for both PAUSA and GAME OVER), `style.css` (dark theme), and `game.js` (all logic, loaded as a plain classic `<script>` at the end of `<body>`; it runs `init()` at load).

`game.js` is a single script with module-level mutable state (`board, current, next, score, lines, level, paused, gameOver, lastTime, dropAccum, dropInterval, animId`) and no classes or modules. Key ideas:

- **Board**: `ROWS × COLS` (20 × 10) array of arrays; `0` = empty, `1–7` = locked cell color index into `COLORS`.
- **Pieces**: `PIECES[type]` are square matrices whose non-zero cells hold the piece's own type number, so the same value is both "occupied" and the color index. `merge()` copies those values straight into `board`. A piece is `{ type, shape, x, y }`; `randomPiece()` deep-copies the shape (uniform random, no 7-bag).
- **Rotation**: `rotateCW()` rotates the whole matrix (clockwise only, no SRS). `tryRotate()` applies simple horizontal wall kicks `[0, -1, 1, -2, 2]`.
- **Collision**: `collide(shape, ox, oy)` is the single source of truth for movement, rotation, gravity, ghost projection (`ghostY()`), and spawn/game-over detection. Cells above the board (`y < 0`) are allowed.
- **Lifecycle**: `lockPiece()` = `merge()` → `clearLines()` → `spawn()`. `spawn()` promotes `next` to `current`, rolls a new `next`, redraws the preview, and calls `endGame()` if the new piece already collides.
- **Game loop**: `loop(ts)` via `requestAnimationFrame` accumulates `dt` into `dropAccum` and applies gravity once it reaches `dropInterval`, then calls `draw()` (grid → locked board → ghost at alpha 0.2 → current piece). Pause/game over stop the loop with `cancelAnimationFrame(animId)`; unpause restarts it by calling `loop()` directly.
- **Scoring/level**: `LINE_SCORES[cleared] * level`; soft drop +1 per row, hard drop +2 per row. `level = floor(lines / 10) + 1`, `dropInterval = max(100, 1000 - (level - 1) * 90)` ms. These are recalculated only in `clearLines()`.
- **Input**: one `keydown` listener (`e.code`): arrows move/soft-drop/rotate, `KeyX` also rotates, `Space` hard-drops, `KeyP` toggles pause. `updateHUD()` runs after every key. The restart button calls `init()`, which resets all state.

## Gotchas

- Canvas size is hardcoded in `index.html` (`300 × 600`). If you change `COLS`, `ROWS` or `BLOCK`, update the `<canvas id="board">` width/height to `COLS*BLOCK × ROWS*BLOCK`. The preview uses its own fixed 30px block on a 4×4 area (`drawNext`).
- Unpausing does not re-hide `#overlay`; `togglePause()` only removes `hidden` when pausing.
- When game over is triggered by gravity inside `loop()`, `endGame()` cancels the current frame id but `loop()` then schedules a new frame, so the loop keeps running after game over (input is blocked by `gameOver`). Game over via hard/soft drop from the keyboard does stop it.
