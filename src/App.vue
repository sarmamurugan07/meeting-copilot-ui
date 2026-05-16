<template>
  <div class="app">
    <header>
      <h1>🎤 Meeting Copilot</h1>
      <p>Real-time AI answers from your meeting audio</p>
    </header>

    <!-- Connection status -->
    <div class="status">
      <span class="dot" :class="{ connected: isConnected }"></span>
      <span>{{ statusText }}</span>
    </div>

    <!-- Setup box -->
    <div v-if="!isConnected && !connecting" class="setup-box">
      <p>Enter your server URL to connect</p>
      <small>Run <code>ngrok http 5000</code> on your PC and paste the URL below</small>
      <input v-model="serverUrl" type="text" placeholder="https://xxxx.ngrok-free.app" @keyup.enter="connect" />
      <button @click="connect">Connect</button>
    </div>

    <!-- Feed -->
    <div class="feed" ref="feedEl">
      <div v-if="results.length === 0" class="empty">Waiting for voice input...</div>
      <TransitionGroup name="card">
        <div v-for="item in results" :key="item.id" class="card">
          <div class="time">{{ item.time }}</div>
          <div class="question"><span>🗣️</span> {{ item.question }}</div>
          <div class="answer" v-html="formatAnswer(item.answer)"></div>
        </div>
      </TransitionGroup>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick, onMounted } from 'vue'
import { io } from 'socket.io-client'

const serverUrl  = ref(new URLSearchParams(window.location.search).get('server') || '')
const isConnected = ref(false)
const connecting  = ref(false)
const statusText  = ref('Not connected')
const results     = ref([])
const feedEl      = ref(null)
let socket        = null

function connect() {
  const url = serverUrl.value.trim()
  if (!url) return
  connecting.value = true
  statusText.value = 'Connecting...'

  socket = io(url, { transports: ['websocket', 'polling'] })

  socket.on('connect', () => {
    isConnected.value = true
    connecting.value  = false
    statusText.value  = 'Connected — listening for AI answers'
  })

  socket.on('disconnect', () => {
    isConnected.value = false
    statusText.value  = 'Disconnected'
  })

  socket.on('ai_result', async (data) => {
    results.value.push({
      id:       Date.now(),
      time:     new Date().toLocaleTimeString(),
      question: data.question,
      answer:   data.answer
    })
    await nextTick()
    feedEl.value?.lastElementChild?.scrollIntoView({ behavior: 'smooth' })
  })
}

function formatAnswer(text) {
  return text
    .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
    .replace(/`([^`]+)`/g, '<code>$1</code>')
}

onMounted(() => {
  if (serverUrl.value) connect()
})
</script>

<style>
* { box-sizing: border-box; margin: 0; padding: 0; }
body {
  font-family: 'Segoe UI', sans-serif;
  background: #0f1117;
  color: #e0e0e0;
  min-height: 100vh;
  padding: 20px;
}
.app { max-width: 860px; margin: 0 auto; }

header { text-align: center; padding: 24px 0 28px; }
header h1 { font-size: 1.9rem; color: #4fc3f7; }
header p  { color: #888; margin-top: 6px; font-size: 0.9rem; }

.status { text-align: center; margin-bottom: 20px; font-size: 0.85rem; }
.dot {
  display: inline-block; width: 10px; height: 10px;
  border-radius: 50%; background: #f44336;
  margin-right: 6px; vertical-align: middle;
  transition: background 0.3s;
}
.dot.connected { background: #4caf50; }

.setup-box {
  background: #1e2130; border-radius: 12px;
  padding: 28px; text-align: center; margin: 20px auto;
  max-width: 500px;
}
.setup-box p    { color: #ccc; margin-bottom: 8px; }
.setup-box small { color: #666; font-size: 0.8rem; }
.setup-box code { background: #2a2d3e; padding: 2px 6px; border-radius: 4px; color: #80cbc4; }
.setup-box input {
  width: 100%; padding: 10px 14px; border-radius: 8px;
  border: 1px solid #333; background: #0f1117; color: #e0e0e0;
  font-size: 0.95rem; margin: 14px 0 12px;
}
.setup-box button {
  background: #4fc3f7; color: #000; border: none;
  padding: 10px 32px; border-radius: 8px;
  cursor: pointer; font-size: 0.95rem; font-weight: 600;
}
.setup-box button:hover { background: #81d4fa; }

.feed { margin-top: 10px; }
.empty { text-align: center; color: #444; margin-top: 80px; font-size: 1rem; }

.card {
  background: #1e2130; border-radius: 12px;
  padding: 18px 22px; margin-bottom: 16px;
  border-left: 4px solid #4fc3f7;
}
.card .time     { font-size: 0.75rem; color: #666; margin-bottom: 8px; }
.card .question { font-size: 0.95rem; color: #90caf9; margin-bottom: 10px; }
.card .question span { margin-right: 6px; }
.card .answer   { font-size: 0.9rem; line-height: 1.6; white-space: pre-wrap; }
.card .answer code {
  background: #2a2d3e; padding: 2px 6px;
  border-radius: 4px; font-family: monospace; color: #80cbc4;
}

.card-enter-active { animation: slideIn 0.3s ease; }
@keyframes slideIn {
  from { opacity: 0; transform: translateY(12px); }
  to   { opacity: 1; transform: translateY(0); }
}
</style>
