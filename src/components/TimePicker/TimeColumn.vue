<script setup lang="ts">
import { ref, computed } from "vue";
import { gsap } from "gsap";
import { ScrollToPlugin } from "gsap/ScrollToPlugin";

gsap.registerPlugin(ScrollToPlugin);

const props = defineProps({
  modelValue: [Number, Boolean],
  min: { type: Number, default: 0 },
  max: { type: Number, default: 59 },
  step: { type: Number, default: 1 },
  items: { type: Array, default: null },
  isBoolean: { type: Boolean, default: false },
  minTime: { type: String, default: "00:00" },
  maxTime: { type: String, default: "23:59" },
  format: { type: String, default: "24h" },
  maxMinutes: { type: Number, default: 59 },
});

const emit = defineEmits<{
  (e: "update:modelValue", value: string): void;
}>();

const selectedIndex = ref(0);
const isScrolling = ref(false);
const columnRef = ref<HTMLElement | null>(null);

const items = computed(() => {
  if (props.isBoolean) return props.items || ["AM", "PM"];

  const result = [];
  if (props.min === 0 && props.max === 59) {
    for (let i = props.min; i <= props.maxMinutes; i += props.step) {
      result.push(i.toString().padStart(2, "0"));
    }
    return result;
  }

  const [minH] = props.minTime.split(":").map(Number);
  const [maxH] = props.maxTime.split(":").map(Number);

  for (let i = props.min; i <= props.max; i += props.step) {
    if (props.format === "24h") {
      if (i < minH || i > maxH) continue;
    } else {
      const hour24 = props.modelValue === false ? (i % 12) + 12 : i % 12;
      if (hour24 < minH || hour24 > maxH) continue;
    }
    result.push(i.toString().padStart(2, "0"));
  }

  return result;
});

function getItemStyle(index: number) {
  const centerIndex = selectedIndex.value;
  const distance = Math.abs(index - centerIndex);
  const maxDistance = 5;

  if (distance > maxDistance) {
    return {
      opacity: 0,
      transform: "scale(0.1)",
      filter: "blur(5px)",
    };
  }

  const ratio = distance / maxDistance;
  const scale = 1 - ratio * 0.5;
  const opacity = 1 - ratio * 0.8;
  const blur = ratio * 3;
  const rotateX = ratio * 60 * (index < centerIndex ? 1 : -1);

  return {
    opacity,
    transform: `scale(${scale}) rotateX(${rotateX}deg)`,
    filter: `blur(${blur}px)`,
    zIndex: maxDistance - distance,
  };
}

function getCurrentCenterIndex() {
  if (!columnRef.value) return -1;

  const container = columnRef.value;
  const containerRect = container.getBoundingClientRect();
  const containerCenter = containerRect.top + containerRect.height / 2;
  const items = container.querySelectorAll(".time-item");

  for (let i = 0; i < items.length; i++) {
    const item = items[i];
    const itemRect = item.getBoundingClientRect();
    const itemCenter = itemRect.top + itemRect.height / 2;

    if (Math.abs(itemCenter - containerCenter) < itemRect.height / 2) {
      return i;
    }
  }

  return -1;
}

function handleScroll() {
  if (!columnRef.value || isScrolling.value) return;

  const newIndex = getCurrentCenterIndex();
  if (newIndex >= 0 && newIndex !== selectedIndex.value) {
    selectedIndex.value = newIndex;
    updateValue(newIndex);
  }
}

function updateValue(index: number) {
  if (index < 0 || index >= items.value.length) return;

  const newValue = props.isBoolean
    ? (index === 1)
    : parseInt(items.value[index], 10);
  emit("update:modelValue", newValue);
}

function snapToIndex(index: number) {
  if (!columnRef.value) return;

  isScrolling.value = true;
  const itemHeight = 40;
  const targetPosition = index * itemHeight;

  gsap.to(columnRef.value, {
    scrollTo: { y: targetPosition },
    duration: 0.1,
    ease: "power2.out",
    onComplete: () => {
      isScrolling.value = false;
      selectedIndex.value = index;
      updateValue(index);
      columnRef.value.scrollTop = targetPosition;
    },
  });
}

function handleItemClick(index: number) {
  snapToIndex(index);
}
</script>

<template>
  <div class="time-column" ref="columnRef" @scroll="handleScroll">
    <div class="column-content">
      <div
        v-for="(item, index) in items"
        :key="index"
        class="time-item"
        :class="{ active: index === selectedIndex }"
        :style="getItemStyle(index)"
        @click="handleItemClick(index)"
      >
        {{ item }}
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss">
.time-column {
  height: 200px;
  overflow-y: auto;
  scroll-snap-type: y mandatory;
  position: relative;
  flex: 1;
  max-width: 80px;
  padding: 80px 0;
  display: flex;
  flex-direction: column;
  align-items: center;

  &::-webkit-scrollbar {
    width: 6px;
  }
}
.time-item {
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  scroll-snap-align: center;
  color: var(--tg-theme-hint-color);
  font-size: 20px;
  cursor: pointer;
  transition: all 0.2s ease;
  user-select: none;

  &.active {
    color: var(--tg-theme-text-color);
    font-weight: 600;
    font-size: 32px;
  }
}
</style>