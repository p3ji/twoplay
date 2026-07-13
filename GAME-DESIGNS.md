# twoplay — New Game Architecture

Design doc for the next batch of two-player games. Everything targets the existing
single-file framework in `index.html`: each game is an IIFE registered on `games.<key>`
exposing `enter() / exit() / update(dt)` plus optional `tap(player,x,y) / drag / release / tapXY(x,y)`.
Input stays split-screen (bottom half = Honey/p1, top half = Mint/p2) unless the game
opts into the shared-board raycast mode.

Existing games (don't duplicate): tug, draw (quickdraw), toe, four, race, match,
hockey (air hockey), joust, firefly, balloon, snowball, chopper.

---

## Framework additions (do these first)

### 1. Per-game input flag instead of `RAYCAST_MODES`
The hardcoded `RAYCAST_MODES` set forces edits in two places per game. Replace with a
flag on the game object:

```js
// game module: return { raycast:true, enter(){...}, ... }
// pointerdown handler:
if(current.raycast){ current.tapXY && current.tapXY(e.clientX, e.clientY); return; }
```

### 2. Aim-and-release input (needed by Pool, Penalty Kicks)
No new events needed — `tap/drag/release` already exist. Add a small shared helper that
turns them into a slingshot vector:

```js
function makeAimer(surfaceMesh){   // raycasts onto a table/ground mesh
  let start=null, cur=null;
  return {
    down(x,y){ start = cur = project(x,y); },
    move(x,y){ if(start) cur = project(x,y); },
    up(){ const v = start && cur ? start.clone().sub(cur) : null; start=cur=null; return v; }, // drag-back = shoot forward
    preview(){ return start && cur ? {from:start, to:cur} : null; }  // for drawing the aim line
  };
  function project(x,y){ /* NDC → raycast onto surfaceMesh → Vector3 */ }
}
```

### 3. 2D ball physics helper (Pool; reusable later)
Circle–circle elastic collision + friction on the XZ plane. ~40 lines, pure functions:

```js
// balls: [{pos:Vector2, v:Vector2, r, mesh, pocketed}]
function stepBalls(balls, dt, {friction=1.4, bounds, pockets, onPocket, onClack}){ ... }
```

Hockey's inline puck code stays as-is; new ball games use this.

### 4. Rally / first-to-N scoring helper
Hockey ends on one goal; Ping Pong and Red Hands need multi-round scoring:

```js
function makeRally(target, onPoint, onMatch){
  const pts = {p1:0, p2:0};
  return { pts, score(p){ pts[p]++; onPoint(p, pts); if(pts[p]>=target) onMatch(p); } };
}
```

Show running score via `setMsgs(\`Honey ${pts.p1}\`, \`Mint ${pts.p2}\`, ...)`.

### 5. Mirrored DOM duel layer (Brain games)
Brain games need readable text/buttons per player, which 3D text makes painful. Add one
overlay with two halves — the top half rotated 180° so Mint reads it face-to-face:

```html
<div id="duel" class="hidden">
  <div class="duelHalf" id="duelP2"></div>  <!-- transform: rotate(180deg) -->
  <div class="duelHalf" id="duelP1"></div>
</div>
```

- Games that use it set `usesDuel:true`; `enterGame`/`backToHub` toggle visibility.
- The global `pointerdown` handler must early-return when `e.target.closest('#duel button')`
  so option buttons don't double-fire as zone taps.
- The 3D scene stays visible behind it (translucent halves) — boxies react to right/wrong
  answers, keeping the world alive.

### 6. Hub categories
The hub list is at 12+ cards. Group cards under three headings — **Action**, **Board**,
**Brain** — plain `<h3>` separators in the existing scroll list. No routing changes.

### 7. PWA
Bump the cache version in `sw.js` with every batch (per AGENTS.md).

---

## Game designs

### A. Boxy Pong (`games.pong`) — Action
Table-tennis, distinct from air hockey by **timing-based swings** instead of drag-defense.

- Table (~4×7) with net at z=0; boxies fixed at each end holding paddles.
- Ball has real height: `pos:Vector3`, gravity on y, bounces off the table
  (`v.y *= -0.75` at table height), arcs over the net.
- Input: **drag** moves your boxy along x (reuse hockey's targetX smoothing);
  **tap** swings. A swing connects if the ball is within reach-radius of the paddle and
  descending on your side. Timing sets direction: early = cross-court, late = down-the-line;
  swing while ball is high = smash (flatter, faster).
- Point when the ball double-bounces or leaves the table on your side. `makeRally(5)`.
- Serve alternates every 2 points; server taps to toss+serve.
- Camera: side-on-ish high angle `(7, 8, 7)` looking at table center so the arc reads.
- Failure animation: sulk + ball rolls off; ace → confetti at the landing spot.

### B. Pocket Pool (`games.pool`) — Board (turn-based)
Simplified 2-player pool. Full 8-ball is too much for a phone screen shared table —
use **3 vs 3 colored balls + 1 cue ball, 4 corner pockets**.

- `raycast:true` turn-based like toe/four: only the active player's input counts
  (ignore which half the touch came from — the active player aims from anywhere,
  same as Boxy Toe's model).
- Input: touch the table and **drag back from the cue ball** → aim line + power meter
  (clamped) drawn with a thin stretched box mesh; release fires. Uses `makeAimer`.
- Physics: `stepBalls` with friction; shot ends when all `v.length() < ε`.
- Rules (kept toddler-simple, matching the game's cozy tone):
  - Pocket one of your color → go again. Pocket opponent's → their turn, ball stays down.
  - Cue ball pocketed → respot at center, turn passes.
  - First to sink all 3 of their color wins. No 8-ball, no fouls beyond scratch.
- Boxies stand at table corners; active player's boxy walks to the cue ball and does a
  little push animation on release.
- Camera: top-down `(0, 14, 4)` like Connect Four.

### C. Red Hands (`games.slap`) — Action
The classic slap game: attacker vs defender, reaction + bluffing.

- Two boxies face each other across the island center, hands extended (arm pivots
  already exist on `makeBoxy`).
- Roles: attacker (starts as Honey) and defender. Attacker **taps to slap**; defender
  **taps to pull hands away**.
- The bluff layer, so it isn't a pure reaction test: after a random 1–3 s tension delay,
  the attacker's boxy does a **feint wind-up** 0–2 times (shoulder twitch animation)
  before the real slap. Defender dodging on a feint = "flinched!" → attacker scores free
  point. Defender dodges within the ~250 ms slap window → clean dodge, roles swap.
  Slap lands → attacker point, roles stay.
- Feints are engine-driven (random), not attacker-controlled — keeps input to one tap
  each and makes it fair on a shared screen.
- `makeRally(5)`. Slap connect = big squash animation + `AudioMod.tap` burst;
  dodge = whoosh + attacker overbalances forward.
- Camera: low and close `(0, 4, 6)` for drama.

### D. Brain pack — three games sharing the duel layer

All three: `usesDuel:true`, countdown via `makeCountdown`, `makeRally(5)` scoring,
boxies visible behind the overlay cheering/sulking per point.

**D1. Quick Math (`games.math`)**
- Same equation shown on both halves (top mirrored): `7 + 6 = 13 ✓/✗` or
  3-choice answers. First correct tap wins the point; wrong answer locks you out
  for that round (prevents spam-both-buttons strategy).
- Difficulty ramps with round number: single digit add → mixed ops → two-step.
- Generator guards: wrong choices within ±3 of truth, never negative.

**D2. Odd One Out (`games.odd`)**
- Shared 4×4 grid of emoji/colored tiles, one subtly different (🐱🐱🐈🐱 /
  hue-shifted swatch). First tap on the odd tile scores; tap a wrong tile → 1 s lockout.
- Rendered in the duel layer as one shared grid (readable from both sides since it's
  imagery, not text) — cheaper than 3D tiles and no raycast needed.
- Difficulty ramp: hue delta shrinks / emoji pairs get sneakier.

**D3. Echo (`games.echo`)** — memory sequence, turn-based
- Shared board of 4 glowing lantern pads (3D, `raycast:true` — this one skips the duel
  layer since it's tap-position based, like match).
- Watch phase: pads flash a growing sequence. Repeat phase: players alternate rounds
  reproducing it. First mistake loses; sequence grows +1 each successful round.
- Reuses Boxy Match's lantern meshes/glow pattern.

---

## Suggested build order

| # | Game | New infra it lands | Est. size |
|---|------|--------------------|-----------|
| 1 | Boxy Pong | rally helper | ~150 lines |
| 2 | Red Hands | (none — pure animation/timing) | ~120 lines |
| 3 | Quick Math | duel layer + hub categories | ~140 lines + ~40 CSS |
| 4 | Odd One Out | (reuses duel layer) | ~100 lines |
| 5 | Pocket Pool | aimer + ball physics | ~220 lines |
| 6 | Echo | (reuses match assets) | ~110 lines |

Rationale: Pong first (highest fun-per-line, only needs the rally helper), duel layer lands
with Quick Math and is immediately amortized by Odd One Out, Pool last since it carries
the most new physics/input machinery.

Each batch: add hub card(s), bump `sw.js` cache version, test portrait on a phone
(dynamic-FOV camera already handles it, but duel-layer font sizes need a 375 px check).
