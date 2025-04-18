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

    <!-- 游戏结束遮罩 -->
    <div v-if="gameOver" class="game-over-overlay">
      <div class="game-over-content">
        <div class="game-over">很遗憾！游戏结束！</div>
        <button @click="restartGame">重新开始</button>
      </div>
    </div>

    <!-- 分数排行榜 -->
    <div class="leaderboard">
      <h3>分数排行榜</h3>
      <ul>
        <li v-for="(entry, index) in leaderboard" :key="index">
          <span class="time">{{ entry.time }}</span>
          <span class="score">{{ entry.score }} 分</span>
        </li>
      </ul>
    </div>
  </div>
</template>

<script setup>
  import { ref, onMounted, nextTick } from 'vue'
  
  /* -------------------------- 常量定义 -------------------------- */
  const numRows = 6 // 棋盘行数
  const numCols = 6 // 棋盘列数
  const numColors = 5 // 方块颜色种类数
  const colors = ['#f44336', '#2196f3', '#4caf50', '#ff9800', '#9c27b0'] // 方块颜色
  const symbols = ['🐹', '🐱', '🐼', '🐶', '🐰', '🦊', '🐯', '🐨', '🐭', '🐻'] // 方块符号
  const BOMB = 'bomb' // 炸弹类型
  const RAINBOW = 'rainbow' // 彩虹类型
  const SPECIAL_CHANCE = 0.1 // 特殊方块生成概率
  
  /* -------------------------- 状态变量 -------------------------- */
  const board = ref([]) // 棋盘数据
  const matched = ref([]) // 匹配状态
  const fallingMap = ref([]) // 下落状态
  const score = ref(0) // 当前得分
  const timeLeft = ref(60) // 剩余时间
  const gameOver = ref(false) // 游戏结束状态
  const comboText = ref('') // 连击提示文本
  const leaderboard = ref([]) // 分数排行榜
  let comboCount = 0 // 连击计数
  
  const dragging = ref(false) // 是否正在拖拽
  const dragStart = ref(null) // 拖拽起始位置
  const dragOffset = ref({ x: 0, y: 0 }) // 拖拽偏移量
  const touchStartPoint = ref(null) // 触控起始点
  
  /* -------------------------- 初始化 -------------------------- */
  /**
   * 初始化棋盘，生成随机棋盘数据，确保有可匹配的方块
   */
  const initBoard = () => {
    do {
      board.value = Array.from({ length: numRows }, () =>
        Array.from({ length: numCols }, () => randomCell())
      )
      matched.value = Array.from({ length: numRows }, () => Array(numCols).fill(false))
      fallingMap.value = Array.from({ length: numRows }, () => Array(numCols).fill(false))
    } while (!findMatches()) // 如果没有匹配的方块，重新生成棋盘
  }
  
  /**
   * 重新开始游戏，重置状态并初始化棋盘
   */
  const restartGame = () => {
    clearInterval(timer)
    saveScoreToLeaderboard() // 保存分数到排行榜
    score.value = 0
    timeLeft.value = 60
    gameOver.value = false
    comboText.value = ''
    initBoard()
    startTimer()
    setTimeout(() => {
      processBoard()
    }, 500) // 延迟处理消除逻辑，确保棋盘渲染完成
  }
  
  /* -------------------------- 分数排行榜逻辑 -------------------------- */
  /**
   * 保存当前分数到排行榜，并存储到 localStorage
   */
  const saveScoreToLeaderboard = () => {
    const now = new Date()
    const formattedTime = `${String(now.getMonth() + 1).padStart(2, '0')}/${String(now.getDate()).padStart(2, '0')} ${String(now.getHours()).padStart(2, '0')}:${String(now.getMinutes()).padStart(2, '0')}`
    const newEntry = { time: formattedTime, score: score.value }
    const storedLeaderboard = JSON.parse(localStorage.getItem('leaderboard')) || []
    storedLeaderboard.push(newEntry)
    storedLeaderboard.sort((a, b) => b.score - a.score) // 按分数降序排序
    storedLeaderboard.splice(10) // 只保留前 10 名
    localStorage.setItem('leaderboard', JSON.stringify(storedLeaderboard))
    leaderboard.value = storedLeaderboard
  }
  
  /**
   * 加载排行榜数据，从 localStorage 获取
   */
  const loadLeaderboard = () => {
    leaderboard.value = JSON.parse(localStorage.getItem('leaderboard')) || []
  }
  
  /* -------------------------- 定时器 -------------------------- */
  let timer = null
  /**
   * 开始倒计时，时间结束时触发游戏结束
   */
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
  /**
   * 判断是否为炸弹方块
   * @param {any} cell 方块值
   * @returns {boolean}
   */
  const isBomb = (cell) => cell === BOMB
  
  /**
   * 判断是否为彩虹方块
   * @param {any} cell 方块值
   * @returns {boolean}
   */
  const isRainbow = (cell) => cell === RAINBOW
  
  /**
   * 获取相邻方块的颜色
   * @param {number} r 行号
   * @param {number} c 列号
   * @returns {number|null} 相邻颜色或 null
   */
  const getAdjacentColor = (r, c) => {
    const dirs = [
      [-1, 0], [1, 0], [0, -1], [0, 1] // 上、下、左、右
    ]
    for (const [dr, dc] of dirs) {
      const nr = r + dr, nc = c + dc
      if (nr >= 0 && nr < numRows && nc >= 0 && nc < numCols) {
        const cell = board.value[nr][nc]
        if (typeof cell === 'number') return cell // 返回相邻的颜色
      }
    }
    return null // 如果没有相邻颜色，返回 null
  }
  
  /**
   * 清除所有指定颜色的方块
   * @param {number} color 颜色值
   */
  const clearAllOfColor = (color) => {
    for (let r = 0; r < numRows; r++) {
      for (let c = 0; c < numCols; c++) {
        if (board.value[r][c] === color) board.value[r][c] = null
      }
    }
  }
  
  /**
   * 清除炸弹周围的方块
   * @param {number} r 行号
   * @param {number} c 列号
   */
  const clearAdjacent = (r, c) => {
    for (let dr = -1; dr <= 1; dr++) {
      for (let dc = -1; dc <= 1; dc++) {
        const nr = r + dr, nc = c + dc
        if (nr >= 0 && nr < numRows && nc >= 0 && nc < numCols) {
          board.value[nr][nc] = null // 清除炸弹周围的方块
          matched.value[nr][nc] = true // 标记为已匹配
        }
      }
    }
  }
  
  /**
   * 清除匹配的方块，并处理特殊方块逻辑
   */
  const clearMatches = () => {
    for (let r = 0; r < numRows; r++) {
      for (let c = 0; c < numCols; c++) {
        if (matched.value[r][c]) {
          const cell = board.value[r][c]
          if (isBomb(cell)) {
            clearAdjacent(r, c) // 清除炸弹周围的方块
          } else if (isRainbow(cell)) {
            const color = getAdjacentColor(r, c) // 获取彩虹方块周围的颜色
            if (color !== null) {
              clearAllOfColor(color) // 清除所有相同颜色的方块
            }
          }
          board.value[r][c] = null // 清除当前方块
          score.value += 10 // 增加分数
        }
      }
    }
  }
  
  /**
   * 让方块下落，填补空缺
   */
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
  
  /**
   * 查找棋盘中是否有匹配的方块
   * @returns {boolean} 是否有匹配
   */
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
  
  /**
   * 处理棋盘逻辑，包括匹配、清除和下落
   */
  const processBoard = async () => {
    comboCount = 0
    while (findMatches()) {
      comboCount++
      comboText.value = `Combo x${comboCount}!`
      await nextTick()
      clearMatches()
      await new Promise(res => setTimeout(res, 300))
      collapseBoard()
    }
    comboText.value = ''
    console.log('得分:', score.value)
  }
  
  /**
   * 检查棋盘是否有匹配的方块
   * @param {Array} grid 棋盘数据
   * @returns {boolean} 是否有匹配
   */
  const hasMatch = (grid) => {
    for (let r = 0; r < numRows; r++) {
      for (let c = 0; c < numCols - 2; c++) {
        if (grid[r][c] === grid[r][c + 1] && grid[r][c] === grid[r][c + 2]) return true
      }
    }
    for (let c = 0; c < numCols; c++) {
      for (let r = 0; r < numRows - 2; r++) {
        if (grid[r][c] === grid[r + 1][c] && grid[r][c] === grid[r + 2][c]) return true
      }
    }
    return false
  }
  
  /* -------------------------- 拖拽逻辑 -------------------------- */
  /**
   * 判断两个位置是否相邻
   * @param {Array} pos1 第一个位置 [行, 列]
   * @param {Array} pos2 第二个位置 [行, 列]
   * @returns {boolean} 是否相邻
   */
  const isAdjacent = ([r1, c1], [r2, c2]) => Math.abs(r1 - r2) + Math.abs(c1 - c2) === 1
  
  /**
   * 尝试交换两个位置的方块
   * @param {Array} pos1 第一个位置 [行, 列]
   * @param {Array} pos2 第二个位置 [行, 列]
   * @returns {Array} 交换后的棋盘数据
   */
  const trySwap = (pos1, pos2) => {
    const copy = board.value.map(row => row.slice())
    const temp = copy[pos1[0]][pos1[1]]
    copy[pos1[0]][pos1[1]] = copy[pos2[0]][pos2[1]]
    copy[pos2[0]][pos2[1]] = temp
    return copy
  }
  
  /**
   * 开始拖拽方块
   * @param {number} r 行号
   * @param {number} c 列号
   * @param {Event} event 拖拽事件
   */
  const startDrag = (r, c, event) => {
    if (gameOver.value) return
    dragging.value = true
    dragStart.value = [r, c]
    const point = event?.touches?.[0] || event
    touchStartPoint.value = { x: point.clientX, y: point.clientY }
    window.addEventListener('mousemove', handleMouseMove)
    window.addEventListener('mouseup', resetDrag)
    window.addEventListener('touchmove', handleTouchMove, { passive: false }) // 添加触控移动事件
    window.addEventListener('touchend', resetDrag)
  }
  
  /**
   * 处理触控移动事件
   * @param {Event} event 触控事件
   */
  const handleTouchMove = (event) => {
    event.preventDefault() // 禁用默认滚动行为
    if (!dragStart.value || !touchStartPoint.value) return
    const touch = event.touches[0]
    const dx = touch.clientX - touchStartPoint.value.x
    const dy = touch.clientY - touchStartPoint.value.y
    const [r, c] = dragStart.value
    if (Math.abs(dx) > 30) endDrag(r, dx > 0 ? c + 1 : c - 1)
    else if (Math.abs(dy) > 30) endDrag(dy > 0 ? r + 1 : r - 1, c)
  }
  
  /**
   * 结束拖拽方块
   * @param {number} r 行号
   * @param {number} c 列号
   */
  const endDrag = (r, c) => {
    if (!dragStart.value || (dragStart.value[0] === r && dragStart.value[1] === c)) return
    if (isAdjacent(dragStart.value, [r, c])) {
      const swapped = trySwap(dragStart.value, [r, c])
      if (hasMatch(swapped)) {
        board.value = swapped // 更新棋盘
        nextTick(() => processBoard()) // 处理消除逻辑
      }
    }
    resetDrag()
  }
  
  /**
   * 重置拖拽状态
   */
  const resetDrag = () => {
    dragging.value = false
    dragStart.value = null
    dragOffset.value = { x: 0, y: 0 }
    touchStartPoint.value = null
    window.removeEventListener('mousemove', handleMouseMove)
    window.removeEventListener('mouseup', resetDrag)
    window.removeEventListener('touchmove', handleTouchMove) // 移除触控移动事件
    window.removeEventListener('touchend', resetDrag)
  }
  
  /**
   * 处理鼠标移动事件
   * @param {Event} event 鼠标事件
   */
  const handleMouseMove = (event) => {
    if (!dragStart.value || !touchStartPoint.value) return
    const dx = event.clientX - touchStartPoint.value.x
    const dy = event.clientY - touchStartPoint.value.y
    const [r, c] = dragStart.value
    if (Math.abs(dx) > 30) endDrag(r, dx > 0 ? c + 1 : c - 1)
    else if (Math.abs(dy) > 30) endDrag(dy > 0 ? r + 1 : r - 1, c)
  }
  
  /* -------------------------- 辅助函数 -------------------------- */
  /**
   * 随机生成颜色索引
   * @returns {number} 颜色索引
   */
  const randomColor = () => Math.floor(Math.random() * numColors)
  
  /**
   * 随机生成方块类型（普通方块、炸弹或彩虹）
   * @returns {string|number} 方块类型
   */
  const randomCell = () => {
    const rnd = Math.random()
    if (rnd < SPECIAL_CHANCE / 2) return BOMB
    if (rnd < SPECIAL_CHANCE) return RAINBOW
    return randomColor()
  }
  
  /**
   * 获取方块的背景颜色
   * @param {string|number} cell 方块值
   * @returns {string} 背景颜色
   */
  const getCellColor = (cell) =>
    cell === BOMB ? '#555' : cell === RAINBOW ? '#ccc' : colors[cell]
  
  /**
   * 获取方块的符号
   * @param {string|number} cell 方块值
   * @returns {string} 方块符号
   */
  const getCellSymbol = (cell) =>
    cell === BOMB ? '💣' : cell === RAINBOW ? '🌈' : symbols[cell]
  
  /* -------------------------- 挂载时初始化 -------------------------- */
  /**
   * 组件挂载时初始化棋盘、定时器和排行榜
   */
  onMounted(() => {
    initBoard()
    startTimer()
    loadLeaderboard() // 加载排行榜
    setTimeout(() => {
      processBoard()
    }, 500) // 延迟处理消除逻辑，确保棋盘渲染完成
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

  /* 分数排行榜样式 */
  .leaderboard {
    min-height: 100px;
    margin-top: 20px;
    padding: 20px;
    background: linear-gradient(135deg, #f8f9fa, #e9ecef);
    border-radius: 16px;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
    transition: transform 0.3s ease, box-shadow 0.3s ease;
  }

  .leaderboard:hover {
    transform: translateY(-5px);
    box-shadow: 0 6px 12px rgba(0, 0, 0, 0.2);
  }

  .leaderboard h3 {
    margin-bottom: 16px;
    font-size: 24px;
    font-weight: bold;
    color: #333;
    text-align: center;
    text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.1);
  }

  .leaderboard ul {
    list-style: none;
    padding: 0;
    margin: 0;
  }

  .leaderboard li {
    font-size: 18px;
    padding: 10px 0;
    border-bottom: 1px solid #ddd;
    display: flex;
    justify-content: space-between;
    align-items: center;
    transition: background-color 0.3s ease;
  }

  .leaderboard li:last-child {
    border-bottom: none;
  }

  .leaderboard li:hover {
    background-color: rgba(0, 0, 0, 0.05);
  }

  .leaderboard li .time {
    font-weight: bold;
    color: #3498db;
  }

  .leaderboard li .score {
    font-weight: bold;
    color: #2ecc71;
  }
  
  /* 游戏结束遮罩层 */
  .game-over-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: linear-gradient(135deg, rgba(0, 0, 0, 0.8), rgba(50, 50, 50, 0.9)); /* 渐变背景 */
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000; /* 确保遮罩层在最上方 */
    animation: fadeIn 0.5s ease; /* 淡入动画 */
  }

  /* 游戏结束内容 */
  .game-over-content {
    text-align: center;
    color: #fff;
    background: rgba(255, 255, 255, 0.2); /* 半透明背景 */
    padding: 40px;
    border-radius: 16px;
    box-shadow: 0 8px 16px rgba(0, 0, 0, 0.3); /* 添加阴影 */
    animation: scaleUp 0.5s ease; /* 放大动画 */
  }

  .game-over {
    font-size: 32px;
    font-weight: bold;
    margin-bottom: 40px;
    text-shadow: 2px 2px 4px rgba(0, 0, 0, 0.5); /* 添加文字阴影 */
  }

  /* 按钮样式 */
  button {
    padding: 14px 28px;
    font-size: 20px;
    font-weight: bold;
    background: linear-gradient(135deg, #42a5f5, #2196f3);
    color: #fff;
    border: none;
    border-radius: 12px;
    cursor: pointer;
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
    transition: all 0.3s ease;
  }

  button:hover {
    background: linear-gradient(135deg, #1e88e5, #1976d2);
    transform: translateY(-3px);
    box-shadow: 0 6px 12px rgba(0, 0, 0, 0.3);
  }

  button:active {
    transform: translateY(0);
    box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
  }

  /* 动画效果 */
  @keyframes fadeIn {
    from {
      opacity: 0;
    }
    to {
      opacity: 1;
    }
  }

  @keyframes scaleUp {
    from {
      transform: scale(0.8);
      opacity: 0;
    }
    to {
      transform: scale(1);
      opacity: 1;
    }
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
