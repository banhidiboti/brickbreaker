<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const emit = defineEmits(['goHome'])
const resumeGame = () => { paused.value = false }
const togglePause = () => {
  if (!gameOver.value && !won.value) paused.value = !paused.value
}

const goHome = () => {
  paused.value = false
  gameOver.value = false
  cancelAnimationFrame(animationFrameId)
  emit('goHome')
}

const canvas = ref(null)
const score = ref(0)
const lives = ref(3)
const gameOver = ref(false)
const paused = ref(false)
const won = ref(false)
const finalScore = ref(0)
const bonusBalls = ref([])

let ctx = null
let animationFrameId = null

// Internal game resolution — all logic runs at these coordinates
const WIDTH = 480
const HEIGHT = 640
const HUD_PAUSE_BREAKPOINT = 900

// Scale factor: fit the game into the window while keeping aspect ratio
const scale = ref(1)
const showHudPause = ref(false)
let draggingPaddle = false
let activePointerId = null

const updateScale = () => {
  const scaleX = window.innerWidth / WIDTH
  const scaleY = window.innerHeight / HEIGHT
  scale.value = Math.min(scaleX, scaleY)
  showHudPause.value = window.innerWidth <= HUD_PAUSE_BREAKPOINT
}

const clamp = (value, min, max) => Math.max(min, Math.min(max, value))

const getGameCoordsFromPointer = (event) => {
  const rect = canvas.value.getBoundingClientRect()
  const x = (event.clientX - rect.left) * (WIDTH / rect.width)
  const y = (event.clientY - rect.top) * (HEIGHT / rect.height)
  return { x, y }
}

const onPointerDown = (event) => {
  if (gameOver.value || won.value) return

  const { x, y } = getGameCoordsFromPointer(event)
  const p = paddle.value
  const isOnPaddle =
    x >= p.x &&
    x <= p.x + p.width &&
    y >= p.y - 10 &&
    y <= p.y + p.height + 10

  if (!isOnPaddle) return

  draggingPaddle = true
  activePointerId = event.pointerId
  p.movingLeft = false
  p.movingRight = false
  p.x = clamp(x - p.width / 2, 0, WIDTH - p.width)
  canvas.value?.setPointerCapture?.(event.pointerId)
}

const onPointerMove = (event) => {
  if (!draggingPaddle || activePointerId !== event.pointerId) return

  const { x } = getGameCoordsFromPointer(event)
  const p = paddle.value
  p.movingLeft = false
  p.movingRight = false
  p.x = clamp(x - p.width / 2, 0, WIDTH - p.width)
}

const onPointerUp = (event) => {
  if (activePointerId !== event.pointerId) return
  draggingPaddle = false
  activePointerId = null
  paddle.value.movingLeft = false
  paddle.value.movingRight = false
  canvas.value?.releasePointerCapture?.(event.pointerId)
}

// ── Game State ─────────────────────────────────────────────────────────────

const paddle = ref({
  x: WIDTH / 2 - 60,
  y: HEIGHT - 40,
  width: 120,
  height: 14,
  speed: 9,
  movingLeft: false,
  movingRight: false
})

const ball = ref({
  x: WIDTH / 2,
  y: HEIGHT - 100,
  radius: 9,
  dx: 3,
  dy: -3
})

const brickRowCount = 6
const brickColumnCount = 10
const brickWidth = 42
const brickHeight = 20
const brickPadding = 5
const brickOffsetTop = 55
const brickOffsetLeft = 8
const BONUS_BRICK_COUNT = 5

const bricks = ref([])

// ── Particles ──────────────────────────────────────────────────────────────

let particles = []

const spawnParticles = (x, y, color) => {
  for (let i = 0; i < 8; i++) {
    const angle = (Math.PI * 2 / 8) * i + (Math.random() - 0.5) * 0.5
    const speed = 1 + Math.random() * 2
    particles.push({
      x,
      y,
      vx: Math.cos(angle) * speed,
      vy: Math.sin(angle) * speed,
      alpha: 1,
      radius: 2 + Math.random() * 2,
      color
    })
  }
}

const updateParticles = () => {
  particles = particles.filter(p => p.alpha > 0.05)
  for (const p of particles) {
    p.x += p.vx
    p.y += p.vy
    p.alpha -= 0.045
    p.radius *= 0.95
  }
}

const drawParticles = () => {
  for (const p of particles) {
    ctx.globalAlpha = p.alpha
    ctx.fillStyle = p.color
    ctx.shadowColor = p.color
    ctx.shadowBlur = 6
    ctx.beginPath()
    ctx.arc(p.x, p.y, p.radius, 0, Math.PI * 2)
    ctx.fill()
  }
  ctx.globalAlpha = 1
  ctx.shadowBlur = 0
}

// ── Ball Trail ─────────────────────────────────────────────────────────────

let trail = []
const TRAIL_LENGTH = 20
const BONUS_TRAIL_LENGTH = 16

const updateTrail = () => {
  trail.push({ x: ball.value.x, y: ball.value.y })
  if (trail.length > TRAIL_LENGTH) trail.shift()
}

const drawTrail = (points, currentRadius, color, shadowColor) => {
  if (!points.length) return

  for (let i = 0; i < points.length; i++) {
    const alpha = (i / points.length) * 0.4
    const radius = currentRadius * (i / points.length) * 0.7
    ctx.globalAlpha = alpha
    ctx.fillStyle = color
    ctx.shadowColor = shadowColor
    ctx.shadowBlur = 8
    ctx.beginPath()
    ctx.arc(points[i].x, points[i].y, radius, 0, Math.PI * 2)
    ctx.fill()
  }
  ctx.globalAlpha = 1
  ctx.shadowBlur = 0
}

// ── Bricks ─────────────────────────────────────────────────────────────────

const makeBricks = () => {
  const grid = Array.from({ length: brickColumnCount }, () =>
    Array.from({ length: brickRowCount }, () => ({ status: 1, isBonus: false }))
  )

  const positions = []
  for (let c = 0; c < brickColumnCount; c++) {
    for (let r = 0; r < brickRowCount; r++) {
      positions.push({ c, r })
    }
  }

  const count = Math.min(BONUS_BRICK_COUNT, positions.length)
  for (let i = 0; i < count; i++) {
    const randomIndex = Math.floor(Math.random() * positions.length)
    const { c, r } = positions.splice(randomIndex, 1)[0]
    grid[c][r].isBonus = true
  }

  return grid
}

const spawnBonusBall = (x, y) => {
  bonusBalls.value.push({
    x,
    y,
    radius: 7,
    dx: (Math.random() > 0.5 ? 1 : -1) * 3.4,
    dy: -3.4,
    trail: []
  })
}

const initGame = () => {
  score.value = 0
  lives.value = 3
  gameOver.value = false
  paused.value = false
  won.value = false
  particles = []
  trail = []
  bonusBalls.value = []

  paddle.value.x = WIDTH / 2 - 60
  paddle.value.movingLeft = false
  paddle.value.movingRight = false

  ball.value.x = WIDTH / 2
  ball.value.y = HEIGHT - 100
  ball.value.dx = 3 * (Math.random() > 0.5 ? 1 : -1)
  ball.value.dy = -3

  bricks.value = makeBricks()
}

// ── Drawing ────────────────────────────────────────────────────────────────

const drawBackground = () => {
  ctx.clearRect(0, 0, WIDTH, HEIGHT)
}

const drawPaddle = () => {
  const p = paddle.value
  const grad = ctx.createLinearGradient(p.x, p.y, p.x, p.y + p.height)
  grad.addColorStop(0, '#00ffcc')
  grad.addColorStop(1, '#00aa88')

  ctx.shadowColor = '#00ffcc'
  ctx.shadowBlur = 18
  ctx.fillStyle = grad
  ctx.beginPath()
  ctx.roundRect(p.x, p.y, p.width, p.height, 7)
  ctx.fill()
  ctx.shadowBlur = 0
}

const drawBall = (b, innerColor, outerColor, shadowColor) => {
  ctx.shadowColor = shadowColor
  ctx.shadowBlur = 24
  const grad = ctx.createRadialGradient(b.x - 2, b.y - 2, 1, b.x, b.y, b.radius)
  grad.addColorStop(0, innerColor)
  grad.addColorStop(1, outerColor)
  ctx.fillStyle = grad
  ctx.beginPath()
  ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2)
  ctx.fill()
  ctx.shadowBlur = 0
}

// Row 0 = top (red = 30pts) ... Row 5 = bottom (purple = 5pts)
const BRICK_COLORS = [
  ['#ff4466', '#cc1133'], // row 0 — red    — 30 pts
  ['#ff7733', '#cc4400'], // row 1 — orange — 25 pts
  ['#ffcc00', '#cc9900'], // row 2 — yellow — 20 pts
  ['#44ff88', '#00cc55'], // row 3 — green  — 15 pts
  ['#44ccff', '#0099cc'], // row 4 — blue   — 10 pts
  ['#bb66ff', '#8833cc'], // row 5 — purple —  5 pts
]

const BRICK_POINTS = [30, 25, 20, 15, 10, 5]

const drawBricks = () => {
  for (let c = 0; c < brickColumnCount; c++) {
    for (let r = 0; r < brickRowCount; r++) {
      if (bricks.value[c][r].status !== 1) continue
      const bx = c * (brickWidth + brickPadding) + brickOffsetLeft
      const by = r * (brickHeight + brickPadding) + brickOffsetTop
      const [top, bot] = BRICK_COLORS[r % BRICK_COLORS.length]

      const grad = ctx.createLinearGradient(bx, by, bx, by + brickHeight)
      grad.addColorStop(0, top)
      grad.addColorStop(1, bot)

      ctx.shadowColor = top
      ctx.shadowBlur = 8
      ctx.fillStyle = grad
      ctx.beginPath()
      ctx.roundRect(bx, by, brickWidth, brickHeight, 4)
      ctx.fill()

      ctx.fillStyle = 'rgba(255,255,255,0.15)'
      ctx.beginPath()
      ctx.roundRect(bx + 3, by + 2, brickWidth - 6, 5, 2)
      ctx.fill()

      if (bricks.value[c][r].isBonus) {
        ctx.strokeStyle = 'rgba(255,255,255,0.95)'
        ctx.lineWidth = 1.2
        ctx.beginPath()
        ctx.roundRect(bx + 0.8, by + 0.8, brickWidth - 1.6, brickHeight - 1.6, 4)
        ctx.stroke()
      }

      ctx.shadowBlur = 0
    }
  }
}

const draw = () => {
  drawBackground()
  drawTrail(trail, ball.value.radius, '#ffee44', '#ffee44')
  for (const whiteBall of bonusBalls.value) {
    drawTrail(whiteBall.trail, whiteBall.radius, '#ffffff', '#ffffff')
  }
  drawParticles()
  drawBricks()
  drawPaddle()
  drawBall(ball.value, '#fffbe0', '#ffcc00', '#ffee44')
  for (const whiteBall of bonusBalls.value) {
    drawBall(whiteBall, '#ffffff', '#e9e9ff', '#ffffff')
  }

  // Draw HUD
  ctx.fillStyle = 'rgba(0,255,200,0.85)'
  ctx.font = 'bold 15px monospace'
  ctx.textAlign = 'left'
  ctx.fillText(`SCORE: ${score.value}`, 14, 24)
  ctx.textAlign = 'right'
  const livesTextRight = showHudPause.value ? 108 : 14
  ctx.fillText(`LIVES: ${'♥ '.repeat(lives.value).trim()}`, WIDTH - livesTextRight, 24)
}

// ── Logic ──────────────────────────────────────────────────────────────────

const finishIfWon = () => {
  const remaining = bricks.value.flat().filter(x => x.status === 1).length
  if (remaining === 0) {
    won.value = true
    gameOver.value = true
    const multiplier = lives.value >= 2 ? lives.value : 1
    finalScore.value = score.value * multiplier
  }
}

const collisionDetection = (currentBall) => {
  for (let c = 0; c < brickColumnCount; c++) {
    for (let r = 0; r < brickRowCount; r++) {
      const b = bricks.value[c][r]
      if (b.status !== 1) continue
      const bx = c * (brickWidth + brickPadding) + brickOffsetLeft
      const by = r * (brickHeight + brickPadding) + brickOffsetTop

      if (
        currentBall.x > bx &&
        currentBall.x < bx + brickWidth &&
        currentBall.y > by &&
        currentBall.y < by + brickHeight
      ) {
        currentBall.dy = -currentBall.dy
        b.status = 0
        score.value += BRICK_POINTS[r] ?? 10

        // Spawn small particles at brick center
        spawnParticles(bx + brickWidth / 2, by + brickHeight / 2, BRICK_COLORS[r][0])

        if (b.isBonus) {
          spawnBonusBall(bx + brickWidth / 2, by + brickHeight / 2)
        }

        finishIfWon()
        return
      }
    }
  }
}

const applyBallWallAndPaddleCollisions = (currentBall) => {
  if (currentBall.x + currentBall.radius > WIDTH || currentBall.x - currentBall.radius < 0) {
    currentBall.dx = -currentBall.dx
  }
  if (currentBall.y - currentBall.radius < 0) {
    currentBall.dy = Math.abs(currentBall.dy)
  }

  const p = paddle.value
  if (
    currentBall.y + currentBall.radius > p.y &&
    currentBall.y - currentBall.radius < p.y + p.height &&
    currentBall.x > p.x &&
    currentBall.x < p.x + p.width
  ) {
    currentBall.dy = -Math.abs(currentBall.dy)
    const hitPos = (currentBall.x - (p.x + p.width / 2)) / (p.width / 2)
    currentBall.dx = hitPos * 4
    currentBall.dx = Math.max(-5, Math.min(5, currentBall.dx))
  }
}

const update = () => {
  if (paused.value || gameOver.value) return

  ball.value.x += ball.value.dx
  ball.value.y += ball.value.dy
  for (const whiteBall of bonusBalls.value) {
    whiteBall.x += whiteBall.dx
    whiteBall.y += whiteBall.dy
    whiteBall.trail.push({ x: whiteBall.x, y: whiteBall.y })
    if (whiteBall.trail.length > BONUS_TRAIL_LENGTH) whiteBall.trail.shift()
  }

  updateTrail()
  updateParticles()

  applyBallWallAndPaddleCollisions(ball.value)
  for (const whiteBall of bonusBalls.value) {
    applyBallWallAndPaddleCollisions(whiteBall)
  }

  // Ball fell off bottom
  if (ball.value.y + ball.value.radius > HEIGHT) {
    lives.value--
    trail = []
    if (lives.value <= 0) {
      gameOver.value = true
      won.value = false
      finalScore.value = score.value
    } else {
      ball.value.x = WIDTH / 2
      ball.value.y = HEIGHT - 100
      ball.value.dx = 3 * (Math.random() > 0.5 ? 1 : -1)
      ball.value.dy = -3
    }
  }

  // Bonus white balls fell off bottom: remove them (no life loss, no respawn)
  bonusBalls.value = bonusBalls.value.filter(whiteBall => whiteBall.y - whiteBall.radius <= HEIGHT)

  // Paddle movement
  const p = paddle.value
  if (p.movingLeft && p.x > 0) p.x -= p.speed
  if (p.movingRight && p.x + p.width < WIDTH) p.x += p.speed

  collisionDetection(ball.value)
  for (const whiteBall of bonusBalls.value) {
    collisionDetection(whiteBall)
  }
}

const gameLoop = () => {
  update()
  draw()
  animationFrameId = requestAnimationFrame(gameLoop)
}

// ── Controls ───────────────────────────────────────────────────────────────

const keyDown = (e) => {
  if (e.key === 'ArrowLeft' || e.key.toLowerCase() === 'a') paddle.value.movingLeft = true
  if (e.key === 'ArrowRight' || e.key.toLowerCase() === 'd') paddle.value.movingRight = true
  if ((e.key === ' ' || e.key === 'Escape') && !gameOver.value) {
    paused.value = !paused.value
  }
}

const keyUp = (e) => {
  if (e.key === 'ArrowLeft' || e.key.toLowerCase() === 'a') paddle.value.movingLeft = false
  if (e.key === 'ArrowRight' || e.key.toLowerCase() === 'd') paddle.value.movingRight = false
}

onMounted(() => {
  updateScale()
  ctx = canvas.value.getContext('2d')
  window.addEventListener('keydown', keyDown)
  window.addEventListener('keyup', keyUp)
  window.addEventListener('resize', updateScale)
  initGame()
  gameLoop()
})

onUnmounted(() => {
  cancelAnimationFrame(animationFrameId)
  window.removeEventListener('keydown', keyDown)
  window.removeEventListener('keyup', keyUp)
  window.removeEventListener('resize', updateScale)
})
</script>

<template>
  <div class="screen">
    <div
      class="canvas-wrapper"
      :style="{
        width: '480px',
        height: '640px',
        transform: `scale(${scale})`,
        transformOrigin: 'center center',
      }"
    >
      <canvas
        ref="canvas"
        :width="480"
        :height="640"
        @pointerdown="onPointerDown"
        @pointermove="onPointerMove"
        @pointerup="onPointerUp"
        @pointercancel="onPointerUp"
        @pointerleave="onPointerUp"
      />

      <button
        v-if="showHudPause && !gameOver && !won"
        class="hud-pause-btn"
        type="button"
        @click="togglePause"
      >
        {{ paused ? 'RESUME' : 'PAUSE' }}
      </button>

      <!-- Game Over / Win overlay -->
      <Transition name="fade">
        <div v-if="gameOver" class="overlay">
          <div class="panel" :class="won ? 'panel--win' : 'panel--lose'">
            <div class="panel-icon">{{ won ? '🏆' : '💀' }}</div>
            <h1>{{ won ? 'YOU WIN!' : 'GAME OVER' }}</h1>
            <p class="final-score">{{ finalScore }} points</p>
            <p v-if="won && lives > 1" class="score-bonus">{{ score }} × {{ lives }} life bonus! 🎯</p>
            <div class="btn-group">
              <button class="btn" @click="initGame">New Game</button>
              <button class="btn btn--secondary" @click="goHome">Main Menu</button>
            </div>
          </div>
        </div>
      </Transition>

      <!-- Pause overlay -->
      <Transition name="fade">
        <div v-if="paused && !gameOver && !won" class="overlay">
          <div class="panel panel--pause">
            <div class="panel-icon">⏸️</div>
            <h1>PAUSED</h1>
            <p class="pause-hint">Press SPACE or ESC to continue</p>
            <div class="btn-group">
              <button class="btn" @click="resumeGame">Resume</button>
              <button class="btn btn--secondary" @click="initGame(); resumeGame()">New Game</button>
              <button class="btn btn--secondary" @click="goHome">Main Menu</button>
            </div>
          </div>
        </div>
      </Transition>
    </div>
  </div>
</template>

<style scoped src="./GameCanvas.css"></style>