<template>
  <div class="login-page">
    <div class="login-card">
      <h1>🎯 五子棋 AI 对战</h1>
      <p class="subtitle">经典五子棋 · 人机对战 · 三种难度</p>

      <div class="tabs">
        <button :class="{ active: isLogin }" @click="isLogin = true">登录</button>
        <button :class="{ active: !isLogin }" @click="isLogin = false">注册</button>
      </div>

      <form @submit.prevent="handleSubmit" class="form">
        <div class="input-group">
          <label>昵称</label>
          <input v-model="nickname" placeholder="请输入昵称" required maxlength="50" />
        </div>
        <div class="input-group">
          <label>密码</label>
          <input v-model="password" type="password" placeholder="请输入密码" required />
        </div>
        <div v-if="errorMsg" class="error">{{ errorMsg }}</div>
        <div v-if="successMsg" class="success">{{ successMsg }}</div>
        <button type="submit" class="btn-primary" :disabled="loading">
          {{ loading ? '处理中...' : (isLogin ? '登录' : '注册') }}
        </button>
      </form>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import { userApi } from '../api'

const router = useRouter()
const isLogin = ref(true)
const nickname = ref('')
const password = ref('')
const errorMsg = ref('')
const successMsg = ref('')
const loading = ref(false)

async function handleSubmit() {
  errorMsg.value = ''
  successMsg.value = ''
  loading.value = true

  if (!isLogin.value) {
    // 注册流程：先注册，弹窗提示成功后再自动登录
    try {
      const res = await userApi.register({ nickname: nickname.value, password: password.value })
      if (res.code === 200) {
        successMsg.value = '注册成功！正在自动登录...'
        // 延迟 1.5 秒后自动登录
        await new Promise(r => setTimeout(r, 1500))
        try {
          const loginRes = await userApi.login({ nickname: nickname.value, password: password.value })
          if (loginRes.code === 200) {
            successMsg.value = '登录成功！正在跳转...'
            sessionStorage.setItem('user', JSON.stringify(loginRes.data))
            await new Promise(r => setTimeout(r, 500))
            router.push('/lobby')
          } else {
            errorMsg.value = loginRes.message || '自动登录失败，请手动登录'
            isLogin.value = true
          }
        } catch {
          errorMsg.value = '自动登录失败，请手动登录'
          isLogin.value = true
        }
      } else {
        errorMsg.value = res.message || '注册失败'
      }
    } catch {
      errorMsg.value = '系统维护中'
    } finally {
      loading.value = false
    }
    return
  }

  // 登录流程
  try {
    const res = await userApi.login({ nickname: nickname.value, password: password.value })
    if (res.code === 200) {
      sessionStorage.setItem('user', JSON.stringify(res.data))
      router.push('/lobby')
    } else {
      errorMsg.value = res.message || '操作失败'
    }
  } catch {
    errorMsg.value = '系统维护中'
  } finally {
    loading.value = false
  }
}
</script>

<style scoped>
.login-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: linear-gradient(135deg, #1a1a2e 0%, #16213e 50%, #0f3460 100%);
}
.login-card {
  background: rgba(255,255,255,0.05);
  backdrop-filter: blur(10px);
  border: 1px solid rgba(255,255,255,0.1);
  border-radius: 16px;
  padding: 40px;
  width: 380px;
  text-align: center;
}
h1 {
  font-size: 28px;
  margin-bottom: 8px;
  color: #e94560;
}
.subtitle {
  color: #a0a0b0;
  font-size: 14px;
  margin-bottom: 24px;
}
.tabs {
  display: flex;
  gap: 0;
  margin-bottom: 24px;
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid rgba(255,255,255,0.2);
}
.tabs button {
  flex: 1;
  padding: 10px;
  border: none;
  background: transparent;
  color: #a0a0b0;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.3s;
}
.tabs button.active {
  background: #e94560;
  color: #fff;
}
.form { text-align: left; }
.input-group { margin-bottom: 16px; }
.input-group label {
  display: block;
  margin-bottom: 6px;
  font-size: 13px;
  color: #a0a0b0;
}
.input-group input {
  width: 100%;
  padding: 10px 14px;
  border: 1px solid rgba(255,255,255,0.2);
  border-radius: 8px;
  background: rgba(255,255,255,0.05);
  color: #eee;
  font-size: 14px;
  outline: none;
  transition: border-color 0.3s;
}
.input-group input:focus {
  border-color: #e94560;
}
.error {
  color: #e94560;
  font-size: 13px;
  margin-bottom: 14px;
  text-align: center;
}
.success {
  color: #4ecca3;
  font-size: 14px;
  margin-bottom: 14px;
  text-align: center;
  animation: fadeIn 0.3s ease;
}
@keyframes fadeIn {
  from { opacity: 0; transform: translateY(-5px); }
  to { opacity: 1; transform: translateY(0); }
}
.btn-primary {
  width: 100%;
  padding: 12px;
  border: none;
  border-radius: 8px;
  background: #e94560;
  color: #fff;
  font-size: 16px;
  cursor: pointer;
  transition: opacity 0.3s;
}
.btn-primary:hover { opacity: 0.9; }
.btn-primary:disabled { opacity: 0.6; cursor: not-allowed; }
</style>
