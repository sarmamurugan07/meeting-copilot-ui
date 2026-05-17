<template>
  <div class="app">
    <header>
      <h1>🎤 Meeting Copilot</h1>
      <p>Unified AI assistant — voice, clipboard & typed</p>
    </header>

    <!-- Setup box -->
    <div v-if="!isConnected && !connecting" class="setup-box">
      <p>Enter your server URL to connect</p>
      <small>Run <code>ngrok http 5000</code> on your PC</small>
      <input v-model="serverUrl" type="text" placeholder="https://xxxx.ngrok-free.app" @keyup.enter="connect" />
      <button @click="connect">Connect</button>
    </div>

    <!-- Controls bar -->
    <div v-if="isConnected" class="controls-bar">
      <!-- Status -->
      <span class="dot" :class="{ connected: isConnected }"></span>
      <span class="status-text">Connected</span>

      <!-- Source checkboxes -->
      <div class="sources">
        <label><input type="checkbox" v-model="srcVoice" /> 🎤 Voice</label>
        <label><input type="checkbox" v-model="srcClipboard" /> 📋 Clipboard</label>
        <label><input type="checkbox" v-model="srcTyped" /> ⌨️ Typed</label>
      </div>

      <!-- Language selector -->
      <div class="lang-wrap">
        <span>Language:</span>
        <select v-model="selectedLang">
          <option value="C#">C#</option>
          <option value="Java">Java</option>
          <option value="Python">Python</option>
          <option value="JavaScript">JavaScript</option>
          <option value="TypeScript">TypeScript</option>
          <option value="General">General</option>
        </select>
      </div>

      <!-- Clear chat -->
      <button class="clear-btn" @click="clearChat">🗑 Clear</button>
    </div>

    <!-- Chat feed -->
    <div class="feed" ref="feedEl">
      <div v-if="messages.length === 0" class="empty">
        Enable a source above and start asking questions...
      </div>
      <div v-for="msg in messages" :key="msg.id" class="card" :class="'src-' + msg.source">
        <div class="card-meta">
          <span class="src-badge">{{ srcLabel(msg.source) }}</span>
          <span class="time">{{ msg.time }}</span>
        </div>
        <div class="question">{{ msg.question }}</div>
        <div class="answer" v-html="formatAnswer(msg.answer)"></div>
      </div>
    </div>

    <!-- Typed input -->
    <div v-if="isConnected && srcTyped" class="input-box">
      <input
        v-model="typedQuestion"
        type="text"
        :placeholder="`Ask a ${selectedLang} question...`"
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

const serverUrl     = ref(new URLSearchParams(window.location.search).get('server') || '')
const isConnected   = ref(false)
const connecting    = ref(false)
const messages      = ref([])
const feedEl        = ref(null)
const typedQuestion = ref('')
const sending       = ref(false)
const selectedLang  = ref('C#')

// Source toggles
const srcVoice     = ref(true)
const srcClipboard = ref(true)
const srcTyped     = ref(true)

let socket = null

function srcLabel(source) {
  return { voice: '🎤 Voice', clipboard: '📋 Clipboard', typed: '⌨️ Typed' }[source] || source
}

function connect() {
  const url = serverUrl.value.trim()
  if (!url) return
  connecting.value = true

  socket = io(url, { transports: ['websocket', 'polling'] })

  socket.on('connect', () => {
    isConnected.value = true
    connecting.value  = false
    // Tell server our language preference
    socket.emit('set_lang', { language: selectedLang.value })
  })

  socket.on('disconnect', () => { isConnected.value = false })

  socket.on('ai_result', async (data) => {
    // Filter by source checkbox
    const src = data.source || 'voice'
    if (src === 'voice'     && !srcVoice.value)     return
    if (src === 'clipboard' && !srcClipboard.value) return
    if (src === 'typed'     && !srcTyped.value)     return

    messages.value.push({
      id:       Date.now(),
      time:     new Date().toLocaleTimeString(),
      question: data.question,
      answer:   data.answer,
      source:   src
    })
    await nextTick()
    feedEl.value?.lastElementChild?.scrollIntoView({ behavior: 'smooth' })
  })
}

// Watch language change and notify server
import { watch } from 'vue'
watch(selectedLang, (lang) => {
  if (socket?.connected) socket.emit('set_lang', { language: lang })
})

function formatAnswer(text) {
  return text
    .replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;')
    .replace(/```[\w]*\n?([\s\S]*?)```/g, '<pre><code>$1</code></pre>')
    .replace(/`([^`]+)`/g, '<code>$1</code>')
    .replace(/\*\*(.*?)\*\*/g, '<strong>$1</strong>')
    .replace(/\n/g, '<br>')
}

async function sendTyped() {
  const q = typedQuestion.value.trim()
  if (!q || !socket) return
  sending.value = true
  typedQuestion.value = ''
  socket.emit('ask', { question: q, language: selectedLang.value, source: 'typed' })
  sending.value = false
}

function clearChat() { messages.value = [] }

onMounted(() => { if (serverUrl.value) connect() })
</script>

<style>
* { box-sizing: border-box; margin: 0; padding: 0; }
body {
  font-family: 'Segoe UI', sans-serif;
  background: #0f1117; color: #e0e0e0;
  min-height: 100vh; padding: 20px 20px 100px;
}
.app { max-width: 860px; margin: 0 auto; }

header { text-align: center; padding: 20px 0 24px; }
header h1 { font-size: 1.8rem; color: #4fc3f7; }
header p  { color: #888; margin-top: 5px; font-size: 0.88rem; }

/* Setup */
.setup-box {
  background: #1e2130; border-radius: 12px;
  padding: 28px; text-align: center; margin: 20px auto; max-width: 500px;
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

/* Controls bar */
.controls-bar {
  display: flex; align-items: center; flex-wrap: wrap; gap: 14px;
  background: #1e2130; border-radius: 10px;
  padding: 10px 16px; margin-bottom: 16px; font-size: 0.85rem;
}
.dot {
  width: 10px; height: 10px; border-radius: 50%;
  background: #f44336; flex-shrink: 0;
}
.dot.connected { background: #4caf50; }
.status-text { color: #888; }
.sources { display: flex; gap: 12px; }
.sources label { display: flex; align-items: center; gap: 5px; cursor: pointer; color: #ccc; }
.sources input[type=checkbox] { accent-color: #4fc3f7; width: 14px; height: 14px; }
.lang-wrap { display: flex; align-items: center; gap: 6px; color: #888; margin-left: auto; }
.lang-wrap select {
  background: #0f1117; color: #e0e0e0;
  border: 1px solid #333; border-radius: 6px;
  padding: 3px 10px; font-size: 0.85rem; cursor: pointer;
}
.clear-btn {
  background: transparent; color: #666; border: 1px solid #333;
  padding: 4px 12px; border-radius: 6px; cursor: pointer; font-size: 0.8rem;
}
.clear-btn:hover { color: #f44336; border-color: #f44336; }

/* Feed */
.feed { margin-top: 4px; }
.empty { text-align: center; color: #444; margin-top: 60px; font-size: 1rem; }

.card {
  background: #1e2130; border-radius: 12px;
  padding: 16px 20px; margin-bottom: 14px;
  border-left: 4px solid #4fc3f7;
  animation: slideIn 0.3s ease;
}
.card.src-voice     { border-left-color: #4fc3f7; }
.card.src-clipboard { border-left-color: #ab47bc; }
.card.src-typed     { border-left-color: #66bb6a; }

.card-meta { display: flex; justify-content: space-between; margin-bottom: 8px; }
.src-badge { font-size: 0.75rem; color: #888; }
.time      { font-size: 0.75rem; color: #555; }
.question  { font-size: 0.95rem; color: #90caf9; margin-bottom: 10px; font-weight: 500; }
.answer    { font-size: 0.9rem; line-height: 1.7; }
.answer pre { background: #2a2d3e; padding: 12px; border-radius: 8px; overflow-x: auto; margin: 8px 0; }
.answer code { background: #2a2d3e; padding: 2px 6px; border-radius: 4px; font-family: monospace; color: #80cbc4; }
.answer strong { color: #fff; }

@keyframes slideIn {
  from { opacity: 0; transform: translateY(10px); }
  to   { opacity: 1; transform: translateY(0); }
}

/* Input box */
.input-box {
  position: fixed; bottom: 0; left: 0; right: 0;
  background: #1a1d2e; border-top: 1px solid #2a2d3e;
  padding: 12px 20px; display: flex; gap: 10px;
  max-width: 860px; margin: 0 auto;
}
.input-box input {
  flex: 1; padding: 10px 14px; border-radius: 8px;
  border: 1px solid #333; background: #0f1117; color: #e0e0e0; font-size: 0.95rem;
}
.input-box input:focus { outline: none; border-color: #66bb6a; }
.input-box button {
  background: #66bb6a; color: #000; border: none;
  padding: 10px 22px; border-radius: 8px;
  cursor: pointer; font-size: 0.95rem; font-weight: 600;
}
.input-box button:disabled { background: #333; color: #666; cursor: not-allowed; }
</style>
