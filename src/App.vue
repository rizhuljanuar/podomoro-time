<script setup>
import { computed, reactive, ref } from 'vue';

const timeLeft = ref(25 * 60);
const isRunning = ref(false);
const sessions = ref(0);

const timer = reactive({
  mode: 'work',
  workDuration: 25,
  breakDuration: 5
});

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
  color: #e74c3c;
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
</style>
