<template>
  <div class="space-y-1 text-xs text-gray-500 dark:text-gray-400" data-tokenlab-usage="6">
    <div class="whitespace-nowrap" title="TokenLab 上游月度用量（USD）">已用 ${{ money(snapshot?.used) }} / ${{ money(snapshot?.limit) }}</div>
    <div class="whitespace-nowrap" title="新加坡时间（UTC+8）">重置 {{ date(snapshot?.reset_at) }}</div>
    <div :class="stale ? 'text-amber-600 dark:text-amber-400' : ''">
      {{ stale ? '更新失败 · 上次' : '更新' }} {{ date(snapshot?.fetched_at) }}{{ stale ? '' : ' · 自动' }}
    </div>
  </div>
</template>

<script setup lang="ts">
import { computed, onMounted, onUnmounted, ref, watch } from 'vue'
import { adminAPI } from '@/api/admin'
import type { Account } from '@/types'

type Snapshot = { used?: number; limit?: number; reset_at?: string; fetched_at?: string; status?: string }
const props = defineProps<{ account: Account }>()
const snapshot = ref(props.account.extra?.tokenlab_usage as Snapshot | undefined)
const error = ref(false)
const clock = ref(Date.now())
let timer: ReturnType<typeof setInterval> | undefined
let disposed = false
let loading = false
const money = (value?: number) => typeof value === 'number' && Number.isFinite(value)
  ? new Intl.NumberFormat('en-US', { minimumFractionDigits: 4, maximumFractionDigits: 4 }).format(value) : '—'
const date = (value?: string) => !value || Number.isNaN(Date.parse(value)) ? '尚未提供'
  : new Intl.DateTimeFormat('zh-CN', { timeZone: 'Asia/Singapore', month: '2-digit', day: '2-digit', hour: '2-digit', minute: '2-digit', hour12: false }).format(new Date(value))
const stale = computed(() => error.value || snapshot.value?.status === 'failed' || !snapshot.value?.fetched_at
  || clock.value - Date.parse(snapshot.value.fetched_at) > 180000)
watch(() => props.account.extra?.tokenlab_usage, value => { if (value) snapshot.value = value as Snapshot })
async function load() {
  if (disposed || loading || document.visibilityState === 'hidden') return
  loading = true
  try {
    const account = await adminAPI.accounts.getById(props.account.id)
    if (!disposed) { snapshot.value = account.extra?.tokenlab_usage as Snapshot | undefined; error.value = false }
  } catch { if (!disposed) error.value = true }
  finally { loading = false }
}
const visibility = () => { if (document.visibilityState === 'visible') void load() }
onMounted(() => {
  void load()
  let seconds = 0
  timer = setInterval(() => { clock.value = Date.now(); if (++seconds % 30 === 0) void load() }, 1000)
  document.addEventListener('visibilitychange', visibility)
})
onUnmounted(() => { disposed = true; clearInterval(timer); document.removeEventListener('visibilitychange', visibility) })
</script>
