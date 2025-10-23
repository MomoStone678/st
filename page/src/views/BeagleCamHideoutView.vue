<script setup>
import { ref, reactive, computed, onMounted, onUnmounted } from 'vue'

const canvasRef = ref(null)
const context = ref(null)
const animationFrame = ref(null)
const lastTimestamp = ref(0)

const room = {
  width: 900,
  height: 540,
  padding: 32,
}

const keys = reactive({
  up: false,
  down: false,
  left: false,
  right: false,
  hide: false,
})

const state = reactive({
  gameStatus: 'running', // running | caught | success
  timeSurvived: 0,
  snapshots: 0,
  hidden: false,
  flashTimer: 0,
  vignettePulse: 0,
  message: '利用家具的阴影躲避主人的视线，撑过 75 秒。',
  messageTimer: 4,
})

const baseDogSpeed = 180

const player = reactive({
  x: 140,
  y: 380,
  width: 42,
  height: 26,
  facing: 0,
})

const owner = reactive({
  x: 640,
  y: 200,
  radius: 28,
  speed: 90,
  heading: Math.PI,
  visionAngle: Math.PI / 2.4,
  visionDistance: 260,
  targetIndex: 0,
  waitTimer: 0,
})

const patrolPoints = [
  { x: 640, y: 200 },
  { x: 760, y: 360 },
  { x: 520, y: 430 },
  { x: 420, y: 220 },
  { x: 520, y: 130 },
]

const furniture = [
  {
    name: '床头柜',
    x: 260,
    y: 130,
    width: 80,
    height: 60,
    color: '#c8b197',
    solid: true,
  },
  {
    name: '床架',
    x: 320,
    y: 180,
    width: 280,
    height: 140,
    color: '#b08f74',
    solid: true,
  },
  {
    name: '床垫',
    x: 332,
    y: 192,
    width: 256,
    height: 84,
    color: '#f0e8dc',
    solid: false,
  },
  {
    name: '床下阴影',
    x: 332,
    y: 250,
    width: 256,
    height: 64,
    color: 'rgba(20, 20, 35, 0.65)',
    solid: false,
    hideZone: true,
  },
  {
    name: '衣柜',
    x: 110,
    y: 80,
    width: 140,
    height: 140,
    color: '#a37b5a',
    solid: true,
  },
  {
    name: '沙发',
    x: 130,
    y: 360,
    width: 160,
    height: 90,
    color: '#8796a5',
    solid: true,
  },
  {
    name: '茶几',
    x: 520,
    y: 320,
    width: 120,
    height: 60,
    color: '#d7c4aa',
    solid: true,
  },
  {
    name: '地毯',
    x: 480,
    y: 120,
    width: 180,
    height: 120,
    color: 'rgba(250, 235, 210, 0.8)',
    solid: false,
  },
]

const hideZones = furniture.filter((item) => item.hideZone)
const solidFurniture = furniture.filter((item) => item.solid)

const formattedTime = computed(() => {
  const seconds = Math.max(0, Math.floor(state.timeSurvived))
  const mm = String(Math.floor(seconds / 60)).padStart(2, '0')
  const ss = String(seconds % 60).padStart(2, '0')
  return `${mm}:${ss}`
})

const survivalProgress = computed(() => Math.min(100, (state.timeSurvived / 75) * 100))

function resetGame() {
  state.gameStatus = 'running'
  state.timeSurvived = 0
  state.snapshots = 0
  state.hidden = false
  state.flashTimer = 0
  state.vignettePulse = 0
  state.message = '利用家具的阴影躲避主人的视线，撑过 75 秒。'
  state.messageTimer = 4
  player.x = 140
  player.y = 380
  player.facing = 0
  owner.x = patrolPoints[0].x
  owner.y = patrolPoints[0].y
  owner.heading = Math.PI
  owner.targetIndex = 1
  owner.waitTimer = 0.8
  lastTimestamp.value = 0
}

function handleKeyDown(event) {
  const key = event.key.toLowerCase()
  if (['arrowup', 'arrowdown', 'arrowleft', 'arrowright', ' '].includes(event.key.toLowerCase()) || ['w', 'a', 's', 'd'].includes(key)) {
    event.preventDefault()
  }

  switch (key) {
    case 'arrowup':
    case 'w':
      keys.up = true
      break
    case 'arrowdown':
    case 's':
      keys.down = true
      break
    case 'arrowleft':
    case 'a':
      keys.left = true
      break
    case 'arrowright':
    case 'd':
      keys.right = true
      break
    case ' ':
      keys.hide = true
      break
    case 'f':
      triggerSnapshot()
      break
    case 'r':
      if (state.gameStatus !== 'running') {
        resetGame()
      }
      break
    default:
      break
  }
}

function handleKeyUp(event) {
  const key = event.key.toLowerCase()
  switch (key) {
    case 'arrowup':
    case 'w':
      keys.up = false
      break
    case 'arrowdown':
    case 's':
      keys.down = false
      break
    case 'arrowleft':
    case 'a':
      keys.left = false
      break
    case 'arrowright':
    case 'd':
      keys.right = false
      break
    case ' ':
      keys.hide = false
      break
    default:
      break
  }
}

function triggerSnapshot() {
  if (state.gameStatus !== 'running') return
  state.flashTimer = 0.18
  state.snapshots += 1
  state.message = '咔嚓！镜头捕捉到新的藏身点。'
  state.messageTimer = 1.6
}

function rectsIntersect(a, b) {
  return (
    a.x < b.x + b.width &&
    a.x + a.width > b.x &&
    a.y < b.y + b.height &&
    a.y + a.height > b.y
  )
}

function resolveCollision(rect, obstacle) {
  if (!rectsIntersect(rect, obstacle)) return

  const overlapX1 = rect.x + rect.width - obstacle.x
  const overlapX2 = obstacle.x + obstacle.width - rect.x
  const overlapY1 = rect.y + rect.height - obstacle.y
  const overlapY2 = obstacle.y + obstacle.height - rect.y
  const minOverlapX = Math.min(overlapX1, overlapX2)
  const minOverlapY = Math.min(overlapY1, overlapY2)

  if (minOverlapX < minOverlapY) {
    if (rect.x < obstacle.x) {
      rect.x -= overlapX1
    } else {
      rect.x += overlapX2
    }
  } else {
    if (rect.y < obstacle.y) {
      rect.y -= overlapY1
    } else {
      rect.y += overlapY2
    }
  }
}

function updateGame(delta) {
  if (state.gameStatus !== 'running') return

  state.timeSurvived += delta
  state.vignettePulse = (state.vignettePulse + delta * 2) % (Math.PI * 2)

  if (state.flashTimer > 0) {
    state.flashTimer = Math.max(0, state.flashTimer - delta)
  }

  if (state.messageTimer > 0) {
    state.messageTimer = Math.max(0, state.messageTimer - delta)
    if (state.messageTimer === 0) {
      state.message = state.hidden ? '继续潜伏，别让主人发现。' : '寻找阴影，准备随时钻到床下。'
    }
  }

  const move = { x: 0, y: 0 }
  if (keys.up) move.y -= 1
  if (keys.down) move.y += 1
  if (keys.left) move.x -= 1
  if (keys.right) move.x += 1

  const isMoving = move.x !== 0 || move.y !== 0
  const speedModifier = state.hidden ? 0.45 : 1
  const speed = baseDogSpeed * speedModifier

  let nextRect = {
    x: player.x,
    y: player.y,
    width: player.width,
    height: player.height,
  }

  if (isMoving) {
    const magnitude = Math.hypot(move.x, move.y) || 1
    const dirX = move.x / magnitude
    const dirY = move.y / magnitude
    nextRect.x += dirX * speed * delta
    nextRect.y += dirY * speed * delta
    player.facing = Math.atan2(dirY, dirX)
  }

  // 房间边界
  nextRect.x = Math.max(room.padding, Math.min(nextRect.x, room.width - room.padding - player.width))
  nextRect.y = Math.max(room.padding, Math.min(nextRect.y, room.height - room.padding - player.height))

  solidFurniture.forEach((item) => resolveCollision(nextRect, item))

  player.x = nextRect.x
  player.y = nextRect.y

  const playerCenter = {
    x: player.x + player.width / 2,
    y: player.y + player.height / 2,
  }

  // 藏身判定
  const insideHideZone = hideZones.some((zone) => rectsIntersect(nextRect, zone))
  state.hidden = insideHideZone && keys.hide

  // 巡逻主人
  if (owner.waitTimer > 0) {
    owner.waitTimer = Math.max(0, owner.waitTimer - delta)
  } else {
    const target = patrolPoints[owner.targetIndex]
    const dx = target.x - owner.x
    const dy = target.y - owner.y
    const distance = Math.hypot(dx, dy)

    if (distance < 6) {
      owner.targetIndex = (owner.targetIndex + 1) % patrolPoints.length
      owner.waitTimer = 0.6 + Math.random() * 0.6
    } else if (distance > 0) {
      const dirX = dx / distance
      const dirY = dy / distance
      owner.x += dirX * owner.speed * delta
      owner.y += dirY * owner.speed * delta
      owner.heading = Math.atan2(dirY, dirX)
    }
  }

  // 视野检测
  if (!state.hidden) {
    const ownerCenter = { x: owner.x, y: owner.y }
    const diffX = playerCenter.x - ownerCenter.x
    const diffY = playerCenter.y - ownerCenter.y
    const distanceToPlayer = Math.hypot(diffX, diffY)

    if (distanceToPlayer < owner.visionDistance) {
      const angleToPlayer = Math.atan2(diffY, diffX)
      let angleDiff = angleToPlayer - owner.heading
      angleDiff = Math.atan2(Math.sin(angleDiff), Math.cos(angleDiff))
      if (Math.abs(angleDiff) < owner.visionAngle / 2) {
        state.gameStatus = 'caught'
        state.message = '糟糕，被发现了！按 R 再试一次。'
      }
    }
  }

  // 胜利条件
  if (state.gameStatus === 'running' && state.timeSurvived >= 75) {
    state.gameStatus = 'success'
    state.message = '成功躲避主人！按 R 重新挑战。'
  }
}

function drawGame() {
  const ctx = context.value
  if (!ctx) return

  ctx.clearRect(0, 0, room.width, room.height)

  // 地板
  const gradient = ctx.createLinearGradient(0, 0, 0, room.height)
  gradient.addColorStop(0, '#3b2f2f')
  gradient.addColorStop(1, '#251d1c')
  ctx.fillStyle = gradient
  ctx.fillRect(0, 0, room.width, room.height)

  // 木质纹理线条
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.03)'
  ctx.lineWidth = 1
  for (let y = room.padding; y < room.height - room.padding; y += 24) {
    ctx.beginPath()
    ctx.moveTo(room.padding, y)
    ctx.lineTo(room.width - room.padding, y)
    ctx.stroke()
  }

  // 家具
  furniture.forEach((item) => {
    ctx.fillStyle = item.color
    ctx.fillRect(item.x, item.y, item.width, item.height)
    if (item.name === '床架') {
      ctx.strokeStyle = 'rgba(0, 0, 0, 0.15)'
      ctx.lineWidth = 3
      ctx.strokeRect(item.x, item.y, item.width, item.height)
    }
    if (item.hideZone) {
      ctx.fillStyle = 'rgba(0, 0, 0, 0.35)'
      ctx.fillRect(item.x, item.y, item.width, item.height)
    }
  })

  // 主人的视野
  ctx.save()
  ctx.translate(owner.x, owner.y)
  ctx.fillStyle = 'rgba(255, 210, 120, 0.16)'
  ctx.beginPath()
  ctx.moveTo(0, 0)
  ctx.arc(0, 0, owner.visionDistance, owner.heading - owner.visionAngle / 2, owner.heading + owner.visionAngle / 2)
  ctx.closePath()
  ctx.fill()

  ctx.fillStyle = '#ffe0a0'
  ctx.beginPath()
  ctx.arc(0, 0, owner.radius, 0, Math.PI * 2)
  ctx.fill()
  ctx.fillStyle = '#2d1f1f'
  ctx.beginPath()
  ctx.arc(-8, -4, 5, 0, Math.PI * 2)
  ctx.arc(10, -4, 5, 0, Math.PI * 2)
  ctx.fill()
  ctx.fillRect(-6, 8, 12, 6)
  ctx.restore()

  // 比格犬（俯视）
  ctx.save()
  ctx.translate(player.x + player.width / 2, player.y + player.height / 2)
  ctx.rotate(player.facing)
  ctx.fillStyle = '#b47a3c'
  ctx.beginPath()
  ctx.ellipse(0, 0, player.width / 2, player.height / 2, 0, 0, Math.PI * 2)
  ctx.fill()
  ctx.fillStyle = '#f2d7b1'
  ctx.beginPath()
  ctx.ellipse(8, 0, player.width / 4, player.height / 2.4, 0, 0, Math.PI * 2)
  ctx.fill()
  ctx.fillStyle = '#4a2c16'
  ctx.beginPath()
  ctx.arc(-player.width / 2, -player.height / 2.2, 8, 0, Math.PI * 2)
  ctx.arc(-player.width / 2, player.height / 2.2, 8, 0, Math.PI * 2)
  ctx.fill()
  ctx.fillStyle = 'rgba(15, 15, 20, 0.55)'
  ctx.fillRect(6, -6, 12, 12)
  ctx.restore()

  // 相机界面 HUD
  ctx.save()
  ctx.strokeStyle = 'rgba(255, 255, 255, 0.45)'
  ctx.lineWidth = 2
  ctx.strokeRect(room.padding - 12, room.padding - 12, room.width - room.padding * 2 + 24, room.height - room.padding * 2 + 24)

  ctx.strokeStyle = 'rgba(255, 255, 255, 0.25)'
  ctx.beginPath()
  ctx.moveTo(room.width / 2 - 40, room.height / 2)
  ctx.lineTo(room.width / 2 + 40, room.height / 2)
  ctx.moveTo(room.width / 2, room.height / 2 - 40)
  ctx.lineTo(room.width / 2, room.height / 2 + 40)
  ctx.stroke()

  ctx.fillStyle = '#ff5a5a'
  ctx.beginPath()
  ctx.arc(room.padding + 24, room.padding + 16, 6, 0, Math.PI * 2)
  ctx.fill()
  ctx.fillStyle = 'rgba(255, 255, 255, 0.9)'
  ctx.font = '14px "Segoe UI", sans-serif'
  ctx.fillText('REC', room.padding + 36, room.padding + 20)
  ctx.fillText(`00:${String(state.snapshots).padStart(2, '0')}`, room.width - room.padding - 70, room.padding + 20)
  ctx.fillText(`${Math.round(survivalProgress.value)}% STEALTH`, room.width - room.padding - 150, room.height - room.padding + 18)
  ctx.restore()

  // 隐藏时的暗角
  if (state.hidden) {
    const darkness = 0.55 + Math.sin(state.vignettePulse) * 0.1
    const radialGradient = ctx.createRadialGradient(room.width / 2, room.height / 2, 50, room.width / 2, room.height / 2, 380)
    radialGradient.addColorStop(0, 'rgba(0, 0, 0, 0)')
    radialGradient.addColorStop(1, `rgba(0, 0, 0, ${darkness})`)
    ctx.fillStyle = radialGradient
    ctx.fillRect(0, 0, room.width, room.height)
  }

  if (state.flashTimer > 0) {
    const alpha = Math.min(0.6, state.flashTimer * 4)
    ctx.fillStyle = `rgba(255, 255, 255, ${alpha})`
    ctx.fillRect(0, 0, room.width, room.height)
  }

  if (state.gameStatus !== 'running') {
    ctx.fillStyle = 'rgba(0, 0, 0, 0.55)'
    ctx.fillRect(0, 0, room.width, room.height)
    ctx.fillStyle = '#ffffff'
    ctx.font = 'bold 42px "Segoe UI", sans-serif'
    ctx.textAlign = 'center'
    ctx.fillText(state.gameStatus === 'success' ? '躲避成功！' : '被主人发现', room.width / 2, room.height / 2 - 10)
    ctx.font = '18px "Segoe UI", sans-serif'
    ctx.fillText('按 R 重新开始', room.width / 2, room.height / 2 + 32)
  }
}

function loop(timestamp) {
  if (!context.value) return
  if (!lastTimestamp.value) {
    lastTimestamp.value = timestamp
  }
  const delta = (timestamp - lastTimestamp.value) / 1000
  lastTimestamp.value = timestamp

  updateGame(delta)
  drawGame()

  animationFrame.value = requestAnimationFrame(loop)
}

onMounted(() => {
  resetGame()
  const canvas = canvasRef.value
  if (!canvas) return
  canvas.width = room.width
  canvas.height = room.height
  context.value = canvas.getContext('2d')
  window.addEventListener('keydown', handleKeyDown)
  window.addEventListener('keyup', handleKeyUp)
  animationFrame.value = requestAnimationFrame(loop)
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeyDown)
  window.removeEventListener('keyup', handleKeyUp)
  if (animationFrame.value) {
    cancelAnimationFrame(animationFrame.value)
  }
})
</script>

<template>
  <div class="beagle-game-page">
    <section class="info-panel">
      <h1>🎥 比格犬潜行日志</h1>
      <p class="subtitle">以相机视角记录比格犬在家中穿梭与床下潜伏的紧张瞬间。</p>
      <div class="hud">
        <div class="hud-item">
          <span class="label">潜伏计时</span>
          <span class="value">{{ formattedTime }}</span>
        </div>
        <div class="hud-item">
          <span class="label">快门次数</span>
          <span class="value">{{ state.snapshots }}</span>
        </div>
        <div class="hud-item" :class="{ active: state.hidden }">
          <span class="label">床下隐蔽</span>
          <span class="value">{{ state.hidden ? '✓ 已潜伏' : '未进入阴影' }}</span>
        </div>
      </div>
      <div class="progress-bar">
        <div class="progress" :style="{ width: `${survivalProgress}%` }"></div>
      </div>
      <ul class="instructions">
        <li><kbd>WASD</kbd> / 方向键：移动比格犬</li>
        <li><kbd>Space</kbd>：钻入床下并保持隐蔽</li>
        <li><kbd>F</kbd>：按下快门，记录新的藏身点</li>
        <li>躲避巡逻的主人，撑过 75 秒即可成功</li>
      </ul>
    </section>

    <div class="canvas-wrapper">
      <canvas ref="canvasRef" class="game-canvas"></canvas>
      <transition name="fade">
        <div v-if="state.message" class="status-message">{{ state.message }}</div>
      </transition>
    </div>
  </div>
</template>

<style scoped>
.beagle-game-page {
  display: flex;
  flex-direction: column;
  gap: 24px;
  padding: 32px 24px 48px;
  max-width: 1200px;
  margin: 0 auto;
  color: #1e1d2d;
}

.info-panel {
  background: linear-gradient(135deg, rgba(255, 255, 255, 0.9), rgba(240, 242, 255, 0.95));
  border-radius: 20px;
  padding: 24px;
  box-shadow: 0 18px 45px rgba(49, 55, 86, 0.12);
}

.info-panel h1 {
  margin: 0;
  font-size: 2rem;
  letter-spacing: 0.02em;
}

.subtitle {
  margin-top: 6px;
  margin-bottom: 16px;
  color: rgba(39, 46, 82, 0.75);
}

.hud {
  display: flex;
  gap: 18px;
  flex-wrap: wrap;
  margin-bottom: 12px;
}

.hud-item {
  background: rgba(255, 255, 255, 0.8);
  border-radius: 14px;
  padding: 12px 18px;
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.65), 0 8px 22px rgba(61, 72, 124, 0.12);
  min-width: 140px;
}

.hud-item.active {
  background: rgba(100, 211, 152, 0.22);
  box-shadow: inset 0 1px 0 rgba(255, 255, 255, 0.5), 0 8px 22px rgba(75, 181, 117, 0.18);
}

.label {
  display: block;
  font-size: 0.82rem;
  color: rgba(35, 40, 71, 0.6);
}

.value {
  font-size: 1.4rem;
  font-weight: 600;
  color: #2d325a;
}

.progress-bar {
  position: relative;
  height: 10px;
  background: rgba(68, 87, 142, 0.12);
  border-radius: 999px;
  overflow: hidden;
  margin-bottom: 20px;
}

.progress {
  height: 100%;
  background: linear-gradient(90deg, #57b894, #2d8fdd);
  border-radius: inherit;
  transition: width 0.3s ease;
}

.instructions {
  list-style: none;
  margin: 0;
  padding: 0;
  display: flex;
  gap: 18px;
  flex-wrap: wrap;
  color: rgba(39, 46, 82, 0.75);
}

.instructions li {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.95rem;
  background: rgba(255, 255, 255, 0.7);
  padding: 8px 12px;
  border-radius: 12px;
}

.instructions kbd {
  background: rgba(28, 31, 51, 0.82);
  color: #fff;
  font-size: 0.78rem;
  padding: 4px 8px;
  border-radius: 6px;
  box-shadow: inset 0 -2px 0 rgba(0, 0, 0, 0.2);
}

.canvas-wrapper {
  position: relative;
  border-radius: 24px;
  overflow: hidden;
  box-shadow: 0 24px 65px rgba(18, 21, 45, 0.35);
}

.game-canvas {
  width: 100%;
  display: block;
  background: #111318;
}

.status-message {
  position: absolute;
  bottom: 18px;
  left: 50%;
  transform: translateX(-50%);
  background: rgba(12, 14, 30, 0.75);
  color: #f7f9ff;
  padding: 12px 20px;
  border-radius: 999px;
  box-shadow: 0 8px 18px rgba(0, 0, 0, 0.35);
  font-size: 0.98rem;
  letter-spacing: 0.01em;
  backdrop-filter: blur(6px);
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.4s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

@media (max-width: 960px) {
  .beagle-game-page {
    padding: 24px 16px 36px;
  }

  .info-panel h1 {
    font-size: 1.6rem;
  }

  .instructions {
    flex-direction: column;
    align-items: flex-start;
  }

  .canvas-wrapper {
    border-radius: 18px;
  }
}
</style>
