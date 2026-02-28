<script setup>
import { ref } from 'vue'
import WelcomeScreen from './components/WelcomeScreen.vue'
import GameCanvas from './components/GameCanvas.vue'

const gameStarted = ref(false)
</script>

<template>
  <WelcomeScreen v-if="!gameStarted" @start="gameStarted = true" />
  <GameCanvas v-else @go-home="gameStarted = false" />
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