<script setup>
import { computed, onUnmounted, reactive, ref } from 'vue';

const timeLeft = ref(25 * 60);
const isRunning = ref(false);
const sessions = ref(0);

const timer = reactive({
  mode: 'work',
  workDuration: 25,
  breakDuration: 5,
  longBreakDuration: 15,
  sessionsBeforeLongBreak: 4
});

const history = ref([]);
let intervalId = null;

const displayTime = computed(() => {
  const minutes = Math.floor(timeLeft.value / 60);
  const seconds = timeLeft.value % 60;

  const formattedMinutes = String(minutes).padStart(2, 0);
  const formattedSeconds = String(seconds).padStart(2, 0);

  return `${formattedMinutes}:${formattedSeconds}`;
});

const modeLabel = computed(() => {
  if (timer.mode === 'work') return 'Fokus!';
  if (timer.mode === 'break') return 'Istirahat!';
  return 'Long Break';
});

const progress = computed(() => {
  let totalSeconds;

  if (timer.mode === 'work') {
    totalSeconds = timer.workDuration * 60;
  } else if (timer.mode === 'longBreak') {
    totalSeconds = timer.longBreakDuration * 60;
  } else {
    totalSeconds = timer.breakDuration * 60;
  }

  return ((totalSeconds - timeLeft.value) / totalSeconds) * 100;
});

const progressColor = computed(() => {
  if (timer.mode === 'work') {
    return 'linear-gradient(90deg, #e75c3c, #c0392b)';
  } 

  if (timer.mode === 'longBreak') {
    return 'linear-gradient(90deg, #3498db, #2980b9)';
  }
  
  return 'linear-gradient(90deg, #e75c3c, #c0392b)';
});

const isNextLongBreak = computed(() => {
  return (sessions.value + 1) % timer.sessionsBeforeLongBreak === 0;
});

const switchLabel = computed(() => {
  if (timer.mode === 'work') {
    return isNextLongBreak.value ? 'Skip ke Long Break' : 'Skip ke Istirahat';
  }

  return 'Skip ke Fokus';
});

function startTimer() {
  if (isRunning.value || timeLeft.value <= 0) return;

  isRunning.value = true;

  intervalId =setInterval(() => {
    timeLeft.value--;

    if (timeLeft.value <= 0) {
      stopTimer();
      addHistory();
      autoSwitch();
    }
  }, 1000);
}

function pauseTimer() {
  if (!isRunning.value) return;
  isRunning.value = false;
  clearInterval(intervalId);
  intervalId = null;
}

function stopTimer() {
  isRunning.value = false;
  clearInterval(intervalId);
  intervalId = null;
}

function resetTimer() {
  stopTimer()

  if (timer.mode === 'work') {
    timeLeft.value = timer.workDuration * 60;
  } else if (timer.mode === 'longBreak') {
    timeLeft.value = timer.longBreakDuration * 60;
  } else {
    timeLeft.value = timer.breakDuration * 60;
  }
}

function switchMode() {
  stopTimer();

  if (timer.mode === 'work') {
    if ((sessions.value + 1) % timer.sessionsBeforeLongBreak === 0) {
      timer.mode = 'longBreak';
      timeLeft.value = timer.longBreakDuration * 60;
    } else {
      timer.mode = 'break';
      timeLeft.value = timer.breakDuration * 60;
    }
  } else {
    timer.mode = 'work';
    timeLeft.value = timer.workDuration * 60;
  }
}

function autoSwitch() {
  if (timer.mode === 'work') {
    sessions.value++;

    if (sessions.value % timer.sessionsBeforeLongBreak === 0) {
      timer.mode = 'longBreak';
      timeLeft.value = timer.longBreakDuration * 60;
    } else {
      timer.mode = 'break';
      timeLeft.value = timer.breakDuration * 60;
    }
  } else {
    timer.mode = 'work';
    timeLeft.value = timer.workDuration * 60;
  }
}

function addHistory() {
  const now = new Date();
  const timeString = now.toLocaleTimeString('id-ID', {
    hour: '2-digit',
    minute: '2-digit'
  });

  history.value.unshift({
    id: Date.now(),
    mode: timer.mode,
    time: timeString,
    session: sessions.value + 1
  });

  if (history.value.length > 10) {
    history.value.pop();
  }
}

onUnmounted(() => {
  if (intervalId) {
    clearInterval(intervalId);
  }
});
</script>

<template>
  <div class="app">
    <h1>Podomoro Time</h1>

    <h2 :class="timer.mode">{{ modeLabel }}</h2>

    <div class="timer-display">
      {{ displayTime }}
    </div>

    <div class="progress-bar">
      <div
        class="progress-fill"
        :style="{ width: progress + '%', background: progressColor }"
      >
      </div>
    </div>

    <div class="session-info">
      <p>Sesi: <strong>{{ sessions }}</strong> / {{ timer.sessionsBeforeLongBreak }} </p>
      <p v-if="isNextLongBreak" class="nex-long">
        Sesi berikutnya: Long Break!
      </p>
    </div>

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

      <button
        @click="resetTimer"
        class="btn btn-reset"
      >
        Reset
      </button>

      <button
        @click="switchMode"
        class="btn btn-switch"
      >
        {{ switchLabel }}
      </button>
    </div>

    <div v-if="history.length > 0" class="history">
      <h3>Riwayat Sesi</h3>
      <ul>
        <li
          v-for="item in history"
          :key="item.id"
          :class="item.mode"
        >
          Sesi {{ item.session }} ({{ item.mode }}) - {{ item.time }}
        </li>
      </ul>
    </div>

  </div>
</template>

<style scoped>
.app {
  text-align: center;
  padding: 2rem;
  font-family: Arial, sans-serif;
  max-width: 400px;
  margin: 0 auto;
}

h1 {
  color: #333;
  margin-bottom: 0.5rem;
}

h2 {
  font-size: 1.2rem;
  text-transform: uppercase;
  letter-spacing: 2px;
  transition: color 0.3s ease;
}

h3 {
  color: #555;
  font-size: 1rem;
  margin-bottom: 0.5rem;
}

.work {
  color: #e74c3c;
}

.break {
  color: #27ae60;
}

.longBreak {
  color: #3498db;
}

.timer-display {
  font-size: 5rem;
  font-weight: bold;
  margin: 1rem 0;
  font-variant-numeric: tabular-nums;
  color: #2c3e50;
}

.progress-bar {
  width: 100%;
  height: 8px;
  background: #ecf0f1;
  border-radius: 4px;
  margin: 1rem 0;
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #e74c3c, #c0392b);
  border-radius: 4px;
  transition: width 1s linear;
}

session-info {
  margin: 0.5rem 0;
  color: #555;
  font-size: 0.9rem;
}

.next-long {
  color: #3498db;
  font-weight: bold;
}

.controls {
  display: flex;
  gap: 1rem;
  justify-content: center;
  flex-wrap: wrap;
  margin: 1.5rem 0;
}

.btn {
  padding: 0.75rem 2rem;
  font-size: 1rem;
  font-weight: bold;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s ease;
}

.btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.btn:active {
  transform: translateY(0);
}

.btn-start {
  background: #27ae60;
  color: white;
}

.btn-pause {
  background: #f39c12;
  color: white;
}

.btn-reset {
  background: #95a5a6;
  color: white;
}

.btn-switch {
  background: #3498db;
  color: white;
}

.history {
  margin-top: 1.5rem;
  text-align: left;
  padding: 1rem;
  background: #f8f9fa;
  border-radius: 8px;
}

.history ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.history li {
  padding: 0.4rem 0;
  font-size: 0.85rem;
  border-bottom: 1px solid #eee;
}

.history li:last-child {
  border-bottom: none;
}

.history li.work {
  color: #e74c3c;
}

.history li.break {
  color: #27ae60;
}

.history li.longBreak {
  color: #3498db;
}
</style>
