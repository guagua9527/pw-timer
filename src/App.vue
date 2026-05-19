<script setup>
import { ref, onMounted } from 'vue'
import { NConfigProvider } from 'naive-ui'
import Pc from './components/Pc.vue'
import Mobile from './components/Mobile.vue'

const isMobile = ref(false)

const checkDevice = () => {
  const userAgent = navigator.userAgent.toLowerCase()
  const mobileKeywords = ['android', 'webos', 'iphone', 'ipad', 'ipod', 'blackberry', 'iemobile', 'opera mini']
  isMobile.value = mobileKeywords.some(keyword => userAgent.includes(keyword))
}

onMounted(() => {
  checkDevice()
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
    <Pc v-if="!isMobile" />
    <Mobile v-else />
  </NConfigProvider>
</template>
