<script setup lang="ts">
import { ref, computed } from "vue";
import TimePickerModal from "./TimePickerModal.vue";

const props = defineProps({
  modelValue: { type: String, default: null },
  title: { type: String, default: "Выбрать время" },
  format: { type: String as () => "12h" | "24h", default: "24h" },
  minTime: { type: String, default: "00:00" },
  maxTime: { type: String, default: "23:59" },
  disableOutsideClick: { type: Boolean, default: false },
});

const emit = defineEmits<{
  (e: "update:modelValue", value: string): void;
}>();

const isModalOpen = ref(false);
const displayValue = computed(() => props.modelValue || "--:--");

function handleTimeSelect(time: string) {
  emit("update:modelValue", time);
  isModalOpen.value = false;
}

function openModal() {
  if (!props.modelValue) {
    emit("update:modelValue", null);
  }
  isModalOpen.value = true;
}
</script>

<template>
  <div class="time-picker-container">
    <span class="title">{{ title }}</span>
    <button class="time-button" @click="openModal">
      {{ displayValue }}
    </button>
    <TimePickerModal
      v-if="isModalOpen"
      :selected-time="modelValue"
      :format="format"
      :min-time="minTime"
      :max-time="maxTime"
      :disable-outside-click="disableOutsideClick"
      @close="isModalOpen = false"
      @select="handleTimeSelect"
    />
  </div>
</template>

<style scoped lang="scss">
@import "@/assets/styles/variables.scss";
.time-picker-container {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background-color: var(--tg-theme-secondary-bg-color);
  border-radius: 10px;
}

.title {
  color: var(--tg-theme-text-color);
  font-size: 15px;
}

.time-button {
  color: var(--tg-theme-button-color);
  font-size: 16px;
  background: none;
  border: none;
  padding: 8px 12px;
  border-radius: 8px;
  cursor: pointer;

  &:active {
    background-color: rgba(89, 183, 255, 0.1);
  }
}
</style>