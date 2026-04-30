<script setup>
import { computed, onUnmounted, reactive, ref } from 'vue';

const timeLeft = ref(25 * 60);
const isRunning = ref(false);
const sessions = ref(0);

const timer = reactive({
  mode: 'work',
  workDuration: 25,
  breakDuration: 5
});

let intervalId = null;

const displayTime = computed(() => {
  const minutes = Math.floor(timeLeft.value / 60);
  const seconds = timeLeft.value % 60;

  const formattedMinutes = String(minutes).padStart(2, 0);
  const formattedSeconds = String(seconds).padStart(2, 0);

  return `${formattedMinutes}:${formattedSeconds}`;
});

const modeLabel = computed(() => {
  return timer.mode === 'work' ? 'Fokus!' : 'Istirahat!';
});

const progress = computed(() => {
  const totalSeconds = timer.mode === 'work'
    ? timer.workDuration * 60
    : timer.breakDuration * 60

  return ((totalSeconds - timeLeft.value) / totalSeconds) * 100;
});

function startTimer() {
  if (isRunning.value || timeLeft.value <= 0) return;

  isRunning.value = true;

  intervalId =setInterval(() => {
    timeLeft.value--;

    if (timeLeft.value <= 0) {
      stopTimer();
    }
  }, 1000);
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
  } else {
    timeLeft.value = timer.breakDuration * 60;
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
        :style="{ width: progress + '%' }"
      >
      </div>
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
    </div>

    <p>Sesi selesai: {{ sessions }}</p>
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
}

.work {
  color: #e74c3c;
}

.break {
  color: #27ae60;
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
}

.progress-fill {
  height: 100%;
  background: linear-gradient(90deg, #e74c3c, #c0392b);
  border-radius: 4px;
  transition: width 1s linear;
}

.controls {
  display: flex;
  gap: 1rem;
  justify-content: center;
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

.sessions {
  color: #7f8c8d;
  font-size: 0.9rem;
}
</style>
