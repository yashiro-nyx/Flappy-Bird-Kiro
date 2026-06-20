# Game Mechanics — Flappy Kiro

## Physics Constants

All values live in `js/config.js`. Never hardcode these in game logic.

| Constant | Value | Unit | Purpose |
|----------|-------|------|---------|
| gravity | 0.6 | px/frame² | Downward acceleration per frame |
| flapImpulse | -10 | px/frame | Upward velocity on input (negative = up) |
| terminalVelocity | 12 | px/frame | Maximum downward speed |
| frameTime | 16.67 | ms | Target frame duration (60 FPS) |
| baseSpeed | 3 | px/frame | Pipe scroll speed at multiplier 1.0 |

## Ghosty Movement Physics

### Gravity Model

Ghosty uses simple Euler integration with frame-rate independence:

```
velocity += gravity × dtFactor
velocity = min(velocity, terminalVelocity)
position.y += velocity × dtFactor
```

Where `dtFactor = clampedDt / frameTime` (1.0 at exactly 60 FPS).

### Flap Behavior

- Flap sets velocity to `flapImpulse` regardless of current velocity
- No additive force — velocity is replaced, not accumulated
- This gives consistent jump height independent of current fall speed
- Maximum one flap processed per frame

### Horizontal Position

- Ghost x is fixed at 300px (1/3 of canvas width)
- The world scrolls, not the ghost
- Ghost x must never be modified by any physics update

### Rotation Interpolation

Ghost rotation visually indicates velocity direction:

```
normalizedVelocity = (velocity - flapImpulse) / (terminalVelocity - flapImpulse)
clampedNorm = clamp(normalizedVelocity, 0, 1)
rotation = maxUp + clampedNorm × (maxDown - maxUp)
```

- `maxUp` = -30° (nose up during ascent)
- `maxDown` = 90° (nose down at terminal velocity)
- Interpolation is linear, applied every frame after velocity update

### Delta-Time Clamping

After tab backgrounding, dt can spike to seconds. Clamp to prevent tunneling:

```
maxDt = frameTime × 3  (≈50ms)
clampedDt = min(rawDt, maxDt)
```

This means the game "freezes" rather than fast-forwarding when returning from a background tab.

## Pipe Generation Algorithm

### Spawn Trigger

A new pipe pair spawns when the rightmost active pipe's x position ≤ `canvasWidth - spacing`:

```
if (rightmostPipe.x <= 900 - 280) → spawn new pair
```

If no pipes exist, spawn immediately.

### Gap Positioning

Gap center is randomized within bounds that ensure minimum pipe visibility:

```
minGapCenter = minTopHeight + gapHeight/2     = 50 + 75 = 125
maxGapCenter = canvasHeight - scoreBarHeight - minBottomHeight - gapHeight/2
             = 600 - 40 - 50 - 75 = 435
gapCenter = random in [125, 435]
```

This guarantees at least 50px of pipe visible at top and bottom.

### Pipe Pair Structure

Each pipe pair acquires TWO objects from the pool:
- Top pipe: extends from y=0 down to `gapCenter - gapHeight/2`
- Bottom pipe: starts at `gapCenter + gapHeight/2` down to score bar

Both share the same x position, gapCenter, and scored flag.

### Pipe Dimensions

- Body width: 60px
- Cap width: 68px (4px overhang each side)
- Cap height: 15px
- Gap height: 150px (constant, never varies)
- Horizontal spacing: 280px (constant regardless of speed)

### Scrolling

```
pipeSpeed = baseSpeed × speedMultiplier × dtFactor
pipe.x -= pipeSpeed
```

### Cleanup

Release pipe back to pool when fully off-screen: `pipe.x + pipe.width < 0`

## Scoring System

### Score Condition

Score increments when a pipe's right edge crosses the ghost's x position:

```
if (pipe.x + pipe.width < ghost.x && !pipe.scored) → score++
```

Mark both pipes in the pair as scored to prevent double-counting.

### Speed Multiplier Formula

```
speedMultiplier = min(1.0 + score × 0.02, 2.0)
```

- Starts at 1.0 (3 px/frame effective speed)
- Increases 0.02 per point
- Caps at 2.0 (6 px/frame effective speed, reached at score 50)

### High Score Persistence

- Key: `'flappyKiroHighScore'` in localStorage
- Load: parse as integer, validate non-negative, fallback to 0
- Save: on game over if current > stored, wrapped in try/catch
- Never block gameplay on storage failure

## Collision Detection

### Ghost Hitbox (Circle)

```
centerX = ghost.x + ghost.width / 2
centerY = ghost.y + ghost.height / 2
radius = ghost.width / 2 - hitboxMargin    (= 20 - 4 = 16px)
```

The 4px margin makes collisions feel fair — near misses at pipe corners don't register.

### Pipe Hitbox (AABB with Cap Extension)

```
capExtension = (capWidth - width) / 2 = 4px

Top pipe:
  left:   pipe.x - capExtension
  top:    0
  right:  pipe.x + pipe.width + capExtension
  bottom: pipe.topHeight + capHeight

Bottom pipe:
  left:   pipe.x - capExtension
  top:    pipe.bottomY - capHeight
  right:  pipe.x + pipe.width + capExtension
  bottom: canvasHeight - scoreBarHeight
```

### Circle-vs-Rectangle Algorithm

Find closest point on rectangle to circle center, check distance:

```javascript
closestX = clamp(circle.centerX, rect.left, rect.right)
closestY = clamp(circle.centerY, rect.top, rect.bottom)
dx = circle.centerX - closestX
dy = circle.centerY - closestY
collision = (dx² + dy²) < radius²
```

Always compare squared distances — never use `Math.sqrt()`.

### Boundary Collisions

- Floor: `centerY + radius > canvasHeight - scoreBarHeight` (= 560 - radius threshold)
- Ceiling: `centerY - radius < 0`

### Invincibility

- Duration: 300ms after collision triggers
- During invincibility: skip all collision checks (early return)
- Visual: ghost opacity flashes between 0.3 and 1.0 every 100ms

## Input Responsiveness

### Flap Processing

- Input captured immediately on keydown/click/touchstart
- At most one flap per frame (flag-based deduplication)
- Flag reset by engine at start of each frame, before input events can fire again
- No input buffering or queuing — instant response

### Pause Toggle

- Escape or P key toggles pause
- Only these keys can resume — Space/Click while paused is ignored
- Prevents accidental flap when returning from pause

### Game Over Lockout

- 500ms lockout after game-over event
- Prevents accidental restart from the same input that caused death
- Timer decremented by raw clamped dt each frame

### Auto-Pause

- `visibilitychange` event with `document.hidden === true` triggers pause
- Only triggers from Playing state
- Prevents unfair death while tab is backgrounded

## Smooth Animation Interpolation

### Score Pulse

On score increment:
```
scorePulse.scale = 1.3
scorePulse.timer = 200ms
```
Scale linearly decays back to 1.0 over the timer duration.

### Score Popup

On score increment:
```
popup.y += floatSpeed × dtFactor   (floatSpeed = -2, upward)
popup.opacity = 1 - (age / lifetime)
```
Removed after 500ms lifetime.

### Screen Shake

On collision:
```
shake.timer = 300ms
shake.offsetX = random(-maxDisplacement, maxDisplacement) × (timer/duration)
shake.offsetY = random(-maxDisplacement, maxDisplacement) × (timer/duration)
```
Decays linearly to zero. Max displacement: 5px in each axis.

### Ghost Menu Bob

While in menu state:
```
bobOffset = sin(bobTimer / bobPeriod × 2π) × bobAmplitude
```
- Amplitude: 5px
- Period: 2000ms full cycle

### Particle Trail

- Emit one particle every 3 frames from ghost center-rear
- Particle drifts left at -1.5 px/frame
- Opacity fades from 0.6 to 0 over 400ms lifetime
- Radius: random 2–4px

### Cloud Parallax

- 3–6 clouds with individual speeds (30%–70% of pipe speed)
- Opacity correlates with speed: slower = more transparent (farther)
- Reposition to right edge with random y when scrolling off left
