<template>
  <div class="game-page">
    <header>
      <button class="btn-back" @click="goBack">← 返回大厅</button>
      <h1>五子棋 · {{ difficultyLabel }}</h1>
      <div class="header-info">
        <span>悔棋: {{ undoRemaining }}/2</span>
      </div>
    </header>

    <div class="game-area">
      <div class="board-container">
        <!-- 对话气泡 -->
        <div v-if="showBubble && currentBubble" class="bubble-wrapper">
          <div class="bubble" :class="currentBubble.sender">
            <span class="bubble-avatar">{{ currentBubble.sender === 'player' ? '🧑' : '🤖' }}</span>
            <div class="bubble-text">{{ currentBubble.text }}</div>
          </div>
        </div>

        <canvas
          ref="canvasRef"
          :width="canvasSize"
          :height="canvasSize"
          @click="handleClick"
          @mousemove="handleMouseMove"
          @mouseleave="hoverPos = null"
        ></canvas>
      </div>

      <div class="side-panel">
        <div class="status-box" :class="statusClass">
          <div class="status-icon">{{ statusIcon }}</div>
          <div class="status-text">{{ message }}</div>
          <div v-if="turnLabel" class="turn-label">{{ turnLabel }}</div>
        </div>

        <div class="controls">
          <button v-if="status === 'PLAYER_TURN'" class="btn-ctrl" @click="doPause">
            ⏸ 暂停
          </button>
          <button v-if="status === 'PAUSED'" class="btn-ctrl resume" @click="doResume">
            ▶ 继续
          </button>
          <button
            v-if="['PLAYER_TURN', 'GRAY_TURN', 'AI_WIN', 'WHITE_WIN', 'GRAY_WIN'].includes(status) && undoRemaining > 0"
            class="btn-ctrl undo"
            @click="doUndo"
            :disabled="undoLoading"
          >
            ↩ 悔棋 ({{ undoRemaining }})
          </button>
          <button
            v-if="['PLAYER_WIN', 'AI_WIN', 'WHITE_WIN', 'GRAY_WIN', 'DRAW'].includes(status)"
            class="btn-ctrl new-game"
            @click="goBack"
          >
            🆕 新游戏
          </button>
        </div>

        <div class="legend">
          <div class="legend-item"><span class="dot black"></span> 你 (黑)</div>
          <div class="legend-item"><span class="dot white"></span> AI (白)</div>
          <div v-if="gameMode === 'THREE' || gameMode === 'THREE_OBSTACLE'" class="legend-item"><span class="dot gray"></span> AI (灰)</div>
          <div v-if="gameMode === 'OBSTACLE' || gameMode === 'THREE_OBSTACLE'" class="legend-item">
            <span class="dot terrain-obstacle"></span> 假山（可包围消除）
          </div>
          <div v-if="gameMode === 'OBSTACLE' || gameMode === 'THREE_OBSTACLE'" class="legend-item">
            <span class="dot terrain-water"></span> 水潭（落子滑走）
          </div>
        </div>

        <!-- 玩家快捷语回击 -->
        <div class="player-phrases">
          <div class="phrases-title">快捷语回击</div>
          <div class="phrases-grid">
            <button
              v-for="(phrase, idx) in playerPhrases"
              :key="idx"
              class="phrase-btn"
              @click="onPlayerPhrase(phrase)"
            >{{ phrase }}</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted, onUnmounted, computed } from 'vue'
import { useRouter } from 'vue-router'
import { gameApi } from '../api'

const router = useRouter()
const canvasRef = ref(null)

// 棋盘配置
const PADDING = 30

const gameMode = ref('TWO')
const boardSize = computed(() => gameMode.value === 'THREE' || gameMode.value === 'THREE_OBSTACLE' ? 25 : 15)
const cellSize = computed(() => gameMode.value === 'THREE' || gameMode.value === 'THREE_OBSTACLE' ? 22 : 36)
const stoneRadius = computed(() => gameMode.value === 'THREE' || gameMode.value === 'THREE_OBSTACLE' ? 9 : 15)
const canvasSize = computed(() => PADDING * 2 + cellSize.value * (boardSize.value - 1))

// 星位点（动态计算）
const starPoints = computed(() => {
  const size = boardSize.value
  const edge = Math.max(3, Math.floor(size / 5))
  const center = Math.floor(size / 2)
  return [edge, center, size - 1 - edge]
})

function createEmptyBoard(size) {
  return Array.from({ length: size }, () => Array(size).fill(0))
}

// 游戏状态
const board = ref(createEmptyBoard(15))
const terrain = ref(null) // 地形数据（障碍物模式）
const status = ref('PLAYER_TURN')
const difficulty = ref('EASY')
const undoRemaining = ref(2)
const message = ref('请落子')
const hoverPos = ref(null)
const undoLoading = ref(false)
const slideFrom = ref(null) // 水潭滑动动画起点
const slideTo = ref(null)   // 水潭滑动动画终点
let slideAnimTimer = null

// ===== 对话系统 =====
const dialogues = ref([])           // 对话历史记录
const currentBubble = ref(null)      // 当前显示的气泡 { sender, text, avatar }
const showBubble = ref(false)        // 气泡显示控制
let thinkTimer = null                 // 思考计时器
let randomTimer = null                // 随机发言计时器
let bubbleTimer = null                // 气泡自动消失计时器

// 文案池：AI 催促（玩家思考过久）
const aiHurryUpPhrases = [
  '快点儿，我等的花儿都谢了🌸',
  '你是去泡茶了吗？赶紧的！',
  '思考人生呢？该你落子啦～',
  '等你等到天长地久...',
  'speed 点！我棋力已经溢出啦！',
  '再不下子我就当你认输咯😏',
]

// 文案池：AI 嘲讽（AI 落子后占优）
const aiTauntPhrases = [
  '就这水平？呵呵～',
  '你这步棋... 挺有创意的🤭',
  '我让让你吧，可惜你不领情',
  '这局我已经赢定了！',
  '你的防线漏洞百出呀',
  '我赌你撑不过10手',
]

// 文案池：AI 欢呼（AI 形成连珠）
const aiCheerPhrases = [
  '哈哈哈这步妙手！',
  '四连珠！你挡得住吗～',
  '感觉良好，胜利在望！',
  '你是不是慌了？我感觉到了😆',
  '我这棋力，没谁了！',
]

// 文案池：AI 随机评论
const aiRandomPhrases = [
  '这棋盘真好看，可惜你要输了',
  '要不要我教你几招？',
  '认真点嘛，我在认真下呢',
  '你的棋风... 很特别',
  '我在想今天晚饭吃什么🍜',
  '这局挺有意思的，继续！',
]

// 玩家快捷回击语
const playerPhrases = [
  '别得意太早！',
  '你等着，我马上翻盘',
  '哼，让你几步而已',
  '失误失误，下次不会了',
  '你AI也会得意忘形？',
  '放马过来吧！',
]

// 工具函数：从数组随机取一条
function randomPick(arr) {
  return arr[Math.floor(Math.random() * arr.length)]
}

// 显示气泡（3秒后自动消失）
function showBubbleMsg(sender, text, avatar = null) {
  currentBubble.value = { sender, text, avatar }
  showBubble.value = true
  if (bubbleTimer) clearTimeout(bubbleTimer)
  bubbleTimer = setTimeout(() => {
    showBubble.value = false
  }, 3000)
}

// 添加对话记录
function addDialogue(sender, text) {
  dialogues.value.push({
    id: Date.now(),
    sender,
    text,
    time: new Date().toLocaleTimeString(),
  })
  // 保留最近20条
  if (dialogues.value.length > 20) dialogues.value.shift()
}

// ===== 对话系统结束 =====

// ===== 计时器逻辑 =====
// 监听游戏状态变化：玩家回合启动时启动思考计时器
watch(status, (newVal) => {
  // 清除旧计时器
  if (thinkTimer) { clearTimeout(thinkTimer); thinkTimer = null }
  if (newVal === 'PLAYER_TURN') {
    startThinkTimer()
  }
  // 游戏结束时清除随机计时器
  if (['PLAYER_WIN', 'AI_WIN', 'WHITE_WIN', 'GRAY_WIN', 'DRAW'].includes(newVal)) {
    if (randomTimer) { clearTimeout(randomTimer); randomTimer = null }
  }
}, { immediate: true })

function startThinkTimer() {
  if (thinkTimer) clearTimeout(thinkTimer)
  thinkTimer = setTimeout(() => {
    // 10秒思考过久，AI 催促
    const text = randomPick(aiHurryUpPhrases)
    showBubbleMsg('ai', text)
    addDialogue('ai', text)
  }, 10000)
}

function startRandomTimer() {
  if (randomTimer) clearTimeout(randomTimer)
  // 15~30秒随机间隔
  const delay = 15000 + Math.random() * 15000
  randomTimer = setTimeout(() => {
    const text = randomPick(aiRandomPhrases)
    showBubbleMsg('ai', text)
    addDialogue('ai', text)
    // 循环：继续下一个随机计时
    startRandomTimer()
  }, delay)
}

function clearAllTimers() {
  if (thinkTimer) { clearTimeout(thinkTimer); thinkTimer = null }
  if (randomTimer) { clearTimeout(randomTimer); randomTimer = null }
  if (bubbleTimer) { clearTimeout(bubbleTimer); bubbleTimer = null }
  clearSlideAnim()
}

// ===== 计时器逻辑结束 =====

// 计算属性
const difficultyLabel = computed(() => {
  const map = { EASY: '初级', MEDIUM: '中级', HARD: '高级' }
  return map[difficulty.value] || '初级'
})
const statusClass = computed(() => {
  const map = {
    PLAYER_WIN: 'win', AI_WIN: 'lose', WHITE_WIN: 'lose', GRAY_WIN: 'lose',
    DRAW: 'draw', AI_THINKING: 'thinking', GRAY_TURN: 'thinking', PAUSED: 'paused'
  }
  return map[status.value] || ''
})
const statusIcon = computed(() => {
  const map = {
    PLAYER_TURN: '🤔', AI_THINKING: '💭', GRAY_TURN: '💭',
    PLAYER_WIN: '🎉', AI_WIN: '😢', WHITE_WIN: '🤖', GRAY_WIN: '🤖',
    DRAW: '🤝', PAUSED: '⏸'
  }
  return map[status.value] || '🎯'
})
const turnLabel = computed(() => {
  const map = {
    PLAYER_TURN: '⚫ 你的回合（黑子）',
    AI_THINKING: '⚪ 白子AI 回合',
    GRAY_TURN: '🔘 灰子AI 回合',
  }
  return map[status.value] || ''
})

onMounted(() => {
  // 先读取模式以确定棋盘大小
  gameMode.value = sessionStorage.getItem('gameMode') || 'TWO'
  const saved = sessionStorage.getItem('gameState')
  if (saved) {
    const state = JSON.parse(saved)
    board.value = state.board || createEmptyBoard(boardSize.value)
    terrain.value = state.terrain || null
    status.value = state.status || 'PLAYER_TURN'
    difficulty.value = state.difficulty || 'EASY'
    undoRemaining.value = state.undoRemaining ?? 2
    message.value = state.message || '请落子'
    // 从保存状态恢复 gameMode
    if (state.gameMode) gameMode.value = state.gameMode
    sessionStorage.removeItem('gameState')
  } else {
    difficulty.value = sessionStorage.getItem('difficulty') || 'EASY'
    board.value = createEmptyBoard(boardSize.value)
  }
  drawBoard()
  // 启动随机对话计时器
  startRandomTimer()
})

onUnmounted(() => {
  // 清除所有计时器
  clearAllTimers()
  clearSlideAnim()
  // 保存当前状态
  const state = {
    board: board.value,
    terrain: terrain.value,
    status: status.value,
    difficulty: difficulty.value,
    undoRemaining: undoRemaining.value,
    message: message.value,
    gameMode: gameMode.value
  }
  sessionStorage.setItem('gameState', JSON.stringify(state))
})

// 绘制棋盘
function drawBoard() {
  const canvas = canvasRef.value
  if (!canvas) return
  const ctx = canvas.getContext('2d')
  const cs = canvasSize.value
  const bs = boardSize.value
  const cl = cellSize.value
  const sp = starPoints.value

  // === 赛博朋克风格背景 ===
  // 深色渐变背景
  const bgGrad = ctx.createRadialGradient(cs / 2, cs / 2, 0, cs / 2, cs / 2, cs * 0.7)
  bgGrad.addColorStop(0, '#0d1b2a')
  bgGrad.addColorStop(1, '#050a18')
  ctx.fillStyle = bgGrad
  ctx.fillRect(0, 0, cs, cs)

  // 添加微妙的网格背景光晕
  ctx.save()
  ctx.globalAlpha = 0.03
  const glowGrad = ctx.createRadialGradient(cs / 2, cs / 2, 0, cs / 2, cs / 2, cs * 0.6)
  glowGrad.addColorStop(0, '#00f0ff')
  glowGrad.addColorStop(1, 'transparent')
  ctx.fillStyle = glowGrad
  ctx.fillRect(0, 0, cs, cs)
  ctx.restore()

  // 添加细微电路纹理（赛博朋克风格）
  ctx.save()
  ctx.globalAlpha = 0.04
  ctx.strokeStyle = '#00f0ff'
  ctx.lineWidth = 0.5
  // 绘制随机电路线（基于固定种子）
  const seed = 42
  for (let i = 0; i < 12; i++) {
    const sx = PADDING + ((seed * (i + 1) * 7) % (bs - 1)) * cl
    const sy = PADDING + ((seed * (i + 1) * 13) % (bs - 1)) * cl
    const ex = sx + (((seed * (i + 1) * 3) % 3) - 1) * cl * (2 + ((seed * i) % 3))
    const ey = sy + (((seed * (i + 1) * 5) % 3) - 1) * cl * (2 + ((seed * i * 2) % 3))
    ctx.beginPath()
    ctx.moveTo(sx, sy)
    ctx.lineTo(ex, sy)
    ctx.lineTo(ex, ey)
    ctx.stroke()
    // 节点
    ctx.beginPath()
    ctx.arc(sx, sy, 1.5, 0, Math.PI * 2)
    ctx.fillStyle = '#00f0ff'
    ctx.fill()
  }
  ctx.restore()

  // === 霓虹网格线 ===
  ctx.strokeStyle = 'rgba(0, 240, 255, 0.25)'
  ctx.lineWidth = bs === 25 ? 0.6 : 1.2
  for (let i = 0; i < bs; i++) {
    const pos = PADDING + i * cl
    // 横线
    ctx.beginPath()
    ctx.moveTo(PADDING, pos)
    ctx.lineTo(PADDING + cl * (bs - 1), pos)
    ctx.stroke()
    // 竖线
    ctx.beginPath()
    ctx.moveTo(pos, PADDING)
    ctx.lineTo(pos, PADDING + cl * (bs - 1))
    ctx.stroke()
  }

  // 网格线发光效果（边缘线加粗发光）
  ctx.save()
  ctx.globalAlpha = 0.08
  ctx.strokeStyle = '#00f0ff'
  ctx.lineWidth = 0.5
  ctx.beginPath()
  ctx.moveTo(PADDING, PADDING)
  ctx.lineTo(PADDING + cl * (bs - 1), PADDING)
  ctx.lineTo(PADDING + cl * (bs - 1), PADDING + cl * (bs - 1))
  ctx.lineTo(PADDING, PADDING + cl * (bs - 1))
  ctx.closePath()
  ctx.stroke()
  ctx.restore()

  // === 星位点：霓虹粉色 ===
  sp.forEach(r => {
    sp.forEach(c => {
      const cx = PADDING + r * cl
      const cy = PADDING + c * cl
      const radius = bs === 25 ? 3 : 5
      // 外发光
      ctx.save()
      ctx.beginPath()
      ctx.arc(cx, cy, radius + 3, 0, Math.PI * 2)
      ctx.fillStyle = 'rgba(255, 0, 128, 0.15)'
      ctx.fill()
      ctx.restore()
      // 星位圆点
      ctx.beginPath()
      ctx.arc(cx, cy, radius, 0, Math.PI * 2)
      const starGrad = ctx.createRadialGradient(cx - 1, cy - 1, 0, cx, cy, radius)
      starGrad.addColorStop(0, '#ff66cc')
      starGrad.addColorStop(1, '#ff0080')
      ctx.fillStyle = starGrad
      ctx.fill()
    })
  })

  // 地形渲染（障碍物模式 / 三人障碍物模式）
  if (terrain.value && (gameMode.value === 'OBSTACLE' || gameMode.value === 'THREE_OBSTACLE')) {
    for (let r = 0; r < Math.min(bs, terrain.value.length); r++) {
      for (let c = 0; c < Math.min(bs, terrain.value[r] ? terrain.value[r].length : 0); c++) {
        const t = terrain.value[r][c]
        if (t === 4) drawObstacle(ctx, r, c)  // 假山
        if (t === 5) drawWater(ctx, r, c)       // 水潭
      }
    }
  }

  // 棋子
  for (let r = 0; r < bs; r++) {
    for (let c = 0; c < bs; c++) {
      if (board.value[r] && board.value[r][c] !== 0) {
        drawStone(ctx, r, c, board.value[r][c])
      }
    }
  }

  // 悬停预览
  if (hoverPos.value && status.value === 'PLAYER_TURN') {
    const hr = hoverPos.value.r, hc = hoverPos.value.c
    if (board.value[hr]?.[hc] === 0) {
      if (terrain.value && terrain.value[hr]?.[hc] === 4) {
        // 假山：霓虹红色叉号
        drawStone(ctx, hr, hc, 1, 0.15)
        ctx.save()
        const hx = PADDING + hc * cellSize.value
        const hy = PADDING + hr * cellSize.value
        const r = stoneRadius.value * 0.6
        ctx.strokeStyle = 'rgba(255, 0, 128, 0.7)'
        ctx.lineWidth = 2
        ctx.beginPath()
        ctx.moveTo(hx - r, hy - r)
        ctx.lineTo(hx + r, hy + r)
        ctx.moveTo(hx + r, hy - r)
        ctx.lineTo(hx - r, hy + r)
        ctx.stroke()
        ctx.restore()
      } else if (terrain.value && terrain.value[hr]?.[hc] === 5) {
        // 水潭：霓虹蓝色半透明预览
        drawStone(ctx, hr, hc, 1, 0.35)
        ctx.save()
        const hx = PADDING + hc * cellSize.value
        const hy = PADDING + hr * cellSize.value
        ctx.fillStyle = 'rgba(0, 240, 255, 0.5)'
        ctx.font = `${stoneRadius.value * 1.5}px sans-serif`
        ctx.textAlign = 'center'
        ctx.textBaseline = 'middle'
        ctx.fillText('◇', hx, hy)
        ctx.restore()
      } else {
        // 赛博朋克风格悬停预览：霓虹青色光标
        const hx = PADDING + hc * cellSize.value
        const hy = PADDING + hr * cellSize.value
        const r = stoneRadius.value
        ctx.save()
        // 外发光
        ctx.beginPath()
        ctx.arc(hx, hy, r + 5, 0, Math.PI * 2)
        ctx.fillStyle = 'rgba(0, 240, 255, 0.08)'
        ctx.fill()
        // 霓虹圆环
        ctx.beginPath()
        ctx.arc(hx, hy, r, 0, Math.PI * 2)
        ctx.strokeStyle = 'rgba(0, 240, 255, 0.5)'
        ctx.lineWidth = 2
        ctx.stroke()
        // 中心小点
        ctx.beginPath()
        ctx.arc(hx, hy, 3, 0, Math.PI * 2)
        ctx.fillStyle = 'rgba(0, 240, 255, 0.6)'
        ctx.fill()
        ctx.restore()
      }
    }
  }
}

function drawStone(ctx, row, col, color, alpha = 1) {
  const cl = cellSize.value
  const x = PADDING + col * cl
  const y = PADDING + row * cl
  const radius = stoneRadius.value
  const hlOffset = Math.max(2, radius * 0.25)
  const hlSize = Math.max(2, radius * 0.33)

  ctx.save()
  ctx.globalAlpha = alpha

  // === 赛博朋克风格棋子 ===
  // 外发光效果
  let glowColor, stoneColor1, stoneColor2, highlightColor, glowAlpha
  if (color === 1) {
    // 黑子 → 深紫蓝色 + 青色霓虹发光
    glowColor = 'rgba(0, 240, 255, 0.3)'
    stoneColor1 = '#1a0a3e'
    stoneColor2 = '#0a0520'
    highlightColor = 'rgba(0, 240, 255, 0.25)'
    glowAlpha = 0.4
  } else if (color === 3) {
    // 灰子 → 紫色霓虹
    glowColor = 'rgba(255, 0, 200, 0.3)'
    stoneColor1 = '#4a2060'
    stoneColor2 = '#1a0a30'
    highlightColor = 'rgba(255, 0, 200, 0.3)'
    glowAlpha = 0.35
  } else {
    // 白子 → 白蓝霓虹
    glowColor = 'rgba(100, 200, 255, 0.3)'
    stoneColor1 = '#e8f4ff'
    stoneColor2 = '#80b8e0'
    highlightColor = 'rgba(200, 240, 255, 0.6)'
    glowAlpha = 0.3
  }

  // 绘制外发光
  ctx.beginPath()
  ctx.arc(x, y, radius + 4, 0, Math.PI * 2)
  ctx.fillStyle = glowColor
  ctx.fill()

  // 绘制棋子主体
  const gradient = ctx.createRadialGradient(x - hlOffset, y - hlOffset, 1, x, y, radius)
  gradient.addColorStop(0, stoneColor1)
  gradient.addColorStop(1, stoneColor2)

  ctx.beginPath()
  ctx.arc(x, y, radius, 0, Math.PI * 2)
  ctx.fillStyle = gradient
  ctx.fill()

  // 霓虹描边
  ctx.beginPath()
  ctx.arc(x, y, radius, 0, Math.PI * 2)
  ctx.strokeStyle = glowColor
  ctx.lineWidth = 1
  ctx.stroke()

  // 高光
  ctx.beginPath()
  ctx.arc(x - hlOffset, y - hlOffset, hlSize, 0, Math.PI * 2)
  ctx.fillStyle = highlightColor
  ctx.fill()

  ctx.restore()
}

// ===== 地形绘制（赛博朋克风格）=====
function drawObstacle(ctx, row, col) {
  const cl = cellSize.value
  const x = PADDING + col * cl
  const y = PADDING + row * cl
  const r = stoneRadius.value

  ctx.save()
  // 假山：赛博朋克风格 - 深色金属质感
  const grad = ctx.createLinearGradient(x - r, y - r, x + r, y + r)
  grad.addColorStop(0, '#2a2a4a')
  grad.addColorStop(0.3, '#1a1a3a')
  grad.addColorStop(0.7, '#0a0a2a')
  grad.addColorStop(1, '#15153a')

  ctx.beginPath()
  // 不规则多边形（使用行列种子保持形状一致）
  const pts = 7
  const seed = row * 31 + col * 17
  for (let i = 0; i < pts; i++) {
    const angle = (i / pts) * Math.PI * 2 - Math.PI / 2
    const seg = ((seed * (i + 1) + 37) % 30) / 100
    const rr = r * (0.7 + seg)
    const px = x + Math.cos(angle) * rr
    const py = y + Math.sin(angle) * rr
    if (i === 0) ctx.moveTo(px, py)
    else ctx.lineTo(px, py)
  }
  ctx.closePath()
  ctx.fillStyle = grad
  ctx.fill()
  // 霓虹描边
  ctx.strokeStyle = 'rgba(255, 0, 128, 0.5)'
  ctx.lineWidth = 1.5
  ctx.stroke()

  // 障碍物图标 🔷
  ctx.fillStyle = 'rgba(255, 0, 200, 0.8)'
  ctx.font = `${r * 1.2}px sans-serif`
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillText('⬡', x, y)

  // 外发光
  ctx.beginPath()
  ctx.arc(x, y, r + 3, 0, Math.PI * 2)
  ctx.fillStyle = 'rgba(255, 0, 200, 0.08)'
  ctx.fill()
  ctx.restore()
}

function drawWater(ctx, row, col) {
  const cl = cellSize.value
  const x = PADDING + col * cl
  const y = PADDING + row * cl
  const r = stoneRadius.value

  ctx.save()
  // 水潭：赛博朋克风格 - 霓虹蓝色
  const grad = ctx.createRadialGradient(x - r * 0.2, y - r * 0.2, r * 0.1, x, y, r)
  grad.addColorStop(0, 'rgba(0, 240, 255, 0.6)')
  grad.addColorStop(0.5, 'rgba(0, 150, 255, 0.4)')
  grad.addColorStop(1, 'rgba(0, 50, 150, 0.3)')

  ctx.beginPath()
  ctx.arc(x, y, r * 0.9, 0, Math.PI * 2)
  ctx.fillStyle = grad
  ctx.fill()
  // 霓虹描边
  ctx.strokeStyle = 'rgba(0, 240, 255, 0.6)'
  ctx.lineWidth = 1.5
  ctx.stroke()

  // 水面波纹（赛博风格）
  ctx.beginPath()
  ctx.ellipse(x, y - r * 0.25, r * 0.5, r * 0.15, 0, 0, Math.PI * 2)
  ctx.strokeStyle = 'rgba(0, 240, 255, 0.4)'
  ctx.lineWidth = 0.8
  ctx.stroke()

  // 水潭图标 💎
  ctx.fillStyle = 'rgba(0, 240, 255, 0.9)'
  ctx.font = `${r * 1.0}px sans-serif`
  ctx.textAlign = 'center'
  ctx.textBaseline = 'middle'
  ctx.fillText('◇', x, y)

  // 外发光
  ctx.beginPath()
  ctx.arc(x, y, r + 3, 0, Math.PI * 2)
  ctx.fillStyle = 'rgba(0, 240, 255, 0.08)'
  ctx.fill()
  ctx.restore()
}

function clearSlideAnim() {
  if (slideAnimTimer) { clearTimeout(slideAnimTimer); slideAnimTimer = null }
  slideFrom.value = null
  slideTo.value = null
}

function getBoardPos(clientX, clientY) {
  const canvas = canvasRef.value
  const rect = canvas.getBoundingClientRect()
  const cs = canvasSize.value
  const bs = boardSize.value
  const cl = cellSize.value
  const scaleX = cs / rect.width
  const scaleY = cs / rect.height
  const x = (clientX - rect.left) * scaleX
  const y = (clientY - rect.top) * scaleY

  const col = Math.round((x - PADDING) / cl)
  const row = Math.round((y - PADDING) / cl)

  if (row >= 0 && row < bs && col >= 0 && col < bs) {
    // 检查是否接近交叉点
    const cx = PADDING + col * cl
    const cy = PADDING + row * cl
    const threshold = stoneRadius.value + 1
    if (Math.abs(x - cx) < threshold && Math.abs(y - cy) < threshold) {
      return { r: row, c: col }
    }
  }
  return null
}

function handleMouseMove(e) {
  const pos = getBoardPos(e.clientX, e.clientY)
  hoverPos.value = pos
  drawBoard()
}

async function handleClick(e) {
  if (status.value !== 'PLAYER_TURN') return
  const pos = getBoardPos(e.clientX, e.clientY)
  if (!pos) return

  // 假山处不能点击落子
  if (terrain.value && terrain.value[pos.r]?.[pos.c] === 4) return
  // 已有棋子的位置不能落子
  if (board.value[pos.r][pos.c] !== 0) return

  // 记住原始点击位置（用于水潭滑动动画）
  const clickRow = pos.r, clickCol = pos.c

  // 清除思考计时器
  if (thinkTimer) { clearTimeout(thinkTimer); thinkTimer = null }
  // 不直接改状态，等后端返回后再更新（避免玩家点击后立即显示"思考中"）
  drawBoard()

  try {
    const res = await gameApi.move(clickRow, clickCol)
    if (res.code === 200) {
      const data = res.data

      // 更新棋盘和地形
      board.value = data.board
      terrain.value = data.terrain || null
      undoRemaining.value = data.undoRemaining

      // 双人/障碍物模式：玩家落子后返回 AI_TURN，前端自动请求 AI 落子
      // 三人/三人障碍物模式：玩家落子后返回 GRAY_TURN，前端自动请求灰AI落子
      if (data.status === 'AI_TURN') {
        status.value = 'AI_TURN'
        message.value = '白子AI 思考中...'
        drawBoard()
        // 延迟 400ms 再请求 AI 落子，让玩家感知到 AI 在"思考"
        await new Promise(r => setTimeout(r, 400))
        await doAiMove()
      } else {
        status.value = data.status
        message.value = data.message || ''
      }

      // 水潭滑动动画：检测落子位置是否变化
      if ((gameMode.value === 'OBSTACLE' || gameMode.value === 'THREE_OBSTACLE') && data.status !== 'PLAYER_WIN') {
        // 找到实际落子位置（与点击位置不同的最近黑子）
        for (let r = 0; r < board.value.length; r++) {
          for (let c = 0; c < board.value[r].length; c++) {
            if (board.value[r][c] === 1 && !(r === clickRow && c === clickCol)) {
              // 检查是否是新落子（附近没有旧黑子逻辑...简化：检查是否在点击位置附近）
              const dist = Math.abs(r - clickRow) + Math.abs(c - clickCol)
              if (dist === 1 && data.message && data.message.includes('滑入')) {
                slideFrom.value = { r: clickRow, c: clickCol }
                slideTo.value = { r, c }
                drawBoard()
                // 延迟播放滑动动画
                if (slideAnimTimer) clearTimeout(slideAnimTimer)
                slideAnimTimer = setTimeout(() => {
                  drawBoard()
                  clearSlideAnim()
                }, 600)
              }
            }
          }
        }
      }

      // AI 落子后对话
      if (data.status === 'PLAYER_TURN') {
        if (Math.random() < 0.5) {
          const text = randomPick(aiTauntPhrases)
          showBubbleMsg('ai', text)
          addDialogue('ai', text)
        }
      } else if (data.status.includes('WIN')) {
        const text = randomPick(aiCheerPhrases)
        showBubbleMsg('ai', text)
        addDialogue('ai', text)
      }

      // 三人/三人障碍物模式：白子落完后自动触发灰子回合
      if (data.status === 'GRAY_TURN') {
        status.value = 'GRAY_TURN'
        message.value = '灰子AI 思考中...'
        drawBoard()
        // 白AI落子后延迟 500ms 再触发灰AI，让玩家感知到"回合流转"
        await new Promise(r => setTimeout(r, 500))
        await doAiMove()
      }
    } else {
      status.value = 'PLAYER_TURN'
      message.value = res.message || '操作失败'
    }
    drawBoard()
  } catch {
    status.value = 'PLAYER_TURN'
    message.value = '系统维护中'
    drawBoard()
  }
}

async function doAiMove() {
  try {
    const res = await gameApi.aiMove()
    if (res.code === 200) {
      const data = res.data
      board.value = data.board
      terrain.value = data.terrain || null
      status.value = data.status
      undoRemaining.value = data.undoRemaining
      message.value = data.message || ''

      // AI 落子后对话
      if (data.status === 'PLAYER_TURN') {
        if (Math.random() < 0.5) {
          const text = randomPick(aiTauntPhrases)
          showBubbleMsg('ai', text)
          addDialogue('ai', text)
        }
      } else if (data.status.includes('WIN')) {
        const text = randomPick(aiCheerPhrases)
        showBubbleMsg('ai', text)
        addDialogue('ai', text)
      }

      // 三人/三人障碍物模式：白子落完返回 GRAY_TURN，自动触发灰子AI落子
      if (data.status === 'GRAY_TURN') {
        status.value = 'GRAY_TURN'
        message.value = '灰子AI 思考中...'
        drawBoard()
        // 延迟 500ms 让玩家感知到回合流转
        await new Promise(r => setTimeout(r, 500))
        await doAiMove()
      }
    } else {
      status.value = 'PLAYER_TURN'
      message.value = res.message || '操作失败'
    }
    drawBoard()
  } catch {
    status.value = 'PLAYER_TURN'
    message.value = '系统维护中'
    drawBoard()
  }
}

async function doUndo() {
  undoLoading.value = true
  try {
    const res = await gameApi.undo()
    if (res.code === 200) {
      board.value = res.data.board
      terrain.value = res.data.terrain || null
      status.value = res.data.status
      undoRemaining.value = res.data.undoRemaining
      message.value = res.data.message || ''
      clearSlideAnim()
      drawBoard()
    }
  } catch {
    message.value = '系统维护中'
  } finally {
    undoLoading.value = false
  }
}

async function doPause() {
  const res = await gameApi.pause()
  if (res.code === 200) {
    status.value = res.data.status
    message.value = res.data.message
  }
}

async function doResume() {
  const res = await gameApi.resume()
  if (res.code === 200) {
    board.value = res.data.board
    terrain.value = res.data.terrain || null
    status.value = res.data.status
    message.value = res.data.message
    drawBoard()
  }
}

// 玩家发送快捷语
function onPlayerPhrase(phrase) {
  showBubbleMsg('player', phrase)
  addDialogue('player', phrase)
}

function goBack() {
  router.push('/lobby')
}
</script>

<style scoped>
.game-page {
  min-height: 100vh;
  background:
    /* 赛博朋克网格背景 */
    linear-gradient(135deg, #0a0a1a 0%, #0d1b2a 50%, #050510 100%),
    /* 细微网格线 */
    linear-gradient(rgba(0, 240, 255, 0.03) 1px, transparent 1px),
    linear-gradient(90deg, rgba(0, 240, 255, 0.03) 1px, transparent 1px);
  background-size: 100% 100%, 40px 40px, 40px 40px;
  position: relative;
}
/* 霓虹角装饰 */
.game-page::before {
  content: '';
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
  background:
    /* 左上角霓虹 */
    linear-gradient(135deg, rgba(0, 240, 255, 0.1) 0%, transparent 30%),
    /* 右下角霓虹 */
    linear-gradient(315deg, rgba(255, 0, 128, 0.08) 0%, transparent 30%);
  z-index: 0;
}
header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px 24px;
  background: rgba(10, 10, 30, 0.9);
  border-bottom: 1px solid rgba(0, 240, 255, 0.15);
  position: relative;
  z-index: 2;
}
header h1 { 
  font-size: 18px; 
  color: #00f0ff; 
  text-shadow: 0 0 10px rgba(0, 240, 255, 0.4);
  letter-spacing: 2px;
}
.btn-back {
  padding: 6px 14px;
  border: 1px solid rgba(0, 240, 255, 0.25);
  border-radius: 6px;
  background: rgba(10, 10, 40, 0.6);
  color: #8899aa;
  cursor: pointer;
  font-size: 13px;
  transition: all 0.3s;
  text-shadow: 0 0 3px rgba(0, 240, 255, 0.2);
}
.btn-back:hover { border-color: #00f0ff; color: #00f0ff; box-shadow: 0 0 10px rgba(0, 240, 255, 0.15); }
.header-info { color: #667; font-size: 13px; }

.game-area {
  display: flex;
  justify-content: center;
  align-items: flex-start;
  gap: 32px;
  padding: 30px 20px;
  position: relative;
  z-index: 1;
}

.board-container {
  position: relative;
  border: 2px solid rgba(0, 240, 255, 0.4);
  border-radius: 4px;
  overflow: hidden;
  box-shadow:
    0 0 15px rgba(0, 240, 255, 0.15),
    0 0 30px rgba(0, 240, 255, 0.08),
    inset 0 0 20px rgba(0, 240, 255, 0.05),
    0 8px 32px rgba(0, 0, 0, 0.6);
  animation: neonPulse 3s ease-in-out infinite;
}
/* 扫描线效果 */
.board-container::after {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
  background: repeating-linear-gradient(
    0deg,
    transparent,
    transparent 2px,
    rgba(0, 240, 255, 0.015) 2px,
    rgba(0, 240, 255, 0.015) 4px
  );
  z-index: 5;
}

@keyframes neonPulse {
  0%, 100% { border-color: rgba(0, 240, 255, 0.4); box-shadow: 0 0 15px rgba(0, 240, 255, 0.15), 0 0 30px rgba(0, 240, 255, 0.08), inset 0 0 20px rgba(0, 240, 255, 0.05), 0 8px 32px rgba(0, 0, 0, 0.6); }
  50% { border-color: rgba(0, 240, 255, 0.6); box-shadow: 0 0 20px rgba(0, 240, 255, 0.25), 0 0 40px rgba(0, 240, 255, 0.12), inset 0 0 25px rgba(0, 240, 255, 0.08), 0 8px 32px rgba(0, 0, 0, 0.6); }
}
canvas { display: block; cursor: pointer; }

.side-panel {
  width: 220px;
}

.status-box {
  padding: 20px;
  background: rgba(10, 10, 30, 0.8);
  border: 1px solid rgba(0, 240, 255, 0.2);
  border-radius: 8px;
  text-align: center;
  margin-bottom: 20px;
  transition: all 0.3s;
  box-shadow: 0 0 10px rgba(0, 240, 255, 0.05), inset 0 0 15px rgba(0, 240, 255, 0.02);
}
.status-box.win { border-color: #4ecca3; background: rgba(78,204,163,0.08); box-shadow: 0 0 15px rgba(78,204,163,0.15); }
.status-box.lose { border-color: #e94560; background: rgba(233,69,96,0.08); box-shadow: 0 0 15px rgba(233,69,96,0.15); }
.status-box.draw { border-color: #f0a500; background: rgba(240,165,0,0.08); box-shadow: 0 0 15px rgba(240,165,0,0.15); }
.status-box.thinking { border-color: rgba(0, 240, 255, 0.3); animation: pulse 1.5s infinite; box-shadow: 0 0 15px rgba(0, 240, 255, 0.15); }
.status-icon { font-size: 36px; margin-bottom: 8px; }
.status-text { font-size: 14px; color: #ccc; line-height: 1.5; }
.turn-label {
  margin-top: 10px;
  padding: 8px 12px;
  background: rgba(255,255,255,0.08);
  border-radius: 6px;
  font-size: 13px;
  color: #f0a500;
  animation: fadeIn 0.3s ease;
}

@keyframes pulse {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

.controls {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 20px;
}
.btn-ctrl {
  width: 100%;
  padding: 12px;
  border: 1px solid rgba(0, 240, 255, 0.3);
  border-radius: 6px;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.3s;
  color: #fff;
  background: rgba(10, 10, 40, 0.8);
  text-shadow: 0 0 5px rgba(0, 240, 255, 0.3);
}
.btn-ctrl:hover { 
  opacity: 0.9; 
  border-color: rgba(0, 240, 255, 0.6);
  box-shadow: 0 0 12px rgba(0, 240, 255, 0.2), inset 0 0 12px rgba(0, 240, 255, 0.05);
}
.btn-ctrl.resume { border-color: rgba(78,204,163,0.5); text-shadow: 0 0 5px rgba(78,204,163,0.3); }
.btn-ctrl.resume:hover { border-color: rgba(78,204,163,0.8); box-shadow: 0 0 12px rgba(78,204,163,0.2); }
.btn-ctrl.undo { border-color: rgba(240,165,0,0.5); text-shadow: 0 0 5px rgba(240,165,0,0.3); }
.btn-ctrl.undo:hover { border-color: rgba(240,165,0,0.8); box-shadow: 0 0 12px rgba(240,165,0,0.2); }
.btn-ctrl.new-game { border-color: rgba(233,69,96,0.5); text-shadow: 0 0 5px rgba(233,69,96,0.3); }
.btn-ctrl.new-game:hover { border-color: rgba(233,69,96,0.8); box-shadow: 0 0 12px rgba(233,69,96,0.2); }
.btn-ctrl:disabled { opacity: 0.4; cursor: not-allowed; border-color: rgba(255,255,255,0.1); }

.legend {
  padding: 16px;
  background: rgba(10, 10, 40, 0.6);
  border: 1px solid rgba(0, 240, 255, 0.12);
  border-radius: 8px;
}
.legend-item {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
  color: #aaa;
  font-size: 13px;
  text-shadow: 0 0 3px rgba(0, 240, 255, 0.1);
}
.dot {
  width: 20px;
  height: 20px;
  border-radius: 50%;
}
.dot.black { background: radial-gradient(circle at 35% 35%, #2a0a5e, #0a0520); box-shadow: 0 0 6px rgba(0, 240, 255, 0.3); }
.dot.white { background: radial-gradient(circle at 35% 35%, #e8f4ff, #80b8e0); box-shadow: 0 0 6px rgba(100, 200, 255, 0.3); border: 1px solid rgba(100, 200, 255, 0.3); }
.dot.gray { background: radial-gradient(circle at 35% 35%, #4a2060, #1a0a30); box-shadow: 0 0 6px rgba(255, 0, 200, 0.3); }
.dot.terrain-obstacle {
  background: linear-gradient(135deg, #2a2a4a, #0a0a2a);
  border: 1px solid rgba(255, 0, 200, 0.4);
  box-shadow: 0 0 6px rgba(255, 0, 200, 0.2);
}
.dot.terrain-water {
  background: radial-gradient(circle at 35% 35%, rgba(0, 240, 255, 0.4), rgba(0, 50, 150, 0.3));
  border: 1px solid rgba(0, 240, 255, 0.4);
  box-shadow: 0 0 6px rgba(0, 240, 255, 0.2);
}

@keyframes fadeIn {
  from { opacity: 0; transform: translateY(-5px); }
  to { opacity: 1; transform: translateY(0); }
}

/* 对话气泡 */
.bubble-wrapper {
  position: absolute;
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
  z-index: 10;
  animation: bubbleIn 0.4s ease;
}
@keyframes bubbleIn {
  0% { opacity: 0; transform: translateX(-50%) translateY(-10px) scale(0.8); }
  100% { opacity: 1; transform: translateX(-50%) translateY(0) scale(1); }
}
.bubble {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 14px;
  border-radius: 8px;
  max-width: 280px;
  font-size: 13px;
  line-height: 1.4;
  box-shadow: 0 4px 16px rgba(0,0,0,0.5);
}
.bubble.ai {
  background: rgba(10, 10, 40, 0.95);
  color: #ccddff;
  border: 1px solid rgba(0, 240, 255, 0.2);
  box-shadow: 0 0 12px rgba(0, 240, 255, 0.1), 0 4px 16px rgba(0,0,0,0.5);
}
.bubble.player {
  background: rgba(20, 10, 40, 0.95);
  color: #ffddff;
  border: 1px solid rgba(255, 0, 200, 0.25);
  box-shadow: 0 0 12px rgba(255, 0, 200, 0.1), 0 4px 16px rgba(0,0,0,0.5);
}
.bubble-avatar {
  font-size: 18px;
  flex-shrink: 0;
}
.bubble-text {
  word-break: break-all;
}

/* 玩家快捷语 */
.player-phrases {
  margin-top: 16px;
  padding: 12px;
  background: rgba(10, 10, 40, 0.6);
  border: 1px solid rgba(0, 240, 255, 0.12);
  border-radius: 8px;
}
.phrases-title {
  font-size: 13px;
  color: #667;
  margin-bottom: 8px;
  text-shadow: 0 0 3px rgba(0, 240, 255, 0.1);
}
.phrases-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 6px;
}
.phrase-btn {
  padding: 6px 4px;
  border: 1px solid rgba(0, 240, 255, 0.15);
  border-radius: 6px;
  background: rgba(10, 10, 40, 0.6);
  color: #8899aa;
  font-size: 12px;
  cursor: pointer;
  transition: all 0.2s;
  white-space: nowrap;
  text-shadow: 0 0 2px rgba(0, 240, 255, 0.1);
}
.phrase-btn:hover {
  background: rgba(0, 240, 255, 0.1);
  border-color: rgba(0, 240, 255, 0.4);
  color: #00f0ff;
  box-shadow: 0 0 8px rgba(0, 240, 255, 0.1);
}
</style>
