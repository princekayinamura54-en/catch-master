<script setup>import { ref, computed, onBeforeUnmount } from 'vue'

const gameArea = ref(null)
const score = ref(0)
const combo = ref(0)
const timer = ref(45)
const status = ref('Press Start to begin')
const message = ref('Click the target quickly after it arrives to build combos.')
const isRunning = ref(false)
const isPowerZone = ref(false)
const targetStyle = ref({ left: '0px', top: '0px', transition: 'none' })
const targetSize = 70
const lastClickTime = ref(0)
const arrivalTimer = ref(null)
const moveEndTimer = ref(null)
const gameTimer = ref(null)
const bonus50 = ref(false)
const bonus100 = ref(false)
const bonus150 = ref(false)

const multiplier = computed(() => 1 + Math.max(0, combo.value - 1) * 0.5)
const currentSpeed = computed(() => Math.min(750, 220 + score.value * 2))
const timeRemaining = computed(() => Math.max(0, timer.value))
const gameLabel = computed(() => isRunning.value ? 'Game Active' : 'Ready')

function getGameRect() {
  const area = gameArea.value?.getBoundingClientRect()
  if (!area) return null
  return {
    width: Math.max(300, area.width - targetSize),
    height: Math.max(200, area.height - targetSize),
  }
}

function clamp(value, min, max) {
  return Math.min(max, Math.max(min, value))
}

function randomTargetPosition() {
  const rect = getGameRect()
  if (!rect) return { x: 0, y: 0 }
  return {
    x: Math.floor(Math.random() * rect.width),
    y: Math.floor(Math.random() * rect.height),
  }
}

function checkPowerZone(x, y) {
  const area = gameArea.value?.getBoundingClientRect()
  if (!area) return false
  const margin = 100
  return x <= margin || x >= area.width - targetSize - margin || y <= margin || y >= area.height - targetSize - margin
}

function updateTargetPosition(x, y, durationMs) {
  targetStyle.value.transition = durationMs > 0 ? `left ${durationMs}ms linear, top ${durationMs}ms linear` : 'none'
  targetStyle.value.left = `${x}px`
  targetStyle.value.top = `${y}px`
}

function scheduleArrival(durationMs) {
  clearTimeout(arrivalTimer.value)
  arrivalTimer.value = window.setTimeout(() => {
    if (!isRunning.value) return
    status.value = 'Target arrived! Click within 3 seconds or lose points.'
    arrivalTimer.value = window.setTimeout(handleMiss, 3000)
  }, durationMs)
}

function moveTargetToNextPosition(initial = false) {
  if (!gameArea.value) return
  clearTimers()

  const nextPos = randomTargetPosition()
  isPowerZone.value = checkPowerZone(nextPos.x, nextPos.y)

  const currentX = parseFloat(targetStyle.value.left) || 0
  const currentY = parseFloat(targetStyle.value.top) || 0
  const deltaX = nextPos.x - currentX
  const deltaY = nextPos.y - currentY
  const distance = Math.hypot(deltaX, deltaY)
  const speed = Math.max(220, currentSpeed.value)
  const durationMs = initial ? 0 : Math.max(200, Math.round((distance / speed) * 1000))

  updateTargetPosition(nextPos.x, nextPos.y, durationMs)

  if (initial) {
    status.value = 'Get ready...'
    arrivalTimer.value = window.setTimeout(() => {
      status.value = 'Target arrived! Click within 3 seconds or lose points.'
      arrivalTimer.value = window.setTimeout(handleMiss, 3000)
    }, 50)
  } else {
    status.value = `Moving to the next spot at ${Math.round(speed)} px/s...`
    scheduleArrival(durationMs)
  }
}

function handleMiss() {
  if (!isRunning.value) return
  score.value = Math.max(0, score.value - 5)
  combo.value = 0
  status.value = 'Missed! -5 points and combo reset.'
  moveTargetToNextPosition()
}

function awardBonusTime() {
  if (score.value >= 50 && !bonus50.value) {
    bonus50.value = true
    timer.value += 5
    message.value = 'Bonus +5 seconds earned at 50 points!'
  }
  if (score.value >= 100 && !bonus100.value) {
    bonus100.value = true
    timer.value += 5
    message.value = 'Bonus +5 seconds earned at 100 points!'
  }
  if (score.value >= 150 && !bonus150.value) {
    bonus150.value = true
    timer.value += 5
    message.value = 'Bonus +5 seconds earned at 150 points!'
  }
}

function clickTarget() {
  if (!isRunning.value) return
  const now = performance.now()
  const isFastClick = lastClickTime.value && now - lastClickTime.value <= 1000
  combo.value = isFastClick ? combo.value + 1 : 1
  lastClickTime.value = now

  let points = 10
  if (isPowerZone.value) {
    points *= 2
  }
  points = Math.round(points * multiplier.value)
  score.value += points
  status.value = `Hit! +${points} points ${isPowerZone.value ? '(Power Zone)' : ''}`
  awardBonusTime()
  moveTargetToNextPosition()
}

function startGame() {
  if (isRunning.value) return
  isRunning.value = true
  score.value = 0
  combo.value = 0
  timer.value = 45
  status.value = 'Catch the target to start your combo!'
  message.value = 'Corners are Power Zones: double points.'
  bonus50.value = false
  bonus100.value = false
  bonus150.value = false
  lastClickTime.value = 0
  moveTargetToNextPosition(true)

  clearInterval(gameTimer.value)
  gameTimer.value = window.setInterval(() => {
    if (!isRunning.value) return
    timer.value -= 1
    if (timer.value <= 0) {
      timer.value = 0
      stopGame()
    }
  }, 1000)
}

function stopGame() {
  isRunning.value = false
  status.value = 'Game over — press Start to play again.'
  clearTimers()
  clearInterval(gameTimer.value)
}

function clearTimers() {
  clearTimeout(arrivalTimer.value)
  clearTimeout(moveEndTimer.value)
}

function resetGame() {
  stopGame()
  score.value = 0
  combo.value = 0
  timer.value = 45
  status.value = 'Press Start to begin'
  message.value = 'Click Start to play Catch Master.'
  isPowerZone.value = false
  targetStyle.value = { left: '0px', top: '0px', transition: 'none' }
}

onBeforeUnmount(() => {
  clearTimers()
  clearInterval(gameTimer.value)
})
</script>

<template>  <section class="game-shell">
    <header class="game-header">
      <div>
        <h1>Catch Master</h1>
        <p>Click the moving target before it settles. Fast clicks build combo multipliers.</p>
      </div>
      <div class="metrics">
        <div class="metric"><span>Score</span><strong>{{ score }}</strong></div>
        <div class="metric"><span>Combo</span><strong>{{ combo }}</strong></div>
        <div class="metric"><span>Timer</span><strong>{{ timeRemaining }}s</strong></div>
        <div class="metric"><span>Speed</span><strong>{{ Math.round(currentSpeed) }} px/s</strong></div>
      </div>
    </header>

    <div ref="gameArea" class="game-area">
      <button
        type="button"
        class="target"
        :class="{ power: isPowerZone }"
        :style="targetStyle"
        @click.stop="clickTarget"
        aria-label="Target"
      >
        {{ isPowerZone ? '🔥' : '🎯' }}
      </button>
      <div class="power-tip">Power Zone when target is near the edge.</div>
    </div>
    <footer class="game-footer">
      <div class="status">{{ status }}</div>
      <div class="message">{{ message }}</div>
      <div class="controls">
        <button @click="startGame" :disabled="isRunning">Start</button>
        <button @click="resetGame">Reset</button>
      </div>
      <div class="milestones">
        <span :class="{ achieved: bonus50 }">50+</span>
        <span :class="{ achieved: bonus100 }">100+</span>
        <span :class="{ achieved: bonus150 }">150+</span>
      </div>
    </footer>
  </section></template>

<style scoped>
.game-shell {
  max-width: 940px;
  margin: 24px auto;
  padding: 16px;
  border: 2px solid #ddd;
  border-radius: 18px;
  background: #fff;
  box-shadow: 0 20px 40px rgba(0, 0, 0, 0.08);
}

.game-header {
  display: flex;
  flex-wrap: wrap;
  justify-content: space-between;
  gap: 20px;
  margin-bottom: 24px;
}

.game-header h1 {
  margin: 0 0 6px;
  font-size: 2rem;
}

.game-header p {
  margin: 0;
  color: #555;
}

.metrics {
  display: grid;
  grid-template-columns: repeat(4, minmax(100px, 1fr));
  gap: 12px;
}

.metric {
  background: #f7f5ff;
  border: 1px solid #e5e1fb;
  border-radius: 14px;
  padding: 12px 14px;
  text-align: left;
}

.metric span {
  display: block;
  font-size: 0.85rem;
  color: #777;
}

.metric strong {
  display: block;
  margin-top: 6px;
  font-size: 1.25rem;
  color: #240d4c;
}

.game-area {
  position: relative;
  height: 420px;
  border: 2px solid #ddd;
  border-radius: 18px;
  background: linear-gradient(180deg, #faf8ff 0%, #f4f1ff 100%);
  overflow: hidden;
}

.target {
  position: absolute;
  width: 70px;
  height: 70px;
  border-radius: 50%;
  border: 4px solid #5c3ab3;
  background: #7d5fff;
  color: white;
  font-size: 1.15rem;
  cursor: pointer;
  display: grid;
  place-items: center;
  box-shadow: 0 14px 28px rgba(92, 58, 179, 0.25);
}

.target.power {
  background: #ef4444;
  border-color: #b91c1c;
  box-shadow: 0 14px 28px rgba(239, 68, 68, 0.28);
}

.power-tip {
  position: absolute;
  bottom: 12px;
  left: 16px;
  color: #4b4b4b;
  font-size: 0.95rem;
}

.game-footer {
  margin-top: 22px;
  display: grid;
  gap: 14px;
}

.status {
  font-weight: 600;
  color: #2d2b33;
}

.message {
  color: #6c6b78;
}

.controls {
  display: flex;
  gap: 12px;
  flex-wrap: wrap;
}

button {
  min-width: 100px;
  padding: 12px 18px;
  border: none;
  border-radius: 999px;
  color: white;
  background: #5c3ab3;
  cursor: pointer;
  font-weight: 600;
}

button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

button:nth-child(2) {
  background: #6b7280;
}

.milestones {
  display: flex;
  gap: 10px;
  color: #444;
  font-weight: 600;
}

.milestones span {
  padding: 6px 12px;
  border: 1px solid #d1d5db;
  border-radius: 999px;
  background: #f9fafb;
}

.milestones span.achieved {
  border-color: #10b981;
  background: #d1fae5;
  color: #065f46;
}

@media (max-width: 820px) {
  .game-header {
    flex-direction: column;
  }
  .metrics {
    grid-template-columns: repeat(2, minmax(100px, 1fr));
  }
}
</style>
