<script setup>
import { ref } from 'vue'
import WelcomeScreen from './components/WelcomeScreen.vue'
import GameCanvas from './components/GameCanvas.vue'
import GameCanvasEndless from './components/GameCanvasEndless.vue'

const gameStarted = ref(false)
const gameMode = ref('classic')

const startGame = (mode = 'classic') => {
  gameMode.value = mode
  gameStarted.value = true
}
</script>

<template>
  <WelcomeScreen v-if="!gameStarted" @start="startGame" />
  <GameCanvas v-else-if="gameMode === 'classic'" key="classic" @go-home="gameStarted = false" />
  <GameCanvasEndless v-else key="endless" @go-home="gameStarted = false" />
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

</style>