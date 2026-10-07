# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Vanilla JavaScript Tetris on HTML5 Canvas. No dependencies, no `package.json`, no build step, no tests, no linter. The UI text and README are in Spanish; keep user-facing strings in Spanish.

## Running

Open `index.html` directly in a browser, or serve the folder statically (recommended):

```bash
python -m http.server 8000   # then http://localhost:8000
npx serve .
```

## Architecture

Three files: `index.html` (DOM + two canvases + overlay), `style.css` (dark theme), and `game.js`, which holds all the logic.

`game.js` is one script with no modules and no classes. All game state lives in module-level `let` variables (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `lastTime`, `dropAccum`, `dropInterval`, `animId`). `init()` resets every one of them and is also the restart handler. If you add new state, reset it in `init()`.

Key conventions:

- **Cell values double as color indices.** `board` is a `ROWS × COLS` matrix of `0` (empty) or `1–7`. Each piece matrix in `PIECES[type]` is filled with its own `type` number, so `merge()` copies shape values straight into the board and `drawBlock` looks them up in `COLORS`. `PIECES`, `COLORS`, and the type number must stay aligned by index; index `0` is `null` in both arrays.
- **`collide(shape, x, y)` is the single validity check** used for movement, rotation (with horizontal wall kicks `[0,-1,1,-2,2]` in `tryRotate`), ghost projection (`ghostY`), and spawn/game-over detection. Cells with `y < 0` are allowed (above the board).
- **Piece lifecycle:** `lockPiece()` → `merge()` → `clearLines()` (updates score, level, `dropInterval`) → `spawn()` (promotes `next` to `current`, rolls a new `next`, redraws the preview, and calls `endGame()` if the new piece collides immediately).
- **Rendering:** `loop()` runs on `requestAnimationFrame`, accumulates `dt` into `dropAccum` for gravity, and redraws the whole board every frame (`draw()`: grid → locked cells → ghost at alpha 0.2 → current piece). The next-piece canvas is redrawn only in `spawn()`.
- **HUD:** `updateHUD()` writes to the DOM. The keydown handler calls it after every key, so `hardDrop()` relies on that rather than calling it itself.

## Coupled values

- `COLS × BLOCK` and `ROWS × BLOCK` must match the `<canvas id="board">` `width`/`height` in `index.html` (300 × 600).
- The preview canvas (`#next-canvas`, 120 × 120) assumes a 4×4 grid at the hard-coded `NB = 30` in `drawNext()`.
