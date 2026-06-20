# Flappy Kiro Domain — Game State & Session Management

## Game State Machine

### States

| State | Description | Active Systems |
|-------|-------------|----------------|
| `menu` | Title screen, waiting for first input | Rendering, ghost bob animation |
| `playing` | Active gameplay | Physics, collision, scoring, rendering, input |
| `paused` | Gameplay frozen | Rendering (frozen frame + overlay) |
| `gameOver` | Death screen, waiting for restart | Rendering (overlay), timer countdown |

### Valid Transitions

```
menu → playing        (Space/Click/Touch)
playing → paused      (Escape/P or tab blur)
paused → playing      (Escape/P only)
playing → gameOver    (collision detected)
gameOver → playing    (Space/Click/Touch after 500ms lockout)
```

### Transition Guards

- Each transition function checks `gameState.current` before executing
- Invalid transitions are silently ignored (no error thrown)
- `resetGame()` only executes from `gameOver` state

## Session Management

### Game Start (Menu → Playing)

1. Set `gameState.current = 'playing'`
2. Reset score to 0
3. Reset speed multiplier to 1.0
4. Reset frame count to 0
5. Clear all timers and effects
6. Ghost begins falling from start position immediately

### Game Over (Playing → GameOver)

1. Set `gameState.current = 'gameOver'`
2. Activate screen shake (300ms)
3. Activate invincibility frames (300ms)
4. Set input lockout timer (500ms)
5. Play game_over.wav
6. Save high score if current > stored
7. Cease all physics updates

### Restart (GameOver → Playing)

1. Set `gameState.current = 'playing'`
2. Reset score to 0
3. Reset speed multiplier to 1.0
4. Reset frame count
5. Clear screen shake and score pulse
6. Release ALL pipes back to pool (`pipePool.releaseAll()`)
7. Release ALL particles back to pool (`particlePool.releaseAll()`)
8. Clear score popups array
9. Reset ghost: y → startY, velocity → 0, rotation → 0, invincibility → off
10. Preserve `highScore` (never reset)

### What Persists Across Sessions

- `highScore` — survives restarts AND page reloads (localStorage)
- Pool allocations — objects are reused, never deallocated
- Event listeners — bound once at init, never rebound

### What Resets on Restart

- Score, speed multiplier, frame count
- All entity positions and states
- All timers and visual effects
- Ghost position and velocity

## Score Persistence

### Storage Strategy

- Key: `'flappyKiroHighScore'`
- Storage: browser localStorage
- Format: stringified integer

### Load High Score

```
1. Try localStorage.getItem(key)
2. If null → return 0
3. Parse as integer
4. If NaN or negative or non-integer → return 0
5. Return parsed value
```

### Save High Score

```
1. Only save when current score > stored high score
2. Only save on game over (not during gameplay)
3. Wrap in try/catch
4. On failure → continue silently (private browsing, quota exceeded)
```

### High Score Update During Play

- `gameState.highScore` is updated in-memory immediately when score exceeds it
- Score bar reflects the new high score in real-time
- Actual localStorage write happens only on game over

## Difficulty Progression

### Speed Multiplier

```
speedMultiplier = min(1.0 + score × 0.02, 2.0)
```

| Score | Multiplier | Effective Speed | Pipes Per Second |
|-------|-----------|-----------------|------------------|
| 0 | 1.0 | 3 px/frame | ~0.64 |
| 10 | 1.2 | 3.6 px/frame | ~0.77 |
| 25 | 1.5 | 4.5 px/frame | ~0.96 |
| 50+ | 2.0 (cap) | 6 px/frame | ~1.29 |

### What Speeds Up

- Pipe scroll speed: `baseSpeed × multiplier × dtFactor`
- Pipe arrival frequency (consequence of faster scrolling + fixed spacing)

### What Stays Constant

- Gap height: always 150px
- Gap positioning bounds: always 50px minimum visibility
- Horizontal pipe spacing: always 280px (measured in pixels, not time)
- Gravity, flap impulse, terminal velocity
- Cloud speeds (independent of multiplier)

### Difficulty Reset

On restart: `speedMultiplier = 1.0` (from CONFIG.difficulty.startingMultiplier)

## Obstacle Generation Rules

### Pipe Pair Spawn Conditions

A new pair spawns when:
- No active pipes exist (first pipe after game start/restart), OR
- Rightmost active pipe x ≤ `canvasWidth - spacing` (= 900 - 280 = 620)

### Gap Center Calculation

```
playableHeight = canvasHeight - scoreBarHeight = 600 - 40 = 560
minGapCenter = minTopHeight + gapHeight/2 = 50 + 75 = 125
maxGapCenter = playableHeight - minBottomHeight - gapHeight/2 = 560 - 50 - 75 = 435
gapCenter = random uniform in [125, 435]
```

### Pipe Properties on Spawn

```
pipe.x = canvasWidth (900)
pipe.topHeight = gapCenter - gapHeight/2
pipe.bottomY = gapCenter + gapHeight/2
pipe.width = 60
pipe.capWidth = 68
pipe.capHeight = 15
pipe.gapCenter = gapCenter
pipe.scored = false
pipe.active = true
```

### Pool Acquisition

- Each pipe pair acquires 2 objects from pipePool
- Both share the same x, gapCenter, scored status
- Top pipe: topHeight represents the pipe height from y=0 downward
- Bottom pipe: bottomY represents where the pipe starts (gap bottom edge)

### Pipe Removal

- Condition: `pipe.x + pipe.width < 0` (fully off-screen left)
- Action: `pipePool.release(pipe)` — marks inactive, ready for reuse
- Both pipes in a pair are released independently (same condition applies)

## Ghosty Behavior Patterns

### During Menu

- Position: fixed x (300), centered y (300)
- Velocity: 0 (no gravity)
- Animation: vertical bob (±5px, 2s cycle)
- Rotation: 0 (level)
- No collision checks

### During Play

- Gravity: continuously applied each frame
- Flap: replaces velocity with impulse on input
- Rotation: interpolated from velocity
- Hitbox: circle updated each frame after position change
- Particle emission: every 3 frames from rear center

### During Game Over

- Physics frozen (no velocity changes, no position updates)
- Ghost remains at death position
- Invincibility flash active for 300ms
- No input response for 500ms lockout

### During Pause

- Physics frozen
- Ghost remains at paused position
- No particle emission
- No input response except Escape/P to resume

## Collision Response Chain

When collision is detected:

```
1. checkCollisions() returns true
2. Engine calls triggerGameOver():
   a. gameState.current = 'gameOver'
   b. inputLockoutTimer = 500ms
   c. screenShake.active = true, timer = 300ms
3. Engine activates invincibility:
   a. ghost.isInvincible = true
   b. ghost.invincibleTimer = 300ms
4. Engine plays game_over.wav
5. Engine saves high score (if new best)
6. All subsequent frames: render frozen state + game over overlay
7. Timer countdown: inputLockoutTimer decremented each frame
8. After lockout expires: next input triggers resetGame()
```

### Invincibility During Game Over

- Prevents re-triggering collision checks during the transition
- Ghost flashes visually (opacity toggle)
- Lasts 300ms then deactivates
- Only relevant if physics were still running (they're not in gameOver, but serves as a safety net)

## Timer Management

### Active Timers

| Timer | Duration | Decremented By | Purpose |
|-------|----------|----------------|---------|
| inputLockoutTimer | 500ms | clamped dt | Prevent accidental restart |
| screenShake.timer | 300ms | clamped dt | Shake decay |
| invincibleTimer | 300ms | clamped dt | Flash duration |
| scorePulse.timer | 200ms | clamped dt | Score text pulse |

### Timer Rules

- Decremented by raw clamped dt (milliseconds), not dtFactor
- Deactivate effect when timer ≤ 0
- Timers run in ALL states (game over timers must still count down)
- Multiple timers can be active simultaneously (shake + invincibility + lockout)
