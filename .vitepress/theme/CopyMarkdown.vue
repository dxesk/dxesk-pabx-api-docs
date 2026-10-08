<script setup lang="ts">
import { ref } from 'vue'
import { useData } from 'vitepress'

const { page } = useData()
const copied = ref(false)

async function copy() {
  const text = (page.value as { markdown?: string }).markdown ?? ''
  try {
    await navigator.clipboard.writeText(text)
  } catch {
    const area = document.createElement('textarea')
    area.value = text
    area.style.position = 'fixed'
    area.style.opacity = '0'
    document.body.appendChild(area)
    area.select()
    document.execCommand('copy')
    area.remove()
  }
  copied.value = true
  setTimeout(() => (copied.value = false), 2000)
}
</script>

<template>
  <div v-if="(page as any).markdown" class="copy-md">
    <button type="button" class="copy-md-btn" @click="copy">
      {{ copied ? 'Copied' : 'Copy as Markdown' }}
    </button>
  </div>
</template>
