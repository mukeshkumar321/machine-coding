# Tic-Tac-Toe (N × N) & Connect-4 — Machine Coding

> **Difficulty:** Medium | **Time:** 45 min | **Frequency:** ⭐⭐⭐⭐

## Table of Contents

1. [Problem Statement](#1-problem-statement)
2. [Requirements](#2-requirements)
3. [Clarifying Questions to Ask](#3-clarifying-questions-to-ask)
4. [Approach & Component Design](#4-approach--component-design)
5. [Reference Solution](#5-reference-solution)
6. [How It Works](#6-how-it-works)
7. [Edge Cases](#7-edge-cases)
8. [Accessibility](#8-accessibility)
9. [Performance Considerations](#9-performance-considerations)
10. [Follow-up Questions](#10-follow-up-questions)
11. [Common Mistakes](#11-common-mistakes)
12. [Interview Tips & Time Plan](#12-interview-tips--time-plan)
13. [Practice Checklist](#13-practice-checklist)

---

## 1. Problem Statement

Build **Tic-Tac-Toe** on an **N × N** board (N is configurable) with:

- Win detection for any N (N in a row)
- **Undo / Redo** (and jump-to-move) via a history of board snapshots
- A **Connect-4** variant (7 × 6, gravity drop, 4 in a row) reusing the same engine

---

## 2. Requirements

### Functional

- Players alternate X / O (Red / Yellow in Connect-4).
- Click an empty cell (TTT) or a column (Connect-4, piece falls to the lowest empty row).
- Detect winner and highlight the winning cells; detect draw (board full).
- No moves after the game is over; no overwriting occupied cells.
- Undo / Redo; making a move after undo discards the redo branch.
- Reset; change board size N.

### Non-functional

- Game logic = **pure functions** (no React) + a reducer; UI is a thin layer.
- Win check is O(k) per move (only around the last move), not O(N²).
- Derived values (winner, next player, draw) are **not stored**.

---

## 3. Clarifying Questions to Ask

- Win condition on larger boards: N in a row, or fixed K (e.g. 4 on 5×5)?
- Max board size? Rectangular boards?
- Undo/redo, or full "jump to any move" history?
- Two local players only, or an AI opponent / online play?
- Who starts? Score tracking across games?

---

## 4. Approach & Component Design

**State shape — snapshot history**

```js
{ history: [ { cells: Array(rows*cols), move: index|null }, ... ], step: number }
```

| Value | How it's obtained |
|---|---|
| `current` | `history[step]` |
| next player | `step % 2` → `players[0]` or `players[1]` |
| `winLine` | `getWinningLine(current.cells, …, current.move, k)` |
| `winner` | `current.cells[current.move]` if `winLine` |
| `isDraw` | no winner and `step === rows * cols` |
| `canUndo / canRedo` | `step > 0` / `step < history.length - 1` |

Undo/redo only moves the `step` pointer. A new move does `history.slice(0, step + 1)` then appends → redo branch is dropped.

**Win check from the last move** — for each of 4 axes (row, column, 2 diagonals) count contiguous same-symbol cells in both directions from the placed piece; win if count ≥ K.

```
directions: [0,1] → row   [1,0] → column   [1,1] ↘   [1,-1] ↙
            ← 2 cells ← ( X ) → 1 cell →   line length = 2 + 1 + 1 = 4  ✔ (Connect-4)
```

**Component tree**

```
App (mode: tictactoe | connect4, size N)
 └─ Game key={mode-N}       ◄─ key remounts → fresh state when config changes
     ├─ useGame({ rows, cols, k, players, gravity })  ← useReducer + gameLogic
     ├─ Status (turn / winner / draw)
     ├─ Board → (TTT) <Cell/> grid   |   (C4) <Column/> buttons of <Disc/>
     ├─ Controls (Undo · Redo · Reset)
     └─ Moves list (jump to step)
```

---

## 5. Reference Solution

<details>
<summary><strong>Show Full Code (gameLogic.js, useGame.js, Board.jsx, Game.jsx, App.jsx, styles.css)</strong></summary>

```jsx
// gameLogic.js — pure, no React
const DIRECTIONS = [[0, 1], [1, 0], [1, 1], [1, -1]];

export function getWinningLine(cells, rows, cols, index, k) {
  const player = cells[index];
  if (!player) return null;

  const r0 = Math.floor(index / cols);
  const c0 = index % cols;

  for (const [dr, dc] of DIRECTIONS) {
    const line = [index];
    for (const sign of [1, -1]) {
      let r = r0 + dr * sign;
      let c = c0 + dc * sign;
      while (r >= 0 && r < rows && c >= 0 && c < cols && cells[r * cols + c] === player) {
        line.push(r * cols + c);
        r += dr * sign;
        c += dc * sign;
      }
    }
    if (line.length >= k) return line;
  }
  return null;
}

// Connect-4 gravity: lowest empty cell in a column, or -1 if the column is full
export function getDropIndex(cells, rows, cols, col) {
  for (let r = rows - 1; r >= 0; r--) {
    if (cells[r * cols + col] === null) return r * cols + col;
  }
  return -1;
}

export const createInitialState = (rows, cols) => ({
  history: [{ cells: Array(rows * cols).fill(null), move: null }],
  step: 0,
});

export function gameReducer(state, action) {
  const { history, step } = state;

  switch (action.type) {
    case 'play': {
      const { index, rows, cols, k, players } = action;
      const current = history[step];
      if (index < 0 || current.cells[index] !== null) return state;
      if (current.move !== null && getWinningLine(current.cells, rows, cols, current.move, k)) {
        return state; // game already over
      }
      const cells = current.cells.slice();
      cells[index] = players[step % 2];
      return {
        history: [...history.slice(0, step + 1), { cells, move: index }],
        step: step + 1,
      };
    }
    case 'undo':
      return step > 0 ? { ...state, step: step - 1 } : state;
    case 'redo':
      return step < history.length - 1 ? { ...state, step: step + 1 } : state;
    case 'jump':
      return action.step >= 0 && action.step < history.length
        ? { ...state, step: action.step }
        : state;
    case 'reset':
      return createInitialState(action.rows, action.cols);
    default:
      return state;
  }
}
```

```jsx
// useGame.js
import { useReducer } from 'react';
import {
  createInitialState, gameReducer, getDropIndex, getWinningLine,
} from './gameLogic';

export function useGame({ rows, cols, k, players, gravity = false }) {
  const [state, dispatch] = useReducer(gameReducer, undefined, () =>
    createInitialState(rows, cols)
  );
  const { history, step } = state;
  const current = history[step];

  const winLine =
    current.move === null ? null : getWinningLine(current.cells, rows, cols, current.move, k);
  const winner = winLine ? current.cells[current.move] : null;
  const isDraw = !winner && step === rows * cols;

  // `target` is a cell index (tic-tac-toe) or a column (gravity games)
  const play = (target) => {
    const index = gravity ? getDropIndex(current.cells, rows, cols, target) : target;
    if (index === -1) return;
    dispatch({ type: 'play', index, rows, cols, k, players });
  };

  return {
    cells: current.cells,
    history,
    step,
    winLine,
    winner,
    isDraw,
    isOver: Boolean(winner) || isDraw,
    nextPlayer: players[step % 2],
    canUndo: step > 0,
    canRedo: step < history.length - 1,
    play,
    undo: () => dispatch({ type: 'undo' }),
    redo: () => dispatch({ type: 'redo' }),
    jump: (s) => dispatch({ type: 'jump', step: s }),
    reset: () => dispatch({ type: 'reset', rows, cols }),
  };
}
```

```jsx
// Board.jsx
export function TicTacToeBoard({ cells, cols, winLine, disabled, onPlay }) {
  const winSet = new Set(winLine ?? []);

  return (
    <div
      className="ttt-board"
      role="group"
      aria-label="Tic-tac-toe board"
      style={{ '--cols': cols }}
    >
      {cells.map((value, i) => (
        <button
          key={i}
          className={`cell${winSet.has(i) ? ' win' : ''}`}
          disabled={disabled || value !== null}
          aria-label={`Row ${Math.floor(i / cols) + 1}, column ${(i % cols) + 1}, ${value ?? 'empty'}`}
          onClick={() => onPlay(i)}
        >
          {value}
        </button>
      ))}
    </div>
  );
}

export function Connect4Board({ cells, rows, cols, winLine, disabled, onPlay }) {
  const winSet = new Set(winLine ?? []);

  return (
    <div className="c4-board" role="group" aria-label="Connect 4 board">
      {Array.from({ length: cols }, (_, c) => (
        <button
          key={c}
          className="c4-col"
          disabled={disabled || cells[c] !== null}
          aria-label={`Drop a disc in column ${c + 1}`}
          onClick={() => onPlay(c)}
        >
          {Array.from({ length: rows }, (_, r) => {
            const i = r * cols + c;
            const v = cells[i];
            return (
              <span
                key={i}
                aria-hidden="true"
                className={`disc${v ? ` p-${v}` : ''}${winSet.has(i) ? ' win' : ''}`}
              />
            );
          })}
        </button>
      ))}
    </div>
  );
}
```

```jsx
// Game.jsx
import { useGame } from './useGame';
import { TicTacToeBoard, Connect4Board } from './Board';

const NAMES = { X: 'X', O: 'O', R: 'Red', Y: 'Yellow' };

export default function Game({ rows, cols, k, players, gravity }) {
  const game = useGame({ rows, cols, k, players, gravity });
  const { winner, isDraw, isOver, nextPlayer, step, history } = game;

  const status = winner
    ? `${NAMES[winner]} wins!`
    : isDraw
    ? "It's a draw"
    : `Next player: ${NAMES[nextPlayer]}`;

  const boardProps = {
    cells: game.cells, rows, cols,
    winLine: game.winLine, disabled: isOver, onPlay: game.play,
  };

  return (
    <section className="game">
      <p className="status" role="status" aria-live="polite">{status}</p>

      {gravity ? <Connect4Board {...boardProps} /> : <TicTacToeBoard {...boardProps} />}

      <div className="controls">
        <button onClick={game.undo} disabled={!game.canUndo}>↶ Undo</button>
        <button onClick={game.redo} disabled={!game.canRedo}>↷ Redo</button>
        <button onClick={game.reset}>Reset</button>
      </div>

      <ol className="moves" start={0}>
        {history.map((_, s) => (
          <li key={s}>
            <button
              className="link"
              aria-current={s === step ? 'step' : undefined}
              onClick={() => game.jump(s)}
            >
              {s === 0 ? 'Go to start' : `Go to move #${s}`}
            </button>
          </li>
        ))}
      </ol>
    </section>
  );
}
```

```jsx
// App.jsx
import { useState } from 'react';
import Game from './Game';
import './styles.css';

const TTT_PLAYERS = ['X', 'O'];
const C4_PLAYERS = ['R', 'Y'];

export default function App() {
  const [mode, setMode] = useState('ttt');
  const [n, setN] = useState(3);

  return (
    <main className="app">
      <h1>{mode === 'ttt' ? `Tic-Tac-Toe ${n}×${n}` : 'Connect 4'}</h1>

      <div className="config">
        <label>
          Game{' '}
          <select value={mode} onChange={(e) => setMode(e.target.value)}>
            <option value="ttt">Tic-Tac-Toe</option>
            <option value="c4">Connect 4</option>
          </select>
        </label>
        {mode === 'ttt' && (
          <label>
            Size{' '}
            <select value={n} onChange={(e) => setN(Number(e.target.value))}>
              {[3, 4, 5, 6].map((s) => (
                <option key={s} value={s}>{s} × {s}</option>
              ))}
            </select>
          </label>
        )}
      </div>

      {/* changing the key remounts Game, so its reducer state starts fresh */}
      {mode === 'ttt' ? (
        <Game key={`ttt-${n}`} rows={n} cols={n} k={n} players={TTT_PLAYERS} gravity={false} />
      ) : (
        <Game key="c4" rows={6} cols={7} k={4} players={C4_PLAYERS} gravity />
      )}
    </main>
  );
}
```

```css
/* styles.css */
.app { max-width: 560px; margin: 24px auto; font-family: system-ui, sans-serif; text-align: center; }
.config { display: flex; gap: 16px; justify-content: center; margin-bottom: 12px; }
.status { font-size: 18px; font-weight: 600; min-height: 28px; }

.ttt-board { display: inline-grid; grid-template-columns: repeat(var(--cols), 64px); gap: 4px; }
.cell { width: 64px; height: 64px; font-size: 28px; font-weight: 700; background: #fff; border: 2px solid #334155; border-radius: 6px; cursor: pointer; }
.cell:hover:not(:disabled) { background: #f1f5f9; }
.cell:disabled { cursor: not-allowed; color: #0f172a; }
.cell.win { background: #bbf7d0; border-color: #16a34a; }

.c4-board { display: inline-flex; gap: 6px; padding: 8px; background: #1d4ed8; border-radius: 10px; }
.c4-col { display: flex; flex-direction: column; gap: 6px; padding: 0; background: transparent; border: 0; border-radius: 28px; cursor: pointer; }
.c4-col:hover:not(:disabled) { background: rgba(255, 255, 255, 0.2); }
.c4-col:disabled { cursor: not-allowed; }
.disc { width: 44px; height: 44px; border-radius: 50%; background: #fff; box-sizing: border-box; }
.disc.p-R { background: #dc2626; }
.disc.p-Y { background: #facc15; }
.disc.win { box-shadow: 0 0 0 4px #22c55e; }

button:focus-visible { outline: 3px solid #f59e0b; outline-offset: 2px; }
.controls { display: flex; gap: 8px; justify-content: center; margin: 14px 0; }
.controls button { padding: 6px 14px; border: 1px solid #94a3b8; border-radius: 6px; background: #fff; cursor: pointer; }
.controls button:disabled { opacity: 0.5; cursor: not-allowed; }
.moves { list-style: none; padding: 0; display: flex; flex-wrap: wrap; gap: 4px 12px; justify-content: center; font-size: 13px; }
button.link { border: 0; background: none; color: #2563eb; cursor: pointer; padding: 0; }
button.link[aria-current='step'] { font-weight: 700; text-decoration: underline; }
```

</details>

---

## 6. How It Works

- **Single flat array** (`rows * cols`) — index ↔ coordinates: `r = floor(i / cols)`, `c = i % cols`, `i = r * cols + c`. Cheap to copy for snapshots.
- **Snapshot history** — each move stores a *new* `cells` array (`slice()` then set). Nothing is mutated, so old snapshots stay valid for undo/redo/jump.
- **Undo/redo = moving a pointer.** `play` truncates at `step + 1` first → redo branch is discarded, exactly like an editor.
- **Derived state** — `winner`, `isDraw`, `nextPlayer` are computed each render from `history[step]`. Undo after a win automatically un-wins; no extra bookkeeping.
- **`getWinningLine` checks only the last move.** For each axis, count same-symbol cells outward in both directions; ≥ K wins and returns the cell indexes (used for highlighting). O(4·k) instead of scanning the board. Passing `k = n` gives classic N×N; `k = 4` on 7×6 gives Connect-4.
- **Connect-4 gravity** is just a different way to choose `index` (`getDropIndex`) — engine, history, win detection are shared.
- **Reducer guards** reject moves on occupied cells or after a win, so the UI can't corrupt state even if a stale click sneaks through.
- **`key={mode-n}`** remounts `Game` when the config changes → no effect needed to reset state.

---

## 7. Edge Cases

- Click on an occupied cell or after the game is over → ignored (button disabled + reducer guard).
- Draw on the **last move that also wins** → win takes priority (`isDraw` requires no winner).
- Undo after a win, then play a *different* move → history truncated, new game branch.
- Full Connect-4 column → button disabled, `getDropIndex` returns `-1`.
- Winning lines longer than K (e.g. 5 in a row in Connect-4) → still a win; whole line highlighted.
- Diagonal wins in both directions (↘ and ↙) and wins that pass *through* the last move in the middle of the line.
- Board size change mid-game → remount; no stale cells of the old size.
- Odd/even `step` parity must hold after jump → next player derives from `step`, never from a stored flag.
- Tiny boards (N=1, N=2) are trivially won by the first player — clamp the size selector to N ≥ 3.

---

## 8. Accessibility

- Cells/columns are real `<button>`s → focusable, Enter/Space work.
- Descriptive `aria-label`s: "Row 2, column 3, X"; Connect-4 columns: "Drop a disc in column 4" (discs are decorative `aria-hidden`, so screen-reader users learn the board state only from the status text — in a real app also announce each move, e.g. "Red dropped in column 4").
- `role="status"` / `aria-live="polite"` announces turn changes and the result.
- Don't encode players by colour only: use letters in TTT; for Connect-4 add a pattern/label in the status and consider ✕/○ marks inside discs for colour-blind users.
- Disabled buttons after game over; visible focus ring.
- Move list: `aria-current="step"` for the active entry.

---

## 9. Performance Considerations

- Win check O(k) per move; draw check O(1) (`step === rows*cols`).
- Snapshot cost O(N²) per move × moves ≤ N⁴ memory worst-case — trivial for N ≤ 10. For huge boards store only moves (`[index, ...]`) and **replay** to the step (`cells = replay(moves.slice(0, step))`), or use immutable structures.
- Cells re-render on each move; wrap `Cell` in `React.memo` (`value`, `isWin`, stable `onPlay` + index) if N is large. Otherwise unnecessary.
- `new Set(winLine)` per render is O(k) — fine; memoise only if profiling says so.

---

## 10. Follow-up Questions

<details>
<summary><strong>1. Make win detection O(1) per move for N×N tic-tac-toe.</strong></summary>

Keep counters: `rowCount[N]`, `colCount[N]`, `diag`, `anti` per player (+1 for X, −1 for O). After a move at `(r, c)` add the player's value; if any counter reaches `±N`, that player wins. It only works for "N in a row" (K = N), not for K < N like Connect-4, and undo must decrement (or recompute from history).

</details>

<details>
<summary><strong>2. Store only moves instead of full boards?</strong></summary>

`moves = [idx0, idx1, …]`, `step`. The board is derived by replaying `moves.slice(0, step)` with parity-based players (memoised). Memory O(moves) instead of O(moves·N²); undo/redo/jump stay trivial, and moves are serialisable for sharing or replay.

</details>

<details>
<summary><strong>3. How would you add an AI opponent?</strong></summary>

3×3: **minimax** with alpha-beta pruning (small tree). Larger boards / Connect-4: depth-limited minimax with a heuristic (count open lines), move ordering from the centre, or MCTS. Compute inside `setTimeout`/Web Worker so the UI doesn't block; dispatch the AI move as a normal `play`.

</details>

<details>
<summary><strong>4. Online multiplayer?</strong></summary>

WebSocket (or Firebase) broadcasting `{type:'play', index}`; the server is authoritative for turn order and validation. The pure reducer is reused on both ends. Handle reconnect by sending the move list, and optimistic UI with rollback on rejection.

</details>

<details>
<summary><strong>5. Add scores across games & "who starts" alternation.</strong></summary>

Lift `scores` to `App` (or another reducer) updated when `winner` first appears — do it in the event that causes the win (`onGameEnd` callback from the move handler), not in a `useEffect` watching `winner` (avoids double counting under StrictMode). Alternate the starting player with a `startingIndex` used in `players[(step + startingIndex) % 2]`.

</details>

<details>
<summary><strong>6. Why remount with `key` instead of resetting in an effect?</strong></summary>

An effect would render the old state once with the new config (wrong-size board), then reset. A `key` change creates a fresh instance with correct initial state in a single render — the recommended way to "reset state when props change".

</details>

<details>
<summary><strong>7. How do you add keyboard arrow navigation across the grid?</strong></summary>

Roving tabindex: only one cell has `tabIndex=0`; arrow keys move a `focusIndex` state by ±1 / ±cols (clamped) and call `.focus()` on the target via a refs array (or `data-index` + `querySelector`). Use `role="grid"` with `row` wrappers for correct semantics.

</details>

<details>
<summary><strong>8. Animate Connect-4 discs dropping?</strong></summary>

Render the newly placed disc with a CSS animation (`@keyframes drop` from `translateY(-<rows>*cellHeight)`), keyed by move index so it plays only for the last move; don't animate on undo/redo jumps. Respect `prefers-reduced-motion`.

</details>

---

## 11. Common Mistakes

- Mutating the board array (`squares[i] = 'X'`) → no re-render and corrupts history.
- Storing `winner`, `xIsNext` or `isDraw` in state → they drift on undo/redo.
- Scanning only rows/cols and forgetting **both** diagonals; or hard-coding the 8 lines of 3×3.
- Win check that doesn't extend in *both* directions from the last move (misses wins when the last piece is in the middle).
- Not truncating history after an undo, producing a corrupt redo branch.
- Allowing moves after a win or on filled cells.
- Declaring a draw before checking for a winner on the last move.
- Using row/column nested arrays and shallow-copying only the outer array.
- Connect-4: dropping into the clicked row instead of the lowest empty row; indexing rows top-down vs bottom-up inconsistently.
- Resetting state in `useEffect` on config change.

---

## 12. Interview Tips & Time Plan

Write the pure functions first and talk through the index math. Mention the O(1)-counter optimisation, but implement the last-move scan because it generalises to Connect-4.

| Minutes | Task |
|---|---|
| 0–5 | Clarify (K vs N, undo semantics), define `history` + `step` |
| 5–15 | 3×3 board: render, click, alternate players (immutable update) |
| 15–25 | N×N with `getWinningLine` (4 directions), winner highlight, draw |
| 25–33 | History: undo / redo / jump, truncate on new move |
| 33–40 | Connect-4: `gravity` + `getDropIndex`, column UI |
| 40–45 | Edge cases, a11y, complexity discussion, AI/multiplayer follow-ups |

---

## 13. Practice Checklist

- [ ] Index ↔ (row, col) conversions without thinking
- [ ] `getWinningLine` works for all 4 directions, any N and K
- [ ] Derived `winner` / `isDraw` / `nextPlayer` (no extra state)
- [ ] Undo, redo, jump; new move truncates redo
- [ ] Guards: occupied cell, game over, full column
- [ ] Connect-4 reuses the engine (`k=4`, gravity)
- [ ] `key` remount to change size
- [ ] Buttons + `aria-label`s + live status
- [ ] Explain O(1) counter trick and minimax AI
- [ ] Re-do in 30 minutes
