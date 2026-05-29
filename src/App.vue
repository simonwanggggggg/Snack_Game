<script setup>
import { ref, onMounted, onUnmounted, watch } from 'vue';

// --- 配置區 ---
const GRID_SIZE = 20;       
const INITIAL_SPEED = 250; 
const SPEED_STEP = 5;     
const MIN_SPEED = 150;      
const TOTAL_CELLS = GRID_SIZE * GRID_SIZE; 

// 【動態測試開關】測試完勝利畫面後，請把這個數字 5 改回 400 即可正式上線！
const DEBUG_VICTORY_LENGTH = 400; 
// ------------

// 初始狀態為 3 段長度
const snake = ref([
  { x: 10, y: 10, dir: 'UP' }, 
  { x: 10, y: 11, dir: 'UP' }, 
  { x: 10, y: 12, dir: 'UP' }  
]);
const foods = ref([]);
const bombs = ref([]);
const applesEaten = ref(0);

const direction = ref({ x: 0, y: -1, dir: 'UP' }); 
const score = ref(0);
const gameInterval = ref(null);
const currentSpeed = ref(INITIAL_SPEED);
const showRules = ref(false);
const gameState = ref('NOT_STARTED');

// 衝刺與零延遲核心變數
const isBoosting = ref(false);
const tickCounter = ref(0); 

// 一維快取陣列（長度 400）
const boardCache = ref(Array(TOTAL_CELLS).fill('EMPTY'));

// JS 端更新地圖快取
const updateBoardCache = () => {
  const nextCache = Array(TOTAL_CELLS).fill('EMPTY');
  for (let i = 0; i < foods.value.length; i++) {
    nextCache[foods.value[i].y * GRID_SIZE + foods.value[i].x] = 'FOOD';
  }
  for (let i = 0; i < bombs.value.length; i++) {
    nextCache[bombs.value[i].y * GRID_SIZE + bombs.value[i].x] = 'BOMB';
  }
  const len = snake.value.length;
  for (let i = 1; i < len; i++) {
    const s = snake.value[i];
    const idx = s.y * GRID_SIZE + s.x;
    if (idx >= 0 && idx < TOTAL_CELLS) {
      nextCache[idx] = (i === len - 1) ? `TAIL_${i}` : `BODY_${i}`;
    }
  }
  const head = snake.value[0];
  const headIdx = head.y * GRID_SIZE + head.x;
  if (headIdx >= 0 && headIdx < TOTAL_CELLS) {
    nextCache[headIdx] = 'HEAD_' + head.dir;
  }
  boardCache.value = nextCache;
};

const getTargetFoodCount = () => {
    const extraLength = snake.value.length - 3; 
    return Math.max(1, 5 - Math.floor(extraLength / 30)); 
};

const getTargetBombCount = () => {
    if (score.value >= 2500) return 0;
    const extraLength = snake.value.length - 3;
    return Math.max(1, 3 - Math.floor(extraLength / 60));
};

const calculateSpeed = () => {
    const level = Math.floor(snake.value.length / 10);
    return Math.max(INITIAL_SPEED - (level * SPEED_STEP), MIN_SPEED);
};

// 監聽蛇長度變化
watch(() => snake.value.length, () => {
    currentSpeed.value = calculateSpeed();
    
    const targetFoodCount = getTargetFoodCount();
    while (foods.value.length > targetFoodCount) foods.value.pop(); 
    
    const targetBombCount = getTargetBombCount();
    while (bombs.value.length > targetBombCount) bombs.value.pop();

    if (gameState.value === 'PLAYING') {
        generateFood();
        generateBombs();
    }
    updateBoardCache();
}, { flush: 'post' });

const isInFrogSafeZone = (x, y) => {
    const head = snake.value[0];
    return Math.abs(x - head.x) <= 2 && Math.abs(y - head.y) <= 2;
};

const generateFood = () => {
    if (snake.value.length >= TOTAL_CELLS) return;
    const targetCount = getTargetFoodCount();
    while (foods.value.length < targetCount) {
        const newFood = { x: Math.floor(Math.random() * GRID_SIZE), y: Math.floor(Math.random() * GRID_SIZE) };
        const idx = newFood.y * GRID_SIZE + newFood.x;
        if (boardCache.value[idx] === 'EMPTY' && !isInFrogSafeZone(newFood.x, newFood.y)) {
            foods.value.push(newFood);
            updateBoardCache();
        }
        if (snake.value.length + foods.value.length >= TOTAL_CELLS) break;
    }
};

const generateBombs = () => {
    const targetCount = getTargetBombCount();
    while (bombs.value.length < targetCount) {
        const newBomb = { x: Math.floor(Math.random() * GRID_SIZE), y: Math.floor(Math.random() * GRID_SIZE) };
        const idx = newBomb.y * GRID_SIZE + newBomb.x;
        if (boardCache.value[idx] === 'EMPTY' && !isInFrogSafeZone(newBomb.x, newBomb.y)) {
            bombs.value.push(newBomb);
            updateBoardCache();
        }
        if (snake.value.length + foods.value.length + bombs.value.length >= TOTAL_CELLS) break;
    }
};

const relocateOneBomb = () => {
    const targetBombCount = getTargetBombCount();
    while (bombs.value.length > targetBombCount) bombs.value.pop();
    if (bombs.value.length === 0) return;
    const randomIndex = Math.floor(Math.random() * bombs.value.length);
    let attempts = 0;
    while (attempts < 100) {
        const newX = Math.floor(Math.random() * GRID_SIZE);
        const newY = Math.floor(Math.random() * GRID_SIZE);
        const idx = newY * GRID_SIZE + newX;
        if (boardCache.value[idx] === 'EMPTY' && !isInFrogSafeZone(newX, newY)) {
            bombs.value[randomIndex] = { x: newX, y: newY };
            break;
        }
        attempts++;
    }
};

const getBodyOpacity = (type) => {
  if (!type.includes('_')) return 1;
  const partIndex = parseInt(type.split('_')[1]); 
  const totalLength = snake.value.length;
  const distanceFromTail = totalLength - 1 - partIndex; 
  if (distanceFromTail < 5) {
    return 0.2 + (distanceFromTail * 0.16); 
  }
  return 1; 
};

// 核心移動邏輯
const move = () => {
    if (gameState.value !== 'PLAYING') return;

    const head = { x: snake.value[0].x + direction.value.x, y: snake.value[0].y + direction.value.y, dir: direction.value.dir };

    // 判定 1：撞牆
    if (head.x < 0 || head.x >= GRID_SIZE || head.y < 0 || head.y >= GRID_SIZE) {
        triggerGameOver();
        return;
    }

    const targetIdx = head.y * GRID_SIZE + head.x;
    const targetType = boardCache.value[targetIdx];

    // 判定 2：自我碰撞
    if (targetType.startsWith('BODY_') || targetType.startsWith('HEAD_') || 
       (targetType.startsWith('TAIL_') && foods.value.some(f => f.x === head.x && f.y === head.y))) {
        triggerGameOver();
        return;
    }
    // 判定 3：踩到炸彈
    if (targetType === 'BOMB') {
        triggerGameOver();
        return;
    }

    snake.value.unshift(head);
    
    if (targetType === 'FOOD') {
        score.value += 10;
        const foodIndex = foods.value.findIndex(f => f.x === head.x && f.y === head.y);
        if (foodIndex !== -1) foods.value.splice(foodIndex, 1);
        applesEaten.value++;
        
        if (snake.value.length >= DEBUG_VICTORY_LENGTH || snake.value.length === TOTAL_CELLS) {
            updateBoardCache();
            triggerVictory();
            return;
        }
        if (applesEaten.value >= 3) {
            relocateOneBomb();
            applesEaten.value = 0;
        }
        generateFood(); 
        generateBombs(); 
    } else {
        snake.value.pop();
    }
    updateBoardCache();
};

// 高頻率心跳時鐘 (每 10 毫秒執行一次)
const gameTickHandler = () => {
    if (gameState.value !== 'PLAYING') return;

    // 🟢 衝刺速度限制：最高速度 150ms 的 2.5 倍速（最快 60ms 一格）
    const requiredInterval = isBoosting.value 
        ? Math.max(currentSpeed.value * 0.5, MIN_SPEED / 2.5) 
        : currentSpeed.value;
        
    tickCounter.value += 10; 

    if (tickCounter.value >= requiredInterval) {
        move();
        tickCounter.value = 0; 
    }
};

const reverseSnake = () => {
    if (snake.value.length < 2 || gameState.value !== 'PLAYING') return;
    const reversed = [...snake.value].reverse();
    for (let i = 0; i < reversed.length - 1; i++) {
        const current = reversed[i];
        const next = reversed[i + 1];
        if (next.x > current.x) current.dir = 'LEFT';
        else if (next.x < current.x) current.dir = 'RIGHT';
        else if (next.y > current.y) current.dir = 'UP';
        else if (next.y < current.y) current.dir = 'DOWN';
    }
    reversed[reversed.length - 1].dir = reversed[reversed.length - 2].dir;

    const newHead = reversed[0];
    const nextPart = reversed[1];
    const newDx = newHead.x - nextPart.x;
    const newDy = newHead.y - nextPart.y;
    
    let newDirStr = 'UP';
    if (newDx === 1) newDirStr = 'RIGHT';
    else if (newDx === -1) newDirStr = 'LEFT';
    else if (newDy === 1) newDirStr = 'DOWN';
    
    direction.value = { x: newDx, y: newDy, dir: newDirStr };
    snake.value = reversed;
    updateBoardCache();
};

const togglePause = () => {
    if (gameState.value === 'PLAYING') {
        gameState.value = 'PAUSED';
        isBoosting.value = false;
    } else if (gameState.value === 'PAUSED') {
        gameState.value = 'PLAYING';
        tickCounter.value = 0; 
    }
};

const isKeyMatchingCurrentDirection = (key) => {
    if (key === 'ArrowUp' && direction.value.dir === 'UP') return true;
    if (key === 'ArrowDown' && direction.value.dir === 'DOWN') return true;
    if (key === 'ArrowLeft' && direction.value.dir === 'LEFT') return true;
    if (key === 'ArrowRight' && direction.value.dir === 'RIGHT') return true;
    return false;
};

const handleKey = (e) => {
    if (showRules.value) {
        if (e.key === 'Escape') { e.preventDefault(); showRules.value = false; }
        return; 
    }
    const triggerKeys = ['Space', 'Enter'];
    if (gameState.value === 'NOT_STARTED' || gameState.value === 'GAME_OVER' || gameState.value === 'VICTORY') {
        if (triggerKeys.includes(e.code) || triggerKeys.includes(e.key)) { e.preventDefault(); startGame(); }
        return; 
    }
    if (gameState.value === 'PAUSED') {
        if (triggerKeys.includes(e.code) || triggerKeys.includes(e.key)) { e.preventDefault(); togglePause(); return; }
        const directionKeys = ['ArrowUp', 'ArrowDown', 'ArrowLeft', 'ArrowRight'];
        if (directionKeys.includes(e.key)) { e.preventDefault(); togglePause(); }
    }
    if (e.key === 'Enter') { e.preventDefault(); togglePause(); return; }
    if (e.code === 'Space') { e.preventDefault(); reverseSnake(); return; }

    const keys = { ArrowUp: { x: 0, y: -1, dir: 'UP' }, ArrowDown: { x: 0, y: 1, dir: 'DOWN' }, ArrowLeft: { x: -1, y: 0, dir: 'LEFT' }, ArrowRight: { x: 1, y: 0, dir: 'RIGHT' } };
    const target = keys[e.key];
    
    if (target) {
        if (target.x !== -direction.value.x || target.y !== -direction.value.y) {
            if (target.dir === direction.value.dir) {
                isBoosting.value = true;
                move();
                tickCounter.value = 0; 
            } else {
                direction.value = target;
                isBoosting.value = false;
                move();
                tickCounter.value = 0; 
            }
        }
    }
};

const handleKeyUp = (e) => {
    if (gameState.value !== 'PLAYING') return;
    if (isKeyMatchingCurrentDirection(e.key)) {
        isBoosting.value = false;
    }
};

const startGame = () => {
    if (gameInterval.value) clearInterval(gameInterval.value);

    snake.value = [
      { x: 10, y: 10, dir: 'UP' },
      { x: 10, y: 11, dir: 'UP' },
      { x: 10, y: 12, dir: 'UP' }
    ];
    direction.value = { x: 0, y: -1, dir: 'UP' };
    score.value = 0;
    applesEaten.value = 0; 
    currentSpeed.value = INITIAL_SPEED;
    isBoosting.value = false;
    tickCounter.value = 0;
    foods.value = [];
    bombs.value = []; 
    updateBoardCache();
    generateFood(); 
    generateBombs(); 
    gameState.value = 'PLAYING';

    gameInterval.value = setInterval(gameTickHandler, 10);
};

const triggerGameOver = () => { 
    stopGame(); 
    gameState.value = 'GAME_OVER'; 
};

const triggerVictory = () => { 
    stopGame(); 
    gameState.value = 'VICTORY'; 
};

const stopGame = () => { 
    isBoosting.value = false; 
    if (gameInterval.value) {
        clearInterval(gameInterval.value);
        gameInterval.value = null; 
    }
};

onMounted(() => { 
    window.addEventListener('keydown', handleKey); 
    window.addEventListener('keyup', handleKeyUp);
    updateBoardCache(); 
});
onUnmounted(() => { 
    window.removeEventListener('keydown', handleKey); 
    window.removeEventListener('keyup', handleKeyUp);
    stopGame(); 
});
</script>

<template>
  <div class="main-wrapper">
    
    <div class="screen-title-abs">
      <h1>貪食蛇遊戲</h1>
    </div>

    <div class="screen-score-box-abs">
      <div class="score-label">目前得分</div>
      <div class="score-value">{{ score }}</div>
    </div>

    <div class="game-container">
        <div class="game-board neon-black-board" :class="{ 'neon-black-board-boosting': isBoosting }">
            <div v-for="(type, idx) in boardCache" :key="idx" class="cell">
              <div v-if="type.startsWith('HEAD_')" 
                   class="frog-head" 
                   :class="'frog-dir-' + type.split('_')[1].toLowerCase()">
                🐸
              </div>

              <div v-else-if="type.startsWith('TAIL_')" 
                   class="snake-body-circle"
                   :class="{ 'snake-body-boosting': isBoosting }"
                   :style="{ opacity: getBodyOpacity(type) }">
              </div>

              <div v-else-if="type.startsWith('BODY_')" 
                   class="snake-body-circle"
                   :class="{ 'snake-body-boosting': isBoosting }"
                   :style="{ opacity: getBodyOpacity(type) }">
              </div>

              <div v-else-if="type === 'FOOD'" class="food-emoji">🍎</div>

              <div v-else-if="type === 'BOMB'" class="bomb-emoji">💣</div>
            </div>

            <div v-if="gameState === 'NOT_STARTED' || gameState === 'PAUSED'" class="game-overlay-panel">
              <div class="overlay-title">
                {{ gameState === 'NOT_STARTED' ? '準備好了嗎？' : '遊戲暫停' }}
              </div>
              <div class="btn-group-vertical">
                <button class="start-btn" @click="startGame">
                  {{ gameState === 'PAUSED' ? '重新開始' : '開始遊戲' }}
                </button>
                <button class="rules-btn" @click="showRules = true">遊戲玩法</button>
              </div>
              <div class="pause-hint-text">
                [ 按下 <span class="key-highlight">Space</span> 或 <span class="key-highlight">Enter</span> 即可開始遊戲 ]
              </div>
            </div>

            <div v-else-if="gameState === 'GAME_OVER'" class="game-overlay-panel game-over-bg">
              <div class="overlay-title game-over-title">💥 遊戲結束 💥</div>
              <div class="final-score-box">
                <div class="final-label">最終得分</div>
                <div class="final-value">{{ score }}</div>
              </div>
              <div class="btn-group-vertical">
                <button class="start-btn restart-theme" @click="startGame">再試一次</button>
                <button class="rules-btn" @click="showRules = true">遊戲玩法</button>
              </div>
              <div class="pause-hint-text">
                [ 按下 <span class="key-highlight">Space</span> 或 <span class="key-highlight">Enter</span> 可重新開始 ]
              </div>
            </div>

            <div v-else-if="gameState === 'VICTORY'" class="game-overlay-panel victory-bg">
              <div class="overlay-title victory-title">👑 恭喜完美通關 👑</div>
              <div class="final-score-box victory-score-box">
                <div class="final-label" style="color: #6d4c41;">最高紀錄</div>
                <div class="final-value" style="color: #b71c1c;">{{ score }}</div>
              </div>
              <div class="btn-group-vertical">
                <button class="start-btn victory-btn" @click="startGame">再挑戰一次</button>
                <button class="rules-btn" @click="showRules = true">遊戲玩法</button>
              </div>
              <div class="pause-hint-text" style="color: #5d4037;">
                [ 按下 <span class="key-highlight" style="color: #b71c1c;">Space</span> 或 <span class="key-highlight" style="color: #b71c1c;">Enter</span> 重新開啟遊戲 ]
              </div>
            </div>
        </div>
    </div>

    <div v-if="showRules" class="modal-overlay" @click.self="showRules = false">
      <div class="modal-content">
        <h2>🎮 遊戲玩法說明</h2>
        <hr />
        <ul class="rules-list">
          <li><strong>遊戲開始：</strong> 在未開始、暫停或遊戲結束畫面，按 <strong>Space 鍵</strong> 或 <strong>Enter 鍵</strong> 可直接開始遊玩！</li>
          <li><strong>關閉說明：</strong> 隨時按下盤上的 <strong>Esc 鍵</strong> 即可快速關閉本說明書。</li>
          <li><strong>暫停功能：</strong> 遊戲進行中，隨時按下 <strong>Enter 鍵</strong> 即可暫停遊戲。</li>
          <li><strong>方向鍵控制：</strong> 控制蛇前進的方向。</li>
          <li><strong>⚡ 按住/狂連點同方向鍵加速：</strong> 長按目前移動方向鍵會進入高速衝刺狀態（最快被鎖定在 60ms 限制內）；<strong>放開即恢復常速</strong>。在前進的大直線上，狂按同方向鍵，蛇頭會同步瘋狂往前突進！</li>
          <li><strong>空白鍵技能：</strong> 遊戲進行中按下可施展特殊技能。讓蛇頭與蛇尾<strong>「對調位置」</strong>並逆向移動。</li>
          <li><strong>吃到蘋果🍎：</strong> 得分 +10，且身體長度增加 1 節。</li>
          <li><strong>動態難度：</strong> 一開始場上會 5 顆蘋果，隨著長度每長 30 節少一顆，最後剩 1 顆。</li>
          <li><strong>危險炸彈💣：</strong> 一開始場上會有 3 顆炸彈，每長 60 節少一顆。每吃 3 顆蘋果重新定位，且絕對不會出現在蛇頭周圍 5x5 的範圍內。當分數到達 2500 分時，炸彈完全消失！</li>
          <li><strong>自動加速：</strong> 蛇的長度每增加 10 節，移動速度就會自動加快一次。</li>
          <li><strong>失敗條件：</strong> 踩到地圖上的炸彈💣、撞到牆壁，或者蛇頭撞到自己的身體。</li>
          <li><strong>完美通關：</strong> 當蛇的身軀填滿整個遊戲畫面（共 400 節），即達成完美通關！</li>
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
:root, .main-wrapper {
  --board-total-size: 82vh;
  --custom-cell-size: calc(var(--board-total-size) / 20);
}

.main-wrapper {
  display: flex; flex-direction: column; justify-content: center; align-items: center; min-height: 100vh; background-color: #efefef; color: #333333; font-family: 'Segoe UI', sans-serif; position: relative; overflow: hidden; 
}
.screen-title-abs { position: fixed; top: 25px; left: 30px; z-index: 100; }
.screen-title-abs h1 { margin: 0; color: #2e7d32; letter-spacing: 2px; font-size: 2.2rem; }
.screen-score-box-abs { position: fixed; top: 25px; right: 30px; background-color: #ffffff; border: 3px solid #333; border-radius: 6px; padding: 8px 20px; text-align: center; min-width: 90px; box-shadow: 0 4px 10px rgba(0,0,0,0.08); z-index: 100; }
.score-label { font-size: 0.75rem; font-weight: 900; color: #888; letter-spacing: 1px; margin-bottom: 2px; }
.score-value { font-size: 1.6rem; font-weight: bold; color: #222; font-family: monospace; }
.game-container { display: flex; flex-direction: column; align-items: center; }

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
  box-shadow: 0 0 25px rgba(57, 255, 20, 0.2); 
  transition: box-shadow 0.2s ease;
}

.neon-black-board-boosting {
  box-shadow: 0 0 35px rgba(57, 255, 20, 0.6), inset 0 0 15px rgba(57, 255, 20, 0.2);
}

.cell { width: var(--custom-cell-size); height: var(--custom-cell-size); display: flex; justify-content: center; align-items: center; position: relative; }

.game-overlay-panel { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background-color: rgba(30, 30, 30, 0.85); display: flex; flex-direction: column; justify-content: center; align-items: center; z-index: 200; color: white; text-align: center; }
.game-over-bg { background-color: rgba(25, 15, 15, 0.92); }

.victory-bg { background-color: rgba(255, 215, 0, 0.94); color: #5d4037; }
.victory-title { color: #b71c1c; font-size: calc(var(--custom-cell-size) * 1.3); text-shadow: 0 2px 4px rgba(255,255,255,0.6); animation: victory-glow 1.5s infinite alternate; }
.victory-score-box { background-color: rgba(255, 255, 255, 0.6); border: 2px solid #b71c1c; }

@keyframes victory-glow {
  0% { transform: scale(1); }
  100% { transform: scale(1.05); }
}

.overlay-title { font-size: calc(var(--custom-cell-size) * 1.2); font-weight: bold; letter-spacing: 1px; color: #81c784; margin-bottom: 25px; text-shadow: 0 2px 4px rgba(0,0,0,0.5); }
.game-over-title { color: #ff5252; font-size: calc(var(--custom-cell-size) * 1.4); }
.final-score-box { background-color: rgba(250, 250, 250, 0.08); border: 2px solid rgba(255, 255, 255, 0.2); border-radius: 6px; padding: 8px calc(var(--custom-cell-size) * 1.5); margin-bottom: 25px; text-align: center; }
.final-label { font-size: calc(var(--custom-cell-size) * 0.45); color: #ccc; font-weight: bold; letter-spacing: 1px; margin-bottom: 4px; }
.final-value { font-size: calc(var(--custom-cell-size) * 1.4); font-weight: 900; color: #ffd54f; font-family: monospace; line-height: 1; }
.btn-group-vertical { display: flex; flex-direction: column; gap: 12px; width: calc(var(--custom-cell-size) * 7); }

/* 按鈕樣式 */
.start-btn { padding: calc(var(--custom-cell-size) * 0.35) 0; font-size: calc(var(--custom-cell-size) * 0.65); font-weight: bold; cursor: pointer; background-color: #007bff; color: white; border: 2px solid #ffffff; border-radius: 4px; box-shadow: none; transition: background-color 0.15s ease; }
.start-btn:hover { background-color: #0069d9; }
.start-btn:active { background-color: #0056b3; }
.restart-theme { background-color: #ff7043; border: 2px solid #ffffff; box-shadow: none; transition: background-color 0.15s ease; }
.restart-theme:hover { background-color: #f4511e; }
.restart-theme:active { background-color: #d84315; }

.victory-btn { background-color: #b71c1c; }
.victory-btn:hover { background-color: #7f0000; }
.victory-btn:active { background-color: #4c0000; }

.rules-btn { padding: calc(var(--custom-cell-size) * 0.35) 0; font-size: calc(var(--custom-cell-size) * 0.65); font-weight: bold; cursor: pointer; background-color: #6c757d; color: white; border: 2px solid #ffffff; border-radius: 4px; transition: background-color 0.15s ease; }
.rules-btn:hover { background-color: #5a6268; }
.rules-btn:active { background-color: #434a50; }
.pause-hint-text { margin-top: 25px; font-size: calc(var(--custom-cell-size) * 0.45); color: #aaa; }
.key-highlight { color: #ffb74d; font-weight: bold; }

/* ==========================================
   蛇身與青蛙頭：完美網格霓虹融合版
   ========================================== */
.snake-body-circle { 
  width: 112%; 
  height: 112%; 
  background-color: #39ff14; 
  border-radius: 50%;        
  box-shadow: 0 0 10px #39ff14, 0 0 20px rgba(57, 255, 20, 0.5); 
  transition: opacity 0.15s ease-in-out;
  animation: snake-pulse 2.2s infinite ease-in-out; 
}

.snake-body-boosting {
  animation: snake-pulse-fast 0.5s infinite ease-in-out !important;
  box-shadow: 0 0 14px #39ff14, 0 0 25px #39ff14 !important;
}

@keyframes snake-pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(0.96); }
}

@keyframes snake-pulse-fast {
  0%, 100% { transform: scale(0.96); }
  50% { transform: scale(0.90); }
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
  z-index: 50;
  filter: drop-shadow(0 0 6px #39ff14) drop-shadow(0 0 12px rgba(57, 255, 20, 0.6));
  transition: filter 0.2s ease;
}

/* ⚡ 蛇頭衝刺發光加強 */
.neon-black-board-boosting .frog-head {
  filter: drop-shadow(0 0 10px #39ff14) drop-shadow(0 0 20px #39ff14) drop-shadow(0 0 30px #ffffff);
}

/* 蛇舌頭 */
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
  clip-path: polygon(0% 0%, 100% 0%, 100% 60%, 65% 100%, 100% 60%, 100% 100%, 50% 65%, 0% 100%, 0% 60%);
  animation: snake-tongue-flicker 0.3s infinite ease-in-out alternate;
  z-index: 10;
}

@keyframes snake-tongue-flicker {
  0% { height: 2px; bottom: 0px; }
  100% { height: 9px; bottom: -6px; }
}

/* 青蛙頭專用旋轉與上移複合樣式 */
.frog-dir-up    { transform: rotate(180deg) translateY(calc(var(--custom-cell-size) * -0.10)); }
.frog-dir-down  { transform: rotate(0deg) translateY(calc(var(--custom-cell-size) * -0.10)); }
.frog-dir-left  { transform: rotate(90deg) translateY(calc(var(--custom-cell-size) * -0.10)); }
.frog-dir-right { transform: rotate(-90deg) translateY(calc(var(--custom-cell-size) * -0.10)); }

/* ==========================================
   食物、炸彈發光特效區 (精確貼合 Emoji 輪廓)
   ========================================== */
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

.modal-overlay { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background-color: rgba(0, 0, 0, 0.55); display: flex; justify-content: center; align-items: center; z-index: 999; }
.modal-content { background-color: #ffffff; padding: 35px 40px; border-radius: 12px; width: 700px; max-width: 90%; box-shadow: 0 10px 30px rgba(0,0,0,0.3); text-align: left; }
.modal-content h2 { margin-top: 0; color: #2e7d32; font-size: 1.8rem; text-align: center; }
.modal-content hr { border: 0; border-top: 2px solid #eee; margin: 20px 0; }
.rules-list { padding-left: 20px; margin-bottom: 25px; }
.rules-list li { margin-bottom: 16px; font-size: 1.05rem; line-height: 1.6; color: #333333; }
.close-btn { display: block; width: 100%; padding: 12px 0; background-color: #2e7d32; color: white; border: none; border-radius: 6px; font-size: 1.1rem; font-weight: bold; cursor: pointer; transition: background-color 0.15s ease; }
.close-btn:hover { background-color: #1b5e20; }
.close-btn:active { background-color: #0d3c12; }
</style>