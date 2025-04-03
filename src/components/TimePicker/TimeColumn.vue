<script setup lang="ts">
import { ref, onMounted, watch, computed } from 'vue'
import { gsap } from 'gsap'

const props = defineProps({
  modelValue: [Number, Boolean],
  min: { type: Number, default: 0 },
  max: { type: Number, default: 59 },
  step: { type: Number, default: 1 },
  items: { type: Array, default: null },
  isBoolean: { type: Boolean, default: false }
})
function getItemStyle(index: number) {
  const centerIndex = selectedIndex.value
  const distance = Math.abs(index - centerIndex)
  const maxDistance = 5
  
  // Эффект "всасывания" в черные дыры
  if (distance > maxDistance) {
    return {
      opacity: 0,
      transform: 'scale(0.1)',
      filter: 'blur(5px)'
    }
  }
  
  // Плавное изменение для ближайших элементов
  const ratio = distance / maxDistance
  const scale = 1 - ratio * 0.5
  const opacity = 1 - ratio * 0.8
  const blur = ratio * 3
  const rotateX = ratio * 60 * (index < centerIndex ? 1 : -1)
  
  return {
    opacity,
    transform: `scale(${scale}) rotateX(${rotateX}deg)`,
    filter: `blur(${blur}px)`,
    zIndex: maxDistance - distance
  }
}
const emit = defineEmits(['update:modelValue'])

const columnRef = ref<HTMLElement | null>(null)
const items = computed(() => {
  if (props.isBoolean) return props.items || ['AM', 'PM']
  
  const result = []
  for (let i = props.min; i <= props.max; i += props.step) {
    result.push(i.toString().padStart(2, '0'))
  }
  return result
})

const selectedIndex = ref(0)
const isScrolling = ref(false)

// Вычисляем центральную позицию (где находится selection-highlight)
function getCurrentCenterIndex() {
  if (!columnRef.value) return -1
  
  const container = columnRef.value
  const containerRect = container.getBoundingClientRect()
  const containerCenter = containerRect.top + containerRect.height / 2
  
  const items = container.querySelectorAll('.time-item')
  for (let i = 0; i < items.length; i++) {
    const item = items[i]
    const itemRect = item.getBoundingClientRect()
    const itemCenter = itemRect.top + itemRect.height / 2
    
    if (Math.abs(itemCenter - containerCenter) < itemRect.height / 2) {
      return i
    }
  }
  return -1
}

function handleScroll() {
  if (!columnRef.value || isScrolling.value) return
  
  const newIndex = getCurrentCenterIndex()
  if (newIndex >= 0 && newIndex !== selectedIndex.value) {
    selectedIndex.value = newIndex
    updateValue(newIndex)
  }
}

function updateValue(index: number) {
  const newValue = props.isBoolean 
    ? index === 1 
    : parseInt(items.value[index])
  
  emit('update:modelValue', newValue)
}

function snapToIndex(index: number) {
  if (!columnRef.value) return
  
  isScrolling.value = true
  const itemHeight = 40
  gsap.to(columnRef.value, {
    scrollTo: { y: index * itemHeight },
    duration: 0.4,
    ease: 'back.out(1.5)',
    onComplete: () => {
      isScrolling.value = false
      selectedIndex.value = index
      updateValue(index)
    }
  })
}

function handleItemClick(index: number) {
  snapToIndex(index)
}

onMounted(() => {
  // Инициализация начального значения
  const initialIndex = props.isBoolean 
    ? props.modelValue ? 1 : 0 
    : items.value.indexOf(props.modelValue?.toString().padStart(2, '0') || '00')
  
  if (initialIndex >= 0) {
    selectedIndex.value = initialIndex
    // Небольшая задержка для корректного позиционирования
    setTimeout(() => {
      snapToIndex(initialIndex)
    }, 50)
  }
})

</script>

<template>
  <div
    class="time-column"
    ref="columnRef"
    @scroll="handleScroll"
    @touchstart.passive="handleTouchStart"
    @touchmove.passive="handleTouchMove"
    @touchend.passive="handleTouchEnd"
  >

    
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
  overflow-y: scroll;
  scroll-snap-type: y mandatory;
  position: relative;
  flex: 1;
  max-width: 80px;
  padding: 80px 0;
  perspective: 300px;
  display: flex;
  flex-direction: column;
  align-items: center;

  &::-webkit-scrollbar {
    display: none;
  }
}

.column-content {
  display: flex;
  flex-direction: column;
  align-items: center;
  position: relative;
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
  transform-origin: center;
  opacity: 0.5;
  transform: scale(0.8) rotateX(30deg);
}

.time-item.active {
  color: var(--tg-theme-text-color);
  font-weight: 600;
  font-size: 32px;
  transform: scale(1.1) rotateX(0deg);
  opacity: 1;
}

.time-item:nth-child(n+1):nth-child(-n+2),
.time-item:nth-last-child(n+1):nth-last-child(-n+2) {
  transform: scale(0.9) rotateX(15deg);
  opacity: 0.7;
}

.time-item:nth-child(n+3):nth-child(-n+4),
.time-item:nth-last-child(n+3):nth-last-child(-n+4) {
  transform: scale(0.85) rotateX(25deg);
  opacity: 0.6;
}

.time-item:nth-child(n+5),
.time-item:nth-last-child(n+5) {
  transform: scale(0.8) rotateX(30deg);
  opacity: 0.5;
}
</style>