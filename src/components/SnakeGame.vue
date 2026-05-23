<template>
  <div id="game-container">
    <!-- 階段一：遊戲初始入口畫面 -->
    <div v-if="gameState === 'START'" class="screen">
      <h1>🐍 貪食蛇：魔王討伐戰 ⚔️</h1>
      <p class="description">
        這不是傳統的貪食蛇！在這裡，你吃下的每一顆能量豆，都會化為攻擊魔王的砲火。
        你可以消耗自身長度施放強力技能，在魔王的彈幕夾縫中求生吧！
      </p>
      <button @click="startGameStage">進入戰場</button>
    </div>

    <!-- 階段一：遊戲主戰場 UI -->
    <div v-show="gameState === 'PLAYING'" id="stage">
      <div id="hud">
        <div class="hud-item">🐍 蛇身長度 (HP): <span>{{ snakeLength }}</span></div>
        <div class="hud-item">👹 Boss 血量: <span>{{ bossHp }}/100</span></div>
        <div class="hud-item">⚡ 技能狀態: <span>{{ skillStatus }}</span></div>
      </div>
      
      <!-- 遊戲核心畫布 -->
      <canvas ref="gameCanvas" width="600" height="400"></canvas>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

// 遊戲狀態：'START' 或 'PLAYING'
const gameState = ref('START')

// HUD 數據庫
const snakeLength = ref(3)
const bossHp = ref(100)
const skillStatus = ref('就緒 (空白鍵)')

// Canvas 引用
const gameCanvas = ref(null)
let ctx = null

// 點擊按鈕切換到遊戲畫面
const startGameStage = () => {
  gameState.value = 'PLAYING'
  // 確保 DOM 更新後再繪製 Canvas
  setTimeout(() => {
    initCanvas()
    drawStaticStage()
  }, 50)
}

const initCanvas = () => {
  if (gameCanvas.value) {
    ctx = gameCanvas.value.getContext('2d')
  }
}

// 繪製靜態物件
const drawStaticStage = () => {
  if (!ctx) return

  // 1. 清空畫布
  ctx.clearRect(0, 0, gameCanvas.value.width, gameCanvas.value.height)

  // 2. 繪製靜態的蛇 (綠色，長度3)
  ctx.fillStyle = '#2ed573'
  ctx.fillRect(100, 200, 20, 20) // 蛇頭
  ctx.fillRect(80, 200, 20, 20)  // 蛇身1
  ctx.fillRect(60, 200, 20, 20)  // 蛇身2

  // 3. 繪製靜態的 Boss (紅色大方塊，置中在上方)
  ctx.fillStyle = '#ff4757'
  ctx.fillRect(gameCanvas.value.width / 2 - 30, 30, 60, 60) 

  // 4. 繪製能量豆子 (金色)
  ctx.fillStyle = '#ffa502'
  ctx.beginPath()
  ctx.arc(300, 250, 8, 0, Math.PI * 2)
  ctx.fill()
}
</script>

<style scoped>
#game-container {
  position: relative;
  border: 4px solid #333;
  background-color: #111;
  padding: 20px;
  border-radius: 10px;
  box-shadow: 0 0 20px rgba(0,0,0,0.5);
  font-family: 'Arial', sans-serif;
  color: #fff;
  width: 640px;
  margin: 0 auto;
}

.screen {
  text-align: center;
  padding: 40px 20px;
}

h1 { color: #ff4757; margin-top: 0; }
.description { color: #ccc; line-height: 1.6; margin-bottom: 30px; }

button {
  background-color: #ff4757;
  color: white;
  border: none;
  padding: 12px 24px;
  font-size: 18px;
  cursor: pointer;
  border-radius: 5px;
  transition: 0.2s;
}
button:hover { background-color: #ff6b81; transform: scale(1.05); }

#hud {
  display: flex;
  justify-content: space-between;
  margin-bottom: 10px;
  font-weight: bold;
  font-size: 16px;
  background: #222;
  padding: 10px;
  border-radius: 5px;
}

#hud span {
  color: #ffca28;
}

canvas {
  background-color: #050505;
  border: 2px solid #444;
  display: block;
}
</style>