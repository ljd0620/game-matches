<template>
  <div class="container">
    <!-- 状态栏 -->
    <div class="top-panel">
      <div class="status-bar">
        <div class="score">得分：{{ score }}</div>
        <div class="timer" :class="{ low: timeLeft < 10 }">剩余时间：{{ timeLeft }} 秒</div>
      </div>
      <div class="time-bar-wrapper">
        <div
          class="time-bar"
          :class="{ low: timeLeft < 10 }"
          :style="{ width: (timeLeft / 60) * 100 + '%' }"
        />
      </div>
    </div>

    <!-- 游戏区域 -->
    <div class="game-area">
      <div class="board">
        <div v-for="(row, rowIndex) in board" :key="rowIndex" class="row">
          <div
            v-for="(cell, colIndex) in row"
            :key="colIndex"
            class="cell"
            :class="{ bomb: cell === BOMB, rainbow: cell === RAINBOW }"
            :style="{ backgroundColor: getCellColor(cell) }"
            @mousedown.passive="startDrag(rowIndex, colIndex, $event)"
            @mouseup.passive="endDrag(rowIndex, colIndex)"
            @touchstart.passive="startDrag(rowIndex, colIndex, $event)"
            @touchend.passive="endDrag(rowIndex, colIndex)"
          >
            {{ getCellSymbol(cell) }}
          </div>
        </div>
      </div>
    </div>

    <!-- combo 动画 -->
    <div v-if="comboText" class="combo-text">{{ comboText }}</div>

    <!-- 游戏结束提示 -->
    <div v-if="gameOver" class="game-over">游戏结束！</div>

    <!-- 重新开始按钮 -->
    <button v-if="gameOver" @click="restartGame">重新开始</button>
  </div>
</template>

<script setup>
import { ref, onMounted, nextTick } from 'vue'

/* -------------------------- 常量定义 -------------------------- */
const numRows = 6
const numCols = 6
const numColors = 5
const colors = ['#f44336', '#2196f3', '#4caf50', '#ff9800', '#9c27b0']
const BOMB = 'bomb'
const RAINBOW = 'rainbow'
const SPECIAL_CHANCE = 0.1

/* -------------------------- 状态变量 -------------------------- */
const board = ref([])
const matched = ref([])
const fallingMap = ref([])
const score = ref(0)
const timeLeft = ref(120)
const gameOver = ref(false)
const comboText = ref('')
let comboCount = 0

const dragging = ref(false)
const dragStart = ref(null)
const dragOffset = ref({ x: 0, y: 0 })
const touchStartPoint = ref(null)

/* -------------------------- 初始化 -------------------------- */
const initBoard = () => {
  do {
    // 生成随机棋盘
    board.value = Array.from({ length: numRows }, () =>
      Array.from({ length: numCols }, () => randomCell())
    );
    matched.value = Array.from({ length: numRows }, () => Array(numCols).fill(false));
    fallingMap.value = Array.from({ length: numRows }, () => Array(numCols).fill(false));
  } while (!findMatches()); // 如果没有匹配的方块，重新生成棋盘
};

const restartGame = () => {
  clearInterval(timer)
  score.value = 0
  timeLeft.value = 60
  gameOver.value = false
  comboText.value = ''
  initBoard()
  processBoard()
  startTimer()
}

/* -------------------------- 定时器 -------------------------- */
let timer = null
const startTimer = () => {
  timer = setInterval(() => {
    if (timeLeft.value > 0) timeLeft.value--
    else {
      clearInterval(timer)
      gameOver.value = true
    }
  }, 1000)
}

/* -------------------------- 消除逻辑 -------------------------- */
const isBomb = (cell) => cell === BOMB
const isRainbow = (cell) => cell === RAINBOW

const getAdjacentColor = (r, c) => {
  const dirs = [
    [-1, 0], [1, 0], [0, -1], [0, 1] // 上、下、左、右
  ];
  for (const [dr, dc] of dirs) {
    const nr = r + dr, nc = c + dc;
    if (nr >= 0 && nr < numRows && nc >= 0 && nc < numCols) {
      const cell = board.value[nr][nc];
      if (typeof cell === 'number') return cell; // 返回相邻的颜色
    }
  }
  return null; // 如果没有相邻颜色，返回 null
}

const clearAllOfColor = (color) => {
  for (let r = 0; r < numRows; r++) {
    for (let c = 0; c < numCols; c++) {
      if (board.value[r][c] === color) board.value[r][c] = null
    }
  }
}

const clearAdjacent = (r, c) => {
  for (let dr = -1; dr <= 1; dr++) {
    for (let dc = -1; dc <= 1; dc++) {
      const nr = r + dr, nc = c + dc;
      if (nr >= 0 && nr < numRows && nc >= 0 && nc < numCols) {
        board.value[nr][nc] = null; // 清除炸弹周围的方块
        matched.value[nr][nc] = true; // 标记为已匹配
      }
    }
  }
}

const clearMatches = () => {
  for (let r = 0; r < numRows; r++) {
    for (let c = 0; c < numCols; c++) {
      if (matched.value[r][c]) {
        const cell = board.value[r][c];
        if (isBomb(cell)) {
          clearAdjacent(r, c); // 清除炸弹周围的方块
        } else if (isRainbow(cell)) {
          const color = getAdjacentColor(r, c); // 获取彩虹方块周围的颜色
          if (color !== null) {
            clearAllOfColor(color); // 清除所有相同颜色的方块
          }
        }
        board.value[r][c] = null; // 清除当前方块
        score.value += 10; // 增加分数
      }
    }
  }
}

const collapseBoard = () => {
  for (let c = 0; c < numCols; c++) {
    let pointer = numRows - 1
    for (let r = numRows - 1; r >= 0; r--) {
      if (board.value[r][c] !== null) {
        board.value[pointer][c] = board.value[r][c]
        fallingMap.value[pointer][c] = false
        pointer--
      }
    }
    for (let r = pointer; r >= 0; r--) {
      board.value[r][c] = randomCell()
      fallingMap.value[r][c] = true
    }
  }
}

const findMatches = () => {
  matched.value = matched.value.map(row => row.map(() => false))
  let found = false
  for (let r = 0; r < numRows; r++) {
    for (let c = 0; c < numCols - 2; c++) {
      const val = board.value[r][c]
      if (val === board.value[r][c + 1] && val === board.value[r][c + 2]) {
        matched.value[r][c] = matched.value[r][c + 1] = matched.value[r][c + 2] = true
        found = true
      }
    }
  }
  for (let c = 0; c < numCols; c++) {
    for (let r = 0; r < numRows - 2; r++) {
      const val = board.value[r][c]
      if (val === board.value[r + 1][c] && val === board.value[r + 2][c]) {
        matched.value[r][c] = matched.value[r + 1][c] = matched.value[r + 2][c] = true
        found = true
      }
    }
  }
  return found
}

const processBoard = async () => {
  comboCount = 0;
  while (findMatches()) {
    comboCount++;
    comboText.value = `Combo x${comboCount}!`;
    await nextTick();
    clearMatches();
    await new Promise(res => setTimeout(res, 300));
    collapseBoard();
  }
  comboText.value = '';
}

const hasMatch = (grid) => {
  for (let r = 0; r < numRows; r++) {
    for (let c = 0; c < numCols - 2; c++) {
      if (grid[r][c] === grid[r][c + 1] && grid[r][c] === grid[r][c + 2]) return true;
    }
  }
  for (let c = 0; c < numCols; c++) {
    for (let r = 0; r < numRows - 2; r++) {
      if (grid[r][c] === grid[r + 1][c] && grid[r][c] === grid[r + 2][c]) return true;
    }
  }
  return false;
}

/* -------------------------- 拖拽逻辑 -------------------------- */
const isAdjacent = ([r1, c1], [r2, c2]) => Math.abs(r1 - r2) + Math.abs(c1 - c2) === 1

const trySwap = (pos1, pos2) => {
  const copy = board.value.map(row => row.slice())
  const temp = copy[pos1[0]][pos1[1]]
  copy[pos1[0]][pos1[1]] = copy[pos2[0]][pos2[1]]
  copy[pos2[0]][pos2[1]] = temp
  return copy
}

const startDrag = (r, c, event) => {
  if (gameOver.value) return;
  dragging.value = true;
  dragStart.value = [r, c];
  const point = event?.touches?.[0] || event;
  touchStartPoint.value = { x: point.clientX, y: point.clientY };
  window.addEventListener('mousemove', handleMouseMove);
  window.addEventListener('mouseup', resetDrag);
  window.addEventListener('touchmove', handleTouchMove, { passive: false }); // 添加触控移动事件
  window.addEventListener('touchend', resetDrag);
};

const handleTouchMove = (event) => {
  event.preventDefault(); // 禁用默认滚动行为
  if (!dragStart.value || !touchStartPoint.value) return;
  const touch = event.touches[0];
  const dx = touch.clientX - touchStartPoint.value.x;
  const dy = touch.clientY - touchStartPoint.value.y;
  const [r, c] = dragStart.value;
  if (Math.abs(dx) > 30) endDrag(r, dx > 0 ? c + 1 : c - 1);
  else if (Math.abs(dy) > 30) endDrag(dy > 0 ? r + 1 : r - 1, c);
};

const endDrag = (r, c) => {
  if (!dragStart.value || (dragStart.value[0] === r && dragStart.value[1] === c)) return;
  if (isAdjacent(dragStart.value, [r, c])) {
    const swapped = trySwap(dragStart.value, [r, c]);
    if (hasMatch(swapped)) {
      board.value = swapped; // 更新棋盘
      nextTick(() => processBoard()); // 处理消除逻辑
    }
  }
  resetDrag();
};

const resetDrag = () => {
  dragging.value = false;
  dragStart.value = null;
  dragOffset.value = { x: 0, y: 0 };
  touchStartPoint.value = null;
  window.removeEventListener('mousemove', handleMouseMove);
  window.removeEventListener('mouseup', resetDrag);
  window.removeEventListener('touchmove', handleTouchMove); // 移除触控移动事件
  window.removeEventListener('touchend', resetDrag);
};

const handleMouseMove = (event) => {
  if (!dragStart.value || !touchStartPoint.value) return
  const dx = event.clientX - touchStartPoint.value.x
  const dy = event.clientY - touchStartPoint.value.y
  const [r, c] = dragStart.value
  if (Math.abs(dx) > 30) endDrag(r, dx > 0 ? c + 1 : c - 1)
  else if (Math.abs(dy) > 30) endDrag(dy > 0 ? r + 1 : r - 1, c)
}

/* -------------------------- 辅助函数 -------------------------- */
const randomColor = () => Math.floor(Math.random() * numColors)
const randomCell = () => {
  const rnd = Math.random()
  if (rnd < SPECIAL_CHANCE / 2) return BOMB
  if (rnd < SPECIAL_CHANCE) return RAINBOW
  return randomColor()
}

const getCellColor = (cell) =>
  cell === BOMB ? '#555' : cell === RAINBOW ? '#ccc' : colors[cell]
const getCellSymbol = (cell) =>
  cell === BOMB ? '💣' : cell === RAINBOW ? '🌈' : ''

/* -------------------------- 挂载时初始化 -------------------------- */
onMounted(() => {
  initBoard()
  processBoard()
  startTimer()
})
</script>

<style scoped>
  html, body {
    touch-action: manipulation; /* 禁用双击缩放 */
  }

  .container {
    padding: 10px;
    max-width: 100%;
    box-sizing: border-box;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  }
  
  /* 状态栏样式优化 */
  .status-bar {
    display: flex;
    justify-content: space-between;
    align-items: center;
    background-color: #f8f9fa;
    padding: 8px 16px;
    border-radius: 10px;
    box-shadow: 0 2px 6px rgba(0, 0, 0, 0.08);
    margin-bottom: 10px;
    font-weight: 600;
    font-size: 16px;
    color: #333;
    transition: all 0.3s ease;
  }
  
  .status-bar > div {
    flex: 1;
    text-align: center;
  }
  
  .score {
    color: #2ecc71;
  }
  
  .timer {
    color: #3498db;
  }
  
  .timer.low {
    color: #e74c3c;
  }
  
  /* 倒计时条 */
  .time-bar-wrapper {
    width: 100%;
    height: 12px;
    background-color: #eee;
    border-radius: 6px;
    overflow: hidden;
    margin-bottom: 12px;
  }
  
  .time-bar {
    height: 100%;
    background: linear-gradient(to right, #4caf50, #f44336);
    transition: width 1s linear;
  }
  
  .time-bar.low {
    animation: blink 0.8s infinite alternate;
  }
  
  @keyframes blink {
    from { opacity: 1; }
    to { opacity: 0.3; }
  }
  
  /* 游戏结束提示 */
  .game-over {
    text-align: center;
    font-size: 28px; /* 增大字体 */
    color: #fff;
    font-weight: bold;
    margin: 20px 0;
    padding: 10px 20px;
    background: #e74c3c;
    border-radius: 10px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    animation: fade-in 0.8s ease; /* 添加淡入动画 */
  }
  
  @keyframes fade-in {
    from {
      opacity: 0;
      transform: scale(0.9);
    }
    to {
      opacity: 1;
      transform: scale(1);
    }
  }
  
  /* 棋盘样式 */
  .board {
    display: inline-block;
    margin-top: 10px;
    user-select: none;
    touch-action: manipulation;
  }
  
  /* 行 */
  .row {
    display: flex;
  }
  
  /* 格子样式 */
  .cell {
    width: 50px;
    height: 50px;
    margin: 2px;
    border-radius: 8px;
    cursor: pointer;
    transition: all 0.3s ease, box-shadow 0.2s ease;
    display: flex;
    justify-content: center;
    align-items: center;
    box-sizing: border-box;
    font-size: 24px;
    color: white;
    text-align: center;
    line-height: 50px;
    touch-action: manipulation;
    position: relative;
    box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1); /* 添加阴影 */
    will-change: transform, box-shadow; /* 提高动画性能 */
  }
  
  .cell:hover {
    transform: scale(1.1); /* 悬停时放大 */
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2); /* 悬停时阴影增强 */
  }
  
  .cell.exploding {
    animation: explode 0.4s ease;
    opacity: 0.6;
  }
  
  @keyframes explode {
    0% { transform: scale(1); opacity: 1; }
    50% { transform: scale(1.3); opacity: 0.6; }
    100% { transform: scale(0.8); opacity: 0.3; }
  }
  
  .cell.falling {
    animation: fall 0.3s ease;
  }
  
  @keyframes fall {
    from { transform: translateY(-50px); }
    to { transform: translateY(0); }
  }
  
  .cell.drag-target {
    box-shadow: 0 0 8px rgba(255, 255, 255, 0.8);
    z-index: 2;
  }

  .cell.rainbow {
    background: linear-gradient(45deg, red, orange, yellow, green, blue, indigo, violet);
    animation: rainbow-glow 1.5s infinite;
  }

  @keyframes rainbow-glow {
    0% { box-shadow: 0 0 5px red; }
    50% { box-shadow: 0 0 15px yellow; }
    100% { box-shadow: 0 0 5px blue; }
  }

  .cell.bomb {
    background-color: #555;
    animation: bomb-glow 1s infinite;
  }

  @keyframes bomb-glow {
    0% { box-shadow: 0 0 5px #ff0000; }
    50% { box-shadow: 0 0 15px #ff0000; }
    100% { box-shadow: 0 0 5px #ff0000; }
  }
  
  /* combo 提示文字 */
  .combo-text {
    position: absolute;
    top: 10%;
    left: 50%;
    transform: translateX(-50%);
    font-size: 28px;
    font-weight: bold;
    color: #ff9800;
    text-shadow: 1px 1px 2px rgba(0,0,0,0.2);
    animation: combo-bounce 0.8s ease;
    pointer-events: none;
  }
  
  @keyframes combo-bounce {
    0% { transform: translateX(-50%) scale(1); opacity: 0; }
    40% { transform: translateX(-50%) scale(1.2); opacity: 1; }
    100% { transform: translateX(-50%) scale(1); opacity: 0; }
  }
  
  /* 重新开始按钮 */
  button {
    display: block;
    margin: 16px auto;
    padding: 12px 24px; /* 增加按钮的点击区域 */
    font-size: 16px;
    font-weight: bold; /* 加粗字体 */
    background: linear-gradient(135deg, #2196f3, #42a5f5); /* 渐变背景 */
    color: #fff;
    border: none;
    border-radius: 8px; /* 更圆润的边角 */
    cursor: pointer;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1); /* 添加阴影 */
    transition: all 0.3s ease;
  }
  
  button:hover {
    background: linear-gradient(135deg, #1976d2, #2196f3); /* 悬停时的渐变效果 */
    transform: translateY(-2px); /* 悬停时的轻微上移 */
    box-shadow: 0 6px 8px rgba(0, 0, 0, 0.15); /* 悬停时的阴影增强 */
  }
  
  /* 响应式适配 */
  @media (max-width: 600px) {
    .cell {
      width: 40px;
      height: 40px;
      font-size: 20px;
      line-height: 40px;
    }
  
    .status-bar {
      flex-direction: column;
      gap: 8px; /* 增加间距 */
      font-size: 14px; /* 调整字体大小 */
    }

    .score,
    .timer {
      text-align: center;
    }
  }

  @media (pointer: coarse) {
    .cell {
      width: 60px; /* 增大触控设备上的格子大小 */
      height: 60px;
      font-size: 28px; /* 调整字体大小 */
      line-height: 60px;
    }
  
    button {
      padding: 16px 32px; /* 增大按钮的点击区域 */
      font-size: 18px;
    }
  }
  </style>
