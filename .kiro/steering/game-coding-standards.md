# Game Coding Standards — Flappy Kiro

## Architecture

### Module Structure

- Use ES modules (`import`/`export`) with no bundler
- One responsibility per file: physics, rendering, input, audio, state, collision, scoring, entities
- All game constants live in `js/config.js` — no magic numbers in logic modules
- Modules communicate via function calls with shared state objects passed as arguments

### State Management

- Single `gameState` object holds all mutable game state
- Finite state machine with explicit transitions: `menu → playing → paused/gameOver`
- Transition functions guard against invalid source states
- Entity containers live inside `gameState.entities`

### Entity-Component Pattern

- Game objects (Ghost, Pipe, Cloud, Particle, ScorePopup) are plain data objects — no classes
- System functions (physics, collision, scoring, rendering) operate on entity data
- Factory functions (`createPipe()`, `createGhost()`, etc.) produce entity objects with default values from CONFIG

## Naming Conventions

- **Files:** lowercase kebab-case is not used; use single-word lowercase (`physics.js`, `entities.js`, `config.js`)
- **Classes:** PascalCase (`ObjectPool`)
- **Factory functions:** camelCase prefixed with `create` (`createPipe`, `createGhost`, `createScorePopup`)
- **System functions:** camelCase verbs (`updatePhysics`, `checkCollisions`, `updateScoring`, `render`)
- **State transitions:** camelCase verbs (`startGame`, `pauseGame`, `resetGame`, `triggerGameOver`)
- **Constants:** UPPER_SNAKE_CASE only for true constants defined at module scope; CONFIG properties use nested camelCase objects
- **Boolean fields:** prefix with `is`, `has`, or `active` (`isInvincible`, `active`, `scored`)

## Game Loop

### Structure

```javascript
function tick(timestamp) {
  const dt = timestamp - lastTimestamp;
  lastTimestamp = timestamp;

  if (gameState.current === 'playing') {
    updatePhysics(gameState, dt);
    generatePipes(gameState);
    cleanupPipes(gameState);
    checkCollisions(ghost, activePipes);
    updateScoring(gameState);
  }

  render(gameState);
  requestAnimationFrame(tick);
}
```

### Delta-Time Normalization

- All motion uses `dtFactor = clampedDt / CONFIG.physics.frameTime`
- Clamp dt to max 3× frame duration to prevent physics explosion after tab backgrounding
- Position updates: `entity.position += entity.velocity * dtFactor`
- Never assume a fixed 60 FPS — always multiply by dtFactor

### Frame Independence

- Render every frame regardless of game state (paused still draws the frozen frame)
- Physics updates are gated by `gameState.current === 'playing'`
- Timer decrements use raw clamped dt in milliseconds (not dtFactor)

## Memory Management

### Object Pooling

- Pre-allocate pools for frequently created/destroyed objects (Pipes, Particles)
- Use `acquire()` to get an inactive object, reset its properties, then use it
- Use `release(obj)` to mark it inactive when done
- Use `releaseAll()` on game restart — never deallocate pool contents
- Pool sizes configured in `CONFIG.pools` — sized for worst-case active count with headroom

### Zero-Allocation Hot Path

- No `new` in per-frame update or render functions
- No object literals created in the game loop (use pre-allocated scratch objects)
- Array `filter()` in `getActive()` is the one allocation — acceptable for pool sizes under 100
- Score popups use a simple array with splice removal (small count, infrequent)

### Garbage Collection Avoidance

- Reuse entity objects from pools rather than creating fresh ones
- Avoid string concatenation in hot paths; pre-format where possible
- Minimize closure creation inside loops

## Canvas API Patterns

### Rendering Pipeline

- Clear canvas once per frame with single `fillRect` background
- Draw in strict layer order: background → clouds → pipes → particles → ghost → popups → score bar → overlays
- Use `ctx.save()`/`ctx.restore()` only for screen shake transform wrapping and sprite rotation

### Render Batching

- Group draws by style to minimize context state changes:
  1. Set cloud fill → draw all clouds
  2. Set pipe body fill → draw all pipe bodies → set cap fill → draw all caps → set stroke → stroke all
  3. Set particle composite → draw all particles
- This reduces state changes from O(entities) to O(entity-types)

### Efficient Drawing

- Use `ctx.translate()` + `ctx.rotate()` for sprite rotation, not manual matrix math
- For hand-drawn effect: offset stroke path by 1–3px random per edge, keep fill stable
- Screen shake: single `ctx.translate(shakeX, shakeY)` before all draws, restore after

### Animation Frame Handling

- Use `requestAnimationFrame` — never `setInterval` or `setTimeout` for the game loop
- Store previous timestamp to compute dt
- Handle first frame (dt = 0) gracefully

## Collision Detection

### Circle-vs-Rectangle (Primary Algorithm)

```javascript
function circleIntersectsRect(circle, rect) {
  const closestX = Math.max(rect.left, Math.min(circle.centerX, rect.right));
  const closestY = Math.max(rect.top, Math.min(circle.centerY, rect.bottom));
  const dx = circle.centerX - closestX;
  const dy = circle.centerY - closestY;
  return (dx * dx + dy * dy) < (circle.radius * circle.radius);
}
```

- Compare squared distances — never call `Math.sqrt()` in collision checks
- Ghost uses circular hitbox (forgiving at corners), pipes use AABBs
- Include cap extensions in pipe AABBs (±4px width overhang)

### Boundary Checks

- Floor: `centerY + radius > canvasHeight - scoreBarHeight`
- Ceiling: `centerY - radius < 0`
- Both use the circle hitbox, not the sprite bounding box

### Performance Rules

- Skip collision checks during invincibility frames (early return)
- Evaluate collisions once per frame, after position updates, before rendering
- No broad-phase needed at this entity count (< 20 active pipes)

## Event Handling

### Input Design

- Bind events once at initialization via `initInput(callbacks)`
- Use callback pattern — input module calls engine-provided functions (`onFlap`, `onRestart`)
- One flap per frame maximum (flag reset each frame by engine)
- `visibilitychange` for auto-pause on tab blur

### Input State by Game State

| Game State | Space/Click/Touch | Escape/P |
|------------|-------------------|----------|
| Menu       | Start game + flap | Ignored  |
| Playing    | Flap              | Pause    |
| Paused     | Ignored           | Resume   |
| Game Over  | Restart (after lockout) | Ignored |

### Audio Events

- AudioContext created lazily, resumed on first user gesture
- All playback wrapped in try/catch — failures set `audioEnabled = false`
- Jump sound: restart-if-playing (stop previous, start new)
- Score tone: synthesized via OscillatorNode (no file dependency)

## Performance Monitoring

- Track consecutive slow frames (> 20ms / below 50 FPS)
- Log console warning after 10+ consecutive slow frames
- Reset counter on any normal-speed frame
- No runtime performance fixes — warning is diagnostic only

## Testing

- Use property-based testing (fast-check) for universal correctness properties
- Test physics, collision, scoring, and pool logic as pure functions
- Mock browser APIs (Canvas, localStorage, AudioContext) in test environment
- Property tests validate invariants across randomized inputs, not specific examples
