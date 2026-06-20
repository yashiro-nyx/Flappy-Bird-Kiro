# Visual Design — Flappy Kiro

## Rendering Architecture

### Layer Order (Back to Front)

1. Background fill (`#87CEEB` light blue)
2. Clouds (parallax, semi-transparent)
3. Pipes (body → caps → strokes)
4. Particles (ghost trail)
5. Ghost (sprite or fallback circle, rotated)
6. Score popups (floating +1 text)
7. Score bar (dark bottom bar)
8. UI overlays (menu, pause, game over)

Screen shake offset wraps layers 1–7 via `ctx.translate()`.

### Canvas Setup

- Internal resolution: 900×600 (fixed for all game logic)
- Display scales responsively to viewport (3:2 aspect ratio)
- `image-rendering: pixelated` on the canvas element for retro crispness
- Background color on body: `#1a1a2e` (dark, frames the game canvas)

## Ghosty Character

### Sprite Rendering

- Source: `assets/ghosty.png` (40×40px)
- Drawn with rotation applied via `ctx.translate()` + `ctx.rotate()`
- Rotation center: sprite center (x + width/2, y + height/2)

```javascript
ctx.save();
ctx.translate(ghost.centerX, ghost.centerY);
ctx.rotate(ghost.rotation);
ctx.drawImage(sprite, -ghost.width/2, -ghost.height/2, ghost.width, ghost.height);
ctx.restore();
```

### Fallback Rendering

If sprite fails to load, render a white filled circle:
- Center: (ghost.centerX, ghost.centerY)
- Radius: `CONFIG.ghost.fallbackRadius` (18px)
- Color: `CONFIG.colors.ghostFallback` (#ffffff)

### Invincibility Flash

During invincibility frames (300ms after collision):
- Toggle `ghost.flashVisible` every 100ms
- When visible: `globalAlpha = 1.0`
- When hidden: `globalAlpha = 0.3`
- Reset alpha after drawing ghost

### Menu Idle Animation

While in menu state:
- Ghost bobs vertically: `y + sin(timer / period × 2π) × amplitude`
- Amplitude: 5px, Period: 2000ms
- No rotation during bob (stays level)

## Pipe Rendering

### Structure

Each pipe has a body and a cap:
- Body: green filled rectangle (`#228B22`)
- Cap: darker green rectangle (`#165B16`), 68px wide × 15px tall
- Top pipe: cap at the bottom edge (gap side)
- Bottom pipe: cap at the top edge (gap side)

### Hand-Drawn Effect

Apply random stroke offset (1–3px) to pipe edges each frame:

```javascript
const offset = () => CONFIG.handDrawn.strokeOffset.min + 
  Math.random() * (CONFIG.handDrawn.strokeOffset.max - CONFIG.handDrawn.strokeOffset.min);

ctx.strokeStyle = CONFIG.colors.pipeStroke;
ctx.strokeRect(
  pipe.x + offset(), 
  pipeTop + offset(), 
  pipe.width + offset(), 
  pipeHeight + offset()
);
```

- Fill remains stable (no offset on fillRect)
- Stroke wobbles subtly each frame for a sketched look
- Offset recalculated every frame (not cached)

### Render Batching

Draw all pipes in batched passes to minimize state changes:
1. Set `fillStyle = pipeBody` → draw all pipe bodies
2. Set `fillStyle = pipeCap` → draw all pipe caps
3. Set `strokeStyle = pipeStroke` → stroke all pipe outlines

## Background and Clouds

### Cloud Rendering

- Shape: rounded rectangle (white, semi-transparent)
- Opacity: 0.4–0.8 (correlates with speed for parallax depth)
- Corner radius: `cloud.height / 2` (pill shape)
- Use `ctx.globalAlpha` for transparency, reset after batch

### Cloud Parallax

- Slower clouds = more transparent (appear farther away)
- Faster clouds = more opaque (appear closer)
- Speed range: 30%–70% of pipe scroll speed
- Count: 3–6 clouds on screen

### Cloud Recycling

When `cloud.x + cloud.width < 0`:
- Reset x to `CONFIG.canvas.width + random offset`
- Randomize y within playable area (above score bar)
- Keep original speed/opacity (maintains layer consistency)

## Particle Effects

### Ghost Trail

- Emission: one particle every 3 frames during Playing state
- Origin: ghost center-rear (center minus half width for x)
- Initial properties:
  - Radius: random 2–4px
  - Opacity: 0.6
  - Drift: -1.5 px/frame (leftward)
  - Lifetime: 400ms

### Particle Rendering

```javascript
ctx.globalAlpha = particle.opacity;
ctx.fillStyle = '#ffffff';
ctx.beginPath();
ctx.arc(particle.x, particle.y, particle.radius, 0, Math.PI * 2);
ctx.fill();
```

- Batch all particles with single style set
- Set `globalAlpha` per particle (varies as they fade)
- Reset `globalAlpha = 1.0` after particle batch

### Particle Lifecycle

- Age incremented by clamped dt each frame
- Opacity: `initialOpacity × (1 - age/lifetime)` (linear fade)
- Released back to pool when age ≥ lifetime

## Screen Shake

### Implementation

```javascript
if (gameState.screenShake.active) {
  const decay = gameState.screenShake.timer / CONFIG.screenShake.duration;
  const maxOff = CONFIG.screenShake.maxDisplacement;
  gameState.screenShake.offsetX = (Math.random() * 2 - 1) * maxOff * decay;
  gameState.screenShake.offsetY = (Math.random() * 2 - 1) * maxOff * decay;
}

ctx.save();
ctx.translate(gameState.screenShake.offsetX, gameState.screenShake.offsetY);
// ... draw all game layers ...
ctx.restore();
```

### Parameters

- Duration: 300ms
- Max displacement: 5px per axis
- Decay: linear (displacement × timer/duration)
- Trigger: pipe collision

## Score Bar

### Layout

- Position: bottom of canvas, full width
- Height: 40px
- Background: `#2d2d2d`
- Text color: `#ffffff`
- Font size: 18px

### Content

- Left side: "Score: X"
- Right side: "High: X"

### Score Pulse Animation

On score increment:
- Scale text to 1.3× for 200ms
- Apply via `ctx.scale()` around the score text draw
- Decay linearly back to 1.0

```javascript
if (gameState.scorePulse.active) {
  const progress = gameState.scorePulse.timer / CONFIG.scoreBar.pulseDuration;
  const scale = 1 + (CONFIG.scoreBar.pulseScale - 1) * progress;
  ctx.save();
  ctx.translate(scoreX, scoreY);
  ctx.scale(scale, scale);
  ctx.fillText(scoreText, 0, 0);
  ctx.restore();
}
```

## Score Popups

### Appearance

- Text: "+1"
- Font size: 24px
- Color: white
- Initial position: ghost's current position

### Animation

- Float upward: -2 px/frame (dtFactor normalized)
- Fade out: opacity from 1.0 to 0.0 over 500ms
- Remove from array when lifetime exceeded

## UI Overlays

### Menu Screen

- Title: "Flappy Kiro" — centered horizontally, upper third
  - Font: 48px, white
- High score: "Best: X" — centered below title
- Prompt: "Click or Press Space to Start" — lower third
  - Font: 20px, white
- Ghost with idle bob animation on left third

### Pause Overlay

- Semi-transparent dark fill over entire canvas: `rgba(0, 0, 0, 0.5)`
- "PAUSED" text: centered, large white font
- "Press Escape or P to Resume" below

### Game Over Screen

- "Game Over" text: centered, large
- Final score: prominently displayed
- "New Best!" label: shown when score ≥ highScore
- "Click or Press Space to Restart" prompt
- Displayed after 500ms input lockout

## Audio-Visual Integration

### Sound-to-Visual Mapping

| Event | Sound | Visual |
|-------|-------|--------|
| Flap | jump.wav (restart-if-playing) | Ghost velocity reset, rotation snaps up |
| Score | Oscillator 600→900Hz, 100ms | Score pulse + popup + bar update |
| Collision | game_over.wav | Screen shake + invincibility flash |
| Game over | (same as collision) | State overlay appears after lockout |

### Audio Timing

- Sounds play immediately on event (no delay)
- Jump sound restarts from beginning if already playing
- Score tone is synthesized (no file load latency)
- All audio is non-blocking — failures are silent

### Visual Feedback Priority

When multiple effects overlap (collision triggers shake + flash + sound):
1. Screen shake applies first (wraps all rendering)
2. Invincibility flash applies to ghost draw only
3. Game over overlay renders on top of everything
4. Sound plays independently of visual frame

## Drawing Optimization

### State Change Minimization

- Group by fill/stroke style, not by entity
- Set `globalAlpha` only when needed (particles, clouds, flash)
- Use `save()`/`restore()` sparingly — only for transforms (rotation, shake)

### Avoid Per-Frame Allocations

- No object creation in render path
- Reuse string templates for score text (or accept minor string concat)
- No `new Path2D()` in hot path

### Font Rendering

- Set font once per text batch, not per text draw
- Use `textAlign` and `textBaseline` to simplify positioning
- Avoid measuring text width repeatedly (cache or estimate)
