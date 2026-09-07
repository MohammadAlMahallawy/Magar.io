# Handoff Prompts — one per phase

How to use this file: open a **new conversation per phase** with the builder
model. Paste the "Universal preamble" once, then the prompt for the current
phase, then attach the **current** `index.html` (the real file, after your
own review of the previous phase — not from memory). Do not paste future
phases. Do not let the model see or plan ahead past the current phase.

After the model responds, actually open the file in a browser yourself and
run the "Test" checklist before starting the next phase's conversation.

---

## Universal preamble (paste at the top of every phase conversation)

```
You are extending an existing Agar.io clone, one phase at a time. Rules:

1. Do NOT skip ahead or combine phases. Only implement what this message's
   phase asks for. If something from a later phase seems necessary, stub it
   minimally and say so — don't build it.
2. Single file: index.html with embedded <style> and <script>. No build
   tools, no npm, no frameworks, no modules. Must run by double-clicking.
3. Game loop: requestAnimationFrame only, never setInterval/setTimeout.
4. Delta-time everything: never `x += 5`, always `x += speed * deltaTime`
   (deltaTime in seconds).
5. Plain objects in arrays for all game entities (player, bots, food,
   viruses, ejected mass). No classes, no deep inheritance.
6. World space vs screen space: all game logic (movement, collisions,
   distances) happens in WORLD coordinates. Only the final draw call
   converts to screen coordinates via the existing worldToScreen() function.
   Never do game logic in screen space.
7. Reuse the constants already defined at the top of the script. If you need
   a new constant, add it near the others with a comment, don't hardcode
   magic numbers inline.
8. At the end of your response, describe exactly what you tested and what
   you'd expect to see, matching the "Test" section of this prompt. Then
   STOP — do not start the next phase.

Here is the current index.html: [ATTACH FILE]
Here is the project's constants/spec reference: [ATTACH GAME_SPEC.md]
```

---

## Phase 2 — Food

```
PHASE 2: Food.

Add:
- A `food` array. On startup, spawn FOOD_COUNT food objects
  ({x, y, radius: FOOD_RADIUS, color}) at random positions within world
  bounds, each a random color from a small saturated palette.
- Each frame, check distance from the player's center to each food's center
  (world space). If distance < player.radius, mark that food as eaten.
- Remove eaten food and increase player.mass by FOOD_MASS_GAIN per food
  eaten. Recompute player.radius from mass using the existing
  updateRadiusFromMass-style formula (sqrt(mass/PI) * MASS_RADIUS_SCALE) —
  do NOT grow radius linearly with mass.
- For every eaten food, spawn exactly one replacement food at a new random
  position, retrying (up to ~20 attempts) if the new position would land
  inside the player's current radius (there are no bots yet, so only check
  against the player).
- Render all food circles through worldToScreen(), same as the player.

Avoid the classic bugs:
- Don't remove-while-iterating with a plain for...of loop; filter/rebuild
  the array or iterate backwards by index.
- Don't let radius grow linearly with mass.
- Don't forget the "don't spawn inside the player" retry logic — this exact
  pattern gets reused for virus respawning in Phase 5, so get it right now.

Test: Eat food; confirm the player visibly grows (mass number and radius
both increase, at the sqrt-scaled rate — growth should feel like it slows
down relative to mass gained as the blob gets bigger). Food count on the
map should stay constant. No new food should ever appear inside your own
circle.
```

---

## Phase 3 — Bots

```
PHASE 3: Bots.

Add BOT_COUNT bot objects, same shape as player blobs plus
{state, difficulty, targetId, nextEvalAt, boredUntil, isBot: true}. Assign
difficulty on spawn using BOT_DIFFICULTY_WEIGHTS (50/35/15 easy/medium/hard).

State machine (re-evaluate on a per-bot timer using
BOT_REEVAL_INTERVAL[difficulty] — NOT every frame):

- SEEKING_FOOD (default): move toward nearest food within
  BOT_DETECTION_RADIUS, same movement/speed rules as the player. If nothing
  is nearby, wander (pick a random point within detection radius and head
  toward it, re-picking when reached).
- HUNTING: if a blob (player or another bot) within BOT_DETECTION_RADIUS has
  mass < this bot's mass / BOT_EAT_MARGIN, switch to HUNTING that blob's id
  and move toward it. Track when the hunt started.
- FLEEING: if a blob within BOT_DETECTION_RADIUS has mass > this bot's mass
  * BOT_EAT_MARGIN, switch to FLEEING and move directly away from it
  (overrides HUNTING/SEEKING_FOOD while the threat is in range).
- BORED: if a bot has been HUNTING the same targetId for longer than a
  randomized value in BOT_BORED_AFTER without catching it, switch to BORED:
  clear targetId, behave like SEEKING_FOOD, and set boredUntil = now + a
  randomized value from BOT_BORED_COOLDOWN. While now < boredUntil, this bot
  may not re-target the player specifically (it can still flee/hunt other
  bots normally once cooldown-relevant logic allows — keep it simple: just
  block re-targeting the same entity it was just bored of).

Difficulty differences:
- easy: re-evaluates every 1.0s, moves straight at target ignoring
  everything else, never splits/ejects (splitting doesn't exist yet —
  just leave a `canSplit`/`canEject` style flag or comment for Phase 4 to
  use later).
- medium: re-evaluates every 0.5s.
- hard: re-evaluates every 0.2s. (Path-around-viruses behavior comes in
  Phase 5 — not needed yet, just leave the difficulty tier in place.)

Bots follow identical physical rules to the player: same speed formula
based on their own mass, same food-eating, no invincibility.

Avoid the classic bugs:
- Do not re-evaluate targets every frame — this causes jittery flip-flopping.
  Use nextEvalAt timestamps.
- Don't give bots knowledge of the whole map — always gate by
  BOT_DETECTION_RADIUS.
- Boredom must be a real cooldown timer (boredUntil), not a random chance
  re-rolled every frame.

Test: With ~10-15 bots on screen, confirm you can visually distinguish: some
calmly eating food, some fleeing when you're bigger, some hunting you when
they're bigger, and at least one bot that chases for a while, visibly gives
up, and wanders off eating food instead — without flickering between states.
```

---

## Phase 4 — Split and mass ejection

```
PHASE 4: Split and eject.

Add:
- Space key: if the pressing entity's mass >= SPLIT_MIN_MASS, split it into
  two blobs, each with half the mass (recompute radius from mass for both).
  New blob spawns slightly offset in the aim direction (toward mouse for the
  player; toward current target/movement direction for bots) with an
  initial velocity kick of SPLIT_KICK_SPEED in that direction. Both blobs
  share the same ownerId as the original (generate one if the original
  didn't have one yet) and get mergeLockUntil = now + MERGE_COOLDOWN.
- Kick velocity must decay over time (multiply vx/vy toward 0 each frame
  using SPLIT_KICK_DECAY * dt) so pieces settle back to normal
  mouse-following movement — they should not fly forever.
- Blobs sharing an ownerId must never eat each other, and should visually
  move "together" (each still follows the same mouse-direction input
  independently, which is how agar.io actually feels).
- Merging: if two blobs share an ownerId, are both past their
  mergeLockUntil, and their distance < sum of radii (or some reasonable
  overlap threshold), combine them into one blob (sum mass, recompute
  radius, average position) and remove the duplicate.
- Eject key (use 'w'): if mass > EJECT_MASS_COST + some safety margin,
  subtract EJECT_MASS_COST from the ejecting blob's mass (recompute radius),
  spawn an ejectedMass object ({x,y,vx,vy,mass:EJECTED_MASS_MASS,radius,
  ownerId}) at the blob's edge in the aim direction with speed
  EJECTED_MASS_SPEED, decaying via EJECTED_MASS_DECAY each frame. Any blob
  NOT sharing that ejected mass's ownerId can eat it on contact
  (mass += EJECTED_MASS_MASS, remove the ejected mass object). The
  originating player can also eat their own ejected mass if they catch up
  to it later.
- Update the leaderboard/display concept (even if no visible leaderboard UI
  yet) to sum mass across all blobs sharing an ownerId — you'll wire this
  into an actual leaderboard in Phase 6, but keep a helper function like
  getTotalMass(ownerId) now so it's ready.

Avoid the classic bugs:
- Without ownerId grouping, split pieces will eat each other, or you won't
  be able to sum mass correctly later — don't skip it.
- Without mergeLockUntil, splitting has no risk/cost, which breaks balance.
- Without kick decay, split pieces fly off the map.

Test: Split into two; confirm both pieces move roughly together toward the
mouse, can't immediately re-merge, and DO merge back into one blob after
MERGE_COOLDOWN once brought into contact. Confirm ejected mass travels,
decelerates to a stop, and can be eaten by something else (or by yourself
later).
```

---

## Phase 5 — Viruses

```
PHASE 5: Viruses and the feeding/firing mechanic.

Add:
- A `viruses` array: VIRUS_COUNT virus objects
  {x, y, mass: VIRUS_BASE_MASS, radius (derived from mass same formula),
  vx: 0, vy: 0}, spawned at random positions (reuse the Phase 2 "don't spawn
  inside a blob" retry logic).
- Render viruses as a spiky shape (VIRUS_SPIKES points alternating between
  radius and radius*1.25 or similar, arranged in a circle) in green with a
  darker green border.
- Collision rule: if a blob's mass > virus.mass and they touch, the virus
  "pops" that blob: replace it with VIRUS_POP_MIN_PIECES to
  VIRUS_POP_MAX_PIECES new blobs (count scaled by how much bigger the blob
  was than the virus), each getting an even share of the original mass,
  scattered in random directions with a brief outward velocity kick (reuse
  the split-kick decay pattern from Phase 4). All resulting pieces share the
  original blob's ownerId and get a merge lock like a normal split. The
  virus is then removed and respawns elsewhere after a short delay (reuse
  the same safe-spawn retry logic).
- Blobs with mass <= virus.mass are unaffected by collision with it (pick
  "pass through harmlessly" — do not let them be affected at all).
- Feeding: when an ejectedMass object (from Phase 4) hits a virus, remove
  the ejected mass and add its mass to the virus's mass (recompute virus
  radius). Once virus.mass >= VIRUS_FIRE_THRESHOLD, the virus "fires": give
  it vx/vy in the same direction the ejected mass was traveling when it fed
  the virus, scaled so it travels roughly VIRUS_FIRE_DISTANCE before
  stopping (decelerate it like ejected mass does). While moving, a fired
  virus uses the exact same "pop bigger blobs it touches" collision rule as
  a stationary one. After it travels its full distance (or after firing
  once), reset its mass to VIRUS_BASE_MASS and let it sit in place — do not
  let it end up in a permanent no-collision or endlessly-flying broken
  state.

Avoid the classic bugs:
- Forgetting the "smaller blobs are unaffected" rule causes infinite
  re-splitting of your own small pieces on contact.
- The fired virus's direction must match the ejection direction that fed
  it — that's the entire point of the aiming mechanic.
- Make sure every virus that pops or fires actually respawns/resets — audit
  that there's no code path where a virus is removed without being replaced,
  or fires without ever resetting.

Test: Confirm a large blob gets split into several pieces by touching
a virus, while a small blob passes through unaffected. Confirm feeding a
virus with ejected mass grows it, and once VIRUS_FIRE_THRESHOLD is crossed,
it fires along the feed direction and can split a bigger player caught in
its path.
```

---

## Phase 6 — Camera zoom and leaderboard

```
PHASE 6: Camera zoom + leaderboard (final polish pass).

Add:
- getTotalMass(ownerId) (if not already added in Phase 4) summing mass
  across all blobs with that ownerId.
- Camera zoom target: targetZoom = clamp(
    BASE_ZOOM / Math.sqrt(getTotalMass(player.ownerId) / BASE_ZOOM_MASS),
    MIN_ZOOM, MAX_ZOOM). Each frame, interpolate the actual camera.zoom
  toward targetZoom: camera.zoom += (targetZoom - camera.zoom) * ZOOM_LERP.
  Never snap zoom directly to the target.
- Camera position: once the player has multiple blobs (post-split), set
  camera.x/camera.y to the average x/y of all blobs sharing the player's
  ownerId, and make sure targetZoom accounts for their spread so all pieces
  stay roughly in view (e.g. factor in the max distance from center to any
  owned blob, not just total mass, if pieces are very spread out).
- Leaderboard: an HTML overlay div (absolutely positioned, top-right,
  semi-transparent dark background, white text) listing the top
  LEADERBOARD_SIZE distinct ownerIds by getTotalMass, updated on a timer
  every LEADERBOARD_REFRESH_MS (use setInterval just for this UI refresh —
  it's fine here since it's not the game loop/physics, only a display
  refresh; the game loop itself still must stay on requestAnimationFrame).

Avoid the classic bugs:
- Snapping zoom instantly on any mass change — always interpolate.
- Re-sorting/rebuilding the leaderboard list every single rAF frame — throttle
  it to the refresh interval above.

Test: Grow significantly by eating food/bots; camera should smoothly zoom
out, never jump. Split; camera should reframe to keep both pieces in view.
Leaderboard should update every ~250ms and correctly reflect current mass
standings including split players' summed mass.
```
