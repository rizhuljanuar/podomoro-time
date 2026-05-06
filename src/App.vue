<script setup>
import { ref, reactive, computed, watch, onMounted, onUnmounted } from 'vue'

// === STATE TIMER ===
const timeLeft = ref(25 * 60)
const isRunning = ref(false)
const sessions = ref(0)
const showSettings = ref(false)
const soundEnabled = ref(true)  // BARU: Toggle sound


const timer = reactive({
  mode: 'work',

  workDuration: 25,

  breakDuration: 5,
  longBreakDuration: 15,
  sessionsBeforeLongBreak: 4
})

const history = ref([])
let intervalId = null

// === COMPUTED PROPERTIES ===
const displayTime = computed(() => {
  const minutes = Math.floor(timeLeft.value / 60)
  const seconds = timeLeft.value % 60
  const formattedMinutes = String(minutes).padStart(2, '0')
  const formattedSeconds = String(seconds).padStart(2, '0')
  return `${formattedMinutes}:${formattedSeconds}`
})

const modeLabel = computed(() => {
  if (timer.mode === 'work') return 'Fokus!'
  if (timer.mode === 'break') return 'Istirahat!'
  return 'Long Break!'
})

const progress = computed(() => {
  let totalSeconds
  if (timer.mode === 'work') {
    totalSeconds = timer.workDuration * 60
  } else if (timer.mode === 'longBreak') {
    totalSeconds = timer.longBreakDuration * 60
  } else {
    totalSeconds = timer.breakDuration * 60
  }
  return ((totalSeconds - timeLeft.value) / totalSeconds) * 100
})

const progressColor = computed(() => {
  if (timer.mode === 'work') {
    return 'linear-gradient(90deg, #e74c3c, #c0392b)'
  }
  if (timer.mode === 'longBreak') {
    return 'linear-gradient(90deg, #3498db, #2980b9)'
  }
  return 'linear-gradient(90deg, #27ae60, #2ecc71)'
})

const isNextLongBreak = computed(() => {
  return (sessions.value + 1) % timer.sessionsBeforeLongBreak === 0
})

const switchLabel = computed(() => {
  if (timer.mode === 'work') {
    return isNextLongBreak.value ? 'Skip ke Long Break' : 'Skip ke Istirahat'
  }
  return 'Skip ke Fokus'
})

// BARU: Total waktu kerja hari ini (dalam menit)
const totalWorkTime = computed(() => {
  return history.value
    .filter(item => item.mode === 'work')
    .length * timer.workDuration
})

// === WATCH ===
// Pantau perubahan timer settings, simpan ke localStorage
watch(
  () => ({
    work: timer.workDuration,
    break: timer.breakDuration,
    long: timer.longBreakDuration,
    sessions: timer.sessionsBeforeLongBreak,
    sound: soundEnabled.value
  }),
  (newSettings) => {
    localStorage.setItem('pomodoro-settings', JSON.stringify(newSettings))
  },
  { deep: true }
)

// === TIMER FUNCTIONS ===
function startTimer() {
  if (isRunning.value || timeLeft.value <= 0) return
  isRunning.value = true

  intervalId = setInterval(() => {
    timeLeft.value--
    if (timeLeft.value <= 0) {
      stopTimer()
      addHistory()
      playSound()    // BARU: Putar suara saat timer habis
      autoSwitch()
    }
  }, 1000)
}

function pauseTimer() {
  if (!isRunning.value) return

  isRunning.value = false
  clearInterval(intervalId)
  intervalId = null
}

function stopTimer() {
  isRunning.value = false

  clearInterval(intervalId)
  intervalId = null
}

function resetTimer() {
  stopTimer()
  timeLeft.value = timer.mode === 'work'
    ? timer.workDuration * 60
    : timer.mode === 'longBreak'
      ? timer.longBreakDuration * 60
      : timer.breakDuration * 60
}

function switchMode() {
  stopTimer()

  if (timer.mode === 'work') {
    if ((sessions.value + 1) % timer.sessionsBeforeLongBreak === 0) {
      timer.mode = 'longBreak'
      timeLeft.value = timer.longBreakDuration * 60
    } else {
      timer.mode = 'break'

      timeLeft.value = timer.breakDuration * 60
    }
  } else {
    timer.mode = 'work'
    timeLeft.value = timer.workDuration * 60
  }

}


function autoSwitch() {
  if (timer.mode === 'work') {
    sessions.value++
    if (sessions.value % timer.sessionsBeforeLongBreak === 0) {
      timer.mode = 'longBreak'
      timeLeft.value = timer.longBreakDuration * 60
    } else {
      timer.mode = 'break'
      timeLeft.value = timer.breakDuration * 60
    }
  } else {
    timer.mode = 'work'
    timeLeft.value = timer.workDuration * 60
  }
}

function addHistory() {
  const now = new Date()
  const timeString = now.toLocaleTimeString('id-ID', {
    hour: '2-digit',
    minute: '2-digit'
  })
  history.value.unshift({
    id: Date.now(),
    mode: timer.mode,
    time: timeString,
    session: sessions.value + 1
  })
  if (history.value.length > 10) {
    history.value.pop()
  }
}

function applySettings() {
  stopTimer()
  timer.mode = 'work'
  sessions.value = 0
  history.value = []
  timeLeft.value = timer.workDuration * 60
  showSettings.value = false
}

function toggleSettings() {
  showSettings.value = !showSettings.value
}

// BARU: Putar suara beep menggunakan Web Audio API
// Tidak butuh file audio external!
function playSound() {
  if (!soundEnabled.value) return  // Jangan mainkan kalau sound dimatikan

  // Buat konteks audio
  const audioContext = new (window.AudioContext || window.webkitAudioContext)()

  // Mainkan 3 beep berurutan
  const beepDuration = 0.2  // 0.2 detik per beep
  const frequency = 800     // Frekuensi suara (Hz)

  for (let i = 0; i < 3; i++) {
    // Buat oscillator (sumber suara)
    const oscillator = audioContext.createOscillator()
    const gainNode = audioContext.createGain()

    // Sambungkan: oscillator → gain → speaker
    oscillator.connect(gainNode)

    gainNode.connect(audioContext.destination)

    // Atur suara
    oscillator.frequency.value = frequency
    oscillator.type = 'sine'  // Jenis gelombang: sine = bunyi lembut

    // Atur volume (fade in/out supaya tidak tajam)
    const startTime = audioContext.currentTime + (i * 0.3)
    gainNode.gain.setValueAtTime(0.3, startTime)
    gainNode.gain.exponentialRampToValueAtTime(0.01, startTime + beepDuration)

    // Putar suara
    oscillator.start(startTime)
    oscillator.stop(startTime + beepDuration)
  }
}

// BARU: Muat settings dari localStorage saat komponen pertama kali tampil
function loadSettings() {
  const saved = localStorage.getItem('pomodoro-settings')
  if (saved) {
    try {
      const settings = JSON.parse(saved)
      timer.workDuration = settings.work || 25
      timer.breakDuration = settings.break || 5
      timer.longBreakDuration = settings.long || 15
      timer.sessionsBeforeLongBreak = settings.sessions || 4
      soundEnabled.value = settings.sound !== false
      timeLeft.value = timer.workDuration * 60
    } catch (e) {
      // Kalau data corrupt, biarkan default
      console.error('Gagal memuat settings:', e)
    }
  }
}

// === LIFECYCLE HOOKS ===
onMounted(() => {
  loadSettings()  // Muat settings saat komponen pertama tampil
})

onUnmounted(() => {
  if (intervalId) {
    clearInterval(intervalId)
  }
})
</script>

<template>
  <div class="app" :data-mode="timer.mode">
    <div class="header">
      <h1>Pomodoro Timer</h1>
      <div class="header-actions">
        <button
          @click="soundEnabled = !soundEnabled"
          class="btn-icon"
          :title="soundEnabled ? 'Matikan suara' : 'Nyalakan suara'"
        >
          {{ soundEnabled ? '🔊' : '🔇' }}
        </button>
        <button @click="toggleSettings" class="btn-icon" title="Settings">
          ⚙️
        </button>
      </div>
    </div>

    <!-- Settings Panel -->
    <div v-if="showSettings" class="settings-panel">
      <h3>Settings</h3>

      <div class="setting-item">
        <label for="work">Durasi Fokus (menit):</label>
        <input
          id="work"
          type="number"
          v-model.number="timer.workDuration"
          min="1"
          max="60"
        />
      </div>

      <div class="setting-item">
        <label for="break">Durasi Istirahat (menit):</label>
        <input
          id="break"
          type="number"
          v-model.number="timer.breakDuration"
          min="1"
          max="30"
        />
      </div>

      <div class="setting-item">
        <label for="longBreak">Durasi Long Break (menit):</label>
        <input
          id="longBreak"
          type="number"
          v-model.number="timer.longBreakDuration"
          min="1"
          max="60"
        />
      </div>

      <div class="setting-item">
        <label for="sessions">Sesi sebelum Long Break:</label>
        <input
          id="sessions"
          type="number"
          v-model.number="timer.sessionsBeforeLongBreak"
          min="2"
          max="10"
        />
      </div>

      <button @click="applySettings" class="btn btn-apply">
        Terapkan & Reset
      </button>
    </div>

    <!-- Mode Label -->
    <h2 :class="timer.mode">{{ modeLabel }}</h2>

    <!-- Timer Display -->
    <div class="timer-circle">
      <div class="timer-display">
        {{ displayTime }}

      </div>
    </div>


    <!-- Progress Bar -->
    <div class="progress-bar">
      <div
        class="progress-fill"
        :style="{ width: progress + '%', background: progressColor }"
      ></div>
    </div>

    <!-- Session Info -->
    <div class="session-info">
      <p>Sesi: <strong>{{ sessions }}</strong> / {{ timer.sessionsBeforeLongBreak }}</p>
      <p v-if="isNextLongBreak" class="next-long">
        Sesi berikutnya: Long Break!
      </p>
      <p class="total-work">
        Total waktu kerja hari ini: {{ totalWorkTime }} menit
      </p>
    </div>

    <!-- Controls -->
    <div class="controls">
      <button
        v-if="!isRunning"
        @click="startTimer"
        class="btn btn-start"
      >

        Start
      </button>


      <button
        v-else
        @click="pauseTimer"
        class="btn btn-pause"
      >
        Pause
      </button>

      <button @click="resetTimer" class="btn btn-reset">
        Reset
      </button>

      <button @click="switchMode" class="btn btn-switch">

        {{ switchLabel }}
      </button>
    </div>

    <!-- History -->
    <div v-if="history.length > 0" class="history">
      <h3>Riwayat Sesi</h3>
      <ul>
        <li
          v-for="item in history"
          :key="item.id"
          :class="item.mode"
        >
          Sesi {{ item.session }} ({{ item.mode }}) — {{ item.time }}
        </li>
      </ul>
    </div>
  </div>
</template>

<style scoped>
.app {
  text-align: center;
  padding: 2rem;
  font-family: 'Segoe UI', Arial, sans-serif;

  max-width: 420px;
  margin: 2rem auto;
  background: #ffffff;
  border-radius: 16px;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
}

/* Header */
.header {
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 1rem;
}

.header h1 {
  color: #2c3e50;
  margin: 0;
  font-size: 1.5rem;
}

.header-actions {
  display: flex;
  gap: 0.25rem;
}

.btn-icon {
  background: none;
  border: none;
  font-size: 1.3rem;
  cursor: pointer;
  padding: 0.25rem 0.5rem;

  border-radius: 6px;
  transition: background 0.2s ease;
}

.btn-icon:hover {
  background: #ecf0f1;
}

h2 {
  font-size: 1rem;
  text-transform: uppercase;
  letter-spacing: 3px;
  transition: color 0.3s ease;
  margin-top: 1rem;
}

h3 {
  color: #555;
  font-size: 1rem;
  margin-bottom: 0.5rem;
}

.work { color: #e74c3c; }
.break { color: #27ae60; }

.longBreak { color: #3498db; }

/* Timer Circle */
.timer-circle {
  display: flex;
  justify-content: center;
  align-items: center;
  margin: 1.5rem auto;
}


.timer-display {
  font-size: 4.5rem;
  font-weight: bold;
  font-variant-numeric: tabular-nums;

  color: #2c3e50;
  background: #f8f9fa;
  width: 220px;
  height: 220px;

  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  border: 6px solid #ecf0f1;
  transition: border-color 0.3s ease, box-shadow 0.3s ease;
}

/* Warna border lingkaran sesuai mode */
.app[data-mode="work"] .timer-display {
  border-color: #e74c3c;
  box-shadow: 0 0 20px rgba(231, 76, 60, 0.15);
}

.app[data-mode="break"] .timer-display {
  border-color: #27ae60;
  box-shadow: 0 0 20px rgba(39, 174, 96, 0.15);
}


.app[data-mode="longBreak"] .timer-display {
  border-color: #3498db;
  box-shadow: 0 0 20px rgba(52, 152, 219, 0.15);
}

/* Progress Bar */
.progress-bar {
  width: 100%;

  height: 6px;
  background: #ecf0f1;
  border-radius: 3px;
  margin: 0 1rem;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  border-radius: 3px;
  transition: width 1s linear, background 0.3s ease;
}


/* Session Info */
.session-info {
  margin: 1rem 0;
  color: #555;
  font-size: 0.85rem;

}


.session-info p {
  margin: 0.25rem 0;
}

.next-long {
  color: #3498db;
  font-weight: bold;
}


.total-work {
  color: #7f8c8d;
  font-style: italic;
}


/* Controls */
.controls {
  display: flex;
  gap: 0.5rem;
  justify-content: center;
  flex-wrap: wrap;
  margin: 1.5rem 0;
}

.btn {
  padding: 0.7rem 1.5rem;
  font-size: 0.9rem;
  font-weight: 600;
  border: none;
  border-radius: 10px;
  cursor: pointer;
  transition: all 0.2s ease;
  letter-spacing: 0.5px;
}

.btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.15);
}

.btn:active {
  transform: translateY(0);
}

.btn-start { background: #27ae60; color: white; }
.btn-pause { background: #f39c12; color: white; }
.btn-reset { background: #95a5a6; color: white; }
.btn-switch { background: #3498db; color: white; }
.btn-apply { background: #8e44ad; color: white; width: 100%; margin-top: 0.5rem; }

/* Settings Panel */
.settings-panel {
  background: #f8f9fa;
  border-radius: 12px;
  padding: 1.25rem;
  margin: 1rem 0;
  text-align: left;
  border: 1px solid #ecf0f1;
}

.setting-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 0.75rem;
}

.setting-item label {
  font-size: 0.85rem;
  color: #555;
}

.setting-item input {
  width: 70px;
  padding: 0.4rem;
  border: 1px solid #ddd;
  border-radius: 6px;
  text-align: center;
  font-size: 0.9rem;
  transition: border-color 0.2s ease, box-shadow 0.2s ease;
}

.setting-item input:focus {
  outline: none;
  border-color: #3498db;
  box-shadow: 0 0 0 3px rgba(52, 152, 219, 0.15);
}

/* History */
.history {

  margin-top: 1.5rem;
  text-align: left;
  padding: 1rem;
  background: #f8f9fa;
  border-radius: 12px;
  border: 1px solid #ecf0f1;
}

.history ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.history li {
  padding: 0.4rem 0;
  font-size: 0.8rem;
  border-bottom: 1px solid #eee;
}

.history li:last-child { border-bottom: none; }
.history li.work { color: #e74c3c; }
.history li.break { color: #27ae60; }
.history li.longBreak { color: #3498db; }
</style>
