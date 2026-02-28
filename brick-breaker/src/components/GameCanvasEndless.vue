<script setup>
import GameCanvas from './GameCanvas.vue'

const emit = defineEmits(['go-home'])
</script>

<template>
  <GameCanvas mode="endless" v-on:goHome="emit('go-home')" />
</template>
