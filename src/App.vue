<template>
  <div class="app">
    <h1>🎮萌宠消消乐</h1>
    <GameBoard :isAudioEnabled="isAudioEnabled" />
    <div class="audio-toggle" @click.stop="toggleAudio">
      {{ isAudioEnabled ? '🔊' : '🔇' }}
    </div>
  </div>
</template>

<script setup>
  import { ref } from 'vue'
  import GameBoard from './components/GameBoard.vue'
  
  const isAudioEnabled = ref(false); // 音效开关状态
  
  /**
   * 切换音效开关
   */
  const toggleAudio = () => {
    isAudioEnabled.value = !isAudioEnabled.value;
  };
</script>

<style scoped>
  .app {
    position: relative; /* 设置相对定位 */
    text-align: center;
    margin-top: 40px;
    font-family: 'Segoe UI', sans-serif;
    cursor: pointer; /* 鼓励用户点击 */
  }
  
  .audio-toggle {
    position: absolute; /* 绝对定位 */
    top: 10px; /* 距离顶部 20px */
    right: 20px; /* 距离右侧 20px */
    width: 40px; /* 图标宽度，适配移动设备 */
    height: 40px; /* 图标高度，适配移动设备 */
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 24px; /* 图标大小，适配移动设备 */
    color: #333; /* 图标颜色 */
    border-radius: 50%; /* 圆形按钮 */
    cursor: pointer;
    touch-action: manipulation; /* 优化触摸设备上的点击行为 */
    transition: transform 0.2s ease-in-out; /* 更快的缩放效果 */
  }

  .audio-toggle:hover {
    transform: scale(1.1); /* 悬停时放大 */
  }

  .audio-toggle:active {
    transform: scale(0.9); /* 点击时缩小 */
  }

  /* 响应式适配 */
  @media (max-width: 600px) {
    .audio-toggle {
      width: 50px; /* 增大按钮宽度 */
      height: 50px; /* 增大按钮高度 */
      font-size: 28px; /* 增大图标大小 */
      top: 0px; /* 调整顶部距离 */
      right: 10px; /* 调整右侧距离 */
    }
  }
</style>