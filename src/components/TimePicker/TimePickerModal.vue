<script setup lang="ts">
import { ref, computed, onMounted } from "vue";
import TimeColumn from "./TimeColumn.vue";

const props = defineProps({
  selectedTime: { type: String, default: null },
  format: { type: String as () => "12h" | "24h", default: "24h" },
  minTime: { type: String, default: "00:00" },
  maxTime: { type: String, default: "23:59" },
});

const emit = defineEmits(["close", "select"]);

const hours = ref<number>(0);
const minutes = ref<number>(0);
const isAm = ref<boolean>(true);


const isTimeValid = computed(() => {
  if (!props.minTime || !props.maxTime) return true;

  const [minH, minM] = props.minTime.split(":").map(Number);
  const [maxH, maxM] = props.maxTime.split(":").map(Number);

  let currentH = hours.value;
  if (props.format === "12h") {
    currentH = isAm.value ? currentH % 12 : (currentH % 12) + 12;
  }

  const currentM = minutes.value;

  // Проверяем, что текущее время между min и max
  return (
    (currentH > minH || (currentH === minH && currentM >= minM)) &&
    (currentH < maxH || (currentH === maxH && currentM <= maxM))
  );
});

// Модифицируем handleSelect для проверки времени
function handleSelect() {
  if (!isTimeValid.value) {
    // Можно показать сообщение об ошибке или просто не закрывать модальное окно
    return;
  }

  let h = hours.value;
  if (props.format === "12h") {
    h = isAm.value ? h % 12 : (h % 12) + 12;
  }

  const time = `${h.toString().padStart(2, "0")}:${minutes.value
    .toString()
    .padStart(2, "0")}`;
  emit("select", time);
}

onMounted(resetTime);

// Функция сброса времени
function resetTime() {
  if (props.format === "12h") {
    hours.value = 12; // 12:00 AM
    minutes.value = 0;
    isAm.value = true;
  } else {
    hours.value = 0; // 00:00
    minutes.value = 0;
  }
}


const formattedTime = computed(() => {
  let h = hours.value;
  let suffix = "";

  if (props.format === "12h") {
    suffix = isAm.value ? "AM" : "PM";
    h = h % 12 || 12;
  } else {
    h = h % 24;
  }

  return `${h.toString().padStart(2, "0")}:${minutes.value
    .toString()
    .padStart(2, "0")}${suffix ? " " + suffix : ""}`;
});

function handleOutsideClick(e: MouseEvent) {
  if ((e.target as HTMLElement).classList.contains("modal-overlay")) {
    // Устанавливаем текущее время пользователя
    const now = new Date();
    hours.value = now.getHours();
    minutes.value = now.getMinutes();

    if (props.format === "12h") {
      isAm.value = hours.value < 12;
      hours.value = hours.value % 12 || 12;
    }
    // Автоматически подтверждаем выбор
    handleSelect();
  }
}
</script>

<template>
  <div class="modal-overlay dark-theme" @click="handleOutsideClick">
    <div class="modal-content">
      <div class="time-display">{{ formattedTime }}</div>

      <div class="time-columns-container">
        <div class="selection-highlight"></div>
        <div class="time-columns">
          <TimeColumn
            v-model="hours"
            :min="format === '12h' ? 1 : 0"
            :max="format === '12h' ? 12 : 23"
            :step="1"
            :min-time="minTime"
            :max-time="maxTime"
          />
          <TimeColumn
            v-model="minutes"
            :min="0"
            :max="59"
            :step="1"
            :min-time="minTime"
            :max-time="maxTime"
          />
          <TimeColumn
            v-if="format === '12h'"
            v-model="isAm"
            :items="['AM', 'PM']"
            is-boolean
          />
          <TimePickerModal
            v-if="isModalOpen"
            :selected-time="modelValue"
            :format="format"
            :min-time="minTime"
            :max-time="maxTime"
            @close="handleModalClose"
            @select="handleTimeSelect"
          />
        </div>
      </div>

      <button
        class="select-button"
        @click="handleSelect"
        :disabled="!isTimeValid"
        :style="{ opacity: isTimeValid ? 1 : 0.5 }"
      >
        Выбрать
      </button>
    </div>
  </div>
</template>

<style scoped lang="scss">
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  justify-content: center;
  align-items: flex-end;
  z-index: 1000;
}

.modal-content {
  width: 90%;
  max-width: 340px;
  background-color: var(--tg-theme-bg-color);
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
}

.time-display {
  text-align: center;
  font-size: 24px;
  margin-bottom: 20px;
  color: var(--tg-theme-text-color);
  font-weight: 500;
}

.select-button {
  width: 100%;
  background-color: var(--tg-theme-button-color);
  color: white;
  border: none;
  border-radius: 8px;
  padding: 12px;
  font-size: 16px;
  font-weight: 500;
  cursor: pointer;
  margin-top: 10px;
  transition: opacity 0.2s;

  &:active {
    opacity: 0.8;
  }
}
.time-columns-container {
  position: relative;
  height: 200px;
  margin: 10px 0;
  overflow: hidden;

  .time-columns {
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
  }

  .selection-highlight {
    position: absolute;
    top: 50%;
    left: 0;
    right: 0;
    height: 40px;
    transform: translateY(-50%);
    background-color: rgba(10, 132, 255, 0.1);
    border-top: 1px solid var(--tg-theme-button-color);
    border-bottom: 1px solid var(--tg-theme-button-color);
    pointer-events: none;
    z-index: 1;
  }
}
</style>