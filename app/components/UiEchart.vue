<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref, watch } from 'vue'
import * as echarts from 'echarts'

type EChartOption = Record<string, unknown>
const props = defineProps<{ option: EChartOption; height?: number | string }>()

const el = ref<HTMLDivElement | null>(null)
let chart: echarts.ECharts | null = null

function getHeightStyle() {
  if (typeof props.height === 'number') return `${props.height}px`
  if (typeof props.height === 'string') return props.height
  return '300px'
}

function resize() {
  chart?.resize()
}

onMounted(() => {
  if (el.value) {
    chart = echarts.init(el.value, undefined, { renderer: 'canvas' })
    chart.setOption(props.option ?? {}, true)
    window.addEventListener('resize', resize)
  }
})

watch(
  () => props.option,
  (opt) => {
    if (chart && opt) chart.setOption(opt, true)
  },
  { deep: true },
)

onBeforeUnmount(() => {
  window.removeEventListener('resize', resize)
  chart?.dispose()
  chart = null
})
</script>

<template>
  <div ref="el" :style="{ width: '100%', height: getHeightStyle() }" />
  
</template>
