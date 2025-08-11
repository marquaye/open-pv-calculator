<script setup lang="ts">
const props = defineProps<{ label: string; unit: string; min: number; max: number; step: number; number: number }>()
const emit = defineEmits<{ 'update:number': [value: number] }>()
</script>

<template>
  <div class="space-y-2">
    <label>{{ props.label }}</label>
    <div class="flex items-center gap-2">
      <input class="border rounded px-2 py-1 w-24" type="number" :value="props.number" @input="(e:any)=> emit('update:number', Number(e.target.value))">
      <span class="text-sm text-gray-500">{{ props.unit }}</span>
    </div>
  <input type="range" :min="props.min" :max="props.max" :step="props.step" :value="props.number" @input="(e:any)=> emit('update:number', Number(e.target.value))">
  </div>
</template>
