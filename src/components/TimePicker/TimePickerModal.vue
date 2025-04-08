<script setup lang="ts">
import { ref, computed, onMounted, nextTick, watch } from "vue";
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
const period = ref<"AM" | "PM">("AM");
const hourColumnRef = ref<HTMLElement | null>(null);
const minuteColumnRef = ref<HTMLElement | null>(null);
const periodColumnRef = ref<HTMLElement | null>(null);

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
    currentH = period.value === "AM" 
      ? (currentH === 12 ? 0 : currentH)
      : (currentH === 12 ? 12 : currentH + 12);
  }

  const currentM = minutes.value;
  return (
    (currentH > minH || (currentH === minH && currentM >= minM)) &&
    (currentH < maxH || (currentH === maxH && currentM <= maxM))
  );
});

function handleSelect() {
  let h = hours.value;
  if (props.format === "12h") {
    if (period.value === "AM") {
      h = h === 12 ? 0 : h;
    } else {
      h = h === 12 ? 12 : h + 12;
    }
  }
  const time = `${h.toString().padStart(2, "0")}:${minutes.value.toString().padStart(2, "0")}`;
  emit("select", time);
}

onMounted(() => {
  resetTime();
  scrollToCurrentValues();
});

function resetTime() {
  if (props.selectedTime) {
    const [h, m] = props.selectedTime.split(":").map(Number);
    hours.value = h;
    minutes.value = m;
    if (props.format === "12h") {
      period.value = h < 12 ? "AM" : "PM";
      hours.value = h % 12 || 12;
    }
  } else {
    const [minH, minM] = props.minTime.split(":").map(Number);
    hours.value = minH;
    minutes.value = minM;
    if (props.format === "12h") {
      period.value = minH < 12 ? "AM" : "PM";
      hours.value = minH % 12 || 12;
    }
  }
}

function handleOutsideClick(e: MouseEvent) {
  if (props.disableOutsideClick) return;

  if ((e.target as HTMLElement).classList.contains("modal-overlay")) {
    const now = new Date();
    let h = now.getHours();
    let m = now.getMinutes();
    
    if (props.format === "12h") {
      period.value = h < 12 ? "AM" : "PM";
      h = h % 12 === 0 ? 12 : h % 12;
    }

    if (props.minTime || props.maxTime) {
      const [minH, minM] = props.minTime.split(":").map(Number);
      const [maxH, maxM] = props.maxTime.split(":").map(Number);
      
      let currentH = h;
      if (props.format === "12h") {
        currentH = period.value === "AM" 
          ? (h === 12 ? 0 : h)
          : (h === 12 ? 12 : h + 12);
      }
      
      currentH = Math.max(minH, Math.min(maxH, currentH));
      if (currentH === minH) m = Math.max(minM, m);
      if (currentH === maxH) m = Math.min(maxM, m);
      
      if (props.format === "12h") {
        period.value = currentH < 12 ? "AM" : "PM";
        h = currentH % 12 === 0 ? 12 : currentH % 12;
      } else {
        h = currentH;
      }
    }
    
    hours.value = h;
    minutes.value = m;
    handleSelect();
    emit("close");
  }
}

function scrollToCurrentValues() {
  nextTick(() => {
    if (props.selectedTime) {
      const [h, m] = props.selectedTime.split(":").map(Number);
      const hourValue = props.format === "12h" ? h % 12 || 12 : h;
      snapToIndex(hourValue, "hours");
      snapToIndex(m, "minutes");
      if (props.format === "12h") {
        const ampmIndex = h < 12 ? 0 : 1;
        snapToIndex(ampmIndex, "ampm");
      }
    }
  });
}

function snapToIndex(value: number, type: "hours" | "minutes" | "ampm") {
  let column: HTMLElement | null = null;
  let items: string[] = [];

  if (type === "hours") {
    column = hourColumnRef.value;
    items = Array.from({ length: props.format === "12h" ? 12 : 24 }, (_, i) =>
      (i + (props.format === "12h" ? 1 : 0)).toString().padStart(2, "0")
    );
  } else if (type === "minutes") {
    column = minuteColumnRef.value;
    items = Array.from({ length: maxMinutes.value + 1 }, (_, i) =>
      i.toString().padStart(2, "0")
    );
  } else if (type === "ampm") {
    column = periodColumnRef.value;
    items = ["AM", "PM"];
  }

  if (column && items.length > 0) {
    const index =
      type === "ampm"
        ? value
        : items.indexOf(value.toString().padStart(2, "0"));
    if (index >= 0) {
      const itemHeight = 40;
      gsap.to(column, {
        scrollTo: { y: index * itemHeight },
        duration: 0.4,
        ease: "back.out(1.5)",
      });
    }
  }
}

const formattedTime = computed(() => {
  let h = hours.value;
  let suffix = "";

  if (props.format === "12h") {
    suffix = ` ${period.value}`;
    h = h % 12 || 12;
  }

  return `${h.toString().padStart(2, "0")}:${minutes.value
    .toString()
    .padStart(2, "0")}${suffix}`;
});
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
            :max-minutes="maxMinutes"
          />
          <TimeColumn
            v-if="format === '12h'"
            ref="periodColumnRef"
            v-model="period"
            :items="['AM', 'PM']"
            type="ampm"
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

.time-column.ampm {
  max-width: 70px;

  .time-item {
    font-size: 18px;
    &.active {
      font-size: 22px;
      color: var(--tg-theme-button-color);
    }
  }
}
</style>