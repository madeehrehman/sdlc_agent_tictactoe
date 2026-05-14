# Tic-Tac-Toe — Project Specification

## 1. Purpose & Context

This document specifies a browser-based Tic-Tac-Toe game. The application itself is intentionally small; its real purpose is to serve as the **first end-to-end work item for an autonomous SDLC agent pipeline**. It is the vehicle for proving that the factory works — issue ingestion, implementation, testing, PR creation, review, and deploy.

**How to read this document (for the backlog analyzer agent):**
- Section 5 (Functional Requirements) is the decomposition surface. Each `FR-n` is sized to become **one GitHub issue**.
- Each `FR-n` carries its own **Acceptance Criteria** checklist — copy these verbatim into the issue body; they are the contract the PR reviewer agent checks against.
- Section 8 provides the **dependency graph**. Use it to set issue ordering and to populate "blocked by" / "blocks" relationships so the master agent can topologically sort the backlog.
- Do not create issues for Section 6 (Non-Functional Requirements) — fold those into every issue's definition of done.

---

## 2. Scope

### In Scope
- A single-page, two-player (local hot-seat) Tic-Tac-Toe game.
- 3×3 board, alternating X and O turns, win/draw detection, game reset.
- All game logic unit-tested; UI interaction tested at component level.

### Out of Scope
- No AI / computer opponent.
- No backend, no network play, no persistence (no `localStorage`, no database).
- No authentication, no accounts, no score history across sessions.
- No responsive/mobile optimization beyond "usable on a standard desktop viewport."
- No animations or sound.

Keeping the above out of scope is deliberate — it keeps the loop closeable. Do not let issues expand into these areas.

---

## 3. Technology Stack

| Concern | Choice | Notes |
|---|---|---|
| Language | TypeScript | Strict mode on. |
| Framework | React 18 | Functional components + hooks only. |
| Build tool | Vite | Dev server + production build. |
| Test runner | Vitest | Unit + component tests. |
| Component testing | React Testing Library | For UI-layer FRs. |
| Package manager | npm | Lockfile committed. |

The game logic (FR-2, FR-3) **must be implemented as a pure, framework-agnostic TypeScript module** with no React imports. This keeps logic tests trivial and fast, and is a hard architectural constraint, not a preference.

> Alternative: a vanilla-JS implementation is acceptable in principle, but this spec assumes the React stack above. Do not mix.

---

## 4. Architecture Overview

Two layers, strictly separated:

1. **Logic layer** — `src/game/` — pure functions and types. Knows nothing about the DOM or React. Fully deterministic.
2. **UI layer** — `src/components/` — React components that render logic-layer state and translate user events into logic-layer calls.

```
src/
  game/
    types.ts        # Player, Cell, Board, GameState, GameStatus
    state.ts        # createInitialState, applyMove
    outcome.ts      # detectWinner, detectDraw, getStatus
  components/
    Board.tsx       # renders the 3x3 grid
    Cell.tsx        # renders a single square
    StatusBar.tsx   # renders whose turn / outcome
    Game.tsx        # wires state + components + reset
  main.tsx          # app entry
```

**The core architectural rule:** UI components never compute game outcomes. They call the logic layer and render what it returns.

---

## 5. Functional Requirements

### FR-1 — Project Scaffold
**Description:** Initialize the repository as a runnable, testable project.

**Acceptance Criteria:**
- [ ] `npm install` succeeds with a committed lockfile.
- [ ] `npm run dev` starts a dev server rendering a placeholder page.
- [ ] `npm run build` produces a production build with no errors.
- [ ] `npm run test` executes Vitest and exits 0 (a trivial smoke test is acceptable).
- [ ] TypeScript strict mode is enabled.
- [ ] A `README.md` documents install / dev / build / test commands.

**Depends on:** none.

---

### FR-2 — Game State Model
**Description:** Implement the pure data model for a game and the function that applies a move.

**Acceptance Criteria:**
- [ ] Types defined: `Player` (`'X' | 'O'`), `Cell` (`Player | null`), `Board` (length-9 array of `Cell`), `GameState` (board + current player).
- [ ] `createInitialState()` returns an empty board with `X` as the starting player.
- [ ] `applyMove(state, index)` returns a **new** state with the cell filled and the current player switched.
- [ ] `applyMove` is immutable — the input state object is not mutated.
- [ ] `applyMove` rejects a move on an occupied cell (returns state unchanged or signals invalid; behavior must be explicit and tested).
- [ ] `applyMove` rejects a move when the game is already over.
- [ ] Unit tests cover valid moves, occupied-cell rejection, post-game rejection, and immutability.

**Depends on:** FR-1.

---

### FR-3 — Win/Draw Detection
**Description:** Implement pure outcome detection over a board.

**Acceptance Criteria:**
- [ ] `detectWinner(board)` returns the winning `Player` or `null`.
- [ ] All 8 winning lines are detected: 3 rows, 3 columns, 2 diagonals.
- [ ] `detectDraw(board)` returns `true` only when the board is full **and** there is no winner.
- [ ] `getStatus(state)` returns a `GameStatus`: `'in_progress'`, `'won'` (with winner), or `'draw'`.
- [ ] Win is detected before draw — a full board that contains a winning line reports `'won'`, not `'draw'`.
- [ ] Unit tests cover every winning line, a draw, and an in-progress board.

**Depends on:** FR-2.

---

### FR-4 — Board Rendering
**Description:** Render the 3×3 grid and its current contents.

**Acceptance Criteria:**
- [ ] `Board` renders 9 `Cell` components in a 3×3 layout.
- [ ] Each `Cell` displays `X`, `O`, or nothing based on board state.
- [ ] The board is rendered from `GameState` passed in as props — the component computes no logic itself.
- [ ] Component test asserts that a given board state renders the correct marks in the correct positions.

**Depends on:** FR-1. (Consumes FR-2 types; can be built in parallel with FR-3.)

---

### FR-5 — Move Interaction
**Description:** Let a player click an empty cell to take a turn.

**Acceptance Criteria:**
- [ ] Clicking an empty cell applies the move via the logic layer and re-renders.
- [ ] Clicking an occupied cell does nothing.
- [ ] Clicking any cell after the game is over does nothing.
- [ ] Turns alternate X → O → X correctly.
- [ ] Component test simulates clicks and asserts board updates and turn alternation.

**Depends on:** FR-2, FR-4.

---

### FR-6 — Game Status Display
**Description:** Show whose turn it is and the game outcome.

**Acceptance Criteria:**
- [ ] While in progress, `StatusBar` shows the current player (e.g. "X's turn").
- [ ] On a win, it shows the winner (e.g. "X wins").
- [ ] On a draw, it shows a draw message.
- [ ] Status is derived from `getStatus()` — no outcome logic in the component.
- [ ] Component test covers in-progress, won, and draw states.

**Depends on:** FR-3, FR-5.

---

### FR-7 — Reset
**Description:** Let players start a new game at any time.

**Acceptance Criteria:**
- [ ] A reset control is always visible.
- [ ] Activating reset restores the initial empty board with X to move.
- [ ] Reset works both mid-game and after a finished game.
- [ ] Component test asserts board clears and status returns to "X's turn".

**Depends on:** FR-5. (FR-6 recommended first for a coherent UI, but not a hard dependency.)

---

## 6. Non-Functional Requirements

Fold these into every issue's definition of done:
- **Type safety:** no `any`; TypeScript strict mode stays on.
- **Test coverage:** every logic-layer function has unit tests; every component has at least one render/interaction test.
- **Determinism:** logic layer has no side effects, no randomness, no time dependence.
- **Build health:** `npm run build` and `npm run test` both pass on every PR.
- **No scope creep:** no dependencies added beyond Section 3 without an explicit decision.

---

## 7. Edge Cases & Rules Reference

The implementing agent's tests are expected to cover:
- Move attempted on an occupied cell → rejected.
- Move attempted after game over → rejected.
- All 8 winning lines → each detected as a win.
- Board full with no line → draw.
- Board full **with** a winning line → win, not draw.
- X always moves first.
- Reset from mid-game and from a finished game → both return to clean initial state.

---

## 8. Suggested Issue Decomposition & Dependency Graph

One issue per functional requirement. Recommended titles and dependencies:

| Issue | Title | Depends on | Can start in parallel with |
|---|---|---|---|
| #1 | Project scaffold (Vite + React + TS + Vitest) | — | — |
| #2 | Game state model and move application | #1 | #4 |
| #3 | Win and draw detection | #2 | #4 |
| #4 | Board and Cell rendering | #1 | #2, #3 |
| #5 | Move interaction (click to play) | #2, #4 | — |
| #6 | Game status display | #3, #5 | — |
| #7 | Game reset | #5 | — |

**Critical path:** #1 → #2 → #3 → #6 and #1 → #2 → #5 → #6. Issue #4 floats early; #7 closes out.

**Build order for a single-threaded agent:** #1, #2, #3, #4, #5, #6, #7.

### First-pass option
For the very first run of the SDLC agent, consider collapsing all of the above into a single issue — *"Build a playable Tic-Tac-Toe game"* — purely to prove the loop closes end to end. Then use the seven-issue decomposition above as the second pass to exercise sequential, dependency-aware handling.

---

## 9. Definition of Done (Project Level)

The project is complete when:
- [ ] All seven functional requirements are merged.
- [ ] Two players can play a full game to a win and to a draw in the browser.
- [ ] The game can be reset at any point.
- [ ] `npm run build` and `npm run test` pass on `main`.
- [ ] The README documents how to install, run, build, and test.
- [ ] The deployed build is reachable (deploy target defined by the PR/deploy agent, out of scope for this spec).
