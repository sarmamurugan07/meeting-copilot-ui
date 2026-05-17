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
      <div v-if="results.length === 0" class="empty">Waiting for voice input or type a question below...</div>
      <TransitionGroup name="card">
        <div v-for="item in results" :key="item.id" class="card">
          <div class="time">{{ item.time }}</div>
          <div class="question"><span>🗣️</span> {{ item.question }}</div>
          <div class="answer" v-html="formatAnswer(item.answer)"></div>
        </div>
      </TransitionGroup>
    </div>

    <!-- Type input -->
    <div v-if="isConnected" class="input-box">
      <input
        v-model="typedQuestion"
        type="text"
        placeholder="Type a question and press Enter or click Send..."
        @keyup.enter="sendTyped"
        :disabled="sending"
      />
      <button @click="sendTyped" :disabled="sending || !typedQuestion.trim()">
        {{ sending ? '...' : 'Send' }}
      </button>
    </div>
  </div>
</template>

<script setup>
import { ref, nextTick, onMounted } from 'vue'
import { io } from 'socket.io-client'

const serverUrl   = ref(new URLSearchParams(window.location.search).get('server') || '')
const isConnected = ref(false)
const connecting  = ref(false)
const statusText  = ref('Not connected')
const results     = ref([])
const feedEl      = ref(null)
const typedQuestion = ref('')
const sending     = ref(false)
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

async function sendTyped() {
  const q = typedQuestion.value.trim()
  if (!q || !socket) return
  sending.value = true
  typedQuestion.value = ''
  // Send to server — server will ask Claude and emit back ai_result
  socket.emit('ask', { question: q })
  sending.value = false
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

.input-box {
  position: fixed;
  bottom: 0; left: 0; right: 0;
  background: #1a1d2e;
  border-top: 1px solid #2a2d3e;
  padding: 14px 20px;
  display: flex;
  gap: 10px;
  max-width: 860px;
  margin: 0 auto;
}
.input-box input {
  flex: 1;
  padding: 10px 14px;
  border-radius: 8px;
  border: 1px solid #333;
  background: #0f1117;
  color: #e0e0e0;
  font-size: 0.95rem;
}
.input-box input:focus { outline: none; border-color: #4fc3f7; }
.input-box button {
  background: #4fc3f7; color: #000; border: none;
  padding: 10px 22px; border-radius: 8px;
  cursor: pointer; font-size: 0.95rem; font-weight: 600;
}
.input-box button:disabled { background: #333; color: #666; cursor: not-allowed; }
.feed { padding-bottom: 80px; }
</style>
