# Frontier Sim Breach

A single-file, vanilla HTML/CSS/JS top-down stealth game. Everything —
markup, styling, and game logic — lives in `index.html`. There is no
build step, no framework, and no package manager. Open the file in a
browser (or serve the directory statically) and it runs.

This document exists so that another AI picking up this project later
understands the intent behind the mechanics, not just what the code
does. Read this before making gameplay changes — a change that is
technically correct but breaks the core tension (sound over sight) is
still wrong for this game.

## Core concept

The player is a lone operative moving through a dark training facility,
trying to restore two power nodes and reach the exit gate. A single
enemy, **Medic**, patrols the facility. Medic is not a chase-you-by-sight
monster in the traditional sense — his defining trait is that **he is
almost completely blind but has near-perfect hearing**. He reacts to
noise, not to being seen. This is the load-bearing idea of the game:
every mechanic (footsteps, sprinting, power node activation, the call
button) exists to let the player choose when to make noise and accept
the risk, rather than avoid a sightline.

Design implication: don't add mechanics that let Medic "see" the player
from a distance, and don't make stealth about hiding from a cone of
vision. Stealth here is about sound discipline — walk instead of sprint,
avoid triggering objectives while he's nearby, and know that calling
him is a deliberate, costly choice (used to lure him away from an area,
not as a mistake).

## Win / lose conditions

- **Win**: activate both power nodes (`P` tiles), then reach the exit
  gate while both are active. The exit no longer requires a keycard —
  that item was removed from the game entirely.
- **Lose**: Medic makes physical contact with the player. Instant death,
  triggers a jumpscare then a death screen.

## Medic's AI (the part most likely to be touched)

Implemented in `updateEnemy(dt)`. Key numbers, as of this writing:

- `enemySightDistance` — his actual sight range. Deliberately tiny
  (currently ~38 units, less than two tiles) because he is nearly blind.
  Do not casually increase this; it undermines the whole design.
- `enemyHearRadius` — how far he can hear. Sprinting roughly doubles
  his effective hearing radius vs. walking (`enemyHearRadius * 0.55` for
  walking noise). This asymmetry is intentional: sprinting is a genuine
  tradeoff between speed and stealth.
- `enemy.mode` is one of `"patrol"`, `"investigate"`, `"chase"`.
- `enemy.chasingBySound` distinguishes *why* he's in chase mode:
  - `true`: he heard something. He beelines for the **last known sound
    location** (`enemy.lastKnownX/Y`), not the player's live position,
    at `enemy.chargeSpeed` (very fast — "insane speed" per design ask).
    If the player stops making noise, he arrives at a stale location
    and has to re-acquire, which is the player's actual out.
  - `false`: he's actually looking at the player (point-blank sight).
    He tracks live position at the slower `enemy.chaseSpeed`.
- `alertEnemyToSound(x, y)` is the single entry point for "something
  loud happened here." Call this, don't hand-roll mode transitions,
  when adding a new noise-producing mechanic.
- Sound sources currently wired in: player footsteps/sprinting (checked
  every frame via distance + line-of-sight in `updateEnemy`), power
  node activation (`interact()`), and the call-for-Medic button.

## The call-for-Medic mechanic

Pressing `E` first tries to interact with anything in range (power
node, exit gate). If nothing happened, it calls `callForMedic()`
instead — the player actively summons Medic to their position. This is
a risk/reward tool: lure him away from a power node or the exit so you
can slip past, at the cost of him now beelining for exactly where you
are. 6-second cooldown (`callCooldownDuration`). Plays `Spy_call.wav`
plus one random voice line, and shows a blinking/fading icon
(`Spy_sign_noise.png`) above the player.

## Audio-visual conventions worth preserving

- **Animated GIFs used as UI/world elements are real `<img>` tags**,
  positioned absolutely over the canvas, not drawn into the canvas via
  `ctx.drawImage`. This was a deliberate fix: browsers do not reliably
  animate a GIF that's only ever sampled as a canvas texture source
  (they can freeze on one frame, or desync when several `<img>` tags
  share one cached source). If you add a new animated sprite, follow
  the existing `updateEnemySpriteElement` / `updatePlayerSpriteElement`
  / `positionDoorGifEl` pattern: real `<img>` element, positioned in
  screen space each frame from world coordinates via
  `canvas.getBoundingClientRect()` scaling.
- **Distance-based dimming ("circle of light")**: the player has a
  visible light radius (`player.sightRange`). Objects and the Medic
  sprite never fully vanish outside it — they ease toward a dim
  `filter: brightness()` floor (`dimBrightness`) via
  `easeLightBrightness()`, never a hard show/hide. This was a specific
  fix after early versions made things flicker in and out of existence
  at the sight radius edge. Keep new visual elements consistent with
  this: dim, don't disappear.
- **One-shot GIF playback**: door-opening GIFs are timed to their
  actual frame count (`gateOpeningDuration` matches the GIF's real
  playtime) and are hidden, not looped, once the timer expires. If you
  add a new door/animated prop, measure its real duration rather than
  guessing.

## Known asset-dependent features

Some features are wired into the code by filename but the asset may or
may not exist on disk yet at any given time (the project owner adds
art/audio incrementally). These fail gracefully — check
`playerSpriteAssetsOk`-style flags and `.complete`/`naturalWidth`
guards before assuming an image failed to load is a bug. If you see a
404 in the console for a file that matches this project's naming
convention (`Spy_*`, `Cf_*`, `Sm_*`, `Medic_*`), it likely just hasn't
been dropped into the folder yet — don't "fix" it by removing the
reference.

## File layout

Everything is in `index.html`:
- `<style>` block: all CSS, includes the HUD overlay, death screen,
  and sprite/overlay positioning classes.
- Body: canvas + a flat list of absolutely-positioned `<img>`/`<div>`
  overlays layered on top of it (sprites, door GIFs, HUD, death
  screen, jumpscare, start overlay).
- `<script>` block: everything else. Roughly, top to bottom: audio/
  image asset declarations, the tile map (`LEVEL_MAP`), player/enemy
  state objects, `buildMap()`/`resetGame()` (map parsing and state
  reset), input handling, `interact()`, `updateEnemy()` (AI),
  `updatePlayer()`, rendering functions (`drawWorld()` and friends),
  HUD update, death/win screens, and the main `loop()`.

`index.checkpoint-NNN.html` files are point-in-time backups made on
request ("checkpoint") — snapshots to roll back to, not part of the
live game. `index.html` is always the current version.

## Things that look like bugs but are current design

- Medic losing the player quickly after a chase (short `alertTimer`
  windows) is intentional — he's not supposed to track you accurately
  once he loses your sound trail.
- The player character is only ever lit up close to Medic when Medic
  is actually looking at them (point-blank) — beyond that his sprite
  stays dim even mid-chase, because he's chasing a sound memory, not
  watching you.

## Things still in flux / worth confirming with the project owner before big changes

- Exact tuning numbers (speeds, hearing radii, cooldowns) are early
  estimates, not final balance — expect requests to adjust them.
- The map (`LEVEL_MAP`) is a single small level. If asked to add more
  levels/rooms, check whether the existing single-camera assumptions
  (canvas size, camera clamping in `updateCamera()`) need to scale too.
