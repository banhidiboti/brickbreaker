<script setup>
import './WelcomeScreen.css'

const emit = defineEmits(['start'])
const startGame = () => emit('start')

const BRICK_SCORING = [
  { color: '#ff4466', label: 'Red',    points: 30 },
  { color: '#ff7733', label: 'Orange', points: 25 },
  { color: '#ffcc00', label: 'Yellow', points: 20 },
  { color: '#44ff88', label: 'Green',  points: 15 },
  { color: '#44ccff', label: 'Blue',   points: 10 },
  { color: '#bb66ff', label: 'Purple', points:  5 },
]
</script>

<template>
  <div class="welcome-screen">
    <div class="welcome-screen__grid" />

    <div class="welcome-screen__content">
      <div class="welcome-screen__title-block">
        <h1 class="welcome-screen__title">
          BRICK<span class="welcome-screen__title-accent">BREAKER</span>
        </h1>
        <p class="welcome-screen__subtitle">retro arcade · neon edition</p>
      </div>

      <div class="welcome-screen__how-to-play">
        <h2 class="welcome-screen__section-title">HOW TO PLAY</h2>

        <ul class="welcome-screen__instructions">
          <li>
            <span class="welcome-screen__key">A</span>
            <span class="welcome-screen__key">← Left</span>
            <span class="welcome-screen__desc">Move paddle left</span>
          </li>
          <li>
            <span class="welcome-screen__key">D</span>
            <span class="welcome-screen__key">→ Right</span>
            <span class="welcome-screen__desc">Move paddle right</span>
          </li>
          <li>
            <span class="welcome-screen__key">SPACE</span>
            <span class="welcome-screen__key">ESC</span>
            <span class="welcome-screen__desc">Pause / Resume</span>
          </li>
        </ul>

        <div class="welcome-screen__rules">
          <div class="welcome-screen__rule">
            <span class="welcome-screen__rule-icon">💥</span>
            <span>Break all bricks to win</span>
          </div>
          <div class="welcome-screen__rule">
            <span class="welcome-screen__rule-icon">♥</span>
            <span>You have 3 lives — don't let the ball fall!</span>
          </div>
          <div class="welcome-screen__rule welcome-screen__rule--scoring">
            <span class="welcome-screen__rule-icon">🏆</span>
            <div class="welcome-screen__scoring-grid">
              <div
                v-for="brick in BRICK_SCORING"
                :key="brick.label"
                class="welcome-screen__scoring-row"
              >
                <span
                  class="welcome-screen__brick-preview"
                  :style="{ background: brick.color, boxShadow: `0 0 6px ${brick.color}` }"
                />
                <span class="welcome-screen__brick-points">{{ brick.points }} pts</span>
              </div>
            </div>
          </div>
        </div>
      </div>

      <button class="welcome-screen__start-btn" @click="startGame">
        <span class="welcome-screen__btn-text">START GAME</span>
        <span class="welcome-screen__btn-glow" />
      </button>
    </div>
  </div>
</template>