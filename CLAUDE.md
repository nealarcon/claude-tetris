# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Vanilla Tetris implementation using HTML5 Canvas. No dependencies, no build step, no package.json, no test suite.

## Running

Open `index.html` directly in a browser, or serve it statically:

```bash
python3 -m http.server 8000
# or
npx serve .
```

There is no build, lint, or test command — this is plain JS/CSS/HTML consumed as-is by the browser.

## Architecture

Three files, no modules/bundler — `game.js` is loaded directly via `<script src="game.js">` and relies on global scope.

- **`index.html`** — DOM shell: `#board` canvas (300×600, 10×20 grid at `BLOCK=30`px), `#next-canvas` preview, HUD spans (`#score`, `#lines`, `#level`), and the `#overlay` used for both pause and game-over.
- **`style.css`** — dark/retro theme, flexbox layout, no build step.
- **`game.js`** — entire game logic, all state as module-level `let` variables (`board`, `current`, `next`, `score`, `lines`, `level`, `paused`, `gameOver`, `dropInterval`, etc.). No classes, no state management library.

### Core model

- **Board**: `ROWS × COLS` matrix (`createBoard`), each cell is `0` (empty) or a color index `1–8` identifying which piece locked there.
- **Pieces**: `PIECES` array of square matrices (index 0 unused/null so piece type IDs 1–8 index directly into `PIECES`/`COLORS`; ID 8 is the 3×3 "tuerca" ring with an empty center). `randomPiece` picks uniformly from `PIECES.length - 1` types. Rotation (`rotateCW`) is a matrix transpose + reverse, not precomputed rotation states.
- **Collision** (`collide`): bounds + board-cell overlap check, used for movement, rotation, and ghost-piece projection.
- **Wall kicks** (`tryRotate`): after rotating, tries offsets `[0, -1, 1, -2, 2]` columns until one doesn't collide.
- **Lock → clear → spawn pipeline**: `lockPiece()` calls `merge()` (bake current piece into `board`) → `clearLines()` (bottom-up full-row sweep, splice + unshift empty row at top) → `spawn()` (promote `next` to `current`, generate new `next`, check spawn collision → `endGame()`).
- **Game loop** (`loop`, driven by `requestAnimationFrame`): accumulates delta time in `dropAccum`; once it exceeds `dropInterval`, advances the piece one row or locks it. `draw()` runs every frame regardless (grid → locked board → ghost piece at `globalAlpha=0.2` → current piece).
- **Scoring/leveling**: `LINE_SCORES` table `[0,100,300,500,800]` × `level`; hard drop is 2 pts/row, soft drop 1 pt/row. Level = `floor(lines/10)+1`; `dropInterval = max(100, 1000 - (level-1)*90)`.

### Input

Single `keydown` listener switches on `e.code`; movement/rotation/drop are ignored while `paused` or `gameOver` (except `KeyP`/`Escape`, which always toggle pause via `handlePauseMenuKey()`). `togglePause()` drives `#pause-menu`; `endGame()` drives `#overlay`.

### Pause menu

`P`/`Escape` toggle `togglePause()`, which shows the separate `#pause-menu` overlay (the `#overlay` element is now game-over only). Buttons: Reanudar, Reiniciar (calls `init()`, no page reload), Ver controles (toggles a key list), and a Nivel inicial `<select>` (1–10) stored in `startLevel` and persisted in `localStorage('startLevel')`. `init()` starts at `level = startLevel` and `clearLines()` levels with `max(startLevel, floor(lines/10)+1)`. While paused, `handlePauseMenuKey()` swallows all game keys (Arrow Up/Down move focus between menu items); on resume `lastTime`/`dropAccum` are reset and game keys are ignored for 150 ms (`inputBlockedUntil`). Menu buttons `blur()` after click so `Space` doesn't re-trigger them. Code lives in the "Menú de pausa" section at the end of `game.js`.

### Theming

Dark is the default. All colors in `style.css` are CSS variables on `:root` (dark), overridden by `[data-theme="light"]` on `<html>`. `#theme-toggle` (in `index.html`) flips the attribute via `applyTheme()` at the bottom of `game.js`, persists the choice in `localStorage` (`theme`), and calls `blur()` so `Space` doesn't re-trigger the button. The canvas grid color is read from `--grid-color` into `gridColor` (canvas can't use CSS vars directly); the canvas background comes from CSS. New colors should be added as variables in both themes.

### Tunable constants (top of `game.js`)

`COLS`, `ROWS`, `BLOCK`, `COLORS`, `PIECES`, `LINE_SCORES`, initial `dropInterval`. If `COLS`/`ROWS`/`BLOCK` change, update the `#board` canvas `width`/`height` in `index.html` to match (`COLS×BLOCK` by `ROWS×BLOCK`).
