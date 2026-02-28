<script setup>
import { ref, onMounted, onUnmounted, computed } from 'vue'

const canvas = ref(null)
const score = ref(0)
const lives = ref(3)
const gameOver = ref(false)
const paused = ref(false)
const won = ref(false)

let ctx = null
let animationFrameId = null

// Internal game resolution — all logic runs at these coordinates
const WIDTH = 480
const HEIGHT = 640

// Scale factor: fit the game into the window while keeping aspect ratio
const scale = ref(1)

const updateScale = () => {
  const scaleX = window.innerWidth / WIDTH
  const scaleY = window.innerHeight / HEIGHT
  scale.value = Math.min(scaleX, scaleY)
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
const brickOffsetLeft = 13

const bricks = ref([])

const makeBricks = () =>
  Array.from({ length: brickColumnCount }, () =>
    Array.from({ length: brickRowCount }, () => ({ status: 1 }))
  )

const initGame = () => {
  score.value = 0
  lives.value = 3
  gameOver.value = false
  paused.value = false
  won.value = false

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
  const grad = ctx.createLinearGradient(0, 0, 0, HEIGHT)
  grad.addColorStop(0, '#05051a')
  grad.addColorStop(1, '#0a0a2e')
  ctx.fillStyle = grad
  ctx.fillRect(0, 0, WIDTH, HEIGHT)

  // Subtle grid lines for depth effect
  ctx.strokeStyle = 'rgba(60,80,180,0.08)'
  ctx.lineWidth = 1
  for (let x = 0; x < WIDTH; x += 32) {
    ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, HEIGHT); ctx.stroke()
  }
  for (let y = 0; y < HEIGHT; y += 32) {
    ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(WIDTH, y); ctx.stroke()
  }
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

const drawBall = () => {
  const b = ball.value
  ctx.shadowColor = '#ffee44'
  ctx.shadowBlur = 24
  const grad = ctx.createRadialGradient(b.x - 2, b.y - 2, 1, b.x, b.y, b.radius)
  grad.addColorStop(0, '#fffbe0')
  grad.addColorStop(1, '#ffcc00')
  ctx.fillStyle = grad
  ctx.beginPath()
  ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2)
  ctx.fill()
  ctx.shadowBlur = 0
}

const BRICK_COLORS = [
  ['#ff4466', '#cc1133'],
  ['#ff7733', '#cc4400'],
  ['#ffcc00', '#cc9900'],
  ['#44ff88', '#00cc55'],
  ['#44ccff', '#0099cc'],
  ['#bb66ff', '#8833cc'],
]

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

      // Highlight shine on top of brick
      ctx.fillStyle = 'rgba(255,255,255,0.15)'
      ctx.beginPath()
      ctx.roundRect(bx + 3, by + 2, brickWidth - 6, 5, 2)
      ctx.fill()
      ctx.shadowBlur = 0
    }
  }
}

const draw = () => {
  drawBackground()
  drawBricks()
  drawPaddle()
  drawBall()

  // Draw HUD (score + lives)
  ctx.fillStyle = 'rgba(0,255,200,0.85)'
  ctx.font = 'bold 15px monospace'
  ctx.textAlign = 'left'
  ctx.fillText(`SCORE: ${score.value}`, 14, 24)
  ctx.textAlign = 'right'
  ctx.fillText(`LIVES: ${'♥ '.repeat(lives.value).trim()}`, WIDTH - 14, 24)

  if (paused.value && !gameOver.value && !won.value) {
    ctx.fillStyle = 'rgba(0,0,0,0.55)'
    ctx.fillRect(0, 0, WIDTH, HEIGHT)
    ctx.fillStyle = '#ffffff'
    ctx.font = 'bold 44px monospace'
    ctx.textAlign = 'center'
    ctx.fillText('PAUSED', WIDTH / 2, HEIGHT / 2 - 10)
    ctx.font = '16px monospace'
    ctx.fillStyle = '#aaa'
    ctx.fillText('Press SPACE or ESC to continue', WIDTH / 2, HEIGHT / 2 + 30)
  }
}

// ── Logic ──────────────────────────────────────────────────────────────────

const collisionDetection = () => {
  for (let c = 0; c < brickColumnCount; c++) {
    for (let r = 0; r < brickRowCount; r++) {
      const b = bricks.value[c][r]
      if (b.status !== 1) continue
      const bx = c * (brickWidth + brickPadding) + brickOffsetLeft
      const by = r * (brickHeight + brickPadding) + brickOffsetTop

      if (
        ball.value.x > bx &&
        ball.value.x < bx + brickWidth &&
        ball.value.y > by &&
        ball.value.y < by + brickHeight
      ) {
        ball.value.dy = -ball.value.dy
        b.status = 0
        score.value += 10

        const remaining = bricks.value.flat().filter(x => x.status === 1).length
        if (remaining === 0) {
          won.value = true
          gameOver.value = true
        }
      }
    }
  }
}

const update = () => {
  if (paused.value || gameOver.value) return

  ball.value.x += ball.value.dx
  ball.value.y += ball.value.dy

  // Wall collisions (left, right, top)
  if (ball.value.x + ball.value.radius > WIDTH || ball.value.x - ball.value.radius < 0) {
    ball.value.dx = -ball.value.dx
  }
  if (ball.value.y - ball.value.radius < 0) {
    ball.value.dy = Math.abs(ball.value.dy)
  }

  // Paddle collision — reflect ball upward and adjust angle based on hit position
  const p = paddle.value
  const bl = ball.value
  if (
    bl.y + bl.radius > p.y &&
    bl.y - bl.radius < p.y + p.height &&
    bl.x > p.x &&
    bl.x < p.x + p.width
  ) {
    bl.dy = -Math.abs(bl.dy)
    const hitPos = (bl.x - (p.x + p.width / 2)) / (p.width / 2)
    bl.dx = hitPos * 4
    // Clamp dx to prevent extreme angles
    bl.dx = Math.max(-5, Math.min(5, bl.dx))
  }

  // Ball fell off the bottom — lose a life
  if (ball.value.y + ball.value.radius > HEIGHT) {
    lives.value--
    if (lives.value <= 0) {
      gameOver.value = true
      won.value = false
    } else {
      ball.value.x = WIDTH / 2
      ball.value.y = HEIGHT - 100
      ball.value.dx = 3 * (Math.random() > 0.5 ? 1 : -1)
      ball.value.dy = -3
    }
  }

  // Move paddle based on held keys
  if (p.movingLeft && p.x > 0) p.x -= p.speed
  if (p.movingRight && p.x + p.width < WIDTH) p.x += p.speed

  collisionDetection()
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
  <!-- Fills the entire viewport -->
  <div class="screen">
    <!-- Fixed 480x640 wrapper, scaled up via CSS transform -->
    <div
      class="canvas-wrapper"
      :style="{
        width: '480px',
        height: '640px',
        transform: `scale(${scale})`,
        transformOrigin: 'center center',
      }"
    >
      <canvas ref="canvas" :width="480" :height="640" />

      <!-- Game Over / Win overlay -->
      <Transition name="fade">
        <div v-if="gameOver" class="overlay">
          <div class="panel" :class="won ? 'panel--win' : 'panel--lose'">
            <div class="panel-icon">{{ won ? '🏆' : '💀' }}</div>
            <h1>{{ won ? 'YOU WIN!' : 'GAME OVER' }}</h1>
            <p class="final-score">{{ score }} points</p>
            <button class="btn" @click="initGame">New Game</button>
          </div>
        </div>
      </Transition>
    </div>
  </div>
</template>

<style scoped>
/* Fill the entire viewport */
.screen {
  position: fixed;
  inset: 0;
  background: #000;
  display: flex;
  align-items: center;
  justify-content: center;
  overflow: hidden;
}

/* The canvas wrapper has the fixed game size; CSS scale handles the rest */
.canvas-wrapper {
  position: relative;
  border-radius: 12px;
  overflow: hidden;
  box-shadow:
    0 0 0 2px #1a1a4a,
    0 0 60px rgba(0, 120, 255, 0.35),
    0 0 120px rgba(0, 60, 180, 0.2);
}

canvas {
  display: block;
}

.overlay {
  position: absolute;
  inset: 0;
  background: rgba(0, 0, 10, 0.78);
  display: flex;
  align-items: center;
  justify-content: center;
  backdrop-filter: blur(4px);
}

.panel {
  background: rgba(8, 8, 30, 0.95);
  border-radius: 20px;
  padding: 48px 64px;
  text-align: center;
  border: 2px solid transparent;
  box-shadow: 0 0 40px rgba(0,0,0,0.6);
}

.panel--win {
  border-color: #ffcc00;
  box-shadow: 0 0 40px rgba(255, 200, 0, 0.3);
}

.panel--lose {
  border-color: #ff4466;
  box-shadow: 0 0 40px rgba(255, 60, 80, 0.3);
}

.panel-icon {
  font-size: 3.5rem;
  margin-bottom: 8px;
}

.panel h1 {
  margin: 0 0 12px;
  font-size: 2.4rem;
  font-family: monospace;
  letter-spacing: 4px;
  color: #fff;
}

.panel--win h1 { color: #ffcc00; }
.panel--lose h1 { color: #ff4466; }

.final-score {
  font-size: 1.8rem;
  font-family: monospace;
  color: #00ffcc;
  margin: 0 0 32px;
}

.btn {
  padding: 14px 44px;
  font-size: 1.1rem;
  font-family: monospace;
  letter-spacing: 2px;
  background: transparent;
  color: #fff;
  border: 2px solid #44aaff;
  border-radius: 10px;
  cursor: pointer;
  transition: all 0.2s;
}

.btn:hover {
  background: #44aaff22;
  box-shadow: 0 0 20px #44aaff66;
  transform: translateY(-2px);
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s;
}
.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}
</style>