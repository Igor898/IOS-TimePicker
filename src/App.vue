<template>
  <div class="app-container dark-theme">
    <h1>iOS Style TimePicker</h1>
    <div class="demo-section">
      <TimePicker 
        v-model="selectedTime1" 
        title="Выберите время" 
        class="picker"
      />
      <TimePicker 
        v-model="selectedTime2" 
        format="12h" 
        title="12-часовой формат" 
        class="picker"
      />
      
      <TimePicker 
        v-model="selectedTime3" 
        :min-time="'09:00'" 
        :max-time="'18:00'" 
        title="Рабочее время (9:00-18:00)" 
        class="picker"
      />
    </div>
    
    <div class="results">
      <h2>Выбранные значения:</h2>
      <p>24-часовой формат: {{ selectedTime1 || 'не выбрано' }}</p>
      <p>12-часовой формат: {{ selectedTime2 || 'не выбрано' }}</p>
      <p>Ограниченный диапазон: {{ selectedTime3 || 'не выбрано' }}</p>
    </div>
  </div>
</template>

<script setup lang="ts">
//Imports
import { ref } from 'vue'
import TimePicker from './components/TimePicker/TimePicker.vue'

const selectedTime1 = ref<string | null>(null)
const selectedTime2 = ref<string | null>('14:30')
const selectedTime3 = ref<string | null>(null)
</script>

<style scoped lang="scss">
@import "@/assets/styles/variables.scss";
.app-container {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  color: var(--tg-theme-text-color);
  background-color: var(--tg-theme-bg-color);
}

h1 {
  text-align: center;
  margin-bottom: 30px;
  color: var(--tg-theme-text-color);
}

.demo-section {
  display: flex;
  flex-direction: column;
  gap: 20px;
  margin-bottom: 40px;
}

.picker {
  width: 100%;
}

.results {
  padding: 20px;
  background-color: var(--tg-theme-secondary-bg-color);
  border-radius: 12px;
  
  h2 {
    margin-top: 0;
    font-size: 18px;
  }
  
  p {
    margin: 8px 0;
  }
}
</style>