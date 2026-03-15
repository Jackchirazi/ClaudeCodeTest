# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the Games

Open any HTML file directly in a browser — no build step, no server required:

```bash
# Windows
cmd.exe /c start "" "C:\Users\COCK\ClaudeCodeTest\shooter.html"
cmd.exe /c start "" "C:\Users\COCK\ClaudeCodeTest\tictactoe.html"
```

## Git Workflow

After every meaningful change: commit locally with a clean message, then push to GitHub.

```bash
git add <file>
git commit -m "descriptive message"
git push
```

Remote: https://github.com/Jackchirazi/ClaudeCodeTest (branch: `master`)

## Project Structure

Each game is a **single self-contained HTML file** — all CSS, JS, and assets inline. No dependencies, no bundler, no framework.

### shooter.html — Top-Down Shooter

The script is divided into 10 labeled sections (comments mark each):

1. **CONSTANTS & CONFIG** — `CANVAS_W/H` (480×480), `COLORS`, `ENEMY_DEFS` table, `LEVELS` array. Tune gameplay here.
2. **UTILITY** — `circleCollide()`, `screenShake()`, `spawnDeathParticles()`
3. **ENTITY CLASSES** — `Player`, `Enemy`, `Bullet`, `Particle` — all use `alive` flag; dead entries are spliced out backward in the game loop.
4. **PIXEL FONT** — 3×5 bitmap glyphs stored as flat 15-element bit arrays in `FONT` object; rendered by `drawPixelText(ctx, text, x, y, scale, color)`.
5. **DRAW FUNCTIONS** — stateless draw calls (`drawBackground`, `drawPlayer`, `drawEnemy`, `drawBullet`, `drawHUD`, `drawScanlines`, `drawParticles`).
6. **GAME STATE MACHINE** — `GameState.change(state)` calls `state.enter()`. States: `MenuState`, `PlayState`, `LevelCompleteState`, `GameOverState`, `WinState`. Each has `update(dt)` and `draw(ctx)`.
7. **INPUT HANDLER** — `keys` object (keydown/keyup), `mouse` object (`x`, `y`, `down`, `clicked`). Mouse coords always scaled via `(e.clientX - rect.left) * (CANVAS_W / rect.width)`.
8. **GLOBALS** — `player`, `enemies`, `bullets`, `particles`, `score`, `currentLevel`, `killCount`, `shakeTimer`.
9. **GAME LOOP** — `requestAnimationFrame`; delta capped at `1/15s` to handle tab restore spikes.
10. **BOOTSTRAP** — canvas init, `initGame()`, first frame.

**Level clear condition**: `killCount >= lvl.killGoal && enemies.length === 0`

**High score**: stored in `localStorage` key `topdown_hs`.

### tictactoe.html — Tic-Tac-Toe

Standard DOM-based game, no canvas. Self-contained with inline CSS and JS.
