<template>
  <div class="app">
    <header>
      <h1>🎤 Meeting Copilot</h1>
      <p>Unified AI assistant — voice, clipboard, typed & quiz</p>
    </header>

    <!-- Setup box -->
    <div v-if="!isConnected && !connecting" class="setup-box">
      <p>Enter your server URL to connect</p>
      <small>Run <code>ngrok http 5000</code> on your PC</small>
      <input v-model="serverUrl" type="text" placeholder="https://xxxx.ngrok-free.app" @keyup.enter="connect" />
      <button @click="connect">Connect</button>
    </div>

    <template v-if="isConnected">
      <!-- Tab bar -->
      <div class="tabs">
        <button :class="{ active: tab === 'chat' }" @click="tab = 'chat'">💬 Chat</button>
        <button :class="{ active: tab === 'quiz' }" @click="tab = 'quiz'">🧠 Quiz</button>
      </div>

      <!-- ── CHAT TAB ── -->
      <div v-if="tab === 'chat'">
        <div class="controls-bar">
          <span class="dot connected"></span>
          <span class="status-text">Connected</span>
          <div class="sources">
            <label><input type="checkbox" v-model="srcVoice" /> 🎤 Voice</label>
            <label><input type="checkbox" v-model="srcClipboard" /> 📋 Clipboard</label>
            <label><input type="checkbox" v-model="srcTyped" /> ⌨️ Typed</label>
          </div>
          <div class="lang-wrap">
            <span>Languages:</span>
            <div class="multi-lang">
              <!-- Checkboxes for each language -->
              <label v-for="lang in availableLangs" :key="lang" class="lang-chip"
                :class="{ active: selectedLangs.includes(lang) }">
                <input type="checkbox" :value="lang" v-model="selectedLangs" @change="onLangChange" />
                {{ lang }}
              </label>
              <!-- Add custom language -->
              <div class="add-lang">
                <input v-model="newLang" placeholder="+ Add..." @keyup.enter="addLang" maxlength="20" />
                <button @click="addLang" :disabled="!newLang.trim()">+</button>
              </div>
            </div>
          </div>
          <button class="clear-btn" @click="clearChat">🗑 Clear</button>
        </div>

        <div class="feed" ref="feedEl">
          <div v-if="messages.length === 0" class="empty">Enable a source and start asking questions...</div>
          <div v-for="msg in messages" :key="msg.id" class="card" :class="'src-' + msg.source">
            <div class="card-meta">
              <span class="src-badge">{{ srcLabel(msg.source) }}</span>
              <span class="time">{{ msg.time }}</span>
            </div>
            <div class="question">{{ msg.question }}</div>
            <div class="answer" v-html="formatAnswer(msg.answer)"></div>
          </div>
        </div>

        <div v-if="srcTyped" class="input-box">
          <input v-model="typedQuestion" type="text"
            :placeholder="`Ask a ${selectedLang} question...`"
            @keyup.enter="sendTyped" :disabled="sending" />
          <button @click="sendTyped" :disabled="sending || !typedQuestion.trim()">
            {{ sending ? '...' : 'Send' }}
          </button>
        </div>
      </div>

      <!-- ── QUIZ TAB ── -->
      <div v-if="tab === 'quiz'" class="quiz-panel">
        <div class="quiz-controls">
          <div class="lang-wrap">
            <span>Topic:</span>
            <select v-model="quizLang">
              <option value="C#">C#</option>
              <option value="SQL">SQL</option>
              <option value="Java">Java</option>
              <option value="Python">Python</option>
              <option value="JavaScript">JavaScript</option>
              <option value="TypeScript">TypeScript</option>
            </select>
          </div>
          <div class="lang-wrap">
            <span>Difficulty:</span>
            <select v-model="quizDifficulty">
              <option value="beginner">Beginner</option>
              <option value="intermediate">Intermediate</option>
              <option value="advanced">Advanced</option>
            </select>
          </div>
          <button class="gen-btn" @click="generateQuiz" :disabled="quizLoading">
            {{ quizLoading ? '⏳ Generating...' : '🎲 Generate Question' }}
          </button>
        </div>

        <!-- Quiz question card -->
        <div v-if="quiz" class="quiz-card">
          <div class="quiz-topic">{{ quiz.language }} · {{ quiz.difficulty }}</div>
          <div class="quiz-question">{{ quiz.question }}</div>

          <!-- Multiple choice checkboxes -->
          <div class="quiz-options">
            <label
              v-for="(opt, idx) in quiz.options" :key="idx"
              class="quiz-option"
              :class="{
                correct:   quizSubmitted && idx === quiz.correctIndex,
                wrong:     quizSubmitted && selectedOption === idx && idx !== quiz.correctIndex,
                selected:  selectedOption === idx && !quizSubmitted
              }"
            >
              <input
                type="radio"
                :name="'quiz-' + quiz.id"
                :value="idx"
                v-model="selectedOption"
                :disabled="quizSubmitted"
              />
              <span class="opt-letter">{{ ['A','B','C','D'][idx] }}.</span>
              {{ opt }}
            </label>
          </div>

          <!-- Submit / result -->
          <div class="quiz-actions">
            <button
              v-if="!quizSubmitted"
              class="submit-btn"
              @click="submitQuiz"
              :disabled="selectedOption === null"
            >Check Answer</button>

            <div v-if="quizSubmitted" class="quiz-result">
              <span v-if="selectedOption === quiz.correctIndex" class="correct-msg">✅ Correct!</span>
              <span v-else class="wrong-msg">❌ Wrong — correct answer: {{ ['A','B','C','D'][quiz.correctIndex] }}</span>
              <div class="explanation" v-html="formatAnswer(quiz.explanation)"></div>
              <button class="gen-btn" @click="generateQuiz" style="margin-top:12px">Next Question →</button>
            </div>
          </div>
        </div>

        <!-- Quiz history -->
        <div v-if="quizHistory.length > 0" class="quiz-history">
          <h3>📊 Score: {{ quizScore }} / {{ quizHistory.length }}</h3>
          <div v-for="(q, i) in quizHistory" :key="i" class="history-item"
            :class="q.correct ? 'h-correct' : 'h-wrong'">
            <span>{{ q.correct ? '✅' : '❌' }}</span>
            <span>{{ q.question.substring(0, 60) }}...</span>
          </div>
        </div>
      </div>
    </template>
  </div>
</template>

<script setup>
import { ref, nextTick, computed, watch } from 'vue'
import { io } from 'socket.io-client'

const serverUrl     = ref(new URLSearchParams(window.location.search).get('server') || '')
const isConnected   = ref(false)
const connecting    = ref(false)
const tab           = ref('chat')

// Chat
const messages       = ref([])
const feedEl         = ref(null)
const typedQuestion  = ref('')
const sending        = ref(false)
const selectedLang   = computed(() => selectedLangs.value.join(', ') || 'General')
const selectedLangs  = ref(['C#'])
const availableLangs = ref(['C#', 'Java', 'Python', 'JavaScript', 'TypeScript', 'SQL', 'General'])
const newLang        = ref('')
const srcVoice       = ref(true)
const srcClipboard   = ref(true)
const srcTyped       = ref(true)

function addLang() {
  const l = newLang.value.trim()
  if (!l) return
  if (!availableLangs.value.includes(l)) availableLangs.value.push(l)
  if (!selectedLangs.value.includes(l))  selectedLangs.value.push(l)
  newLang.value = ''
  onLangChange()
}

// Quiz
const quizLang       = ref('C#')
const quizDifficulty = ref('intermediate')
const quizLoading    = ref(false)
const quiz           = ref(null)
const selectedOption = ref(null)
const quizSubmitted  = ref(false)
const quizHistory    = ref([])
const quizScore      = computed(() => quizHistory.value.filter(q => q.correct).length)

let socket = null

function srcLabel(s) {
  return { voice: '🎤 Voice', clipboard: '📋 Clipboard', typed: '⌨️ Typed' }[s] || s
}

function connect() {
  const url = serverUrl.value.trim()
  if (!url) return
  connecting.value = true
  socket = io(url, { transports: ['websocket', 'polling'] })

  socket.on('connect', () => {
    isConnected.value = true
    connecting.value  = false
    socket.emit('set_lang', { language: selectedLang.value })
  })
  socket.on('disconnect', () => { isConnected.value = false })

  socket.on('ai_result', async (data) => {
    const src = data.source || 'voice'
    if (src === 'voice'     && !srcVoice.value)     return
    if (src === 'clipboard' && !srcClipboard.value) return
    if (src === 'typed'     && !srcTyped.value)     return
    messages.value.push({
      id: Date.now(), time: new Date().toLocaleTimeString(),
      question: data.question, answer: data.answer, source: src
    })
    await nextTick()
    feedEl.value?.lastElementChild?.scrollIntoView({ behavior: 'smooth' })
  })

  socket.on('quiz_result', (data) => {
    quiz.value = data
    selectedOption.value = null
    quizSubmitted.value  = false
    quizLoading.value    = false
  })
}

function onLangChange() {
  if (socket?.connected) socket.emit('set_lang', { language: selectedLang.value })
}

watch(selectedLang, onLangChange)

function formatAnswer(text) {
  if (!text) return ''
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

function generateQuiz() {
  if (!socket) return
  quizLoading.value = true
  quiz.value = null
  socket.emit('generate_quiz', { language: quizLang.value, difficulty: quizDifficulty.value })
}

function submitQuiz() {
  if (selectedOption.value === null || !quiz.value) return
  quizSubmitted.value = true
  quizHistory.value.push({
    question: quiz.value.question,
    correct:  selectedOption.value === quiz.value.correctIndex
  })
}
</script>

<style>
* { box-sizing: border-box; margin: 0; padding: 0; }
html { height: 100%; }
body {
  font-family: 'Segoe UI', sans-serif;
  background: #0f1117; color: #e0e0e0;
  min-height: 100%; min-height: 100dvh;
  padding: 12px 16px 80px;
}
.app { max-width: 860px; margin: 0 auto; }

header { text-align: center; padding: 14px 0 12px; }
header h1 { font-size: 1.8rem; color: #4fc3f7; }
header p  { color: #888; margin-top: 5px; font-size: 0.88rem; }

.setup-box {
  background: #1e2130; border-radius: 12px;
  padding: 28px; text-align: center; margin: 20px auto; max-width: 500px;
}
.setup-box p { color: #ccc; margin-bottom: 8px; }
.setup-box small { color: #666; font-size: 0.8rem; }
.setup-box code { background: #2a2d3e; padding: 2px 6px; border-radius: 4px; color: #80cbc4; }
.setup-box input {
  width: 100%; padding: 10px 14px; border-radius: 8px;
  border: 1px solid #333; background: #0f1117; color: #e0e0e0;
  font-size: 16px; margin: 14px 0 12px;
}
.setup-box button {
  background: #4fc3f7; color: #000; border: none;
  padding: 10px 32px; border-radius: 8px; cursor: pointer; font-weight: 600;
}

/* Tabs */
.tabs { display: flex; gap: 8px; margin-bottom: 10px; }
.tabs button {
  flex: 1; padding: 10px; border-radius: 8px; border: 1px solid #333;
  background: #1e2130; color: #888; cursor: pointer; font-size: 0.9rem;
}
.tabs button.active { background: #4fc3f7; color: #000; border-color: #4fc3f7; font-weight: 600; }

/* Controls */
.controls-bar {
  display: flex; flex-wrap: wrap; align-items: center; gap: 10px;
  background: #1e2130; border-radius: 10px; padding: 10px 14px;
  margin-bottom: 12px; font-size: 0.85rem;
}
.dot { width: 10px; height: 10px; border-radius: 50%; background: #f44336; flex-shrink: 0; }
.dot.connected { background: #4caf50; }
.status-text { color: #888; }
.sources { display: flex; flex-wrap: wrap; gap: 10px; }
.sources label { display: flex; align-items: center; gap: 5px; cursor: pointer; color: #ccc; }
.sources input[type=checkbox] { accent-color: #4fc3f7; width: 16px; height: 16px; }
.lang-wrap { display: flex; align-items: flex-start; gap: 6px; color: #888; flex-wrap: wrap; width: 100%; }
.lang-wrap > span { padding-top: 4px; white-space: nowrap; }
.multi-lang { display: flex; flex-wrap: wrap; gap: 6px; }
.lang-chip {
  display: flex; align-items: center; gap: 4px;
  background: #0f1117; border: 1px solid #333; border-radius: 20px;
  padding: 5px 12px; cursor: pointer; font-size: 0.8rem; color: #888;
  transition: all 0.2s; user-select: none; -webkit-user-select: none;
}
.lang-chip.active { border-color: #4fc3f7; color: #4fc3f7; background: #1a2a3a; }
.lang-chip input[type=checkbox] { display: none; }
.add-lang { display: flex; gap: 4px; }
.add-lang input {
  width: 80px; padding: 4px 8px; border-radius: 20px;
  border: 1px dashed #444; background: #0f1117; color: #e0e0e0; font-size: 0.8rem;
}
.add-lang input:focus { outline: none; border-color: #4fc3f7; }
.add-lang button {
  background: #4fc3f7; color: #000; border: none;
  width: 26px; height: 26px; border-radius: 50%; cursor: pointer; font-weight: 700; font-size: 1rem;
}
.clear-btn {
  background: transparent; color: #666; border: 1px solid #333;
  padding: 5px 12px; border-radius: 6px; cursor: pointer; font-size: 0.8rem;
}

/* Feed */
.feed { margin-bottom: 12px; }
.empty { text-align: center; color: #444; margin-top: 60px; }
.card {
  background: #1e2130; border-radius: 12px; padding: 16px 18px;
  margin-bottom: 12px; border-left: 4px solid #4fc3f7; animation: slideIn 0.3s ease;
}
.card.src-voice     { border-left-color: #4fc3f7; }
.card.src-clipboard { border-left-color: #ab47bc; }
.card.src-typed     { border-left-color: #66bb6a; }
.card-meta { display: flex; justify-content: space-between; margin-bottom: 8px; }
.src-badge { font-size: 0.75rem; color: #888; }
.time      { font-size: 0.75rem; color: #555; }
.question  { font-size: 0.92rem; color: #90caf9; margin-bottom: 10px; font-weight: 500; }
.answer    { font-size: 0.88rem; line-height: 1.7; }
.answer pre { background: #2a2d3e; padding: 12px; border-radius: 8px; overflow-x: auto; margin: 8px 0; }
.answer code { background: #2a2d3e; padding: 2px 6px; border-radius: 4px; font-family: monospace; color: #80cbc4; }
.answer strong { color: #fff; }

/* Input box — fixed at bottom */
.input-box {
  position: fixed; bottom: 0; left: 0; right: 0; z-index: 100;
  background: #1a1d2e; border-top: 1px solid #2a2d3e;
  padding: 10px 16px;
  display: flex; gap: 8px;
  /* safe area for iPhone notch */
  padding-bottom: max(10px, env(safe-area-inset-bottom));
}
.input-box input {
  flex: 1; padding: 12px 14px; border-radius: 8px;
  border: 1px solid #333; background: #0f1117; color: #e0e0e0;
  font-size: 16px; /* prevents iOS zoom */
  -webkit-appearance: none;
}
.input-box input:focus { outline: none; border-color: #66bb6a; }
.input-box button {
  background: #66bb6a; color: #000; border: none;
  padding: 12px 18px; border-radius: 8px; cursor: pointer; font-weight: 600;
  font-size: 0.9rem; white-space: nowrap; flex-shrink: 0;
}
.input-box button:disabled { background: #333; color: #666; }

/* Quiz */
.quiz-panel { padding-bottom: 20px; }
.quiz-controls {
  display: flex; flex-wrap: wrap; align-items: center; gap: 12px;
  background: #1e2130; border-radius: 10px; padding: 12px 14px; margin-bottom: 16px;
}
.quiz-controls .lang-wrap { margin-left: 0; }
.gen-btn {
  background: #4fc3f7; color: #000; border: none;
  padding: 10px 22px; border-radius: 8px; cursor: pointer; font-weight: 600;
  width: 100%;
}
.gen-btn:disabled { background: #333; color: #666; cursor: not-allowed; }
.quiz-card {
  background: #1e2130; border-radius: 14px; padding: 20px; margin-bottom: 16px;
  border: 1px solid #2a2d3e;
}
.quiz-topic { font-size: 0.78rem; color: #4fc3f7; margin-bottom: 10px; text-transform: uppercase; letter-spacing: 1px; }
.quiz-question { font-size: 1rem; color: #fff; font-weight: 600; margin-bottom: 18px; line-height: 1.5; }
.quiz-options { display: flex; flex-direction: column; gap: 10px; margin-bottom: 18px; }
.quiz-option {
  display: flex; align-items: center; gap: 10px;
  background: #0f1117; border: 1px solid #2a2d3e; border-radius: 8px;
  padding: 12px 14px; cursor: pointer; transition: border-color 0.2s;
  -webkit-tap-highlight-color: transparent;
}
.quiz-option.selected  { border-color: #4fc3f7; background: #1a2a3a; }
.quiz-option.correct   { border-color: #4caf50; background: #1a2e1a; color: #4caf50; }
.quiz-option.wrong     { border-color: #f44336; background: #2e1a1a; color: #f44336; }
.quiz-option input[type=radio] { accent-color: #4fc3f7; width: 18px; height: 18px; flex-shrink: 0; }
.opt-letter { font-weight: 700; color: #4fc3f7; min-width: 20px; }
.submit-btn {
  background: #4fc3f7; color: #000; border: none;
  padding: 12px 28px; border-radius: 8px; cursor: pointer; font-weight: 600; width: 100%;
}
.submit-btn:disabled { background: #333; color: #666; cursor: not-allowed; }
.quiz-result { margin-top: 14px; }
.correct-msg { color: #4caf50; font-size: 1.1rem; font-weight: 700; }
.wrong-msg   { color: #f44336; font-size: 1.1rem; font-weight: 700; }
.explanation { margin-top: 12px; font-size: 0.88rem; line-height: 1.6; color: #ccc; }
.explanation pre { background: #2a2d3e; padding: 10px; border-radius: 8px; overflow-x: auto; margin: 8px 0; }
.explanation code { background: #2a2d3e; padding: 2px 5px; border-radius: 4px; font-family: monospace; color: #80cbc4; }
.quiz-history { background: #1e2130; border-radius: 12px; padding: 16px; margin-top: 16px; }
.quiz-history h3 { color: #4fc3f7; margin-bottom: 12px; font-size: 1rem; }
.history-item { display: flex; gap: 10px; padding: 6px 0; font-size: 0.85rem; color: #888; border-bottom: 1px solid #2a2d3e; }
.history-item.h-correct { color: #4caf50; }
.history-item.h-wrong   { color: #f44336; }

@keyframes slideIn {
  from { opacity: 0; transform: translateY(10px); }
  to   { opacity: 1; transform: translateY(0); }
}
</style>
