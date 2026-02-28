<template>
  <div class="screen">
    <div class="grid" />

    <div class="content">
      <div class="title-block">
        <h1 class="title">BRICK<span class="accent">BREAKER</span></h1>
        <p class="subtitle">retro arcade · neon edition</p>
      </div>

      <div class="how-to-play">
        <h2 class="section-title">HOW TO PLAY</h2>

        <ul class="instructions">
          <li>
            <span class="key">A</span>
            <span class="key">← Left</span>
            <span class="desc">Move paddle left</span>
          </li>
          <li>
            <span class="key">D</span>
            <span class="key">→ Right</span>
            <span class="desc">Move paddle right</span>
          </li>
          <li>
            <span class="key">SPACE</span>
            <span class="key">ESC</span>
            <span class="desc">Pause / Resume</span>
          </li>
        </ul>

        <div class="rules">
          <div class="rule">
            <span class="rule-icon">💥</span>
            <span>Break all bricks to win</span>
          </div>
          <div class="rule">
            <span class="rule-icon">♥</span>
            <span>You have 3 lives — don't let the ball fall!</span>
          </div>
          <div class="rule">
            <span class="rule-icon">🏆</span>
            <span>Each brick is worth 10 points</span>
          </div>
        </div>
      </div>

      <button class="start-btn" @click="$emit('start')">
        <span class="btn-text">START GAME</span>
        <span class="btn-glow" />
      </button>
    </div>
  </div>
</template>

<style scoped src="./WelcomeScreen.css"></style>