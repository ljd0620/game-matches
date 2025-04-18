<template>
  <div class="app">
    <h1>🎮 欢乐消消乐</h1>
    <GameBoard />
  </div>
</template>

<script setup>
import GameBoard from './components/GameBoard.vue'
</script>

<style scoped>
.app {
  text-align: center;
  margin-top: 40px;
  font-family: 'Segoe UI', sans-serif;
}
</style>