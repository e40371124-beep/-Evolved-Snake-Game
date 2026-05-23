<template>
  <div id="game-container">
    <!-- 1. 初始入口畫面 -->
    <div v-if="gameState === 'START'" class="screen">
      <h1>🐍 貪食蛇：極致動作討伐戰 ⚔️</h1>
      <p class="description">
        <b>【機制修復：一鍵重開 ＆ 俐落白框護盾】</b><br>
        ⏱️ <b>完美重開：</b> 已修復無法重新挑戰的 Bug！現在不論戰敗、勝利或暫停都能一鍵秒重開！<br>
        🛡️ <b>白框防護：</b> 護盾全新改版為<b>「粗白色邊框」</b>包裹全蛇身，視覺更清晰、絕無穿牆 Bug！<br>
        🎯 <b>核心判定：</b> 魔王的彈幕只對蛇頭有效，放心地用身體去甩尾走位吧！
      </p>
      <button @click="startGameStage">啟動 BGM 並踏入戰場</button>
    </div>

    <!-- 2. 遊戲主戰場 UI -->
    <div v-show="gameState !== 'START'" id="stage">
      <div id="hud">
        <div class="hud-item">❤️ 血量: <span class="hp-text">{{ playerHp }}/5</span></div>
        <div class="hud-item">🛡️ 護盾: <span class="shield-text">{{ shieldCount }}/2</span></div>
        <div class="hud-item">📏 實際長度: <span style="color: #2ed573">{{ snake.length }}</span></div>
        <div class="hud-item">👹 Boss: <span :class="{ 'rage-text': isBossRaged }">{{ bossHp }}/100 {{ isBossRaged ? '【狂暴】' : '' }}</span></div>
      </div>

      <!-- 技能 CD 顯示條 -->
      <div id="skills-hud">
        <div class="skill-box" :class="{ 'ready': qCooldown === 0 }">
          <span class="key">Q</span> 超載加速
          <div class="cd-overlay" :style="{ width: (qCooldown / 8000) * 100 + '%' }"></div>
          <span class="cd-text" v-if="qCooldown > 0">{{ (qCooldown / 1000).toFixed(1) }}s</span>
        </div>
        <div class="skill-box" :class="{ 'ready': eCooldown === 0 }">
          <span class="key">E</span> 時空結界
          <div class="cd-overlay" :style="{ width: (eCooldown / 12000) * 100 + '%' }"></div>
          <span class="cd-text" v-if="eCooldown > 0">{{ (eCooldown / 1000).toFixed(1) }}s</span>
        </div>
      </div>
      
      <!-- 遊戲畫布與大招字卡、狂暴警告 -->
      <div class="canvas-wrapper">
        <canvas ref="gameCanvas" width="600" height="400"></canvas>
        <div class="skill-flash" :class="{ 'flash-active': isFlashActive }"></div>
        
        <!-- 魔王招式與狂暴警報字卡 -->
        <div class="skill-banner-zone">
          <div v-if="rageAlertTimer > 0" class="rage-alert-box">⚠️ WARNING: BOSS ENTERED RAGE MODE ⚠️</div>
          <div class="boss-skill-banner" :class="{ 'rage-banner': isBossRaged }">🔮 目前招式：{{ bossSkillName }}</div>
        </div>

        <!-- ⏸️ 暫停覆蓋面板 -->
        <div v-if="gameState === 'PAUSED'" class="overlay paused-overlay">
          <h2>⏸️ 遊戲暫停 ⏸️</h2>
          <p>戰局已凍結，請選擇你的下一步操作</p>
          <div class="button-group">
            <button @click="resumeGame" class="resume-btn">繼續戰鬥</button>
            <button @click="resetGame" class="restart-btn">放棄並重開一局</button>
          </div>
        </div>
      </div>

      <!-- 結束與勝利的覆蓋畫面 -->
      <div v-if="gameState === 'GAMEOVER'" class="overlay">
        <h2>❌ 戰敗 ❌</h2>
        <p>身體縮小到極限了... 沉住氣調整節奏再來一次！</p>
        <button @click="resetGame" class="restart-btn">重新挑戰</button>
      </div>

      <div v-if="gameState === 'WIN'" class="overlay win">
        <h2>🎉 討伐成功 🎉</h2>
        <p>太強了！你成功擊碎了進入狂暴死鬥模式的魔王！</p>
        <button @click="resetGame" class="resume-btn">再玩一次</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onUnmounted } from 'vue'

const gameState = ref('START') // 'START', 'PLAYING', 'PAUSED', 'GAMEOVER', 'WIN'
const playerHp = ref(5)
const shieldCount = ref(0)
const bossHp = ref(100)
const bossSkillName = ref('觀察中...')
const isBossRaged = ref(false)
const rageAlertTimer = ref(0)

const qCooldown = ref(0)
const eCooldown = ref(0)
const isFlashActive = ref(false)

const gameCanvas = ref(null)
let ctx = null
let animationFrameId = null

// 🐍 蛇與地圖設定
const gridSize = 20
const snake = ref([])
let velocityX = gridSize
let velocityY = 0
let playerInvincibleTimer = 0
const INVINCIBLE_DURATION = 800 

// 🍎 多功能食物系統（每 1.25 秒一顆球）
const foods = ref([]) 
let foodSpawnTimer = 0
const FOOD_SPAWN_INTERVAL = 1250 

// 👹 魔王設定
const boss = { x: 300, y: 80, width: 60, height: 60 }
let bossShakeTimer = 0 
let bossCastRingRadius = 0 
let bossCastRingMax = 80

// 🎯 核心計時器變數
let lastTime = 0
let snakeMoveTimer = 0
let snakeMoveInterval = 110   
let isSpeedUpActive = false   
let bossAttackTimer = 0

// 戰鬥陣列
const playerBullets = ref([]) 
const bossBullets = ref([])   
const particles = ref([])      

// 🎵 音效系統
let audioCtx = null
let bgmNode = null
let isAudioInitialized = false

const initAudio = () => {
  if (isAudioInitialized) return
  audioCtx = new (window.AudioContext || window.webkitAudioContext)()
  isAudioInitialized = true
}

const playSound = (freq, type, duration) => {
  if (!audioCtx) return
  try {
    const osc = audioCtx.createOscillator()
    const gain = audioCtx.createGain()
    osc.type = type
    osc.frequency.setValueAtTime(freq, audioCtx.currentTime)
    gain.gain.setValueAtTime(0.12, audioCtx.currentTime)
    gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + duration)
    osc.connect(gain)
    gain.connect(audioCtx.destination)
    osc.start()
    osc.stop(audioCtx.currentTime + duration)
  } catch (e) { }
}

const startBPMProgressiveBGM = () => {
  initAudio()
  if (!audioCtx) return
  let noteIndex = 0
  
  const playSequence = () => {
    if (gameState.value !== 'PLAYING' && gameState.value !== 'PAUSED') return
    if (gameState.value === 'PAUSED') {
      bgmNode = setTimeout(playSequence, isBossRaged.value ? 110 : 150)
      return
    }
    const time = audioCtx.currentTime
    
    const speedFactor = isBossRaged.value ? 110 : 150
    const pitchOffset = isBossRaged.value ? 1.5 : 1.0

    const bassLine = [110, 110, 130, 110, 146, 146, 130, 165]
    const melodyLine = [220, 0, 261, 220, 293, 293, 261, 329]

    const oscBass = audioCtx.createOscillator()
    const gainBass = audioCtx.createGain()
    oscBass.type = isBossRaged.value ? 'sawtooth' : 'triangle'
    oscBass.frequency.setValueAtTime(bassLine[noteIndex % bassLine.length] * pitchOffset, time)
    gainBass.gain.setValueAtTime(0.06, time)
    gainBass.gain.exponentialRampToValueAtTime(0.01, time + 0.12)
    oscBass.connect(gainBass)
    gainBass.connect(audioCtx.destination)
    oscBass.start(time)
    oscBass.stop(time + 0.13)

    if (melodyLine[noteIndex % melodyLine.length] > 0) {
      const oscMelody = audioCtx.createOscillator()
      const gainMelody = audioCtx.createGain()
      oscMelody.type = 'square'
      oscMelody.frequency.setValueAtTime(melodyLine[noteIndex % melodyLine.length] * pitchOffset, time)
      gainMelody.gain.setValueAtTime(0.02, time)
      gainMelody.gain.exponentialRampToValueAtTime(0.005, time + 0.1)
      oscMelody.connect(gainMelody)
      gainMelody.connect(audioCtx.destination)
      oscMelody.start(time)
      oscMelody.stop(time + 0.11)
    }

    noteIndex++
    bgmNode = setTimeout(playSequence, speedFactor)
  }
  playSequence()
}

const startGameStage = () => {
  gameState.value = 'PLAYING'
  resetGameData()
  setTimeout(() => {
    if (gameCanvas.value) ctx = gameCanvas.value.getContext('2d')
    window.removeEventListener('keydown', handleKeyDown) // 防重複綁定
    window.addEventListener('keydown', handleKeyDown)
    startBPMProgressiveBGM()
    
    lastTime = performance.now()
    animationFrameId = requestAnimationFrame(mainGameLoop)
  }, 50)
}

const resetGameData = () => {
  playerHp.value = 5
  shieldCount.value = 0
  bossHp.value = 100
  bossSkillName.value = '鎖定目標'
  isBossRaged.value = false
  rageAlertTimer.value = 0
  qCooldown.value = 0
  eCooldown.value = 0
  velocityX = gridSize
  velocityY = 0
  playerInvincibleTimer = 0 
  playerBullets.value = []
  bossBullets.value = []
  particles.value = []
  foods.value = []
  isSpeedUpActive = false
  bossAttackTimer = 0
  bossShakeTimer = 0
  bossCastRingRadius = 0
  foodSpawnTimer = 1000 
  
  snake.value = [
    { x: 200, y: 240 },
    { x: 180, y: 240 },
    { x: 160, y: 240 },
    { x: 140, y: 240 },
    { x: 120, y: 240 }
  ]
}

// 【修改１】一鍵重開主控引擎：能徹底中斷前一次的生命週期，並乾淨地重啟戰局
const resetGame = () => {
  cancelAnimationFrame(animationFrameId)
  if (bgmNode) clearTimeout(bgmNode)
  
  resetGameData()
  gameState.value = 'PLAYING' // 核心：強制將狀態調回遊玩中
  playSound(523, 'sine', 0.15)
  
  lastTime = performance.now()
  animationFrameId = requestAnimationFrame(mainGameLoop)
  startBPMProgressiveBGM()
}

const spawnPeriodicFood = () => {
  if (!gameCanvas.value) return
  let valid = false
  let fx = 0, fy = 0
  
  while (!valid) {
    const maxX = gameCanvas.value.width / gridSize
    const maxY = gameCanvas.value.height / gridSize
    fx = Math.floor(Math.random() * maxX) * gridSize
    fy = Math.floor(Math.random() * maxY) * gridSize
    
    if (!(fx >= boss.x - 60 && fx <= boss.x + 60 && fy <= boss.y + boss.height + 40)) {
      valid = true
    }
  }

  const rand = Math.random()
  let type = 'normal' 
  let color = '#ffa502'
  if (rand < 0.07) {
    type = 'heal'      
    color = '#2ed573'
  } else if (rand < 0.15) {
    type = 'shield'   
    color = '#54a0ff'
  }

  foods.value.push({ x: fx, y: fy, type, color })
}

const handleKeyDown = (e) => {
  const key = e.key.toLowerCase()

  if (key === 'p' || e.key === 'Escape') {
    e.preventDefault()
    if (gameState.value === 'PLAYING') {
      gameState.value = 'PAUSED'
      playSound(300, 'triangle', 0.15)
    } else if (gameState.value === 'PAUSED') {
      resumeGame()
    }
    return
  }

  if (gameState.value !== 'PLAYING') return

  if (key === 'q' && qCooldown.value === 0) {
    isSpeedUpActive = true
    qCooldown.value = 8000 
    playSound(587, 'sawtooth', 0.25)
    setTimeout(() => { isSpeedUpActive = false }, 2000) 
    return
  }

  if (key === 'e' && eCooldown.value === 0) {
    eCooldown.value = 12000 
    isFlashActive.value = true
    playSound(880, 'sine', 0.4)
    bossBullets.value.forEach(b => createExplosion(b.x, b.y, 4, '#00d2d3'))
    bossBullets.value = [] 
    setTimeout(() => { isFlashActive.value = false }, 180)
    return
  }

  switch (key) {
    case 'w': case 'arrowup': if (velocityY === 0) { velocityX = 0; velocityY = -gridSize; } break
    case 's': case 'arrowdown': if (velocityY === 0) { velocityX = 0; velocityY = gridSize; } break
    case 'a': case 'arrowleft': if (velocityX === 0) { velocityX = -gridSize; velocityY = 0; } break
    case 'd': case 'arrowright': if (velocityX === 0) { velocityX = gridSize; velocityY = 0; } break
  }
}

const resumeGame = () => {
  if (gameState.value === 'PAUSED') {
    gameState.value = 'PLAYING'
    playSound(440, 'sine', 0.1)
    lastTime = performance.now() 
  }
}

const mainGameLoop = (timestamp) => {
  if (gameState.value === 'PAUSED') {
    drawGame()
    animationFrameId = requestAnimationFrame(mainGameLoop)
    return
  }
  
  if (gameState.value !== 'PLAYING') return

  const dt = timestamp - lastTime
  lastTime = timestamp

  if (qCooldown.value > 0) qCooldown.value = Math.max(0, qCooldown.value - dt)
  if (eCooldown.value > 0) eCooldown.value = Math.max(0, eCooldown.value - dt)
  if (rageAlertTimer.value > 0) rageAlertTimer.value = Math.max(0, rageAlertTimer.value - dt)
  if (playerInvincibleTimer > 0) playerInvincibleTimer = Math.max(0, playerInvincibleTimer - dt)
  
  if (bossShakeTimer > 0) bossShakeTimer -= dt
  if (bossCastRingRadius > 0) {
    bossCastRingRadius += dt * (isBossRaged.value ? 0.5 : 0.3)
    if (bossCastRingRadius > bossCastRingMax) bossCastRingRadius = 0
  }

  foodSpawnTimer += dt
  if (foodSpawnTimer >= FOOD_SPAWN_INTERVAL) {
    foodSpawnTimer = 0
    if (foods.value.length < 5) { 
      spawnPeriodicFood()
    }
  }

  bossAttackTimer += dt
  const currentBossCD = isBossRaged.value ? 1300 : 2200
  if (bossAttackTimer >= currentBossCD) {
    bossAttackTimer = 0
    triggerRandomBossAttack()
  }

  snakeMoveTimer += dt
  const currentInterval = isSpeedUpActive ? snakeMoveInterval / 2 : snakeMoveInterval
  if (snakeMoveTimer >= currentInterval) {
    snakeMoveTimer = 0
    updateSnakeMovement()
  }

  updateProjectiles()
  updateParticles()
  drawGame()

  animationFrameId = requestAnimationFrame(mainGameLoop)
}

const triggerRandomBossAttack = () => {
  bossCastRingRadius = 1 
  const attackType = Math.floor(Math.random() * 5) + 1
  const head = snake.value[0]

  const dx = head.x - boss.x
  const dy = head.y - (boss.y + 30)
  const baseAngle = Math.atan2(dy, dx)
  
  const speedBonus = isBossRaged.value ? 1.0 : 0.0

  switch (attackType) {
    case 1:
      bossSkillName.value = isBossRaged.value ? '🔥 滅世極限擴散彈' : '🔥 扇形擴散彈(鎖定)'
      playSound(160, 'triangle', 0.15)
      const offsets = [-0.4, -0.2, 0, 0.2, 0.4]
      offsets.forEach(offset => {
        const finalAngle = baseAngle + offset
        const spd = 1.6 + speedBonus
        bossBullets.value.push({ x: boss.x, y: boss.y + 30, vx: Math.cos(finalAngle) * spd, vy: Math.sin(finalAngle) * spd, radius: 6 })
      })
      break

    case 2:
      bossSkillName.value = isBossRaged.value ? '🎯 雷射連環死線狙擊' : '🎯 狙擊精準彈'
      const burstCount = isBossRaged.value ? 5 : 3
      for (let i = 0; i < burstCount; i++) {
        setTimeout(() => {
          if (gameState.value !== 'PLAYING') return
          playSound(280, 'sawtooth', 0.08)
          const curHead = snake.value[0]
          const curDx = curHead.x - boss.x
          const curDy = curHead.y - (boss.y + 30)
          const dist = Math.sqrt(curDx*curDx + curDy*curDy)
          const spd = 2.0 + speedBonus
          bossBullets.value.push({ x: boss.x, y: boss.y + 30, vx: (curDx/dist) * spd, vy: (curDy/dist) * spd, radius: 5 })
        }, i * (isBossRaged.value ? 110 : 180))
      }
      break

    case 3:
      bossSkillName.value = isBossRaged.value ? '💥 終焉全方位星爆' : '💥 全方位星爆'
      playSound(200, 'sawtooth', 0.2)
      const starCount = isBossRaged.value ? 16 : 12
      for (let i = 0; i < starCount; i++) {
        const angle = (Math.PI * 2 / starCount) * i
        const spd = 1.25 + speedBonus
        bossBullets.value.push({ x: boss.x, y: boss.y + 30, vx: Math.cos(angle) * spd, vy: Math.sin(angle) * spd, radius: 6 })
      }
      break

    case 4:
      bossSkillName.value = isBossRaged.value ? '⚡ 天崩地裂夾擊陣' : '⚡ 左右夾擊彈'
      playSound(180, 'triangle', 0.1)
      for (let i = 0; i < 4; i++) {
        const vxSpd = 0.75 + i*0.2 + speedBonus
        const vySpd = 1.5 + speedBonus
        bossBullets.value.push({ x: 40, y: 40, vx: vxSpd, vy: vySpd, radius: 6 })
        bossBullets.value.push({ x: 560, y: 40, vx: -vxSpd, vy: vySpd, radius: 6 })
      }
      break

    case 5:
      bossSkillName.value = isBossRaged.value ? '🌊 海嘯級星體震盪' : '🌊 巨型震盪波'
      playSound(120, 'sawtooth', 0.4)
      const distWave = Math.sqrt(dx*dx + dy*dy)
      const spdWave = 0.9 + speedBonus
      bossBullets.value.push({ x: boss.x, y: boss.y + 30, vx: (dx/distWave) * spdWave, vy: (dy/distWave) * spdWave, radius: isBossRaged.value ? 30 : 24 })
      break
  }
}

const updateSnakeMovement = () => {
  if (snake.value.length === 0) return
  
  const newHead = { x: snake.value[0].x + velocityX, y: snake.value[0].y + velocityY }

  if (newHead.x >= gameCanvas.value.width) newHead.x = 0
  if (newHead.x < 0) newHead.x = gameCanvas.value.width - gridSize
  if (newHead.y >= gameCanvas.value.height) newHead.y = 0
  if (newHead.y < 0) newHead.y = gameCanvas.value.height - gridSize

  if (playerInvincibleTimer === 0) {
    for (let i = 1; i < snake.value.length; i++) {
      if (newHead.x === snake.value[i].x && newHead.y === snake.value[i].y) {
        takeDamage()
        break
      }
    }
  }

  if (gameState.value !== 'PLAYING') return
  
  snake.value.unshift(newHead)
  
  let ateFoodIndex = -1
  for (let i = 0; i < foods.value.length; i++) {
    if (newHead.x === foods.value[i].x && newHead.y === foods.value[i].y) {
      ateFoodIndex = i
      break
    }
  }

  if (ateFoodIndex !== -1) {
    const item = foods.value[ateFoodIndex]
    foods.value.splice(ateFoodIndex, 1) 
    
    if (item.type === 'heal') {
      playerHp.value = Math.min(5, playerHp.value + 1)
      playSound(650, 'sine', 0.12)
    } else if (item.type === 'shield') {
      shieldCount.value = Math.min(2, shieldCount.value + 1)
      playSound(784, 'sine', 0.15)
    } else {
      playSound(523, 'sine', 0.08)
    }
    
    if (item.type === 'normal') {
      shootPlayerBullet(newHead.x, newHead.y)
    }
  }

  while (snake.value.length > playerHp.value) {
    snake.value.pop()
  }
}

const takeDamage = () => {
  if (playerInvincibleTimer > 0) return

  if (shieldCount.value > 0) {
    shieldCount.value--
    playerInvincibleTimer = 400 
    playSound(330, 'triangle', 0.2)
    if (snake.value.length > 0) {
      createExplosion(snake.value[0].x + 10, snake.value[0].y + 10, 12, '#54a0ff')
    }
    return 
  }

  playerHp.value--
  playerInvincibleTimer = INVINCIBLE_DURATION 
  playSound(200, 'sawtooth', 0.35)
  
  if (snake.value.length > 0) {
    createExplosion(snake.value[0].x + 10, snake.value[0].y + 10, 15, '#ff4757')
  }
  
  while (snake.value.length > playerHp.value) {
    snake.value.pop()
  }
  
  if (playerHp.value <= 0 || snake.value.length === 0) {
    gameState.value = 'GAMEOVER'
    cancelAnimationFrame(animationFrameId)
    if (bgmNode) clearTimeout(bgmNode)
  }
}

const shootPlayerBullet = (startX, startY) => {
  playerBullets.value.push({ x: startX + 10, y: startY + 10, targetX: boss.x, targetY: boss.y + 30, speed: 9.5 })
}

const updateProjectiles = () => {
  for (let i = playerBullets.value.length - 1; i >= 0; i--) {
    const b = playerBullets.value[i]
    const dx = b.targetX - b.x
    const dy = b.targetY - b.y
    const dist = Math.sqrt(dx * dx + dy * dy)

    if (dist < 18) {
      bossHp.value = Math.max(0, bossHp.value - 6) 
      playSound(480, 'triangle', 0.05)
      createExplosion(b.x, b.y, 8, '#ffa502')
      bossShakeTimer = 150 
      
      if (bossHp.value < 30 && !isBossRaged.value) {
        isBossRaged.value = true
        rageAlertTimer.value = 3500 
        playSound(90, 'sawtooth', 0.8) 
      }

      playerBullets.value.splice(i, 1)
      if (bossHp.value <= 0) {
        gameState.value = 'WIN'
        cancelAnimationFrame(animationFrameId)
        if (bgmNode) clearTimeout(bgmNode)
      }
    } else {
      b.x += (dx / dist) * b.speed
      b.y += (dy / dist) * b.speed
    }
  }

  if (playerInvincibleTimer === 0) {
    for (let i = bossBullets.value.length - 1; i >= 0; i--) {
      const b = bossBullets.value[i]
      b.x += b.vx
      b.y += b.vy

      if (b.x < -40 || b.x > gameCanvas.value.width + 40 || b.y > gameCanvas.value.height + 40 || b.y < -40) {
        bossBullets.value.splice(i, 1)
        continue
      }

      let hitHead = false
      if (snake.value.length > 0) {
        const head = snake.value[0]
        const centerX = head.x + gridSize / 2
        const centerY = head.y + gridSize / 2
        
        const distX = Math.abs(b.x - centerX)
        const distY = Math.abs(b.y - centerY)

        if (distX <= (gridSize/2 + b.radius) && distY <= (gridSize/2 + b.radius)) {
          hitHead = true
        }
      }

      if (hitHead) {
        bossBullets.value.splice(i, 1)
        takeDamage() 
      }
    }
  }
}

const createExplosion = (x, y, count, color) => {
  for (let i = 0; i < count; i++) {
    particles.value.push({ x: x, y: y, vx: (Math.random() - 0.5) * 6, vy: (Math.random() - 0.5) * 6, life: 18, color: color })
  }
}

const updateParticles = () => {
  for (let i = particles.value.length - 1; i >= 0; i--) {
    const p = particles.value[i]
    p.x += p.vx; p.y += p.vy; p.life--
    if (p.life <= 0) particles.value.splice(i, 1)
  }
}

const drawGame = () => {
  if (!ctx || !gameCanvas.value) return
  ctx.clearRect(0, 0, gameCanvas.value.width, gameCanvas.value.height)

  // 繪製資源球
  foods.value.forEach(f => {
    ctx.fillStyle = f.color
    ctx.shadowBlur = 10
    ctx.shadowColor = f.color
    ctx.beginPath()
    ctx.arc(f.x + 10, f.y + 10, f.type === 'normal' ? 6 : 8, 0, Math.PI * 2)
    ctx.fill()
    ctx.shadowBlur = 0
  })

  let shouldDrawPlayer = true
  if (playerInvincibleTimer > 0) {
    shouldDrawPlayer = Math.floor(performance.now() / 60) % 2 === 0
  }
  
  // 繪製蛇身與【修改２：白色粗邊框護盾】
  snake.value.forEach((part, index) => {
    if (!shouldDrawPlayer) return 

    // 1. 繪製原本的彩色蛇身方塊
    if (isSpeedUpActive) {
      ctx.fillStyle = index === 0 ? '#00d2d3' : '#54a0ff'
    } else {
      ctx.fillStyle = index === 0 ? '#2ed573' : '#26af5f'
    }
    ctx.fillRect(part.x, part.y, gridSize - 2, gridSize - 2)

    // 2.核心優化：有護盾時，直接在每一節方塊周圍補上極其厚實的亮白色粗框
    if (shieldCount.value > 0) {
      ctx.save()
      ctx.strokeStyle = '#ffffff' // 粗白色邊框
      ctx.lineWidth = 3           // 線條加粗
      ctx.shadowBlur = 8          // 白色霓虹環境微光
      ctx.shadowColor = '#ffffff'
      // 在方塊的外圍繪製線條框
      ctx.strokeRect(part.x + 1, part.y + 1, gridSize - 4, gridSize - 4)
      ctx.restore()
    }
  });

  // 魔王繪製
  let bossX = boss.x
  let bossY = boss.y
  if (bossShakeTimer > 0) {
    bossX += (Math.random() - 0.5) * 8
    bossY += (Math.random() - 0.5) * 8
  }

  if (isBossRaged.value) {
    ctx.fillStyle = '#5f27cd' 
    ctx.shadowBlur = 20
    ctx.shadowColor = '#ff0000' 
  } else {
    ctx.fillStyle = bossShakeTimer > 0 ? '#ff6b81' : '#ff4757'
  }
  ctx.fillRect(bossX - 30, bossY, boss.width, boss.height)
  ctx.shadowBlur = 0

  // 魔王雙眼
  if (snake.value.length > 0) {
    const head = snake.value[0]
    const angleLeft = Math.atan2(head.y - (bossY + 20), head.x - (bossX - 10))
    const angleRight = Math.atan2(head.y - (bossY + 20), head.x - (bossX + 10))
    
    const eyeLookDist = 3
    const lookXLeft = Math.cos(angleLeft) * eyeLookDist
    const lookYLeft = Math.sin(angleLeft) * eyeLookDist
    const lookXRight = Math.cos(angleRight) * eyeLookDist
    const lookYRight = Math.sin(angleRight) * eyeLookDist

    ctx.fillStyle = '#fff'
    ctx.fillRect(bossX - 16, bossY + 15, 10, 10)
    ctx.fillRect(bossX + 6, bossY + 15, 10, 10)
    
    ctx.fillStyle = isBossRaged.value ? '#ff0000' : '#000'
    ctx.fillRect(bossX - 14 + lookXLeft, bossY + 17 + lookYLeft, 6, 6)
    ctx.fillRect(bossX + 8 + lookXRight, bossY + 17 + lookYRight, 6, 6)
  }

  // 魔王蓄力環
  if (bossCastRingRadius > 0) {
    ctx.strokeStyle = isBossRaged.value 
      ? `rgba(255, 0, 0, ${1 - bossCastRingRadius / bossCastRingMax})`
      : `rgba(255, 71, 87, ${1 - bossCastRingRadius / bossCastRingMax})`
    ctx.lineWidth = isBossRaged.value ? 5 : 3
    ctx.beginPath()
    ctx.arc(bossX, bossY + 30, bossCastRingRadius, 0, Math.PI * 2)
    ctx.stroke()
  }

  // 魔王血條
  ctx.fillStyle = '#333'
  ctx.fillRect(boss.x - 45, boss.y - 15, 90, 6)
  ctx.fillStyle = isBossRaged.value ? '#ff0000' : '#ff4757'
  ctx.fillRect(boss.x - 45, boss.y - 15, 90 * (bossHp.value / 100), 6)

  // 繪製子彈
  playerBullets.value.forEach(b => {
    ctx.fillStyle = '#fff'
    ctx.beginPath()
    ctx.arc(b.x, b.y, 4, 0, Math.PI * 2)
    ctx.fill()
  })

  bossBullets.value.forEach(b => {
    ctx.fillStyle = isBossRaged.value ? '#ff00fa' : (b.radius > 15 ? '#ff6b81' : '#ff4757')
    ctx.shadowBlur = b.radius > 15 ? 16 : 6
    ctx.shadowColor = isBossRaged.value ? '#ff00fa' : '#ff4757'
    ctx.beginPath()
    ctx.arc(b.x, b.y, b.radius, 0, Math.PI * 2)
    ctx.fill()
    ctx.shadowBlur = 0
  })

  particles.value.forEach(p => {
    ctx.fillStyle = p.color
    ctx.globalAlpha = p.life / 18
    ctx.fillRect(p.x, p.y, 3, 3)
  })
  ctx.globalAlpha = 1.0
}

onUnmounted(() => {
  cancelAnimationFrame(animationFrameId)
  if (bgmNode) clearTimeout(bgmNode)
  window.removeEventListener('keydown', handleKeyDown)
})
</script>

<style>
body { margin: 0; padding: 0; background-color: #121212; display: flex; justify-content: center; align-items: center; height: 100vh; user-select: none; }
#game-container { position: relative; border: 4px solid #3c3c3c; background-color: #181818; padding: 20px; border-radius: 12px; box-shadow: 0 0 30px rgba(0,0,0,0.7); font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; color: #fff; width: 640px; margin: 0 auto; }
.screen { text-align: center; padding: 35px 15px; }
h1 { color: #ff4757; font-size: 28px; margin-top: 0; letter-spacing: 1px; }
.description { color: #aaa; line-height: 1.7; margin-bottom: 25px; font-size: 14px; }
button { background-color: #ff4757; color: white; border: none; padding: 12px 28px; font-size: 16px; font-weight: bold; cursor: pointer; border-radius: 6px; transition: all 0.2s ease; box-shadow: 0 4px 10px rgba(255, 71, 87, 0.4); }
button:hover { background-color: #ff6b81; transform: translateY(-2px); box-shadow: 0 6px 15px rgba(255, 71, 87, 0.6); }
#hud { display: flex; justify-content: space-between; margin-bottom: 12px; font-weight: bold; font-size: 15px; background: #262626; padding: 12px 20px; border-radius: 6px; border-left: 5px solid #ff4757; }
.hp-text { color: #2ed573 !important; }
.shield-text { color: #54a0ff !important; text-shadow: 0 0 4px #54a0ff; }
.rage-text { color: #ff0000 !important; animation: blink-red 0.5s infinite alternate; font-weight: 900; }
#skills-hud { display: flex; gap: 15px; margin-bottom: 12px; }
.skill-box { flex: 1; position: relative; background: #222; border: 2px solid #444; padding: 10px; border-radius: 6px; font-size: 14px; font-weight: bold; text-align: center; overflow: hidden; color: #888; }
.skill-box.ready { color: #fff; border-color: #00d2d3; box-shadow: 0 0 8px rgba(0, 210, 211, 0.2); }
.skill-box .key { background: #444; padding: 2px 6px; border-radius: 4px; color: #fff; margin-right: 5px; }
.skill-box.ready .key { background: #00d2d3; }
.cd-overlay { position: absolute; top: 0; left: 0; height: 100%; background: rgba(255, 255, 255, 0.15); pointer-events: none; }
.cd-text { position: absolute; right: 15px; top: 10px; color: #ff4757; }
.canvas-wrapper { position: relative; display: block; }
canvas { background-color: #0a0a0a; border: 2px solid #3a3a3a; display: block; border-radius: 4px; }
.skill-flash { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0, 210, 211, 0.3); pointer-events: none; opacity: 0; }
.flash-active { opacity: 1; }
.skill-banner-zone { position: absolute; top: 10px; left: 10px; right: 10px; pointer-events: none; display: flex; flex-direction: column; gap: 8px; }
.boss-skill-banner { background: rgba(0, 0, 0, 0.65); padding: 6px 12px; border-radius: 4px; font-size: 13px; font-weight: bold; border-left: 4px solid #ff4757; width: fit-content; }
.boss-skill-banner.rage-banner { border-left-color: #ff00fa; background: rgba(95, 39, 205, 0.4); text-shadow: 0 0 5px #ff00fa; }
.rage-alert-box { background: rgba(255, 0, 0, 0.85); color: #fff; text-align: center; padding: 10px; font-weight: 900; font-size: 16px; border-radius: 4px; border: 2px solid #fff; animation: shake-alert 0.3s infinite, flash-alert 0.5s infinite alternate; box-shadow: 0 0 20px #ff0000; }

.overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: rgba(10, 10, 10, 0.9); display: flex; flex-direction: column; justify-content: center; align-items: center; border-radius: 4px; }
.paused-overlay h2 { color: #ffca28 !important; text-shadow: 0 0 10px rgba(254, 202, 40, 0.5); }
.button-group { display: flex; gap: 20px; margin-top: 10px; }
.resume-btn { background-color: #2ed573; box-shadow: 0 4px 10px rgba(46, 213, 115, 0.4); }
.resume-btn:hover { background-color: #26af5f; box-shadow: 0 6px 15px rgba(46, 213, 115, 0.6); }
.restart-btn { background-color: #ff4757; box-shadow: 0 4px 10px rgba(255, 71, 87, 0.4); }
.restart-btn:hover { background-color: #ff6b81; box-shadow: 0 6px 15px rgba(255, 71, 87, 0.6); }

.overlay h2 { font-size: 38px; color: #ff4757; margin-bottom: 12px; text-shadow: 0 0 10px rgba(255,71,87,0.5); }
.overlay.win h2 { color: #2ed573; text-shadow: 0 0 10px rgba(46,213,115,0.5); }
.overlay p { font-size: 16px; margin-bottom: 25px; color: #bbb; }
@keyframes blink-red { from { opacity: 1; } to { opacity: 0.4; } }
@keyframes flash-alert { from { background: rgba(255, 0, 0, 0.9); } to { background: rgba(150, 0, 0, 0.9); } }
@keyframes shake-alert {
  0% { transform: translate(0, 0); }
  20% { transform: translate(-2px, 2px); }
  40% { transform: translate(-2px, -2px); }
  60% { transform: translate(2px, 2px); }
  80% { transform: translate(2px, -2px); }
  100% { transform: translate(0, 0); }
}
</style>