<template>
  <div class="lobby-page">
    <header>
      <h1>🎯 五子棋 游戏大厅</h1>
      <div class="header-right">
        <span class="username">{{ user?.nickname }}</span>
        <button class="btn-outline" @click="goProfile">个人中心</button>
        <button class="btn-text" @click="logout">退出</button>
      </div>
    </header>

    <div class="content">
      <h2>选择难度开始游戏</h2>
      <div class="difficulty-cards">
        <div class="card" :class="{ selected: difficulty === 'EASY' }" @click="difficulty = 'EASY'">
          <div class="card-icon">😊</div>
          <h3>初级</h3>
          <p>随机防守策略<br/>适合新手入门</p>
        </div>
        <div class="card" :class="{ selected: difficulty === 'MEDIUM' }" @click="difficulty = 'MEDIUM'">
          <div class="card-icon">🤔</div>
          <h3>中级</h3>
          <p>评分算法策略<br/>需要一些技巧</p>
        </div>
        <div class="card" :class="{ selected: difficulty === 'HARD' }" @click="difficulty = 'HARD'">
          <div class="card-icon">🔥</div>
          <h3>高级</h3>
          <p>深度搜索算法<br/>高手挑战级别</p>
        </div>
      </div>

      <div class="mode-selection">
        <h3>游戏模式</h3>
        <div class="mode-buttons">
          <button
            class="mode-btn"
            :class="{ selected: gameMode === 'TWO' }"
            @click="gameMode = 'TWO'"
          >双人模式</button>
          <button
            class="mode-btn"
            :class="{ selected: gameMode === 'THREE' }"
            @click="gameMode = 'THREE'"
          >三人混战</button>
          <button
            class="mode-btn obstacle"
            :class="{ selected: gameMode === 'OBSTACLE' }"
            @click="gameMode = 'OBSTACLE'"
          >🏔️ 障碍物模式</button>
          <button
            class="mode-btn three-obstacle"
            :class="{ selected: gameMode === 'THREE_OBSTACLE' }"
            @click="gameMode = 'THREE_OBSTACLE'"
          >🔺 三人障碍物</button>
        </div>
      </div>

      <div v-if="errorMsg" class="error">{{ errorMsg }}</div>
      <button class="btn-start" @click="startGame" :disabled="loading">
        {{ loading ? '创建游戏中...' : '开始游戏' }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import { gameApi, userApi } from '../api'

const router = useRouter()
const difficulty = ref('EASY')
const gameMode = ref('TWO')  // 新增：游戏模式
const loading = ref(false)
const errorMsg = ref('')

const user = JSON.parse(sessionStorage.getItem('user') || '{}')

function goProfile() { router.push('/profile') }

async function startGame() {
  errorMsg.value = ''
  loading.value = true
  try {
    const res = await gameApi.start(difficulty.value, gameMode.value)
    if (res.code === 200) {
      sessionStorage.setItem('difficulty', difficulty.value)
      sessionStorage.setItem('gameMode', gameMode.value)
      sessionStorage.setItem('gameState', JSON.stringify(res.data))
      router.push('/game')
    } else {
      errorMsg.value = res.message || '开始游戏失败'
    }
  } catch {
    errorMsg.value = '系统维护中'
  } finally {
    loading.value = false
  }
}

async function logout() {
  await userApi.logout()
  sessionStorage.clear()
  router.push('/login')
}
</script>

<style scoped>
.lobby-page {
  min-height: 100vh;
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
}
header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 16px 32px;
  background: rgba(255,255,255,0.03);
  border-bottom: 1px solid rgba(255,255,255,0.08);
}
header h1 { font-size: 20px; color: #e94560; }
.header-right { display: flex; align-items: center; gap: 16px; }
.username { color: #ccc; font-size: 14px; }
.btn-outline {
  padding: 6px 16px;
  border: 1px solid rgba(255,255,255,0.3);
  border-radius: 6px;
  background: transparent;
  color: #ccc;
  cursor: pointer;
  font-size: 13px;
  transition: all 0.3s;
}
.btn-outline:hover { border-color: #e94560; color: #e94560; }
.btn-text {
  padding: 6px 8px;
  border: none;
  background: transparent;
  color: #888;
  cursor: pointer;
  font-size: 13px;
}
.btn-text:hover { color: #e94560; }

.content {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 60px 20px;
}
.content h2 {
  font-size: 22px;
  color: #ccc;
  margin-bottom: 40px;
}
.difficulty-cards {
  display: flex;
  gap: 24px;
  margin-bottom: 40px;
}
.card {
  width: 200px;
  padding: 30px 20px;
  background: rgba(255,255,255,0.05);
  border: 2px solid rgba(255,255,255,0.1);
  border-radius: 12px;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s;
}
.card:hover {
  border-color: rgba(233,69,96,0.5);
  transform: translateY(-4px);
}
.card.selected {
  border-color: #e94560;
  background: rgba(233,69,96,0.1);
}
.card-icon { font-size: 36px; margin-bottom: 12px; }
.card h3 { font-size: 18px; color: #eee; margin-bottom: 8px; }
.card p { font-size: 13px; color: #888; line-height: 1.6; }

.btn-start {
  padding: 14px 60px;
  border: none;
  border-radius: 10px;
  background: #e94560;
  color: #fff;
  font-size: 18px;
  cursor: pointer;
  transition: all 0.3s;
}
.btn-start:hover { opacity: 0.9; transform: scale(1.05); }
.btn-start:disabled { opacity: 0.6; transform: none; cursor: not-allowed; }
.error {
  color: #e94560;
  margin-bottom: 16px;
  font-size: 14px;
}

.mode-selection {
  margin-bottom: 40px;
  text-align: center;
}
.mode-selection h3 {
  font-size: 14px;
  color: #ccc;
  margin-bottom: 16px;
}
.mode-buttons {
  display: flex;
  gap: 16px;
  justify-content: center;
}
.mode-btn {
  flex: 1;
  max-width: 180px;
  padding: 10px 0;
  border: 1px solid rgba(255,255,255,0.2);
  border-radius: 8px;
  background: transparent;
  color: #ccc;
  font-size: 14px;
  cursor: pointer;
  transition: all 0.3s;
}
.mode-btn:hover {
  border-color: rgba(233,69,96,0.5);
}
.mode-btn.selected {
  background: #e94560;
  border-color: #e94560;
  color: #fff;
}
.mode-btn.obstacle.selected {
  background: linear-gradient(135deg, #8B7355, #5C4033);
  border-color: #6B5340;
  color: #fff;
}
.mode-btn.three-obstacle.selected {
  background: linear-gradient(135deg, #e94560, #8B7355);
  border-color: #6B5340;
  color: #fff;
}
</style>
