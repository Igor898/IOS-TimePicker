<script setup lang="ts">
import { ref, computed, onMounted, nextTick } from "vue";
import TimeColumn from "./TimeColumn.vue";
import { gsap } from "gsap";
import { ScrollToPlugin } from "gsap/ScrollToPlugin";

gsap.registerPlugin(ScrollToPlugin);

const props = defineProps({
  selectedTime: { type: String, default: null },
  format: { type: String as () => "12h" | "24h", default: "24h" },
  minTime: { type: String, default: "00:00" },
  maxTime: { type: String, default: "23:59" },
  disableOutsideClick: { type: Boolean, default: false },
});

const emit = defineEmits<{
  (e: "close"): void;
  (e: "select", time: string): void;
}>();

const hours = ref<number>(0);
const minutes = ref<number>(0);
const isAm = ref<boolean>(true);
const hourColumnRef = ref<HTMLElement | null>(null);
const minuteColumnRef = ref<HTMLElement | null>(null);

const maxMinutes = computed(() => {
  const [maxH, maxM] = props.maxTime.split(":").map(Number);
  return hours.value === maxH ? maxM : 59;
});

const isTimeValid = computed(() => {
  if (!props.minTime || !props.maxTime) return true;
  const [minH, minM] = props.minTime.split(":").map(Number);
  const [maxH, maxM] = props.maxTime.split(":").map(Number);

  let currentH = hours.value;
  if (props.format === "12h") {
    currentH = isAm.value ? currentH % 12 : (currentH % 12) + 12;
  }

  const currentM = minutes.value;

  return (
    (currentH > minH || (currentH === minH && currentM >= minM)) &&
    (currentH < maxH || (currentH === maxH && currentM <= maxM))
  );
});

function handleSelect() {
  if (!isTimeValid.value) return;

  let h = hours.value;
  if (props.format === "12h") {
    h = isAm.value ? h % 12 : (h % 12) + 12;
  }
  const time = `${h.toString().padStart(2, "0")}:${minutes.value
    .toString()
    .padStart(2, "0")}`;
  emit("select", time);
}

onMounted(() => {
  resetTime();
  scrollToCurrentValues();
});

// Функция сброса времени
function resetTime() {
  if (props.selectedTime) {
    const [h, m] = props.selectedTime.split(":").map(Number);
    hours.value = h;
    minutes.value = m;

    if (props.format === "12h") {
      isAm.value = h < 12;
      hours.value = h % 12 || 12;
    }
  } else {
    const [minH, minM] = props.minTime.split(":").map(Number);
    hours.value = minH;
    minutes.value = minM;

    if (props.format === "12h") {
      isAm.value = minH < 12;
      hours.value = minH % 12 || 12;
    }
  }
}

function scrollToCurrentValues() {
  nextTick(() => {
    if (props.selectedTime) {
      const [h, m] = props.selectedTime.split(":").map(Number);
      // Прокручиваем колонку часов
      snapToIndex(h % (props.format === "12h" ? 12 : 24), "hours");
      // Прокручиваем колонку минут
      snapToIndex(m, "minutes");
    }
  });
}

function snapToIndex(value: number, type: "hours" | "minutes") {
  const items = itemsForType(type);
  const index = items.indexOf(value.toString().padStart(2, "0"));

  if (index >= 0) {
    const column =
      type === "hours" ? hourColumnRef.value : minuteColumnRef.value;
    if (column) {
      const itemHeight = 40;
      gsap.to(column, {
        scrollTo: { y: index * itemHeight },
        duration: 0.4,
        ease: "back.out(1.5)",
      });
    } else {
      console.warn(`Column reference for ${type} is not defined.`);
    }
  } else {
    console.warn(`Value ${value} not found in items for type ${type}:`, items);
  }
}

function itemsForType(type: "hours" | "minutes"): string[] {
  if (type === "hours") {
    return Array.from({ length: props.format === "12h" ? 12 : 24 }, (_, i) =>
      (i + (props.format === "12h" ? 1 : 0)).toString().padStart(2, "0")
    );
  } else {
    const maxM = maxMinutes.value;
    return Array.from({ length: maxM + 1 }, (_, i) =>
      i.toString().padStart(2, "0")
    );
  }
}

const formattedTime = computed(() => {
  let h = hours.value;
  let suffix = "";

  if (props.format === "12h") {
    suffix = "AM";
    h = h % 12 || 12;
  } else {
    h = h % 24;
  }

  return `${h.toString().padStart(2, "0")}:${minutes.value
    .toString()
    .padStart(2, "0")}${suffix ? " " + suffix : ""}`;
});

function handleOutsideClick(e: MouseEvent) {
  if (props.disableOutsideClick) return;

  if ((e.target as HTMLElement).classList.contains("modal-overlay")) {
    const now = new Date();
    hours.value = now.getHours();
    minutes.value = now.getMinutes();

    if (props.format === "12h") {
      isAm.value = hours.value < 12;
      hours.value = hours.value % 12 || 12;
    }

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
            ref="hourColumnRef"
            v-model="hours"
            :min="format === '12h' ? 1 : 0"
            :max="format === '12h' ? 12 : 23"
            :step="1"
            :min-time="minTime"
            :max-time="maxTime"
            :format="format"
          />
          <TimeColumn
            ref="minuteColumnRef"
            v-model="minutes"
            :min="0"
            :max="59"
            :step="1"
            :maxMinutes="maxMinutes"
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
.time-columns {
  display: flex;
  justify-content: center;
  align-items: center;

  position: relative;
  &::after {
    content: ":";
    position: absolute;
    left: 50%;
    top: 50%;
    transform: translate(-50%, -50%);
    font-size: 24px;
    color: var(--tg-theme-text-color);
  }
}

.time-column:last-child {
  .time-item {
    opacity: 0.5;
    transform: scale(0.8);

    &.active {
      opacity: 1;
      transform: scale(1);
    }
  }
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