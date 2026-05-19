<template>
  <n-popover trigger="click" placement="bottom-start" :show="showPopover" @update:show="showPopover = $event">
    <template #trigger>
      <n-button size="small" style="width: 80px;" :style="{ color: hasValue ? '#3D3D3D' : '#8A8A8A' }">
        {{ displayValue }}
      </n-button>
    </template>
    <n-space :size="8" style="padding: 8px;">
      <n-select
        v-model:value="internalHour"
        :options="hourOptions"
        style="width: 80px;"
        placeholder="HH"
      />
      <n-text style="line-height: 32px;">:</n-text>
      <n-select
        v-model:value="internalMinute"
        :options="minuteOptions"
        style="width: 80px;"
        placeholder="mm"
      />
    </n-space>
  </n-popover>
</template>

<script setup>
import { ref, computed, watch } from 'vue'
import { NPopover, NButton, NSelect, NSpace, NText } from 'naive-ui'

const props = defineProps({
  hour: { type: Number, default: null },
  minute: { type: Number, default: null },
  now: { type: Boolean, default: false },
  placeholder: { type: String, default: '输入' }
})

const emit = defineEmits(['update:hour', 'update:minute'])

const showPopover = ref(false)
const internalHour = ref(props.hour)
const internalMinute = ref(props.minute)

const hasValue = computed(() => internalHour.value !== null && internalMinute.value !== null)

const hourOptions = Array.from({ length: 24 }, (_, i) => ({
  label: i.toString().padStart(2, '0'),
  value: i
}))

const minuteOptions = Array.from({ length: 60 }, (_, i) => ({
  label: i.toString().padStart(2, '0'),
  value: i
}))

const displayValue = computed(() => {
  if (hasValue.value) {
    return `${internalHour.value.toString().padStart(2, '0')}:${internalMinute.value.toString().padStart(2, '0')}`
  }
  return props.placeholder
})

watch(() => props.hour, (val) => {
  internalHour.value = val
})
watch(() => props.minute, (val) => {
  internalMinute.value = val
})

const onTimeChange = () => {
  if (hasValue.value) {
    emit('update:hour', internalHour.value)
    emit('update:minute', internalMinute.value)
    showPopover.value = false
  }
}

watch(showPopover, (show) => {
  if (show && props.now && !hasValue.value) {
    internalHour.value = new Date().getHours()
    internalMinute.value = new Date().getMinutes()
  }
})

watch([internalHour, internalMinute], onTimeChange)
</script>