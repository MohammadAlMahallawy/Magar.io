# Agar.io Clone — Master Spec (Reference Doc)

This is the single source of truth to paste/attach alongside every phase prompt
given to the builder model. It exists so numbers and rules don't drift between
sessions with a model that has no memory of earlier conversations.

**Status:** Phase 0 and Phase 1 are already implemented in `index.html`
(skeleton, resize handling, rAF game loop with delta time, world/camera
system, player movement, grid + boundary rendering). Everything from Phase 2
onward is built by *extending that same file*, phase by phase.

---

## 1. Visual style guide (derived from reference screenshots)

- **Background:** very light blue-gray (`#f2f6fa`), overlaid with a square
  grid in a slightly darker line color (`#dce8f0`), grid cell ≈ 50px in world
  space.
- **World boundary:** a dashed rectangle in a muted blue-gray (`#b0c4d4`) at
  the edges of the 5000×5000 world.
- **Blobs (player/bots):** flat, fully-saturated fill colors (magenta, purple,
  yellow-green, cyan, orange, red, blue) with a **darker border** of the same
  hue (~35-40% darker), stroke width scales with radius
  (`max(2, radius * 0.08)`). No gradients, no shadows on the blob itself.
- **Names:** bold white text, centered on the blob, with a black outline or
  drop shadow so it reads on any fill color. Font size scales with radius
  (roughly `radius * 0.5`, clamped to a reasonable min/max so tiny/huge blobs
  don't get illegible or absurd text).
- **Food:** small flat-colored circles (~5-8px world radius), same saturated
  palette as blobs, no border needed (or a very thin one).
- **Virus:** green fill (`#33ff33`) with a darker green border, rendered as a
  spiky/jagged circle (12-ish spikes), notably larger than food but usually
  smaller than a mid-game player.
- **Leaderboard:** a semi-transparent dark box (`rgba(0,0,0,0.5)`), pinned
  top-right, white text, "Leaderboard" header + numbered top 10 list.
- **HUD:** current score/mass shown bottom-left, simple white/dark text on a
  light translucent background.
- **End-of-match summary modal** (optional polish, not required by the phase
  plan): centered card showing food eaten, time alive, highest mass, cells
  eaten, top leaderboard position.

None of the polish items (end screen, skins, mobile ad banners) are in scope.
Focus entirely on the 7 phases below.

---

## 2. Global constants (do not redefine differently per phase — extend this list)

```js
// World
const WORLD_WIDTH = 5000;
const WORLD_HEIGHT = 5000;
const GRID_SIZE = 50;

// Movement / mass-to-radius
const BASE_SPEED = 260;        // px/sec at REFERENCE_MASS
const REFERENCE_MASS = 20;
const SPEED_FALLOFF = 0.25;    // speed = BASE_SPEED * (REFERENCE_MASS/mass)^SPEED_FALLOFF
const MASS_RADIUS_SCALE = 6;   // radius = sqrt(mass/PI) * MASS_RADIUS_SCALE
const START_MASS = 20;

// Food (Phase 2)
const FOOD_COUNT = 500;
const FOOD_RADIUS = 6;         // world units
const FOOD_MASS_GAIN = 1;      // mass added per food eaten

// Bots (Phase 3)
const BOT_COUNT = 14;
const BOT_DIFFICULTY_WEIGHTS = { easy: 0.50, medium: 0.35, hard: 0.15 };
const BOT_DETECTION_RADIUS = 600;      // world units, "how far a bot can see"
const BOT_EAT_MARGIN = 1.25;           // must be 25% bigger to safely hunt
const BOT_REEVAL_INTERVAL = { easy: 1.0, medium: 0.5, hard: 0.2 }; // seconds
const BOT_BORED_AFTER = [8, 15];       // seconds range, randomized per chase
const BOT_BORED_COOLDOWN = [10, 20];   // seconds range before re-targeting player

// Split & eject (Phase 4)
const SPLIT_MIN_MASS = 40;             // can't split below this
const SPLIT_KICK_SPEED = 500;          // initial velocity boost, decays to 0
const SPLIT_KICK_DECAY = 3.0;          // per-second decay rate
const MERGE_COOLDOWN = 15;             // seconds before split pieces can re-merge
const EJECT_MASS_COST = 14;            // mass removed from ejecting blob
const EJECTED_MASS_MASS = 12;          // mass of the ejected blob itself
const EJECTED_MASS_SPEED = 700;
const EJECTED_MASS_DECAY = 4.0;

// Viruses (Phase 5)
const VIRUS_COUNT = 30;
const VIRUS_BASE_MASS = 100;
const VIRUS_SPIKES = 12;
const VIRUS_FIRE_THRESHOLD = 150;      // mass at which a fed virus fires
const VIRUS_FIRE_DISTANCE = 800;       // world units it launches
const VIRUS_POP_MIN_PIECES = 3;
const VIRUS_POP_MAX_PIECES = 7;

// Camera / leaderboard (Phase 6)
const BASE_ZOOM = 1.0;
const BASE_ZOOM_MASS = START_MASS;
const MIN_ZOOM = 0.35;
const MAX_ZOOM = 1.0;
const ZOOM_LERP = 0.05;
const LEADERBOARD_SIZE = 10;
const LEADERBOARD_REFRESH_MS = 250;
```

Treat these as defaults — tune numbers during playtesting, but always tune
them *in this file* and carry the updated value forward, not just inline in
one phase's code.

---

## 3. Data shapes (plain objects, no classes)

```js
// A "blob" is the shared shape for player pieces, bot pieces, and split pieces.
{
  id: 'p1_a',            // unique per-blob id
  ownerId: 'p1',         // groups multiple blobs of the same player/bot together
  x, y,                  // world position
  vx, vy,                // used only during split-kick / eject decay, else 0
  mass,
  radius,                // always derived from mass, never hand-set
  color, borderColor,
  mergeLockUntil: 0,     // timestamp; can't merge with siblings before this
  isBot: false,
  // bot-only fields:
  state: 'SEEKING_FOOD', // SEEKING_FOOD | HUNTING | FLEEING | BORED
  difficulty: 'easy',    // easy | medium | hard
  targetId: null,
  nextEvalAt: 0,
  boredUntil: 0
}

// Food
{ x, y, radius: FOOD_RADIUS, color }

// Virus
{ x, y, mass: VIRUS_BASE_MASS, radius, vx: 0, vy: 0, firing: false }

// Ejected mass
{ x, y, vx, vy, mass: EJECTED_MASS_MASS, radius, ownerId }
```

`ownerId` is what makes split pieces behave as one player: sum mass across
all blobs sharing an `ownerId` for the leaderboard, camera framing, and
"can't eat your own piece" checks.

---

## 4. File structure

- Keep everything in `index.html` (embedded `<style>` + `<script>`) until it
  passes ~1000 lines. At that point, split into `index.html` + `game.js`
  (plain `<script src="game.js">`, no modules/bundlers).
- Never introduce a build step, npm, or framework.

---

## 5. Phase acceptance criteria (condensed)

| Phase | Done when... |
|---|---|
| 0 (done) | Canvas fills window, resizes cleanly, stable rAF loop, test circle renders. |
| 1 (done) | Player moves smoothly toward mouse, slower at higher mass, camera pans, grid/boundary prove world/screen conversion is correct, player can't leave world bounds. |
| 2 | Fixed food count maintained, player grows via `sqrt` radius formula on eating, food never spawns inside a blob. |
| 3 | 10-15 bots visibly show all 4 states (seeking food, hunting, fleeing, going bored and wandering off) without jitter, with difficulty-based target prediction and boundary-aware movement. |
| 4 | Space splits into two co-controlled pieces sharing `ownerId`; pieces can't re-merge before `MERGE_COOLDOWN`; eject key spawns a decelerating mass blob that others can eat. |
| 5 | Virus pops any bigger blob that touches it into 3-7 pieces; ejecting mass into a virus grows it and it fires along the ejection trajectory once `VIRUS_FIRE_THRESHOLD` is crossed. |
| 6 | Zoom smoothly interpolates out as total owned mass grows (never snaps); camera frames all owned pieces post-split; leaderboard shows top 10 by summed `ownerId` mass, refreshed every ~250ms. |

Full task-level detail for each phase lives in `HANDOFF_PROMPTS.md` — that
file is what actually gets pasted to the builder model, one phase at a time.
