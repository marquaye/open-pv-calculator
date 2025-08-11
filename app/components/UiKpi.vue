<script setup>
defineProps({
  label: { type: String, required: true },
  value: { type: String, required: true },
})
</script>

<template>
  <div class="card p-4">
    <div class="text-xs text-slate-500">{{ label }}</div>
    <div class="text-lg font-semibold">{{ value }}</div>
  </div>
</template>
