<script setup>
import { computed } from 'vue'
import { NConfigProvider, NDialogProvider, NMessageProvider, useOsTheme, darkTheme } from 'naive-ui'
import GamePanel from '@/components/GamePanel.vue'

const osTheme = useOsTheme()
const theme = computed(() => (osTheme.value === 'dark' ? darkTheme : null))
</script>

<template>
  <n-config-provider :theme="theme">
    <n-dialog-provider>
      <n-message-provider>
        <h1>Rock ✊ Paper 🖐️ Scissors ✌️</h1>
        <game-panel></game-panel>
      </n-message-provider>
    </n-dialog-provider>
  </n-config-provider>
</template>
