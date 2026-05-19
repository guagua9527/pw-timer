<script setup>
import { ref, onMounted, computed } from 'vue'
import { NConfigProvider } from 'naive-ui'
import Pc from './components/Pc.vue'
import Mobile from './components/Mobile.vue'

const isMobile = ref(false)
const userPreference = ref(null)
const STORAGE_KEY = 'device_preference'

const checkDevice = () => {
  const userAgent = navigator.userAgent.toLowerCase()
  const mobileKeywords = ['android', 'webos', 'iphone', 'ipad', 'ipod', 'blackberry', 'iemobile', 'opera mini']
  isMobile.value = mobileKeywords.some(keyword => userAgent.includes(keyword))
}

const switchToMobile = () => {
  localStorage.setItem(STORAGE_KEY, 'mobile')
  window.location.reload()
}

const switchToPc = () => {
  localStorage.setItem(STORAGE_KEY, 'pc')
  window.location.reload()
}

const resetPreference = () => {
  localStorage.removeItem(STORAGE_KEY)
  window.location.reload()
}

onMounted(() => {
  const saved = localStorage.getItem(STORAGE_KEY)
  if (saved === 'pc') {
    userPreference.value = 'pc'
  } else if (saved === 'mobile') {
    userPreference.value = 'mobile'
  }
  checkDevice()
})

const showPcView = computed(() => {
  if (userPreference.value === 'pc') return true
  if (userPreference.value === 'mobile') return false
  return !isMobile.value
})
</script>

<template>
  <NConfigProvider :theme-overrides="{
    common: {
      primaryColor: '#E8985E',
      primaryColorHover: '#D4874E',
      primaryColorPressed: '#B8754A',
      borderRadius: '8px'
    }
  }">
    <Pc v-if="showPcView" @switch-to-mobile="switchToMobile" />
    <Mobile v-else @switch-to-pc="switchToPc" @reset-preference="resetPreference" />
  </NConfigProvider>
</template>
