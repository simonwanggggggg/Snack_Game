<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue'

// --- 配置區 ---
const GRID_SIZE = 20
const INITIAL_SPEED = 250 // 初始一格所需毫秒數
const SPEED_STEP = 5
const MIN_SPEED = 150
const TOTAL_CELLS = GRID_SIZE * GRID_SIZE

const DEBUG_VICTORY_LENGTH = 400
// ------------

// 基礎格子座標狀態
const snake = ref([
  { x: 10, y: 10, dir: 'UP' },
  { x: 10, y: 11, dir: 'UP' },
  { x: 10, y: 12, dir: 'UP' },
])

// 🟢 【流暢核心】上一次的格子座標，用來做兩點之間的平滑插值渲染
const prevSnake = ref([
  { x: 10, y: 10, dir: 'UP' },
  { x: 10, y: 11, dir: 'UP' },
  { x: 10, y: 12, dir: 'UP' },
])

const foods = ref([])
const bombs = ref([])
const applesEaten = ref(0)

const direction = ref({ x: 0, y: -1, dir: 'UP' })
const nextDirection = ref({ x: 0, y: -1, dir: 'UP' }) // 🟢 防止同一個 Tick 內連續按鍵導致自殺
const score = ref(0)
const currentSpeed = ref(INITIAL_SPEED)
const showRules = ref(false)
const gameState = ref('NOT_STARTED')

// 衝刺核心變數
const isBoosting = ref(false)

// 🟢 【流暢核心動畫時鐘】
let lastTickTime = 0
let accumulatedTime = 0
let animationFrameId = null
const renderProgress = ref(0) // 記錄當前 Tick 過去了百分之幾 (0.0 到 1.0)

// 一維快取陣列
const boardCache = ref(Array(TOTAL_CELLS).fill('EMPTY'))

const updateBoardCache = () => {
  const nextCache = Array(TOTAL_CELLS).fill('EMPTY')
  for (let i = 0; i < foods.value.length; i++) {
    nextCache[foods.value[i].y * GRID_SIZE + foods.value[i].x] = 'FOOD'
  }
  for (let i = 0; i < bombs.value.length; i++) {
    nextCache[bombs.value[i].y * GRID_SIZE + bombs.value[i].x] = 'BOMB'
  }
  boardCache.value = nextCache
}

const getTargetFoodCount = () => {
  const extraLength = snake.value.length - 3
  return Math.max(1, 5 - Math.floor(extraLength / 30))
}

const getTargetBombCount = () => {
  if (score.value >= 2500) return 0
  const extraLength = snake.value.length - 3
  return Math.max(1, 3 - Math.floor(extraLength / 60))
}

const calculateSpeed = () => {
  const level = Math.floor(snake.value.length / 10)
  return Math.max(INITIAL_SPEED - level * SPEED_STEP, MIN_SPEED)
}

watch(
  () => snake.value.length,
  () => {
    currentSpeed.value = calculateSpeed()
    const targetFoodCount = getTargetFoodCount()
    while (foods.value.length > targetFoodCount) foods.value.pop()
    const targetBombCount = getTargetBombCount()
    while (bombs.value.length > targetBombCount) bombs.value.pop()

    if (gameState.value === 'PLAYING') {
      generateFood()
      generateBombs()
    }
    updateBoardCache()
  },
  { flush: 'post' },
)

const isInFrogSafeZone = (x, y) => {
  const head = snake.value[0]
  return Math.abs(x - head.x) <= 2 && Math.abs(y - head.y) <= 2
}

const generateFood = () => {
  if (snake.value.length >= TOTAL_CELLS) return
  const targetCount = getTargetFoodCount()
  while (foods.value.length < targetCount) {
    const newFood = {
      x: Math.floor(Math.random() * GRID_SIZE),
      y: Math.floor(Math.random() * GRID_SIZE),
    }
    const idx = newFood.y * GRID_SIZE + newFood.x

    // 檢查格子是否被蛇身或炸彈佔據
    const isOccupiedBySnake = snake.value.some((s) => s.x === newFood.x && s.y === newFood.y)
    if (
      boardCache.value[idx] === 'EMPTY' &&
      !isOccupiedBySnake &&
      !isInFrogSafeZone(newFood.x, newFood.y)
    ) {
      foods.value.push(newFood)
      updateBoardCache()
    }
    if (snake.value.length + foods.value.length >= TOTAL_CELLS) break
  }
}

const generateBombs = () => {
  const targetCount = getTargetBombCount()
  while (bombs.value.length < targetCount) {
    const newBomb = {
      x: Math.floor(Math.random() * GRID_SIZE),
      y: Math.floor(Math.random() * GRID_SIZE),
    }
    const idx = newBomb.y * GRID_SIZE + newBomb.x

    const isOccupiedBySnake = snake.value.some((s) => s.x === newBomb.x && s.y === newBomb.y)
    if (
      boardCache.value[idx] === 'EMPTY' &&
      !isOccupiedBySnake &&
      !isInFrogSafeZone(newBomb.x, newBomb.y)
    ) {
      bombs.value.push(newBomb)
      updateBoardCache()
    }
    if (snake.value.length + foods.value.length + bombs.value.length >= TOTAL_CELLS) break
  }
}

const relocateOneBomb = () => {
  const targetBombCount = getTargetBombCount()
  while (bombs.value.length > targetBombCount) bombs.value.pop()
  if (bombs.value.length === 0) return
  const randomIndex = Math.floor(Math.random() * bombs.value.length)
  let attempts = 0
  while (attempts < 100) {
    const newX = Math.floor(Math.random() * GRID_SIZE)
    const newY = Math.floor(Math.random() * GRID_SIZE)
    const idx = newY * GRID_SIZE + newX

    const isOccupiedBySnake = snake.value.some((s) => s.x === newX && s.y === newY)
    if (boardCache.value[idx] === 'EMPTY' && !isOccupiedBySnake && !isInFrogSafeZone(newX, newY)) {
      bombs.value[randomIndex] = { x: newX, y: newY }
      break
    }
    attempts++
  }
}

const getBodyOpacity = (index) => {
  const totalLength = snake.value.length
  const distanceFromTail = totalLength - 1 - index
  if (distanceFromTail < 5) {
    return 0.2 + distanceFromTail * 0.16
  }
  return 1
}

// 🟢 核心遊戲邏輯：此處僅更新純資料座標
const gameTick = () => {
  // 鎖定目前前進方向
  direction.value = nextDirection.value

  // 複製當前狀態到「上一次座標歷史」，供畫面插值平滑使用
  prevSnake.value = snake.value.map((s) => ({ ...s }))

  const head = {
    x: snake.value[0].x + direction.value.x,
    y: snake.value[0].y + direction.value.y,
    dir: direction.value.dir,
  }

  // 判定 1：撞牆
  if (head.x < 0 || head.x >= GRID_SIZE || head.y < 0 || head.y >= GRID_SIZE) {
    triggerGameOver()
    return
  }

  const targetIdx = head.y * GRID_SIZE + head.x
  const targetType = boardCache.value[targetIdx]

  // 判定 2：自我碰撞 (排除最後一節，除非要吃東西)
  const isEating = targetType === 'FOOD'
  const checkLen = isEating ? snake.value.length : snake.value.length - 1
  for (let i = 1; i < checkLen; i++) {
    if (snake.value[i].x === head.x && snake.value[i].y === head.y) {
      triggerGameOver()
      return
    }
  }

  // 判定 3：踩到炸彈
  if (targetType === 'BOMB') {
    triggerGameOver()
    return
  }

  // 更新蛇身陣列資料
  snake.value.unshift(head)

  if (isEating) {
    score.value += 10
    const foodIndex = foods.value.findIndex((f) => f.x === head.x && f.y === head.y)
    if (foodIndex !== -1) foods.value.splice(foodIndex, 1)
    applesEaten.value++

    if (snake.value.length >= DEBUG_VICTORY_LENGTH || snake.value.length === TOTAL_CELLS) {
      triggerVictory()
      return
    }
    if (applesEaten.value >= 3) {
      relocateOneBomb()
      applesEaten.value = 0
    }
    generateFood()
    generateBombs()
  } else {
    snake.value.pop()
  }
}

// 🟢 【流暢核心】高精度硬體幀同步引擎 (60Hz ~ 144Hz+)
const loop = (timestamp) => {
  if (gameState.value !== 'PLAYING') return

  if (!lastTickTime) lastTickTime = timestamp
  const elapsed = timestamp - lastTickTime
  lastTickTime = timestamp

  // 衝刺速度限制計算
  const currentInterval = isBoosting.value
    ? Math.max(currentSpeed.value * 0.5, MIN_SPEED / 2.5)
    : currentSpeed.value

  accumulatedTime += elapsed

  // 如果累積時間大於一格的所需時間，執行邏輯 Tick
  while (accumulatedTime >= currentInterval) {
    gameTick()
    accumulatedTime -= currentInterval
    if (gameState.value !== 'PLAYING') return
  }

  // 計算兩格之間的平滑渲染進度百分比 (0.0 ~ 1.0)
  renderProgress.value = accumulatedTime / currentInterval

  animationFrameId = requestAnimationFrame(loop)
}

// 🟢 【流暢核心】動態計算每節身體跨格子移動的即時像素位移
const getSmoothStyle = (index) => {
  const current = snake.value[index]
  const prev = prevSnake.value[index] || current // 防止長度突變時邊界溢出

  // 處理空白鍵對調位置或重開遊戲時的座標突變（不進行插值）
  if (Math.abs(current.x - prev.x) > 1 || Math.abs(current.y - prev.y) > 1) {
    return {
      transform: `translate3d(${current.x * 100}%, ${current.y * 100}%, 0)`,
    }
  }

  // 線性插值公式：X_now = X_prev + (X_current - X_prev) * progress
  const smoothX = prev.x + (current.x - prev.x) * renderProgress.value
  const smoothY = prev.y + (current.y - prev.y) * renderProgress.value

  return {
    transform: `translate3d(${smoothX * 100}%, ${smoothY * 100}%, 0)`,
  }
}

const reverseSnake = () => {
  if (snake.value.length < 2 || gameState.value !== 'PLAYING') return
  const reversed = [...snake.value].reverse()
  for (let i = 0; i < reversed.length - 1; i++) {
    const current = reversed[i]
    const next = reversed[i + 1]
    if (next.x > current.x) current.dir = 'LEFT'
    else if (next.x < current.x) current.dir = 'RIGHT'
    else if (next.y > current.y) current.dir = 'UP'
    else if (next.y < current.y) current.dir = 'DOWN'
  }
  reversed[reversed.length - 1].dir = reversed[reversed.length - 2].dir

  const newHead = reversed[0]
  const nextPart = reversed[1]
  const newDx = newHead.x - nextPart.x
  const newDy = newHead.y - nextPart.y

  let newDirStr = 'UP'
  if (newDx === 1) newDirStr = 'RIGHT'
  else if (newDx === -1) newDirStr = 'LEFT'
  else if (newDy === 1) newDirStr = 'DOWN'

  direction.value = { x: newDx, y: newDy, dir: newDirStr }
  nextDirection.value = { x: newDx, y: newDy, dir: newDirStr }

  // 翻轉時同步更新 prev 歷史紀錄，防止鏡像拉伸抖動
  snake.value = reversed
  prevSnake.value = reversed.map((s) => ({ ...s }))
  accumulatedTime = 0 // 重置時鐘確保過渡流暢
}

const togglePause = () => {
  if (gameState.value === 'PLAYING') {
    gameState.value = 'PAUSED'
    isBoosting.value = false
    cancelAnimationFrame(animationFrameId)
  } else if (gameState.value === 'PAUSED') {
    gameState.value = 'PLAYING'
    lastTickTime = 0
    accumulatedTime = 0
    animationFrameId = requestAnimationFrame(loop)
  }
}

const handleKey = (e) => {
  if (showRules.value) {
    if (e.key === 'Escape') {
      e.preventDefault()
      showRules.value = false
    }
    return
  }
  const triggerKeys = ['Space', 'Enter']
  if (
    gameState.value === 'NOT_STARTED' ||
    gameState.value === 'GAME_OVER' ||
    gameState.value === 'VICTORY'
  ) {
    if (triggerKeys.includes(e.code) || triggerKeys.includes(e.key)) {
      e.preventDefault()
      startGame()
    }
    return
  }
  if (gameState.value === 'PAUSED') {
    if (triggerKeys.includes(e.code) || triggerKeys.includes(e.key)) {
      e.preventDefault()
      togglePause()
      return
    }
    const directionKeys = ['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight']
    if (directionKeys.includes(e.key)) {
      e.preventDefault()
      togglePause()
    }
  }
  if (e.key === 'Enter') {
    e.preventDefault()
    togglePause()
    return
  }
  if (e.code === 'Space') {
    e.preventDefault()
    reverseSnake()
    return
  }

  const keys = {
    ArrowUp: { x: 0, y: -1, dir: 'UP' },
    ArrowDown: { x: 0, y: 1, dir: 'DOWN' },
    ArrowLeft: { x: -1, y: 0, dir: 'LEFT' },
    ArrowRight: { x: 1, y: 0, dir: 'RIGHT' },
  }
  const target = keys[e.key]

  if (target) {
    // 防止原地回頭自殺判定的優化 (改用 direction 判斷非 nextDirection)
    if (target.x !== -direction.value.x || target.y !== -direction.value.y) {
      if (target.dir === direction.value.dir) {
        isBoosting.value = true
      } else {
        nextDirection.value = target
        isBoosting.value = false
      }
    }
  }
}

const handleKeyUp = (e) => {
  if (gameState.value !== 'PLAYING') return
  if (
    (e.key === 'ArrowUp' && nextDirection.value.dir === 'UP') ||
    (e.key === 'ArrowDown' && nextDirection.value.dir === 'DOWN') ||
    (e.key === 'ArrowLeft' && nextDirection.value.dir === 'LEFT') ||
    (e.key === 'ArrowRight' && nextDirection.value.dir === 'RIGHT')
  ) {
    isBoosting.value = false
  }
}

const startGame = () => {
  snake.value = [
    { x: 10, y: 10, dir: 'UP' },
    { x: 10, y: 11, dir: 'UP' },
    { x: 10, y: 12, dir: 'UP' },
  ]
  prevSnake.value = [
    { x: 10, y: 10, dir: 'UP' },
    { x: 10, y: 11, dir: 'UP' },
    { x: 10, y: 12, dir: 'UP' },
  ]
  direction.value = { x: 0, y: -1, dir: 'UP' }
  nextDirection.value = { x: 0, y: -1, dir: 'UP' }
  score.value = 0
  applesEaten.value = 0
  currentSpeed.value = INITIAL_SPEED
  isBoosting.value = false
  foods.value = []
  bombs.value = []

  updateBoardCache()
  generateFood()
  generateBombs()

  gameState.value = 'PLAYING'
  lastTickTime = 0
  accumulatedTime = 0
  renderProgress.value = 0

  if (animationFrameId) cancelAnimationFrame(animationFrameId)
  animationFrameId = requestAnimationFrame(loop)
}

const triggerGameOver = () => {
  stopGame()
  gameState.value = 'GAME_OVER'
}
const triggerVictory = () => {
  stopGame()
  gameState.value = 'VICTORY'
}

const stopGame = () => {
  isBoosting.value = false
  if (animationFrameId) {
    cancelAnimationFrame(animationFrameId)
    animationFrameId = null
  }
}

onMounted(() => {
  window.addEventListener('keydown', handleKey)
  window.addEventListener('keyup', handleKeyUp)
  updateBoardCache()
})
onUnmounted(() => {
  window.removeEventListener('keydown', handleKey)
  window.removeEventListener('keyup', handleKeyUp)
  stopGame()
})
</script>

<template>
  <div class="main-wrapper">
    <div class="hud-top-bar">
      <div class="screen-title-box">
        <h1>NEON SNAKE</h1>
      </div>
      <div class="screen-score-box neon-score-box">
        <div class="score-label">SCORE</div>
        <div class="score-value">{{ score }}</div>
      </div>
    </div>

    <div class="game-container">
      <div class="game-board neon-black-board" :class="{ 'neon-black-board-boosting': isBoosting }">
        <div v-for="(type, idx) in boardCache" :key="'grid-' + idx" class="cell">
          <div v-if="type === 'FOOD'" class="food-emoji">🍎</div>
          <div v-else-if="type === 'BOMB'" class="bomb-emoji">💣</div>
        </div>

        <div
          v-for="(s, index) in snake"
          :key="'snake-' + index"
          class="smooth-snake-part"
          :style="getSmoothStyle(index)"
        >
          <div v-if="index === 0" class="frog-head" :class="'frog-dir-' + s.dir.toLowerCase()">
            🐸
          </div>
          <div
            v-else
            class="snake-body-circle"
            :class="{ 'snake-body-boosting': isBoosting }"
            :style="{ opacity: getBodyOpacity(index) }"
          ></div>
        </div>

        <div
          v-if="gameState === 'NOT_STARTED' || gameState === 'PAUSED'"
          class="game-overlay-panel neon-overlay"
        >
          <div class="overlay-title neon-text-green">NEON SNAKE</div>
          <div class="btn-group-vertical">
            <button class="start-btn neon-btn-green" @click="startGame">
              {{ gameState === 'PAUSED' ? 'RESUME GAME' : 'START GAME' }}
            </button>
            <button class="rules-btn neon-btn-orange" @click="showRules = true">HOW TO PLAY</button>
          </div>
          <div class="pause-hint-text neon-hint">
            PRESS <span class="key-highlight-green">SPACE</span> OR
            <span class="key-highlight-green">ENTER</span> TO PLAY
          </div>
        </div>

        <div v-else-if="gameState === 'GAME_OVER'" class="game-overlay-panel game-over-bg">
          <div class="overlay-title game-over-title">💥 GAME OVER 💥</div>

          <div class="final-score-box neon-final-score-box">
            <div class="final-label">FINAL SCORE</div>
            <div class="final-value">{{ score }}</div>
          </div>

          <div class="btn-group-vertical">
            <button class="start-btn restart-theme" @click="startGame">TRY AGAIN</button>
            <button class="rules-btn neon-btn-white" @click="showRules = true">HOW TO PLAY</button>
          </div>
          <div class="pause-hint-text">
            [ PRESS <span class="key-highlight">SPACE</span> OR
            <span class="key-highlight">ENTER</span> TO RESTART ]
          </div>
        </div>

        <div v-else-if="gameState === 'VICTORY'" class="game-overlay-panel victory-neon-bg">
          <div class="overlay-title victory-neon-title">👑 VICTORY 👑</div>

          <div class="final-score-box neon-victory-score-box">
            <div class="final-label">HIGH SCORE</div>
            <div class="final-value">{{ score }}</div>
          </div>

          <div class="btn-group-vertical">
            <button class="start-btn neon-btn-gold" @click="startGame">PLAY AGAIN</button>
            <button class="rules-btn neon-btn-white" @click="showRules = true">HOW TO PLAY</button>
          </div>
          <div class="pause-hint-text victory-hint">
            [ PRESS <span class="key-highlight-gold">SPACE</span> OR
            <span class="key-highlight-gold">ENTER</span> TO RESTART ]
          </div>
        </div>
      </div>
    </div>

    <div v-if="showRules" class="modal-overlay" @click.self="showRules = false">
      <div class="modal-content">
        <h2>🎮 遊戲玩法說明 🎮</h2>
        <hr />
        <ul class="rules-list">
          <li>
            <strong>開始🎮：</strong> 按 <strong>Space</strong> 或
            <strong>Enter</strong> 即可開始遊戲！
          </li>
          <li>
            <strong>暫停🛑：</strong> 遊戲進行中，隨時按下<strong>Enter </strong>即可暫停遊戲。
          </li>
          <li><strong>移動🏃：</strong> 方向鍵控制蛇前進的方向。</li>
          <li><strong>衝刺⚡：</strong>長按或連點目前移動方向鍵會加速，放開即恢復。</li>
          <li>
            <strong>技能🔮：</strong>遊戲進行中按下<strong>Space </strong
            >可讓蛇頭/尾<strong>對調位置</strong>並逆向移動。
          </li>
          <li><strong>蘋果🍎：</strong>得分 +10，身體長度增加 1 節。</li>
          <li><strong>難度🚨：</strong> 一開始場上會有 5 顆蘋果，隨著長度減少。最後剩 1 顆。</li>
          <li>
            <strong>炸彈💣：</strong> 一開始場上會有 3 顆炸彈，隨著長度減少。遊戲過程會重新定位。
          </li>
          <li><strong>加速⏭️：</strong> 蛇的長度每增加 10 節，移動速度就會自動加快一次。</li>
          <li><strong>失敗💥：</strong> 踩到炸彈💣、撞到牆壁，或者蛇頭撞到自己的身體。</li>
          <li><strong>通關🏆：</strong> 當蛇身填滿整個遊戲畫面即通關！</li>
        </ul>
        <button class="close-btn" @click="showRules = false">我知道了</button>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* ==========================================
   全域自適應高度計算
   ========================================== */
:root,
.main-wrapper {
  --board-total-size: 76vh;
  --custom-cell-size: calc(var(--board-total-size) / 20);
}

.main-wrapper {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  min-height: 100vh;
  background-color: #0a0a0c;
  background-image:
    linear-gradient(rgba(57, 255, 20, 0.03) 1px, transparent 1px),
    linear-gradient(90deg, rgba(57, 255, 20, 0.03) 1px, transparent 1px);
  background-size: 40px 40px;
  background-position: center;
  color: #ffffff;
  font-family: 'Space Grotesk', 'Segoe UI', sans-serif;
  position: relative;
  overflow: hidden;
}

.hud-top-bar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  width: var(--board-total-size);
  margin-bottom: 25px;
  padding: 0 10px;
  box-sizing: border-box;
}

.screen-title-box h1 {
  margin: 0;
  color: #39ff14;
  letter-spacing: 4px;
  font-size: 2.4rem;
  font-family: 'Orbitron', sans-serif;
  font-weight: 900;
  text-shadow: 0 0 12px rgba(57, 255, 20, 0.6);
}

.screen-score-box {
  background-color: #000000;
  border: 2px solid #39ff14;
  border-radius: 6px;
  padding: 6px 22px;
  text-align: center;
  min-width: 100px;
  box-shadow: 0 0 15px rgba(57, 255, 20, 0.3);
  animation: score-box-pulse 3s infinite ease-in-out;
}
.score-label {
  font-size: 0.75rem;
  font-weight: 900;
  color: #39ff14;
  opacity: 0.8;
  letter-spacing: 2px;
  margin-bottom: 2px;
  font-family: 'Orbitron', sans-serif;
  text-shadow: 0 0 4px rgba(57, 255, 20, 0.5);
}
.score-value {
  font-size: 1.8rem;
  font-weight: bold;
  color: #39ff14;
  font-family: 'Share Tech Mono', monospace;
  text-shadow: 0 0 8px #39ff14;
}

@keyframes score-box-pulse {
  0%,
  100% {
    box-shadow: 0 0 12px rgba(57, 255, 20, 0.2);
    border-color: rgba(57, 255, 20, 0.7);
  }
  50% {
    box-shadow: 0 0 20px rgba(57, 255, 20, 0.5);
    border-color: rgba(57, 255, 20, 1);
  }
}

.game-container {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.game-board {
  display: grid;
  grid-template-columns: repeat(20, var(--custom-cell-size));
  grid-template-rows: repeat(20, var(--custom-cell-size));
  width: var(--board-total-size);
  height: var(--board-total-size);
  border: none;
  position: relative;
}

.neon-black-board {
  background-color: #000000;
  border: 2px solid rgba(57, 255, 20, 0.3);
  box-shadow: 0 0 30px rgba(57, 255, 20, 0.15);
  transition:
    box-shadow 0.2s ease,
    border-color 0.2s ease;
}

.neon-black-board-boosting {
  border-color: rgba(57, 255, 20, 0.8);
  box-shadow:
    0 0 40px rgba(57, 255, 20, 0.5),
    inset 0 0 20px rgba(57, 255, 20, 0.15);
}

.cell {
  width: var(--custom-cell-size);
  height: var(--custom-cell-size);
  display: flex;
  justify-content: center;
  align-items: center;
  position: relative;
}

/* 🟢 新增：平滑移動蛇身的絕對定位圖層，使其完全不受 grid 網格死板限制 */
.smooth-snake-part {
  position: absolute;
  top: 0;
  left: 0;
  width: var(--custom-cell-size);
  height: var(--custom-cell-size);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 100;
  /* 核心技巧：利用 GPU 硬件加速的 translate3d 實現像素級無損移動 */
  will-change: transform;
}

/* 基礎面板配置 */
.game-overlay-panel {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 200;
  color: white;
  text-align: center;
}
.game-over-bg {
  background-color: rgba(5, 5, 5, 0.94);
  border: 2px solid #ff5252;
  box-shadow: inset 0 0 30px rgba(255, 82, 82, 0.15);
}
.neon-overlay {
  background-color: rgba(5, 5, 5, 0.9);
  backdrop-filter: blur(4px);
  border: 2px solid #39ff14;
  box-shadow: inset 0 0 30px rgba(57, 255, 20, 0.15);
}

.victory-neon-bg {
  background-color: rgba(5, 5, 5, 0.95);
  backdrop-filter: blur(4px);
  border: 2px solid #ffd700;
  box-shadow:
    inset 0 0 35px rgba(255, 215, 0, 0.2),
    0 0 25px rgba(255, 215, 0, 0.1);
}

.neon-text-green {
  color: #39ff14 !important;
  font-family: 'Orbitron', monospace;
  font-size: calc(var(--custom-cell-size) * 1.5) !important;
  letter-spacing: 4px;
  text-shadow:
    0 0 10px #39ff14,
    0 0 25px rgba(57, 255, 20, 0.5);
  animation: text-pulse 2s infinite ease-in-out;
}

@keyframes text-pulse {
  0%,
  100% {
    opacity: 1;
    text-shadow:
      0 0 10px #39ff14,
      0 0 25px rgba(57, 255, 20, 0.5);
  }
  50% {
    opacity: 0.85;
    text-shadow:
      0 0 6px #39ff14,
      0 0 15px rgba(57, 255, 20, 0.3);
  }
}

.victory-neon-title {
  color: #ffd700 !important;
  font-size: calc(var(--custom-cell-size) * 1.4);
  font-weight: bold;
  letter-spacing: 3px;
  font-family: 'Orbitron', sans-serif;
  text-shadow:
    0 0 12px #ffd700,
    0 0 25px rgba(255, 215, 0, 0.6);
  animation: victory-neon-pulse 1.8s infinite alternate;
  margin-bottom: 25px;
}

@keyframes victory-neon-pulse {
  0% {
    transform: scale(1);
    text-shadow:
      0 0 10px #ffd700,
      0 0 20px rgba(255, 215, 0, 0.5);
  }
  100% {
    transform: scale(1.04);
    text-shadow:
      0 0 16px #ffd700,
      0 0 35px rgba(255, 215, 0, 0.8),
      0 0 4px #ffffff;
  }
}

.overlay-title {
  font-size: calc(var(--custom-cell-size) * 1.2);
  font-weight: bold;
  letter-spacing: 1px;
  color: #81c784;
  margin-bottom: 25px;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.5);
}
.game-over-title {
  color: #ff5252;
  font-family: 'Orbitron', sans-serif;
  font-size: calc(var(--custom-cell-size) * 1.4);
  text-shadow: 0 0 10px #ff5252;
}

.neon-final-score-box {
  background-color: #000000 !important;
  border: 2px solid #ff5252 !important;
  border-radius: 6px;
  padding: 10px calc(var(--custom-cell-size) * 1.8);
  margin-bottom: 25px;
  text-align: center;
  box-shadow: 0 0 20px rgba(255, 82, 82, 0.4);
}
.neon-final-score-box .final-label {
  font-size: calc(var(--custom-cell-size) * 0.45);
  color: #ff8a8a;
  font-weight: bold;
  letter-spacing: 2px;
  margin-bottom: 4px;
}
.neon-final-score-box .final-value {
  font-size: calc(var(--custom-cell-size) * 1.5);
  font-weight: 900;
  color: #ffd54f;
  font-family: 'Share Tech Mono', monospace;
  line-height: 1;
  text-shadow:
    0 0 12px #ffd54f,
    0 0 4px #ff5252;
}

.neon-victory-score-box {
  background-color: #000000 !important;
  border: 2px solid #ffd700 !important;
  border-radius: 6px;
  padding: 10px calc(var(--custom-cell-size) * 1.8);
  margin-bottom: 25px;
  text-align: center;
  box-shadow: 0 0 20px rgba(255, 215, 0, 0.3);
}
.neon-victory-score-box .final-label {
  font-size: calc(var(--custom-cell-size) * 0.45);
  color: #ffeaa7;
  font-weight: bold;
  letter-spacing: 2px;
  margin-bottom: 4px;
}
.neon-victory-score-box .final-value {
  font-size: calc(var(--custom-cell-size) * 1.5);
  font-weight: 900;
  color: #ffd700;
  font-family: 'Share Tech Mono', monospace;
  line-height: 1;
  text-shadow:
    0 0 12px #ffd700,
    0 0 4px #ffd700;
}

.btn-group-vertical {
  display: flex;
  flex-direction: column;
  gap: 14px;
  width: calc(var(--custom-cell-size) * 8);
}

/* 按鈕樣式區 */
.start-btn,
.rules-btn,
.restart-theme {
  padding: calc(var(--custom-cell-size) * 0.38) 0;
  font-size: calc(var(--custom-cell-size) * 0.58);
  font-family: 'Orbitron', monospace;
  font-weight: 900;
  cursor: pointer;
  background-color: #050505;
  border-radius: 4px;
  transition: all 0.15s ease-in-out;
  box-shadow: none;
  letter-spacing: 1px;
}

.neon-btn-green {
  color: #39ff14;
  border: 2px solid #39ff14;
  box-shadow: 0 0 8px rgba(57, 255, 20, 0.2);
}
.neon-btn-green:hover {
  background-color: #39ff14;
  color: #000000;
  box-shadow: 0 0 18px #39ff14;
}
.neon-btn-green:active {
  transform: scale(0.96);
}

.neon-btn-orange {
  color: #ff9f00;
  border: 2px solid #ff9f00;
  box-shadow: 0 0 8px rgba(255, 159, 0, 0.2);
}
.neon-btn-orange:hover {
  background-color: #ff9f00;
  color: #000000;
  box-shadow: 0 0 18px #ff9f00;
}
.neon-btn-orange:active {
  transform: scale(0.96);
}

.neon-btn-white {
  color: #ffffff;
  border: 2px solid #ffffff;
  box-shadow: 0 0 8px rgba(255, 255, 255, 0.2);
}
.neon-btn-white:hover {
  background-color: #ffffff;
  color: #000000;
  box-shadow: 0 0 18px #ffffff;
}
.neon-btn-white:active {
  transform: scale(0.96);
}

.neon-btn-gold {
  color: #ffd700;
  border: 2px solid #ffd700;
  box-shadow: 0 0 8px rgba(255, 215, 0, 0.2);
}
.neon-btn-gold:hover {
  background-color: #ffd700;
  color: #000000;
  box-shadow: 0 0 20px #ffd700;
}
.neon-btn-gold:active {
  transform: scale(0.96);
}

.restart-theme {
  color: #ff5252;
  border: 2px solid #ff5252;
  box-shadow: 0 0 8px rgba(255, 82, 82, 0.2);
}
.restart-theme:hover {
  background-color: #ff5252;
  color: white;
  box-shadow: 0 0 18px #ff5252;
}

.pause-hint-text {
  margin-top: 25px;
  font-size: calc(var(--custom-cell-size) * 0.42);
  color: #666;
  font-family: 'Share Tech Mono', monospace;
}
.neon-hint {
  color: #888;
  letter-spacing: 1px;
}
.victory-hint {
  color: #888;
}
.key-highlight-green {
  color: #39ff14;
  font-weight: bold;
  text-shadow: 0 0 5px rgba(57, 255, 20, 0.5);
}
.key-highlight {
  color: #ff5252;
  font-weight: bold;
}
.key-highlight-gold {
  color: #ffd700;
  font-weight: bold;
  text-shadow: 0 0 5px rgba(255, 215, 0, 0.4);
}

/* ==========================================
   蛇身與青蛙頭
   ========================================== */
.snake-body-circle {
  width: 90%; /* 🟢 縮小一點點點（112% -> 90%），在滑動時能拉出完美的流線感與關節層次 */
  height: 90%;
  background-color: #39ff14;
  border-radius: 50%;
  box-shadow:
    0 0 8px #39ff14,
    0 0 16px rgba(57, 255, 20, 0.4);
}

.snake-body-boosting {
  box-shadow:
    0 0 12px #39ff14,
    0 0 22px #39ff14 !important;
}

.frog-head {
  font-size: calc(var(--custom-cell-size) * 0.8);
  line-height: 1;
  user-select: none;
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100%;
  position: relative;
  filter: drop-shadow(0 0 6px #39ff14) drop-shadow(0 0 12px rgba(57, 255, 20, 0.6));
}

.neon-black-board-boosting .frog-head {
  filter: drop-shadow(0 0 10px #39ff14) drop-shadow(0 0 20px #39ff14) drop-shadow(0 0 30px #ffffff);
}

.frog-head::after {
  content: '';
  position: absolute;
  bottom: -4px;
  left: 50%;
  transform: translateX(-50%);
  width: 4px;
  height: 8px;
  background-color: #ff2222;
  border-radius: 1px;
  clip-path: polygon(
    0% 0%,
    100% 0%,
    100% 60%,
    65% 100%,
    100% 60%,
    100% 100%,
    50% 65%,
    0% 100%,
    0% 60%
  );
  animation: snake-tongue-flicker 0.3s infinite ease-in-out alternate;
  z-index: 10;
}

@keyframes snake-tongue-flicker {
  0% {
    height: 2px;
    bottom: 0px;
  }
  100% {
    height: 9px;
    bottom: -6px;
  }
}

.frog-dir-up {
  transform: rotate(180deg) translateY(calc(var(--custom-cell-size) * -0.1));
}
.frog-dir-down {
  transform: rotate(0deg) translateY(calc(var(--custom-cell-size) * -0.1));
}
.frog-dir-left {
  transform: rotate(90deg) translateY(calc(var(--custom-cell-size) * -0.1));
}
.frog-dir-right {
  transform: rotate(-90deg) translateY(calc(var(--custom-cell-size) * -0.1));
}

/* 食物、炸彈發光特效區 */
.food-emoji {
  font-size: calc(var(--custom-cell-size) * 0.7);
  line-height: 1;
  user-select: none;
  filter: drop-shadow(0 0 6px #ff2222) drop-shadow(0 0 12px #ff2222);
}

.bomb-emoji {
  font-size: calc(var(--custom-cell-size) * 0.7);
  line-height: 1;
  user-select: none;
  filter: drop-shadow(0 0 6px #00ffff) drop-shadow(0 0 10px #9400d3);
}

.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0, 0, 0, 0.75);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 999;
  backdrop-filter: blur(4px);
}
.modal-content {
  background-color: #151518;
  border: 2px solid #2e7d32;
  padding: 35px 40px;
  border-radius: 12px;
  width: 700px;
  max-width: 90%;
  box-shadow: 0 0 30px rgba(46, 125, 50, 0.3);
  text-align: left;
}
.modal-content h2 {
  margin-top: 0;
  color: #39ff14;
  font-family: 'Orbitron', sans-serif;
  font-size: 1.8rem;
  text-align: center;
  text-shadow: 0 0 8px rgba(57, 255, 20, 0.5);
}
.modal-content hr {
  border: 0;
  border-top: 2px solid #222;
  margin: 20px 0;
}
.rules-list {
  padding-left: 20px;
  margin-bottom: 25px;
}
.rules-list li {
  margin-bottom: 16px;
  font-size: 1.05rem;
  line-height: 1.6;
  color: #cccccc;
}
.rules-list strong {
  color: #39ff14;
}
.close-btn {
  display: block;
  width: 100%;
  padding: 12px 0;
  background-color: #2e7d32;
  color: white;
  border: none;
  border-radius: 6px;
  font-size: 1.1rem;
  font-weight: bold;
  cursor: pointer;
  transition: background-color 0.15s ease;
  font-family: 'Space Grotesk', sans-serif;
  letter-spacing: 1px;
}
.close-btn:hover {
  background-color: #39ff14;
  color: #000;
  box-shadow: 0 0 15px #39ff14;
}
.close-btn:active {
  transform: scale(0.98);
}
</style>
