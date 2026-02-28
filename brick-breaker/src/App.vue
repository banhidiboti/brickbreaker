<script setup>
import { ref } from 'vue'
import WelcomeScreen from './components/WelcomeScreen.vue'
import GameCanvas from './components/GameCanvas.vue'

const gameStarted = ref(false)
</script>

<template>
  <Transition name="switch" mode="out-in">
    <WelcomeScreen v-if="!gameStarted" @start="gameStarted = true" />
    <GameCanvas v-else />
  </Transition>
</template>

<style>
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  background: #000;
  overflow: hidden;
}

.switch-enter-active,
.switch-leave-active {
  transition: opacity 0.4s ease;
}
.switch-enter-from,
.switch-leave-to {
  opacity: 0;
}
</style>